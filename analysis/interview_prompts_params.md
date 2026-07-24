# Interview 계열 LLM 호출의 프롬프트 & 파라미터 전수 조사

> 모든 호출은 `self.llm_adapter.complete(messages, config)` → 기본 설정 시 `ClaudeCodeAdapter` →
> `claude_agent_sdk.query()` 경유. `messages`(system+user)와 `CompletionConfig`를 지점별로 정리.

---

## 0. 공통 파라미터 구조 (`providers/base.py:45`)

```python
@dataclass(frozen=True, slots=True)
class CompletionConfig:
    model: str
    temperature: float = 0.7      # 기본값. 각 호출부에서 오버라이드
    max_tokens: int = 4096
    stop: list[str] | None = None
    top_p: float = 1.0
    response_format: dict | None = None
    role: str | None = None       # llm_profiles 조회 키 (모델 라우팅용)
    profile: str | None = None
    max_turns: int | None = None
    reasoning_effort: Literal["low","medium","high"] | None = None
    model_is_explicit: bool = False
```

**모델 해석 우선순위** (`config/loader.py:get_llm_model_for_role`):
```
explicit_model 인자 있으면 그것 사용
  → 없으면 role → Stage 매핑 (INTERVIEW/REFLECT/그외)
    → Stage.INTERVIEW  → config.clarification.default_model (기본: Opus 계열)
    → Stage.REFLECT    → config.resilience.reflect_model (기본: Opus 계열)
    → 그 외            → config.evaluation.semantic_model (기본: Opus 계열)
```
→ 별도 설정이 없으면 **인터뷰 관련 호출은 전부 기본적으로 Opus급 모델**을 씀 (`DEFAULT_OPUS_MODEL`).

---

## 1. 질문 생성 — `ask_next_question()` (`bigbang/interview.py:846`)

### System Prompt 조립 (`_build_system_prompt`, :1123)

```
base_prompt = load_agent_prompt("socratic-interviewer")   # agents/socratic-interviewer.md 원문 로드
             (suppress_tool_use_prompt_cues=True 인 경우: _TOOLLESS_INTERVIEW_BASE_PROMPT 사용)

dynamic_header (라운드 1일 때):
"""
You are an expert requirements engineer conducting a Socratic interview.

CRITICAL: Start your FIRST response with a DIRECT QUESTION about the project.
Do NOT introduce yourself. Do NOT say "I'll conduct" or "Let me ask".
Just ask a specific, clarifying question immediately.

This is Round {N}. Your ONLY job is to ask questions that reduce ambiguity.

Initial context: {prompt_initial_context}
"""

+ 답변 접두어 안내:
"""
Answer prefixes the caller may use:
- [from-code]: Existing codebase state (factual, read from files).
- [from-user]: Human decisions/judgments.
- [from-research]: Externally researched information (API docs, pricing, compatibility).
"""

+ (Brownfield인 경우) "This is a BROWNFIELD project... Focus on INTENT and DECISIONS, not on discovering what exists."

+ (있으면) 현재 모호도 스냅샷:
"""
## Current Ambiguity Snapshot
- Overall ambiguity: 0.34
- Milestone: **PROGRESSING** — ...
"""

최종 조합: f"{dynamic_header}\n{trimmed_base}\n\n{perspective_panel}"
```

**문자수 예산 관리** (프롬프트 자체가 "결정론적 코드로 강제"되는 지점):
- `_MAX_SYSTEM_PROMPT_CHARS = 3500`
- `_MIN_SYSTEM_PROMPT_CHARS = 1200`
- `_MAX_INITIAL_CONTEXT_SYSTEM_CHARS = 1800`
- 초과 시 `dynamic_header` 우선 보존 → `perspective_panel` → `base_prompt` 순으로 잘라냄 (헤더가 가장 중요하다는 설계 판단)

### User Prompt
= 지금까지의 **대화 히스토리 전체** (`conversation_history`, 질문/답변 쌍들), 예산 초과 시 `_trim_messages_to_budget()`로 트리밍.

### CompletionConfig (:817)
```python
CompletionConfig(
    model=self.model,            # 기본: get_llm_model_for_role("interview") → Opus급
    role="clarification",
    temperature=self.temperature,  # InterviewEngine 기본값 = 0.7
    max_tokens=self.max_tokens,     # InterviewEngine 기본값 = 2048
)
```

**★ temperature=0.7 (창의적 편)** — 다양한 질문을 탐색적으로 생성해야 하므로 다른 지점(채점 0.1, 추출 0.2)보다 높음.

---

## 2. 모호도 채점 — `AmbiguityScorer.score()` (`bigbang/ambiguity.py:481`)

### System Prompt (`_build_scoring_system_prompt`, :599) — Greenfield 버전

```
You are an expert requirements analyst. Evaluate the clarity of software requirements.

Evaluate three components:
1. Goal Clarity (40%): Is the goal specific and well-defined?
2. Constraint Clarity (30%): Are constraints and limitations specified?
3. Success Criteria Clarity (30%): Are success criteria measurable?

Score each from 0.0 (unclear) to 1.0 (perfectly clear). Scores above 0.8 require very specific requirements.

IMPORTANT: If the additional context lists "decide-later" or "deferred" items, these
are INTENTIONAL deferrals — the team has deliberately chosen to postpone those
decisions. Do NOT penalise the clarity score for intentionally deferred items.
Score only what is present and answerable.

RESPOND ONLY WITH VALID JSON. No other text before or after.

Required JSON format:
{"goal_clarity_score": 0.0, "goal_clarity_justification": "string",
 "constraint_clarity_score": 0.0, "constraint_clarity_justification": "string",
 "success_criteria_clarity_score": 0.0, "success_criteria_clarity_justification": "string"}
```

Brownfield 버전은 위에 **Context Clarity (15%)** 차원 + JSON 필드 하나가 추가되고 나머지 세 가중치는 35/25/25로 재조정됨.

### User Prompt (`_build_scoring_user_prompt`, :652)
```
Please evaluate the clarity of the following requirements conversation:

---
{context}   # 인터뷰 전체 대화 내역
---

Additional context (intentional deferrals — do not penalise):
{additional_context}   # decide-later 항목 (있을 때만)

Analyze each component and provide scores with justifications.
```

### CompletionConfig (:473)
```python
CompletionConfig(
    model=self.model,           # 기본: get_llm_model_for_role("clarification")
    role="ambiguity",
    temperature=self.temperature,  # = SCORING_TEMPERATURE = 0.1 (고정)
    max_tokens=current_max_tokens,  # 초기 2048, 잘림(finish_reason=="length") 시 2배씩 증가
)
```

**★ temperature=0.1 (거의 결정론적)** — 책(book.md 2장)에서 강조한 바로 그 수치. "재현 가능한 채점"이 존재 이유.

### 옵션 경로: 차원별 개별 채점 (`per_dimension=True`, 기본 OFF)
- `_build_dimension_system_prompt()` — 위 프롬프트에서 **해당 차원 하나의 rubric만** 남기고, JSON도 `{"clarity_score": 0.0, "justification": "string"}` 단일 필드로 축소.
- `score_per_dimension()`에서 `asyncio.gather`로 **차원 수만큼(3~4개) 병렬 호출**.
- 같은 `temperature=0.1`, 같은 rubric 텍스트 — "패키징(1콜 vs N콜)만 다르고 채점 기준은 동일해야 한다"는 주석 명시.

---

## 3. Seed 요구사항 추출 — `SeedGenerator._extract_requirements()` (`bigbang/seed_generator.py:445`)

### System Prompt
```python
load_agent_prompt("seed-architect")   # agents/seed-architect.md 원문 그대로 사용
```
→ 2장 분석에서 인용한 "AC는 3~7개, 결과 목록이지 구현단계 아님" 등의 규율이 **여기서 그대로 시스템 프롬프트**가 됩니다.

### User Prompt (`_build_extraction_user_prompt`, :652)
```
Extract structured requirements from the following interview conversation.

---
{context}
---

Respond ONLY with the structured format below. Do NOT add explanations, questions,
commentary, or prose. Do NOT wrap in markdown code blocks.

ACCEPTANCE_CRITERIA rule: produce 3-7 outcome-level criteria. Each is one
independently valuable, user-visible outcome — NOT an implementation step. ...
ACCEPTANCE_CRITERIA verify rule: `verify` must be one complete single-line shell
command. Never use heredoc or multiline syntax...
ACCEPTANCE_CRITERIA expect rule: `expect` is ONLY a literal string printed
verbatim in stdout... Use `expect: NONE` for exit-code/status conditions...

GOAL: <clear goal statement>
CONSTRAINTS: <constraint 1> | <constraint 2> | ...
ACCEPTANCE_CRITERIA:
AC: <description> | verify: <command or NONE> | artifacts: <comma-list or NONE> | expect: <output assertion or NONE>
ONTOLOGY_NAME: <name>
ONTOLOGY_DESCRIPTION: <description>
ONTOLOGY_FIELDS: <name>:<type>:<description> | ...
EVALUATION_PRINCIPLES: <name>:<description>:<weight> | ...
EXIT_CONDITIONS: <name>:<description>:<criteria> | ...
```
→ 자유 텍스트 JSON이 아니라 **파이프(`|`) 구분 라인 포맷**을 강제 — `_parse_extraction_response()`가 정해진 접두어(`_KNOWN_PREFIXES`)로 파싱하기 때문. 실패 시 `_MAX_EXTRACTION_RETRIES=1`회 재시도.

### CompletionConfig (:473)
```python
CompletionConfig(
    model=self.model,           # 기본: get_llm_model_for_role("seed_generation")
    role="seed_generation",
    temperature=self.temperature,  # = EXTRACTION_TEMPERATURE = 0.2
    max_tokens=self.max_tokens,     # 기본 4096
)
```

---

## 4. PM 인터뷰 (`bigbang/pm_interview.py`)

PM 인터뷰는 **완전히 새로운 프롬프트 체계가 아니라, 기존 InterviewEngine을 감싸는 레이어**입니다.

### 4-1. 질문 생성 — `PMInterviewEngine.ask_next_question()` (:532)
```python
question_result = await self.inner.ask_next_question(state)
# ↑ self.inner: InterviewEngine — 위 "1. 질문 생성"과 완전히 동일한 프롬프트/파라미터
```
→ 질문 자체는 socratic-interviewer 페르소나로 그대로 생성됨. **PM 인터뷰만의 고유 프롬프트는 그다음 분류 단계에 있음.**

### 4-2. 질문 분류/리프레이밍 — `QuestionClassifier.classify()` (`question_classifier.py:224`)

**System Prompt** (`_CLASSIFICATION_SYSTEM_PROMPT`, :131):
```
You are a question classifier for a Product Requirements Document (PM) interview.

Your job is to determine whether a question generated during a requirements
interview is answerable by a Product Manager (PM), requires deep technical/
development expertise, or is premature and should be deferred to a later stage.

A PRD is a contract between the PM and the developers: success criteria are the
behavior and policy the PM must observe in the delivered feature to accept it as
built. Measuring adoption, conversion, KPI movement, or other post-launch
outcomes is follow-up product work informed by real-world usage — not part of
this contract.

## Categories
**PLANNING** — PM can answer directly: 사업 목표, 수용기준, 우선순위, 유저스토리, 범위결정, 컴플라이언스
**DEVELOPMENT** — 기술 전문성 필요: 아키텍처, 구현 디테일, 인프라, 코드 패턴, 성능/보안 디테일
**DECIDE_LATER** — 시기상조/현재 알 수 없음
```
→ 3장에서 본 "PM 인터뷰 성공 기준을 배포후 KPI가 아니라 인도된 행동에 고정"(최근 커밋 9888ec17)이 **바로 이 시스템 프롬프트**로 구현되어 있음을 확인.

**User Prompt** (:238):
```
Question to classify:
{question}

Interview context so far:
{interview_context}

Codebase context (brownfield):     # brownfield일 때만
{codebase_context[:2000]}
```

**CompletionConfig** (:254):
```python
CompletionConfig(
    model=self.model,      # 기본: get_llm_model_for_role("question_classification")
    role="question_classification",
    temperature=self.temperature,  # = _CLASSIFIER_TEMPERATURE = 0.2
    max_tokens=512,
)
```

### 4-3. PM Seed 추출 — `generate_pm_seed()` (:1065)

**System Prompt** (`_EXTRACTION_SYSTEM_PROMPT`, :95):
```
You are a requirements extraction engine. Given a PM interview transcript,
extract structured product requirements. Preserve uncertainty explicitly: do
not turn uncertain, stakeholder-dependent, or unknown answers into confirmed
requirements. Put tentative claims in assumptions and unresolved choices in
decide_later_items.

Respond ONLY with valid JSON in this exact format:
{
    "product_name": "...", "goal": "...",
    "user_stories": [{"persona": "...", "action": "...", "benefit": "..."}],
    "constraints": [...], "success_criteria": [...],
    "deferred_items": [...], "decide_later_items": [...], "assumptions": [...]
}
```

**CompletionConfig** (:1121):
```python
CompletionConfig(
    model=self.model, role="pm_interview",
    temperature=0.2, max_tokens=4096,
)
```
→ "불확실성을 확정 요구사항으로 둔갑시키지 마라"는 지시가 프롬프트 레벨에서 명시 — 2장 "추측은 계약이 아니다" 원칙의 PM 버전.

---

## 5. Brownfield 저장소 설명 — `generate_desc()` (`bigbang/brownfield.py:282`)

### System Prompt (`_DESC_SYSTEM_PROMPT`, :75)
```
You are a concise technical writer. Given the content of a project's README or
CLAUDE.md, produce exactly ONE short sentence (max 15 words) describing the
project. Reply with only that sentence — no quotes, no bullet points.
```

### User Prompt
```
Project at: {repo_path.name}

{README 또는 CLAUDE.md 내용 (최대 max_chars까지)}
```

### CompletionConfig (:314)
```python
CompletionConfig(
    model=resolved_model,   # 기본: _FRUGAL_MODEL = "anthropic/claude-3-5-haiku-20241022"
    role="brownfield",
    temperature=0.0,         # 완전 결정론
    max_tokens=60,           # 한 문장이면 충분
)
```

**★ Insight ─────────────────────────────────────**
이 지점만 유일하게 **Haiku급(Frugal tier)** 모델을 명시적으로 고정합니다. "여러 저장소를 스캔하며 한 줄 설명 붙이는" 작업은 품질보다 비용이 중요하다고 판단한 것 — 매뉴얼(manual.md)의 PAL Router(Frugal→Standard→Frontier) 사다리를 함수 레벨에서 직접 실천한 사례입니다. `temperature=0.0`도 이 지점과 questionclassifier류처럼 "창의성 불필요, 일관성 필요"인 작업의 공통 패턴입니다.
**─────────────────────────────────────────────────**

---

## 6. 코드베이스 탐색 요약 — `_summarize_with_llm()` (`bigbang/explore.py:411`)

### System Prompt
```
You are a codebase analyst. Produce concise, structured summaries of existing
codebases to help developers understand what already exists before they start
extending it. Be factual and specific.
```

### User Prompt (스캔 결과를 구조화해서 삽입, :429)
```
Summarize this codebase for a developer who needs to extend it.
Be concise (max 300 words). Focus on: tech stack, key types/interfaces,
architectural patterns, protocols, and important conventions.

## Tech Stack
{scan["tech_stack"]}

## Key Dependencies
{scan["dependencies"][:15]}

## Key Type Definitions
{scan["key_types"][:20]}

## Architectural Patterns
{scan["key_patterns"]}

## Config Files
{scan["config_contents"][:3]}   # 파일당 800자 잘림

Output a structured summary with sections: Tech Stack, Key Types, Patterns, Conventions.
```

### CompletionConfig (:463)
```python
CompletionConfig(
    model=self.model,          # ExploreEngine 기본 모델
    role="brownfield_explore",
    temperature=0.2,
    max_tokens=1024,
)
```
실패 시 LLM 없이 스캔 결과를 그대로 이어붙인 fallback 요약으로 대체 (`log.warning` 후 규칙 기반 조합).

---

## 📊 종합 비교표 — Temperature/max_tokens/role 패턴

| 호출 지점 | role | temperature | max_tokens | 모델 티어 | 목적 성격 |
| :--- | :--- | :---: | :---: | :--- | :--- |
| 질문 생성 (`ask_next_question`) | `clarification` | **0.7** | 2048 | Opus급 | 탐색적/창의적 |
| 질문 후보 3인 병렬 (옵션, 기본OFF) | `clarification` | 0.7 | 2048 | Opus급 | 탐색적 |
| 모호도 통합 채점 (`score`) | `ambiguity` | **0.1** | 2048~ (적응형) | Opus급 | 재현가능 채점 |
| 모호도 차원별 채점 (옵션) | `ambiguity` | 0.1 | 2048~ | Opus급 | 재현가능 채점 |
| Seed 요구사항 추출 | `seed_generation` | **0.2** | 4096 | Opus급 | 구조화 추출 |
| 질문 분류/리프레이밍 (PM) | `question_classification` | **0.2** | 512 | Opus급(clarification 상속) | 분류 판단 |
| PM Seed 추출 | `pm_interview` | 0.2 | 4096 | Opus급 | 구조화 추출 |
| Brownfield 한줄 설명 | `brownfield` | **0.0** | 60 | **Haiku (Frugal 고정)** | 결정론적 요약 |
| 코드베이스 탐색 요약 | `brownfield_explore` | 0.2 | 1024 | Opus급(설정에 따름) | 구조화 요약 |

**★ Insight ─────────────────────────────────────**
temperature 값이 세 단계로 뚜렷하게 나뉩니다:
- **0.0~0.2** (채점/추출/분류/요약): "같은 입력 → 같은 출력"이 필요한 작업. 파싱 가능한 구조(JSON, 파이프 구분 라인)를 요구하는 지점과 정확히 겹침.
- **0.7** (질문 생성만 유일): 다양성이 곧 품질인 유일한 작업 — 매 라운드 다른 각도의 질문이 나와야 인터뷰가 진전됨.

이건 5장에서 본 "발산은 뜨겁게(Wonder=0.7), 수렴은 미지근하게(Reflect=0.5), 측정은 차갑게(scoring=0.1)"라는 원칙과 정확히 같은 패턴이 인터뷰 단계에도 반복 적용된 것입니다. **temperature가 "이 호출의 인식론적 역할"(탐색 vs 판정)을 코드 레벨에서 선언하는 장치**로 쓰이고 있습니다.
**─────────────────────────────────────────────────**

---

## 🔑 프롬프트 소스의 두 가지 유형

1. **에이전트 페르소나 파일 로드** (`load_agent_prompt("...")`) — `src/ouroboros/agents/*.md`를 그대로 읽어 system prompt로 사용:
   - `socratic-interviewer` (질문 생성)
   - `seed-architect` (요구사항 추출)
   → 이 페르소나 파일들은 **책(book.md)에서 인용한 규율 텍스트 그 자체**입니다. 즉 "문서에 적힌 원칙"이 아니라 "실제로 매번 LLM에 주입되는 시스템 프롬프트"라는 뜻.

2. **인라인 하드코딩 프롬프트** (모듈 상수) — 나머지 전부(모호도 채점, PM 분류, brownfield 설명, 탐색 요약). 페르소나라기보다 **단일 목적 함수의 스펙**에 가까워서 별도 페르소나 파일 없이 모듈 안에 상수로 박혀 있음.

---

**Created**: 2026-07-23
**연관 문서**: `agent_invocation_map.md` (AC 실행), `interview_agent_calls.md` (호출 횟수/재귀방지)
