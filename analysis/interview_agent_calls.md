# Interview 스킬의 에이전트 호출 분석

> 핵심 발견: **interview는 `claude -p`가 아니라 Claude Agent SDK를 통해 CLI를 호출**한다.
> AC 실행(orchestrator)과 인터뷰(bigbang)는 서로 다른 두 개의 "Claude 서브프로세스 호출 방식"을 갖고 있다.

---

## 🔀 두 가지 서로 다른 메커니즘

| | AC 실행 (orchestrator) | Interview/모호도채점 (bigbang) |
| :--- | :--- | :--- |
| **파일** | `orchestrator/claude_worker_runtime.py` | `providers/claude_code_adapter.py` |
| **호출 방식** | 직접 `asyncio.create_subprocess_exec` | `claude_agent_sdk.query()` (SDK 경유) |
| **CLI 플래그** | `claude -p --output-format json` | `--output-format stream-json --input-format stream-json --verbose` |
| **프로토콜** | stdin에 prompt 문자열 → stdout JSON 한 줄 | stdin/stdout 스트리밍 JSON (SDK 내부 프로토콜) |
| **temperature 등 세밀 제어** | 불가 (CLI 플래그 없음) | SDK 옵션으로 일부 제어 가능 |
| **역할** | 실제 "코드를 쓰는" 작업 에이전트 | "질문하고 채점하는" 순수 텍스트 생성 |

**확인 근거** (SDK 소스, `claude_agent_sdk/_internal/transport/subprocess_cli.py:225,408`):
```python
cmd = [self._cli_path, "--output-format", "stream-json", "--verbose"]
...
cmd.extend(["--input-format", "stream-json"])
```
→ `-p`/`--print` 플래그는 SDK 전체에서 **한 번도 등장하지 않음** (grep 확인).

**★ Insight ─────────────────────────────────────**
같은 "Claude를 서브프로세스로 부른다"는 목표라도 **왜 두 개의 다른 방식**을 썼을까?
- AC 실행은 **한 번에 하나의 완결된 결과**(코드 diff, 파일 변경)만 필요 → 단순한 request/response로 충분.
- Interview/채점은 **SDK가 제공하는 세밀한 옵션**(`disallowed_tools`, `allowed_tools=[]`, `strict_mcp_config`, `max_turns=1` 등)으로 "이 서브 에이전트가 절대 도구를 쓰지 못하게" 봉인해야 함 — 순수 텍스트 생성 워크로드를 에이전틱 CLI에 태우면서 그 에이전틱 능력을 하나씩 꺼버리는 형태. 코드 주석에 스스로 "이 부조화가 구조적 빚(structural debt)"이라고 적혀 있음 (issue #869 관련).
**─────────────────────────────────────────────────**

---

## 📞 Interview 한 라운드당 실제 호출 횟수

**출처**: `mcp/tools/authoring_handlers.py:3235-3320` (ouroboros_interview MCP 핸들러의 답변 처리 로직)

```python
record_result = await engine.record_response(state, answer, pending_question)
# ↑ LLM 호출 없음 — 상태 업데이트만

await engine.save_state(state)
# ↑ LLM 호출 없음 — 디스크 저장만

answered = _count_answered_rounds(state)
if answered >= MIN_ROUNDS_BEFORE_EARLY_EXIT:   # = 3
    # 코드 주석 원문:
    # "Only score ambiguity when completion is actually possible.
    #  Before MIN_ROUNDS_BEFORE_EARLY_EXIT the result cannot trigger
    #  early exit, so the LLM call (~3-8s) is pure waste."
    live_score = await self._score_interview_state(llm_adapter, state)
    # ↑ ① 모호도 채점 LLM 호출 (AmbiguityScorer.score())

    if 조기종료조건_충족:
        return await self._complete_interview_response(...)
        # ↑ Seed 생성 트리거 (seed_generator.py) — 별도 LLM 호출 발생

    question_result = await engine.ask_next_question(state)
    # ↑ ② 다음 질문 생성 LLM 호출
else:
    question_result = await engine.ask_next_question(state)
    # ↑ ① 다음 질문 생성 LLM 호출만 (채점 스킵)
```

### 라운드별 호출 횟수 표

| 라운드 | 답변 처리 | 모호도 채점? | 질문 생성? | 총 LLM 호출 |
| :---: | :--- | :---: | :---: | :---: |
| 1~2 (`MIN_ROUNDS_BEFORE_EARLY_EXIT`=3 미만) | 상태 저장만 | ❌ 스킵 (비용 절약) | ✅ | **1회** |
| 3+ | 상태 저장만 | ✅ | ✅ (조기종료 아닐 시) | **2회** |
| 조기종료 성립 시 (`streak≥2`) | 상태 저장만 | ✅ | ❌ (`_complete_interview_response`로 분기) | **1회 + Seed 생성** |

**★ Insight ─────────────────────────────────────**
1~2라운드에서 채점을 건너뛰는 이유가 **코드 주석에 명시적으로 정당화**되어 있습니다: "이 시점 결과는 어차피 조기종료를 못 만드니, LLM 호출(~3-8초)은 순수 낭비다." — 3장에서 본 "비용 사다리" 원칙이 인터뷰 루프 내부에도 그대로 적용된 사례입니다. **결정론적 카운터(`answered >= 3`)가 LLM 호출 여부를 게이팅**합니다.
**─────────────────────────────────────────────────**

---

## 🎛️ 질문 생성의 두 경로 (기본 vs 옵션)

`bigbang/interview.py:836` — `ask_next_question()` 내부:

```python
if self.question_candidate_panel:      # 기본값: False (interview.py:655)
    candidate = await self._select_question_from_candidates(...)
    # ↳ 3개 페르소나(contrarian, architect, researcher) 병렬 LLM 호출
    #   (interview.py:946, asyncio.gather)
    if candidate is not None:
        return Result.ok(candidate)

result = await self.llm_adapter.complete(messages, config)
# ↑ 기본 경로: 단일 LLM 호출
```

- **기본 경로 (99% 실사용)**: 질문 1개 생성에 **LLM 호출 1회**
- **옵션 경로 (`question_candidate_panel=True`)**: 3개 페르소나가 각자 질문 후보를 동시 생성 → **LLM 호출 3회 병렬** → 결정론적 함수(`select_question_candidate`)가 그중 하나 선택. (이 옵션은 코드 기본값이 꺼져 있어, 현재 기본 `ooo interview` 사용 시엔 적용되지 않음)

---

## 📚 bigbang/ 패키지 전체의 LLM 호출 지점 목록

`grep "llm_adapter.complete("` 로 찾은 모든 지점 — 전부 위와 동일한 `ClaudeCodeAdapter`(기본 설정 시)를 경유:

| 파일 | 함수 | 역할 | 호출 시점 |
| :--- | :--- | :--- | :--- |
| `interview.py:846` | `ask_next_question` | 다음 소크라테스식 질문 생성 | 매 라운드 (조건부) |
| `interview.py:933` | `_generate_question_candidates._one` | 페르소나별 질문 후보 (3개 병렬) | 옵션(`question_candidate_panel`), 기본 OFF |
| `ambiguity.py:481` | `score` | 전체 차원 통합 모호도 채점 | 3라운드 이상부터 |
| `ambiguity.py:911` | `_score_single_dimension` | 차원별 개별 채점 (병렬) | 옵션(`per_dimension=True`), 기본 OFF |
| `seed_generator.py:485` | `_extract_requirements` | 인터뷰 완료 시 Seed 요구사항 추출 | 인터뷰 완료(조기종료/등급 통과) 시 1회 |
| `question_classifier.py:262` | `classify` | 입력이 dev 인터뷰 vs PM 인터뷰 중 어디로 갈지 라우팅 | 인터뷰 시작 시 1회 |
| `brownfield.py:323` | `generate_desc` | Brownfield 저장소 설명 생성 | `ooo brownfield` 스캔 시 |
| `explore.py:471` | `_summarize_with_llm` | 코드베이스 탐색 결과 요약 | Brownfield 인터뷰 컨텍스트 구축 시 |
| `pm_interview.py:1129` | (PM 질문 생성) | `ooo pm` 전용 인터뷰 질문 생성 | PM 인터뷰 라운드마다 |
| `pm_document.py:312` | (PRD 생성) | PRD 문서 생성 | PM 인터뷰 완료 시 |

---

## 🧮 종합: 전형적인 `ooo interview` 세션의 총 호출 횟수 추정

예: 5라운드만에 수렴하는 그린필드 인터뷰

```
Round 1: 질문 생성만          → 1회
Round 2: 질문 생성만          → 1회
Round 3: 채점 + 질문 생성     → 2회
Round 4: 채점 + 질문 생성     → 2회
Round 5: 채점 → 조기종료 성립 → 1회 (채점만, 질문 생성 스킵)
Seed 생성 (_extract_requirements) → 1회 이상 (요구사항 추출 로직에 따라 다회 가능)
──────────────────────────────────
합계: 7~8회 이상의 별도 `claude` 서브프로세스 스폰
(+ question_classifier의 라우팅 호출 1회, 인터뷰 시작 시)
```

**참고**: 이건 전부 **텍스트 생성 전용** 호출입니다. AC 실행(orchestrator, `ooo run`)에서 보던 "실제 코드를 쓰는" `claude -p` 호출과는 완전히 별개 — interview 단계에서는 파일 시스템에 손도 대지 않습니다 (allowed_tools를 비워서 도구 접근 자체를 봉인).

---

## 🔒 재귀 방지 장치 (Interview 전용)

`claude_code_adapter.py`의 `strict_mcp_config` + `_ISOLATION_OVERRIDES`:

```python
_ISOLATION_OVERRIDES = (
    ("setting_sources", []),
    ("skills", []),
    ("agents", {}),
    ("plugins", []),
    ("hooks", {}),
    ("include_hook_events", False),
)
```

인터뷰용 서브 에이전트가 부모 Claude Code 세션의 스킬/플러그인/훅을 상속받으면, 그 서브 에이전트가 다시 `ooo interview`를 트리거해 **무한 재귀**에 빠질 수 있습니다. 이걸 막기 위해 MCP 서버 발견 자체를 차단(`strict_mcp_config`)하고, 스킬/에이전트/플러그인 상속 경로를 하나씩 명시적으로 꺼버립니다.

---

**Created**: 2026-07-23
**연관 문서**: `agent_invocation_map.md` (AC 실행 흐름 — `claude -p` 방식)
