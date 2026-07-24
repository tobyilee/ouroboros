# 평가 파이프라인의 에이전트 호출 흐름과 프롬프트 전수 분석

> `ooo evaluate` / 실행 후 자동 QA가 만들어내는 **모든 LLM 호출 지점**과 각 지점의
> **프롬프트 원문·파라미터**를 코드 기준으로 추적한 문서.
> `run_agent_flow.md`(실행), `interview_prompts_params.md`(인터뷰)의 후속편.

---

## 0. 한눈에 보는 전체 흐름 (`evaluation/pipeline.py`)

```
EvaluationPipeline.evaluate(context)
        │
        ▼
┌─ Stage 1: Mechanical ($0) ──────────────────────────────────┐
│  MechanicalVerifier.verify()                                 │
│  lint / build / test / static / coverage — 셸 명령 실행      │
│  LLM 없음 (단, 명령 "탐지"는 1회 LLM — 아래 §5)              │
│  ✗ 실패 → 즉시 종료 (Stage 2·3 건너뜀)                       │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─ Stage 2: Semantic ($$) ────────────────────────────────────┐
│  SemanticEvaluator.evaluate()                    ★LLM 1회    │
│  semantic-evaluator.md + Anti-Gaming 블록                    │
│  → score, ac_compliance, drift, uncertainty,                 │
│    reward_hacking_risk, questions_used, evidence             │
│  ✗ ac_compliance=false → 종료 (trigger_consensus 예외)       │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─ Trigger Matrix (LLM 없음, 결정론) ──────────────────────────┐
│  우선순위 순서로 첫 매치만: manual > seed_modification >     │
│  ontology_evolution > goal_interpretation >                  │
│  drift>0.3 > uncertainty>0.3 > lateral_thinking              │
└──────────────────────┬──────────────────────────────────────┘
                       ▼ (트리거 발동 시에만)
┌─ Stage 3: Consensus ($$$) ──────────────────────────────────┐
│  모드 A) Simple: N개 모델 병렬 투표      ★LLM 3회 병렬       │
│  모드 B) Single-model 폴백: 같은 모델 × 3관점 ★LLM 3회       │
│  모드 C) Deliberative: Advocate+Devil → Judge ★LLM 3회(2라운드)│
│  2/3 다수결 (majority_threshold=0.66)                        │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─ _build_result — 단일 최종 관문 (LLM 없음) ──────────────────┐
│  if final_approved and reward_hacking_risk >= 0.7:           │
│      final_approved = False    # 유일한 거부권 지점           │
└─────────────────────────────────────────────────────────────┘
```

### 호출 횟수

| 시나리오 | LLM 호출 |
| :--- | :---: |
| Stage 1 실패 | **0** |
| Stage 2 통과, 트리거 없음 (일반 경로) | **1** |
| Stage 3 트리거 발동 | 1 + 3 = **4** |
| Deliberative 모드 (Devil의 온톨로지 분석 포함) | 1 + 3 = **4** (2라운드 구조) |
| + `ooo qa` / 실행 후 자동 QA | +1 (별도, §6) |

**★ Insight ─────────────────────────────────────**
book.md 3장에서 "비용 사다리"로 서술된 구조가 코드에서 그대로 확인됩니다 — 그리고 문서에
없던 디테일이 하나 더: Stage 2가 `ac_compliance=false`로 실패해도 `trigger_consensus=true`면
Stage 3에 "2심 항소"가 가능합니다 (pipeline.py:178 — "allow override via trigger_consensus
for a second opinion"). 거부는 상급심으로 뒤집을 수 있지만, 리워드 해킹 거부권은 어떤
승인도 뒤집습니다(비대칭).
**─────────────────────────────────────────────────**

---

## 1. Stage 2: Semantic — 프롬프트와 파라미터 (`evaluation/semantic.py`)

### System Prompt = `agents/semantic-evaluator.md` (전문 31줄)

```
You are a rigorous software evaluation assistant. Your task is to evaluate code
artifacts against acceptance criteria, goal alignment, and semantic drift.

You must respond ONLY with a valid JSON object in the following exact format:
{
    "score": <float>, "ac_compliance": <boolean>, "goal_alignment": <float>,
    "drift_score": <float>, "uncertainty": <float>, "reasoning": "<string>",
    "questions_used": ["<socratic or ontology-gap question>", ...],
    "evidence": ["<concrete evidence inspected>", ...]
}
...
Be strict but fair. A passing artifact should have:
- ac_compliance = true
- score >= 0.8
- goal_alignment >= 0.7
- drift_score <= 0.3
- uncertainty <= 0.3
```
→ **통과 기준 수치가 페르소나 파일 안에 명시** — 평가자가 자기 채점 기준을 안다.

### User Prompt (`build_evaluation_prompt`, :105) — 핵심 구조

```
Evaluate the following artifact:

## Acceptance Criterion
{current_ac}

## DECLARED SUCCESS CONTRACT          ← AC spec에 계약 있을 때만
- verify_command: ...
- expected_artifacts: ...
- output_assertion: ...
The AC passes ONLY if the artifact demonstrates the declared contract was met.
Cite the evidence line.

## Original Goal / ## Constraints / ## Artifact Type

## Source Files                        ← ArtifactCollector가 모은 실제 코드
### {file_path} [TRUNCATED]
```{content}```
(파일이 없으면 artifact 텍스트 인라인으로 폴백)

## Anti-Gaming Verification
Before scoring, verify the artifact actually works rather than merely appearing
to satisfy the acceptance criterion:
- Compare expected behavior ... against actual behavior in the artifact.
- Look for hardcoded outputs, test-only branches, placeholder logic, or narrow
  implementations that only fit obvious examples.
- Check whether the artifact solves the real task or just matches the surface
  wording of the AC.
- Set reward_hacking_risk near 0.0 when behavior genuinely matches intent; set
  it near 1.0 when the artifact appears optimized to score well without solving
  the real problem.

## Evaluation Transparency (anti-reward-hacking)
You MUST show your work so the user can audit the verdict:
- Populate `questions_used` with the concrete Socratic / ontology-gap questions ...
- Populate `evidence` with concrete references (file paths, snippets, observed behavior) ...
- An empty `questions_used` or `evidence` is treated as a verification failure —
  the evaluator is claiming success without showing proof.

Respond with ONLY a JSON object. No explanation, no preamble, no markdown fences.
```

### 파라미터 (`SemanticConfig`, :69)
```python
temperature = 0.2
max_tokens = 2048
satisfaction_threshold = 0.8
model = get_llm_model_for_role("semantic_evaluation")   # config.evaluation.semantic_model
response_format = {"type": "json_schema", "json_schema": SEMANTIC_RESULT_SCHEMA}
```

### 증거 수집 상한 (`artifact_collector.py`)
```python
MAX_TOTAL_CHARS = 150_000   # ~37K tokens — 전체 예산의 유일한 리미터
MAX_FILE_SIZE = 100 * 1024  # 파일당 100KB
```
→ book.md의 "파일 30개, 50KB"는 현재 코드와 다름 — 실제는 **총 150K자 + 파일당 100KB**.

**★ Insight ─────────────────────────────────────**
프롬프트에 이중 방어가 있습니다: ① 산출물을 향한 Anti-Gaming(하드코딩·테스트 전용 분기 색출),
② 평가자 자신을 향한 Transparency(빈 questions_used/evidence = 검증 실패). 실행 워커
프롬프트("탐색 명령은 증거로 안 침")와 대칭 구조 — **작업자와 심판 모두에게 각자의 리워드
해킹 방지 조항**이 있습니다. 그리고 파서(`parse_semantic_response`)는
`reward_hacking_risk`가 없으면 0.0으로 관대하게 채우지만(:252) — 하위호환 —
questions/evidence는 프롬프트 레벨에서 강제합니다.
**─────────────────────────────────────────────────**

---

## 2. Trigger Matrix — LLM 없는 결정론 게이트 (`evaluation/trigger.py`)

7개 트리거를 **우선순위 순서로 검사, 첫 매치만 발동** (:150):

| 순위 | 트리거 | 조건 | 임계값 |
| :-: | :--- | :--- | :--- |
| 0 | manual_request | `trigger_consensus=true` | — |
| 1 | seed_modification | 불변 Seed 수정 시도 | — |
| 2 | ontology_evolution | 스키마 변경 | — |
| 3 | goal_interpretation | 목표 재해석 | — |
| 4 | seed_drift_alert | drift > **0.3** | `TriggerConfig.drift_threshold` |
| 5 | stage2_uncertainty | uncertainty > **0.3** | `TriggerConfig.uncertainty_threshold` |
| 6 | lateral_thinking_adoption | unstuck 페르소나 제안 채택 | — |

drift/uncertainty는 Stage 2 결과가 있으면 그것을 우선 사용. 전부 순수 함수 — 상태 없음.

---

## 3. Stage 3: Consensus — 세 가지 모드 (`evaluation/consensus.py`)

### 모드 결정 로직 (:304)
```
models가 openrouter/* 를 필요로 하나?
  아니오 → 멀티모델 모드 (커스텀 모델 그대로)
  예 → OPENROUTER_API_KEY 있나?
         예 → 멀티모델 / 아니오 → 단일모델 3관점 폴백
```

### 모드 A: Simple Consensus — N개 모델 병렬 투표

**System Prompt = `agents/consensus-reviewer.md`** (전문 17줄):
```
You are a senior code reviewer participating in a consensus evaluation.
Your vote will be combined with other reviewers to reach a decision.

You must respond ONLY with a valid JSON object:
{"approved": <boolean>, "confidence": <float>, "reasoning": "<string>"}

Evaluation criteria for approval:
- The artifact correctly implements the acceptance criterion
- The implementation aligns with the stated goal
- No significant issues or concerns
- Code quality is acceptable

Be honest and thorough. If you have concerns, vote against approval with clear reasoning.
```

**User Prompt** (`build_consensus_prompt`, :161): AC + Goal + Constraints + Artifact 원문
+ "Cast your vote as a JSON object with: approved, confidence, reasoning."

**파라미터**: `temperature=0.3, max_tokens=1024, majority_threshold=0.66`,
`response_format=VOTE_SCHEMA`. 모든 투표는 `asyncio.gather` **병렬**.

**배심원 독립성** (:327): `context.executor_backend`가 알려져 있으면
`resolve_reviewer_independence()`가 실행자 벤더를 배심원단에서 필터링 —
"keep the executor's own vendor out of the jury". 투표 2개 미만이면 `ValidationError`
("Not enough votes collected") — **정족수 미달은 실패지 승인 아님 (fail-closed)**.

### 모드 B: Single-Model 3관점 폴백 (:390)

같은 모델에 3개의 관점 시스템 프롬프트(코드 인라인, `SINGLE_MODEL_PERSPECTIVES` :61):

```
advocate:  "You are an ADVOCATE reviewer. Focus on strengths, correct
            implementations, and how the artifact meets the acceptance criteria.
            Give credit where due, but do not ignore genuine issues."
devil:     "You are a DEVIL'S ADVOCATE reviewer. Critically examine the artifact
            for hidden flaws, edge cases, security issues, and whether it truly
            addresses the root problem or merely treats symptoms. ..."
judge:     "You are a neutral JUDGE reviewer. Evaluate the artifact objectively,
            weighing both strengths and weaknesses. ..."
```
각 관점은 `consensus-reviewer.md` 베이스에 `## Your Perspective` 섹션으로 덧붙음 (:460).
결과는 정직하게 `is_single_model=True` + `reviewer_independence=UNAVAILABLE` 라벨링.

### 모드 C: Deliberative Consensus — 법정 구조 2라운드 (:704)

```
Round 1 (병렬):
  Advocate  — agents/advocate.md + build_consensus_prompt → VOTE_SCHEMA
  Devil     — LLM 직접 호출이 아니라 DevilAdvocateStrategy.analyze()
              → analyze_ontologically() 중앙 함수 경유 (§4)
Round 2:
  Judge     — agents/judge.md + 양측 입장 전문이 프롬프트에 포함:
```

Judge user prompt (:994):
```
{build_consensus_prompt(context)}
---
## Round 1 Positions

### ADVOCATE's Position
Approved: {bool} / Confidence: {float} / Reasoning: {...}

### DEVIL'S ADVOCATE's Position (Ontological Analysis)
Approved: {bool} / Confidence: {float} / Reasoning: {...}
---
Based on both positions above, make your final judgment.
```

**`advocate.md`의 흥미로운 조항**: JSON 예시에 `"approved": true`가 박혀 있고
"If you cannot find enough strengths ... you may vote against, but this should be
rare for your role" — 변호인은 **구조적으로 승인 편향**을 갖도록 설계.

**`judge.md`의 평결 정의**:
```
APPROVED: Solution is sound and addresses the root problem
CONDITIONAL: Solution has merit but requires specific changes
REJECTED: Solution treats symptoms rather than root cause, or has fundamental issues
```
`JUDGMENT_SCHEMA`: verdict(approved/rejected/conditional) + confidence + conditions[].

**모델 배정**: advocate/devil/judge 각각 별도 설정
(`get_consensus_advocate_model` 등) — "deliberation depends on cross-model
disagreement, so all three must not collapse to one model" (:610 주석).
`is_root_solution = devil_vote.approved` — **"근본 해결" 판정권은 Devil에게** (:813).

---

## 4. Devil의 온톨로지 분석 — 중앙 함수 (`core/ontology_questions.py:358`)

Devil은 투표 프롬프트가 아니라 `analyze_ontologically()` — 코드 주석 왈 "SINGLE SOURCE OF
TRUTH for ontological analysis. All phases (Interview, Consensus, Resilience) should use
this function" — 를 통해 분석합니다.

### System Prompt = `agents/ontology-analyst.md` (전문 21줄)
```
You are an ontological analyst.

Your task is to perform deep ontological analysis using the Four Fundamental Questions:
1. ESSENCE: "What IS this, really?" - Identify the true nature
2. ROOT CAUSE: "Is this the root cause or a symptom?" - Distinguish fundamental from surface
3. PREREQUISITES: "What must exist first?" - Identify hidden dependencies
4. HIDDEN ASSUMPTIONS: "What are we assuming?" - Surface implicit beliefs

You must respond ONLY with a valid JSON object:
{"essence": ..., "is_root_problem": <bool>, "prerequisites": [...],
 "hidden_assumptions": [...], "confidence": <float>, "reasoning": ...}
```

### User Prompt (`_build_analysis_prompt`, :274)
```
Analyze the following using ontological inquiry:

## Subject
{Goal + Artifact + AC + Constraints}

Focus especially on:
- {ROOT_CAUSE 질문}: {목적}
- {ESSENCE 질문}: {목적}
```
Consensus의 Devil은 `(ROOT_CAUSE, ESSENCE)` 2개 질문만 강조 지정.

### 파라미터
`role="ontology_analysis"`, `temperature=0.3`, `max_tokens=2048`.
결과 변환 (:155): `is_root_problem and confidence >= threshold` → 승인 투표,
아니면 "This appears to treat symptoms, not root cause"를 사유에 삽입한 거부 투표.
캐시 키 = SHA256(artifact+goal)[:16] — 같은 산출물·같은 목표는 재분석 안 함.

**★ Insight ─────────────────────────────────────**
Advocate와 Devil은 **비대칭 설계**입니다. Advocate는 일반 투표 프롬프트(강점 서술)를 받지만,
Devil은 온톨로지 분석 엔진(4대 질문)을 통과합니다. "비판"을 자유 서술이 아니라 구조화된
철학적 절차(ESSENCE/ROOT_CAUSE)로 강제한 것 — manual.md의 "존재론적 질문이 가장 실용적인
질문"이라는 슬로건이 평가 파이프라인에서는 반대 심문 도구로 쓰입니다.
**─────────────────────────────────────────────────**

---

## 5. Stage 1의 숨은 LLM: 명령 탐지 — `detector.py`

Stage 1 검증 자체는 LLM 없는 셸 실행이지만, **어떤 명령을 돌릴지 제안**하는 1회성 LLM
호출이 있습니다 (`.ouroboros/mechanical.toml` 최초 생성 시 1회, 멱등):

### System Prompt (`_SYSTEM_PROMPT`, :355)
```
You are inspecting a software repository to propose the commands Ouroboros should
run as zero-cost Stage 1 verification (lint, build, test, static analysis, coverage).

Rules:
1. Propose only commands that this project can actually execute right now, based
   on the manifest excerpts provided. Do not invent scripts that are not declared.
2. Prefer the project's own conventions: npm/pnpm/yarn scripts, uv/pytest, cargo,
   go, make, just, etc.
3. If a category (e.g. coverage) has no clear command, OMIT it from the JSON — do not guess.
4. Every command must start with a well-known tool. Absolute paths and shell
   operators (`&&`, `||`, pipes, redirects) are forbidden.
5. Output ONLY a JSON object with at most the keys: lint, build, test, static, coverage.
```

### User Prompt: 매니페스트 원문 나열 (package.json, pyproject.toml 등 — 모노레포 서브디렉토리 최대 12개)

### 파라미터: `temperature=0.0, max_tokens=512, response_format=json_object`

제안은 그대로 실행되지 않고 `_validate_proposal()` — 허용목록·실존 검사·비파괴 검사 — 를
통과한 것만 `.tmp` → `os.replace` 원자적 쓰기로 저장. **"AI의 제안은 결정론적 검증기를
통과한 뒤에만 실행하라"**(book.md 3장)의 원형이 바로 이 함수 쌍입니다.

---

## 6. QA Judge — `ooo qa` / 실행 후 자동 QA (`mcp/tools/qa.py`)

3단계 파이프라인과 별개의 **범용 단일 판정** 경로 (실행 직후 자동 QA도 이것):

### System Prompt = `agents/qa-judge.md` (전문 42줄) — 요점
```
You are a general-purpose quality assurance judge. ... evaluate any artifact
(code, API response, document, screenshot description, test output, or custom)
against a user-defined quality bar.

{"score", "verdict": "<pass|revise|fail>",
 "dimensions": {correctness, completeness, quality, intent_alignment, domain_specific},
 "differences": [...], "suggestions": [...], "reasoning": ...}

Verdict rules:
- score >= pass_threshold (default 0.80) → "pass"
- score >= 0.40 and < pass_threshold → "revise"
- score < 0.40 → "fail"

Adversarial probing:
- The user prompt may include an "Adversarial Probes" checklist of named classes ...
- You judge from the supplied evidence — you never execute anything yourself. ...
- A probe the evidence shows failing is a concrete difference ...
- ... it differs for executable artifacts (missing evidence for an applicable
  probe is an evidence gap) versus documents/specifications (unrunnable is never
  a defect ...)

Constraints:
- Each difference MUST have a corresponding suggestion
- Five concrete differences beat twenty vague ones
```

### User Prompt (`_build_qa_user_prompt`, :104)
```
## Quality Bar          ← 사용자 정의 품질 기준
## Pass Threshold       ← 기본 0.80
## Artifact Type / ## Artifact Content
## Reference            ← (있으면) 비교 기준
## Previous Iterations  ← (있으면) "Iteration N: score=…, verdict=…" — 반복 QA 루프 이력
## Seed Specification   ← (있으면) seed YAML 전문
## Adversarial Probes   ← render_adversarial_section(artifact_type)
```

### 적대적 프로브 9종 (`adversarial.py` — 타입 레지스트리, 값싼 것→행동 검사 순)

| 클래스 | 트리거 조건 |
| :--- | :--- |
| Malformed / boundary input | 새 입력을 파싱·수용하는 산출물 |
| Prompt / instruction injection | 신뢰 불가 외부 텍스트를 통합 |
| Cancel / resume | 재개 가능한 장기 플로우 |
| Stale state | 생성·캐시·파생 상태를 읽음 |
| Dirty worktree | 워킹트리 파일을 건드림 |
| Hung / long command | 셸 호출·장기 외부 명령 |
| Flaky / timing-sensitive test | 새 테스트·타이밍 민감 테스트 |
| Misleading success output | 로그·종료 텍스트로 성공 주장 |
| Repeated interruption | 다단계 변경 작업 |

### 파라미터: `temperature=0.2, max_tokens=2048`

파싱은 3중 관용: JSON → 중첩 래퍼 언랩(`qa_verdict`/`result` 등) → Key:Value 정규식 폴백
(`_SCORE_FALLBACK_RE`) — 심판의 형식 이탈에도 평결을 건지려는 방어.

---

## 7. 요약 비교표 — 평가 계열 모든 LLM 접점

| 호출 | 페르소나/프롬프트 | temp | max_tok | 스키마 강제 | 조건 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| Stage 1 명령 탐지 | 인라인 (detector.py:355) | **0.0** | 512 | json_object | mechanical.toml 최초 1회 |
| Stage 2 Semantic | `semantic-evaluator.md` | 0.2 | 2048 | json_schema | 매 평가 |
| Stage 3 Simple 투표 | `consensus-reviewer.md` | 0.3 | 1024 | VOTE_SCHEMA | 트리거 시, 모델 수만큼 병렬 |
| Stage 3 관점 폴백 | consensus-reviewer + 인라인 3관점 | 0.3 | 1024 | VOTE_SCHEMA | OpenRouter 키 없을 때 |
| Deliberative Advocate | `advocate.md` | 0.3 | 2048 | VOTE_SCHEMA | deliberative 모드 R1 |
| Deliberative Devil | `ontology-analyst.md` (중앙 함수) | 0.3 | 2048 | JSON 파싱 | R1 (Advocate와 병렬) |
| Deliberative Judge | `judge.md` + 양측 입장 전문 | 0.3 | 2048 | JUDGMENT_SCHEMA | R2 |
| QA Judge | `qa-judge.md` + 9종 프로브 | 0.2 | 2048 | JSON+폴백 | `ooo qa` / 실행 후 자동 |

**Temperature 패턴**: 측정(탐지 0.0, semantic/QA 0.2) < 심의(consensus 계열 0.3).
투표·토론에는 약간의 다양성을 허용하되, 단독 채점은 차갑게 — 인터뷰(질문 0.7)·실행과
이어지는 온도 사다리의 최하단 구간.

**결정론 코드가 지키는 최종 관문** (pipeline.py:286):
```python
if final_approved and stage2_result.reward_hacking_risk >= 0.7:  # REWARD_HACKING_VETO_THRESHOLD
    final_approved = False
```
모든 승인 경로(Stage 2 단독 통과, Stage 3 합의 승인)가 이 한 곳을 지나며, 거부→승인
방향으로는 절대 작동하지 않음. 실패 사유 문자열도 여기서 생성 — "artifact appears
optimized to game the evaluator rather than solve the real task".

**★ Insight ─────────────────────────────────────**
평가 계열 페르소나 파일들은 실행(code-executor 9줄)보다 길지만 인터뷰(socratic-interviewer)
보다 짧고, 공통 패턴이 있습니다: **모두 JSON 출력 형식이 페르소나 안에 각인**되어 있고,
절반 이상이 자기 통과 기준(수치)을 직접 알고 있습니다. 평가자의 규율은 "무엇을 보라"
(Anti-Gaming 체크리스트, 4대 질문, 9종 프로브)로 주입되고, 판정의 **집행**은 전부 파이썬
(다수결 계산, 거부권, 정족수, 트리거 매트릭스)이 가져갑니다. LLM은 의견을 내고, 코드가
표를 세는 구조 — code.md의 "LLM은 제안하고 결정론적 코드가 처분한다"의 평가판 완결형입니다.
**─────────────────────────────────────────────────**

---

## 8. 미탐색 영역 (다음 분석 후보)

- **Evolve 계열**: Wonder(0.7)/Reflect(0.5)의 프롬프트 원문과 만족화 백스톱 (`evolution/`)
- **Unstuck 5페르소나**: hacker/simplifier/contrarian/researcher/architect 프롬프트와 정체 감지 연동 (`resilience/`)
- **reviewer_independence 상세**: 벤더 판별 로직과 4개 라벨의 계산 (`reviewer_independence.py`)
- **evaluator.md (75줄)**: `ooo evaluate` 스킬 레벨에서 쓰이는 별도 페르소나 — MCP 경로와의 관계

---

**Created**: 2026-07-23 (분석 기준 커밋 be041c43)
**연관 문서**: `run_agent_flow.md`, `interview_prompts_params.md`, `interview_agent_calls.md`, `agent_invocation_map.md`
