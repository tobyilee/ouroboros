# `ooo run`의 에이전트 호출 흐름과 프롬프트 전수 분석

> Seed 실행(`ooo run`) 한 번이 만들어내는 **모든 LLM/에이전트 호출 지점**과, 각 지점에서
> 실제로 조립되는 **프롬프트 원문·파라미터**를 코드 기준으로 추적한 문서.
> Interview 분석(`interview_agent_calls.md`, `interview_prompts_params.md`)의 후속편.

---

## 0. 한눈에 보는 전체 흐름

```
[스킬 계층 — 프롬프트]
skills/run/SKILL.md
  frontmatter: mcp_tool: ouroboros_execute_seed
  본문: 백그라운드 잡 시작 → 관찰자(observer) 서브에이전트 위임 → job_wait 폴링 계약
        │
        ▼
[MCP 계층 — 코드]
mcp/tools/execution_handlers.py  →  OrchestratorRunner 생성, detached job 분리
        │
        ▼
[실행 엔진 — 코드]
runner.execute_seed()                          (runner.py:3892)
  → prepare_session()                          세션 영속화, execution contract 구축
  → execute_precreated_session()               system prompt + tool 정책 준비
  → _execute_parallel()                        (runner.py:4690)
        │
        ├─① DependencyAnalyzer.analyze()       ★LLM 1회 (AC 2개 이상일 때)
        │    구조적 분석(결정론) ∪ LLM 분석 → StagedExecutionPlan
        ▼
ParallelACExecutor.execute_parallel()          (parallel_executor.py:1614)
  Stage 루프 (레벨별 순차, 레벨 내 병렬, semaphore=3)
        │
        ├─ AC마다:
        │   ├─② Preflight Decomposition        ★LLM 1회 (mode=preflight일 때, tool-free)
        │   │    → ATOMIC 이면 그대로 / SPLIT 이면 Sub-AC 각각이 별도 세션
        │   ├─③ Atomic 실행                    ★★에이전트 멀티턴 세션 (핵심 비용)
        │   │    AtomicPromptBuilder → adapter.execute_task → claude_agent_sdk.query()
        │   ├─   Verify Gate                    LLM 아님 — verify_command 셸 실행 (결정론)
        │   ├─④ 재시도                          ★실패 시 최대 ac_retry_attempts(기본 2)회
        │   │    실패분류 + 에러꼬리 + (최종시도) 수평사고 지시가 프롬프트에 주입
        │   └─⑤ Bounce 분류                    ★실패 원인 모호할 때만 (tool-free)
        │
        └─ 레벨 경계:
            ├─   파일 충돌 감지                  LLM 아님 — 파일 경로 교집합 (결정론)
            └─⑥ LevelCoordinator 세션          ★충돌 있을 때만 (Read/Bash/Edit 가능)
        │
        ▼
[실행 후]
  ⑦ QA (자동, skip_qa 없으면) + chained 3단계 formal evaluation 잡
```

### 호출 횟수 계산 — AC 3개(독립), 충돌 없음, 전부 1회 성공 시

| 지점 | 횟수 | 비고 |
| :--- | :---: | :--- |
| ① 의존성 분석 | 1 | temperature 0.0, 1000 tokens 상한 |
| ② 분해 판정 | 3 | AC당 1회 (preflight 모드), tool-free |
| ③ AC 실행 | 3 | **에이전틱 멀티턴 — 전체 비용의 대부분** |
| ④⑤⑥ | 0 | 실패/충돌 없으면 발생 안 함 |
| ⑦ QA/평가 | 별도 잡 | evaluate 파이프라인 (다음 분석 대상) |
| **합계** | **7 + QA** | 실패·충돌·분해가 늘수록 증가 |

**★ Insight ─────────────────────────────────────**
Interview와의 결정적 차이: interview의 LLM 호출은 전부 **1턴 텍스트 생성**(도구 봉인)이었지만,
`ooo run`의 ③ AC 실행은 **도구(Read/Write/Edit/Bash/Glob/Grep)를 쥔 멀티턴 에이전트 세션**입니다.
반대로 ①②⑤ 같은 "판단" 호출은 여기서도 `tools=[]`로 봉인됩니다 — "손을 쓰는 세션"과
"판단만 하는 호출"이 같은 어댑터 위에서 tool 정책 하나로 갈라집니다.
**─────────────────────────────────────────────────**

---

## 1. 스킬 계층 — `skills/run/SKILL.md`

frontmatter가 MCP 라우팅을 선언하고(`mcp_tool: ouroboros_execute_seed`), 본문은 호스트
(Claude Code)에게 주는 **운영 절차서**입니다. LLM에게 주입되는 프롬프트라기보다 "실행 감독
매뉴얼"에 가깝습니다:

- `ouroboros_start_execute_seed`로 **백그라운드 잡** 시작 → `job_id`/`session_id`/`execution_id` 즉시 반환
- **관찰자 위임**: `response.meta.job_observer`가 있으면 읽기 전용 서브에이전트(Claude Code의 Task/Agent 1개)를 스폰해 `job_wait` 폴링을 전담시킴 — 메인 세션은 대화 계속 가능
- 효율 정책 질문: `efficiency_mode="adaptive"` vs `"quality_first"` (frugality_assurance는 별도 opt-in)
- 재시도/충돌 방지 규칙: 관찰자가 커서를 독점, 메인 세션은 같은 잡을 폴링하지 않음
- Synapse(실행 중 추가 지시), Conductor attention(사람 판단 필요 이벤트) 처리 절차 포함

---

## 2. 오케스트레이터 System Prompt — `build_system_prompt()` (runner.py:326)

모든 AC 워커 세션에 공통으로 들어가는 시스템 프롬프트. **6개 조각의 조립품**입니다:

```python
prompt = f"""{strategy_fragment}      # ① task_type별 페르소나
{seed_contract}                       # ② Seed 계약 렌더링
{guidance_fragment}                   # ③ 프로젝트 실행 지침 (있을 때만)
{ac_tracking}                         # ④ 진행 마커 프로토콜
{recovery_protocol}"""                # ⑤ 자가 회복 프로토콜
# + conductor_directive (있을 때만)   # ⑥ 후속 실행 지시
# + context_pack_fragment (repo면)    # ⑦ 결정론적 코드베이스 컨텍스트
```

### ① strategy_fragment — task_type별 페르소나 (execution_strategy.py)

| task_type | 페르소나 파일 | tools |
| :--- | :--- | :--- |
| code (기본) | `agents/code-executor.md` | Read, Write, Edit, Bash, Glob, Grep |
| research | `agents/research-agent.md` | Read, Write, Bash, Glob, Grep |
| analysis | `agents/analysis-agent.md` | Read, Write, Bash, Glob, Grep |

`code-executor.md` 전문 (놀랍도록 짧음 — 9줄):
```
You are an autonomous coding agent executing a task for the Ouroboros workflow system.

## Guidelines
- Execute each acceptance criterion thoroughly
- Use the available tools (Read, Edit, Bash, Glob, Grep) to accomplish tasks
- Write clean, well-tested code following project conventions
- Report progress clearly as you work
- If you encounter blockers, explain them clearly
```

### ② seed_contract — `render_seed_contract_for_execution()` (core/seed_contract_prompt.py:142)

```
## Seed Contract
The Seed is the immutable source of truth for this execution. Interpret every
execution decision through this contract.

## Goal
{goal}

## Task Type
{task_type}

{constraints 섹션}
{brownfield 섹션 — context_references 등, 있을 때만}
{ontology lens 섹션}
{evaluation principles 섹션}
{exit conditions 섹션}
```
→ 2장(book.md)의 "불변 계약"이 매 워커 세션의 시스템 프롬프트에 문자 그대로 주입되는 지점.

### ④ ac_tracking — AC_TRACKING_PROMPT (workflow_state.py:1013)

```
## Progress Tracking

As you work through each acceptance criterion, use these markers to track progress:
- When you START working on a criterion: [AC_START: N] (where N is the criterion number)
- When you COMPLETE a criterion: [AC_COMPLETE: N]

Example:
"[AC_START: 1] I'll begin implementing the first criterion..."
"...implementation done. [AC_COMPLETE: 1]"
```
→ 스트림 파서가 이 마커를 읽어 `execution.ac.*` 이벤트를 발행 — **프롬프트 프로토콜과 이벤트
소싱이 만나는 접점**.

### ⑤ recovery_protocol — `get_run_recovery_protocol_prompt()` (resilience/recovery.py:180)

```
## Self-Recovery Protocol
If you notice that the run is stalled, repeating the same failed edit, or making
no acceptance-criterion progress, switch strategy before continuing:
- spinning: stop retrying the same fix; isolate or bypass the blocker.
- no_drift: gather the missing fact or inspect the source of truth.
- diminishing_returns: simplify the task and remove unnecessary moving parts.
- oscillation: choose one architecture and make the smallest coherent step.

When you switch strategy, state the detected pattern and the new concrete next
step briefly, then continue implementing and verifying the acceptance criteria.
```
→ resilience 패키지의 4가지 정체 패턴(외부 감시용)이 **워커 자신의 셀프체크 지시**로도 복제됨.
같은 분류학을 코드(감시)와 프롬프트(자가진단) 양쪽에 심은 이중화.

### ⑥ conductor_directive — 후속 실행 전용 (runner.py:403)

이전 실행이 거부된 뒤 교정 재실행일 때만 추가:
```
## Active Conductor Successor Directive
This is bounded additive context for a successor execution. The Seed above remains
the source of truth. Do not weaken or silently replace its approved direction.

Instruction: {directive.instruction}
Rejected evidence reasons: {reasons}

Preservation contract:
- goal: true / acceptance criteria: true / constraints: true / non-goals: true
...
```

### ⑦ context_pack_fragment — 결정론적 repo 스캔 (LLM 아님)

기존 repo에서 실행하면 스택/verify 명령/레이아웃을 **코드로 스캔**해 붙임("workers are not
primed blind"). 보안 계약: seed에 박힌 경로(LLM 생성 가능)는 신뢰하지 않고 `repo_root` 안에
격리(containment)될 때만 사용.

---

## 3. ① 의존성 분석 — DependencyAnalyzer (dependency_analyzer.py)

**하이브리드**: 결정론적 구조 분석(prerequisites/metadata/shared resources)이 먼저 돌고,
LLM은 **추가 엣지 발견용**으로 1회만 호출. LLM 실패 시 구조 분석만으로 fallback (`structured_fallback`).

### 프롬프트 (DEPENDENCY_ANALYSIS_PROMPT, :233) — system prompt 없음, user 단독

```
Analyze the following acceptance criteria and determine their dependencies.

Acceptance Criteria:
AC 0: {content}
AC 1: {content}
...

Instructions:
1. For each AC, identify which OTHER ACs it depends on (if any)
2. An AC depends on another if:
   - It requires files/code created by the other AC
   - It needs functionality implemented by the other AC
   - It builds upon or extends the other AC's work
3. If ACs are independent (can be done in any order), they have no dependencies

Return ONLY a valid JSON object in this exact format:
{"dependencies": [{"ac_index": 0, "depends_on": []}, ...]}

Rules:
- Use 0-based indexing / 빈 배열 허용 / JSON only / 모든 AC 포함
```

### 파라미터 (:552)
```python
CompletionConfig(role="dependency_analysis", temperature=0.0, max_tokens=1000)
```
→ 결과 합집합: `dependencies[i] = 구조적 ∪ LLM`. LLM은 놓친 엣지만 보태고, 구조적 근거는
절대 LLM이 지울 수 없음.

---

## 4. ② AC 분해 판정 — Preflight Decomposition (parallel_executor.py:3694)

`decomposition_mode="preflight"`(기본)일 때 AC마다 1회. `tools=[]` **tool-free 호출**
(`_dispatch_decomposition_prompt`, :2640), `DECOMPOSITION_TIMEOUT_SECONDS` 타임아웃.

### System Prompt
```
You are a task decomposition expert. Analyze tasks and break them down if needed.
```
(execution_profile이 있으면 `build_decomposition_system_prompt(params)`로 대체 — 분할 축/최소 단위 명시)

### User Prompt (:3720)
```
Analyze this acceptance criterion and determine if it should be decomposed.

## Goal Context
{seed_goal}

## Acceptance Criterion (AC #N)
{ac_content}

## Instructions
Default to ATOMIC. Each sub-AC becomes a separate agent session with its own full
context, so split only when the parent bundles multiple independently valuable
outcomes that can be verified separately.
Decompose into {min}-{max} sub-ACs only when each child is simpler,
independently executable, and owns distinct parent scope. Multiple steps or files
alone are not evidence that a split is warranted.

If the AC is one focused outcome, respond with: ATOMIC

If decomposing, respond with ONLY this structured JSON object:
{"children":[{"description":"...","coverage_claims":["distinct parent scope"],
"verification_hint":"how this child is independently checked"}],
"covers_parent":true,"rationale":"why the children cover the parent without overlap"}
```

**★ Insight ─────────────────────────────────────**
"Default to ATOMIC"과 "Multiple steps or files alone are not evidence" — 분해 편향을
프롬프트 레벨에서 억제합니다. 2장 seed-architect의 "과분해는 결함" 규칙과 정확히 같은 철학이
실행 단계에도 반복됩니다. 분해 = Sub-AC당 별도 에이전트 세션 = 토큰 비용 배수이기 때문.
LLM의 SPLIT 응답도 그대로 믿지 않고 `_verify_generic_decomposition`(구조 검증)을 통과해야
하며, 실패 시 repair 프롬프트로 딱 1회 재시도 — "LLM은 제안하고 코드가 처분한다" 패턴.
**─────────────────────────────────────────────────**

### ⑤ Bounce 분류 (실패 원인 모호할 때만, :2682)
```
system: "You are a conservative execution-recovery classifier."
user: "Classify this failed execution attempt for recovery. Use only the bounded
attempt evidence below. Do not infer complexity from task length or wording.
Return ONLY JSON with cause, reason, evidence_refs, and has_remaining_scope.
cause must be TOO_BIG, BAD_SPEC, ENVIRONMENT, MODEL, or UNKNOWN. TOO_BIG is
allowed only when the trace shows attempted work and distinct parent scope
still remaining.

## Bounded Attempt Trace
{trace.summary}"     # redact_and_truncate — 1,000자 상한
```
→ TOO_BIG 판정만 재분해(bounce_only)로 이어짐. "증거가 있을 때만 TOO_BIG 허용"이라는
보수적 게이트.

---

## 5. ③ AC 워커 프롬프트 — AtomicPromptBuilder (atomic_prompt_builder.py)

**실제 코드를 쓰는 에이전트에게 가는 프롬프트.** 최종 형태 (:282):

```
Execute the following task:

## Working Directory
`{cwd}`

Files present:
- {os.listdir 결과 — 결정론적 스캔, 숨김파일 제외}

**Important**: Use Glob to discover files. Never guess absolute paths.

## Goal Context
{seed_goal}

{auto recursion guard}          # ooo/ouroboros 재호출 금지 가드

{task_section}                  # AC 본문 + 거버닝된 이전 레벨 컨텍스트
                                # + SUCCESS CONTRACT 블록 (아래)
{legacy_context_section}{retry_section}{parallel_section}
{completion_instruction}
```

### SUCCESS CONTRACT 블록 (:33) — AC spec에 계약이 있을 때만

```
SUCCESS CONTRACT for this AC:
- Run locally before completion: {verify_command}. The verify gate re-runs it
  and records authoritative evidence.
- Expected artifacts: {expected_artifacts} — ensure they exist in the workspace
- Expected output: {output_assertion}
```
→ 워커가 **채점 기준을 미리 알고 작업**하게 함. 채점은 어차피 하네스가 같은 명령을 재실행해
결정론적으로 수행하므로, 미리 알려줘도 게이밍이 아니라 정렬(alignment)이 됨.

### 병렬 인지 섹션 (:151) — 같은 레벨에 형제 AC가 있을 때

일반 모드:
```
## Parallel Execution Notice
Other agents are working on sibling tasks concurrently.
Avoid modifying files that other agents are likely editing.
Focus on files directly related to YOUR task.

Sibling tasks in progress:
- {형제 AC 내용, 80자 잘림}
```
fat-harness 모드는 더 강한 스코프 계약:
```
## Current AC Scope Boundary
Sibling/future ACs are listed only to define work that is outside the current
dispatch. Do not satisfy those criteria now, and do not pre-create their files,
tests, docs, or evidence. ...
```

### 재시도 섹션 (:137 + `_build_ac_retry_prompt`, :5592)

```
## Retry Context
This is retry attempt {N} for this acceptance criterion.
Resume from the current shared workspace state, including any
coordinator-reconciled changes already applied.

### Prior failure classification
{failure_class}

### Last error (tail)
{redacted 에러 마지막 500자}
```
**최종 시도에만** 수평사고 지시 추가 (`build_lateral_change_of_approach_directive` — resilience/lateral.py):
"이전 접근이 실패했으니 접근 자체를 바꿔라" — unstuck 페르소나 체계가 재시도 루프에 인라인 주입됨.

### 완료 지시 — 두 가지 모드 (:251, :277)

일반 모드:
```
Use the available tools to accomplish this task. Report your progress clearly.
When complete, explicitly state: [TASK_COMPLETE]
```

fat-harness 모드 (증거 스키마 강제):
```
## Current AC Scope Contract
You are responsible only for the current acceptance criterion in this dispatch. ...
Your final evidence JSON must cite only files, commands, and tests directly
changed or run for this current AC in this runtime session.
For files_touched, cite workspace-relative paths only, never absolute paths ...
For commands_run, include only validation/production commands ...; omit
exploratory discovery commands such as rg, grep, sed, cat, ls, find ...

When complete, emit exactly ONE fenced JSON evidence record as the final
response and then stop. Populate the active profile fields directly
({required_fields}); do not emit a generic command_result wrapper.
Do not prefix it with [TASK_COMPLETE] or any prose; the harness decides
success from typed evidence plus the verifier PASS.
```
+ 문서 전용 AC("tests_passed 넣지 마라"), 검증 전용 AC("files_touched 넣지 마라") 특례 노트.

**★ Insight ─────────────────────────────────────**
fat-harness 완료 지시의 마지막 문장이 핵심입니다: "the harness decides success from typed
evidence plus the verifier PASS" — **워커의 완료 선언은 판정에 쓰이지 않는다**고 워커에게
미리 통보합니다. 3장 "완료는 승인이 아니다"가 프롬프트 문장으로 워커에게 고지되는 순간.
"exploratory 명령(rg/grep/ls)은 증거로 치지 마라"는 규칙은 리워드 해킹(증거 부풀리기)의
가장 흔한 형태를 프롬프트 레벨에서 선제 차단하는 장치입니다.
**─────────────────────────────────────────────────**

---

## 6. 실제 디스패치 — ClaudeAgentAdapter.execute_task (adapter.py:1455)

interview(1턴 봉인 호출)와 달리 **Claude Agent SDK `query()`의 풀 에이전틱 스트리밍**:

```python
options_kwargs = {
    "allowed_tools": effective_tools,      # 기본 DEFAULT_TOOLS = [Read, Write, Edit, Bash, Glob, Grep]
    "permission_mode": effective_permission_mode,
    "cwd": self._cwd,
    "hooks": {"PreToolUse": [HookMatcher(...)]},  # 위임된 execute_seed 도구 컨텍스트 훅
    "model": effective_model,              # 모델 라우터의 per-call 오버라이드가 생성자 핀보다 우선
    "effort": reasoning_effort,            # SDK 네이티브 — "enforced, not advised"
    "resume": current_session_id,          # 재시도/재개 시 세션 이어가기
    "fork_session": True,                  # (핸들 메타데이터에 있을 때)
}
async for sdk_message in query(prompt=prompt, options=options):
    ...  # 스트리밍 — [AC_START]/[AC_COMPLETE] 마커, 도구 호출 이벤트 추적
```

디스패치 직전 게이트 체인 (LLM 아님, 전부 결정론):
1. `_wait_for_memory(label)` — 시스템 여유 메모리 확인 후 스폰
2. `_await_dispatch_rate_budget` — 백엔드 RPM/TPM 예산 페이싱 (Claude는 자체 관리라 면제)
3. `resolve_execute_effort` — effort 라우팅 (분해 자식은 부모 tier 상속, 2회차+ 재시도는 1노치 상승)
4. 모델 라우팅 — `model_router`가 tier 결정 → per-call `model` 오버라이드

비교 — 같은 SDK, 다른 사용법:

| | Interview (`ClaudeCodeAdapter`) | Run (`ClaudeAgentAdapter`) |
| :--- | :--- | :--- |
| 턴 수 | `max_turns=1` | 무제한 (에이전틱 루프) |
| 도구 | `allowed_tools=[]` 봉인 | Read/Write/Edit/Bash/Glob/Grep |
| MCP | `strict_mcp_config` 격리 | disallowedTools로 ouroboros만 차단 |
| 응답 | 최종 텍스트 1개 | 메시지 스트림 (마커/도구 이벤트 파싱) |
| 세션 | 일회성 | resume/fork로 재시도 간 연속성 |

---

## 7. ⑥ 레벨 경계 충돌 조정 — LevelCoordinator (coordinator.py)

**충돌 감지는 결정론**(레벨 내 AC들이 만진 파일 경로 교집합) — 충돌이 있을 때만 LLM 세션 소집:

### System Prompt (:65)
```
You are a Level Coordinator reviewing parallel AC execution results.
Your job is to detect and resolve file conflicts, then provide actionable
guidance for the next level of execution. Be concise and precise.
```

### User Prompt (`_build_review_prompt`, :514)
```
Review the results of Level {N} parallel AC execution.

## Level {N} Results
{level_context.to_prompt_text()}

## File Conflicts Detected
- `{file_path}` modified by: AC 1, AC 3
...

## Your Tasks
1. Read the conflicting files using the Read tool
2. Run `git diff` if needed to understand changes
3. If edits from different ACs conflict, resolve them using the Edit tool
4. Provide your review as a structured JSON response:
{"review_summary": "...", "fixes_applied": [...], ...}
```
→ 도구는 정책 평면(`PolicySessionRole.COORDINATOR`)에서 파생 — 코디네이터도 무엇이든 할 수
있는 게 아니라 역할 기반 envelope 안에서만 동작. **충돌 없으면 비용 0** (매뉴얼의 주장이
코드로 확인됨).

---

## 8. 요약 비교표 — `ooo run` 내 모든 LLM 접점

| # | 호출 | 방식 | tools | temp | 프롬프트 소스 | 조건 |
| :-: | :--- | :--- | :--- | :---: | :--- | :--- |
| ① | 의존성 분석 | 1턴 | — | 0.0 | 인라인 상수 (dependency_analyzer.py:233) | AC ≥ 2 |
| ② | 분해 판정 | 1턴 | `[]` | 어댑터 기본 | 인라인 (parallel_executor.py:3694) | preflight 모드 |
| ③ | **AC 실행** | **멀티턴** | **R/W/E/Bash/Glob/Grep** | — | build_system_prompt + AtomicPromptBuilder | AC/Sub-AC마다 |
| ④ | 재시도 실행 | 멀티턴 | 동일 | — | ③ + 실패분류/에러꼬리/수평사고 | 실패 시 ≤2회 |
| ⑤ | Bounce 분류 | 1턴 | `[]` | — | 인라인 (:2682) | 실패 원인 모호 시 |
| ⑥ | 충돌 조정 | 멀티턴 | 정책 파생 (Read/Bash/Edit) | — | 인라인 (coordinator.py:65,537) | 파일 충돌 시 |
| ⑦ | QA/평가 | — | — | — | evaluation/ 파이프라인 | 실행 후 자동 |

시스템 프롬프트에 로드되는 페르소나 파일: `code-executor.md`(9줄) / `research-agent.md` /
`analysis-agent.md` — task_type에 따라 택1.

**★ Insight ─────────────────────────────────────**
페르소나 파일이 인터뷰(socratic-interviewer.md는 수십 줄의 정교한 규율)와 달리 code-executor는
9줄로 극단적으로 짧습니다. 이유는 구조에 있습니다: 실행 워커의 "규율"은 페르소나가 아니라
**AtomicPromptBuilder가 조립하는 태스크 프롬프트**(SUCCESS CONTRACT, 스코프 경계, 증거 스키마,
재시도 컨텍스트)에 실려 있고, 시스템 프롬프트에는 Seed Contract·진행 마커·자가회복 프로토콜이
붙습니다. 즉 인터뷰는 "누구인가"로 통제하고, 실행은 "계약이 무엇인가"로 통제합니다 —
페르소나 기반 통제 vs 계약 기반 통제의 대비가 이 코드베이스에서 가장 선명한 지점.
**─────────────────────────────────────────────────**

---

## 9. 미탐색 영역 (다음 분석 후보)

- **⑦ 평가 파이프라인 프롬프트**: Stage 2 Semantic(`semantic-evaluator.md`), Stage 3
  Consensus(`advocate.md`/`judge.md`/`consensus-reviewer.md`), QA(`qa-judge.md`) — evaluate 분석에서
- **모델/effort 라우팅 수치**: model_routing.py의 tier 결정 함수 (frugal→standard→frontier)
- **Sub-AC 실행 트리**: 분해 후 재귀 깊이(`max_decomposition_depth`)와 노드 identity 체계
- **비-Claude 런타임의 동일 프롬프트 전달**: codex_cli_runtime 등이 이 프롬프트를 어떻게 변환하는지

---

**Created**: 2026-07-23 (분석 기준 커밋 be041c43)
**연관 문서**: `agent_invocation_map.md`, `interview_agent_calls.md`, `interview_prompts_params.md`
