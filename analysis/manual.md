# Ouroboros 매뉴얼 — 동작 원리와 활용법

> 이 문서는 README와 `docs/` 하위 문서를 기반으로 Ouroboros의 **동작 원리**와 **실전 활용법**을 한국어로 정리한 매뉴얼입니다.
> 원문 문서: [README.md](./README.md) · [docs/getting-started.md](./docs/getting-started.md) · [docs/architecture.md](./docs/architecture.md) · [docs/cli-reference.md](./docs/cli-reference.md) · [docs/config-reference.md](./docs/config-reference.md)

---

## 목차

1. [Ouroboros란?](#1-ouroboros란)
2. [핵심 철학: 왜 "명세 우선"인가](#2-핵심-철학-왜-명세-우선인가)
3. [동작 원리](#3-동작-원리)
   - [전체 라이프사이클](#31-전체-라이프사이클-the-loop)
   - [Interview — 소크라테스식 질문](#32-interview--소크라테스식-질문)
   - [Ambiguity Score — 코드 작성의 관문](#33-ambiguity-score--코드-작성의-관문)
   - [Seed — 불변 명세](#34-seed--불변-명세)
   - [Execute — Double Diamond 실행](#35-execute--double-diamond-실행)
   - [Evaluate — 3단계 검증 게이트](#36-evaluate--3단계-검증-게이트)
   - [Evolve — 진화 루프와 수렴](#37-evolve--진화-루프와-수렴)
   - [Ralph — 멈추지 않는 루프](#38-ralph--멈추지-않는-루프)
4. [아키텍처 내부](#4-아키텍처-내부)
5. [설치](#5-설치)
6. [시작하기](#6-시작하기)
7. [명령어 레퍼런스](#7-명령어-레퍼런스)
8. [설정](#8-설정)
9. [모니터링과 관측](#9-모니터링과-관측)
10. [활용 시나리오](#10-활용-시나리오)
11. [베스트 프랙티스](#11-베스트-프랙티스)
12. [문제 해결](#12-문제-해결)
13. [더 읽을 문서](#13-더-읽을-문서)

---

## 1. Ouroboros란?

**Ouroboros**는 AI 코딩 에이전트를 위한 **명세 우선(specification-first) 워크플로 엔진**입니다. "프롬프트를 멈추고, 명세하라(Stop prompting. Start specifying.)"는 슬로건 그대로, 모호한 아이디어를 **검증된 동작 코드베이스**로 바꾸는 것을 목표로 합니다.

Claude Code, Codex CLI, OpenCode, Hermes, Gemini, Kiro, GitHub Copilot CLI, Pi, Zcode 등 **다양한 AI 코딩 CLI 위에서 동일한 워크플로**를 실행할 수 있는 로컬 우선(local-first) 런타임 계층이며, 비결정적인 에이전트 작업을 **재현 가능하고(replayable), 관측 가능하고(observable), 정책에 묶인(policy-bound) 실행 계약**으로 바꿉니다.

### Agent OS 스택

Ouroboros는 3개 저장소로 구성된 "Agent OS" 스택의 **커널(OS 계층)** 입니다.

| 계층 | 저장소 | 역할 |
| :--- | :--- | :--- |
| **Shell** (터미널 클라이언트) | [`Q00/ourocode`](https://github.com/Q00/ourocode) | 여러 AI CLI를 한 세션에서 다루는 네이티브 터미널 UI |
| **Apps** (도메인 워크플로) | [`Q00/ouroboros-plugins`](https://github.com/Q00/ouroboros-plugins) | PR 운영, Jira 동기화, 릴리스 등 설치형 도메인 플러그인 |
| **OS** (이 저장소) | [`Q00/ouroboros`](https://github.com/Q00/ouroboros) | Seed · Ledger · Runtime · MCP · 안전 경계를 소유한 코어 |

커널이 계약을 소유합니다: **모든 행위는 Seed에 묶이고, 원장(ledger)에 기록되며, 재현 가능한 이벤트**가 됩니다 — 어떤 LLM이 실행하든 관계없이.

---

## 2. 핵심 철학: 왜 "명세 우선"인가

> 대부분의 AI 코딩 실패는 **출력이 아니라 입력에서** 발생한다. 병목은 AI의 능력이 아니라 인간의 명확성이다.

| 문제 | 일어나는 일 | Ouroboros의 해법 |
| :--- | :--- | :--- |
| 모호한 프롬프트 | AI가 의도를 추측하고, 재작업 발생 | 소크라테스식 인터뷰가 숨은 가정을 노출 |
| 명세 부재 | 빌드 도중 아키텍처가 표류 | 불변 Seed 명세가 코드 이전에 의도를 고정 |
| 수동 QA | "괜찮아 보임"은 검증이 아님 | 3단계 자동 평가 게이트 |

철학적 엔진은 소크라테스적입니다: 모든 좋은 질문은 결국 **존재론적(ontological)** 질문 — "이것을 어떻게 하지?"가 아니라 "이것은 대체 **무엇**인가?" — 으로 이어집니다. "태스크란 무엇인가? 삭제 가능한가, 보관 가능한가?"에 답하는 순간 재작업의 한 부류 전체가 사라집니다. **존재론적 질문이 가장 실용적인 질문입니다.**

---

## 3. 동작 원리

### 3.1 전체 라이프사이클 (The Loop)

우로보로스(자기 꼬리를 삼키는 뱀)는 장식이 아니라 아키텍처 그 자체입니다:

```
    Interview → Seed → Execute → Evaluate
        ↑                           │
        └──── Evolutionary Loop ────┘
```

각 사이클은 반복이 아니라 **진화**입니다. 평가의 출력이 다음 세대 Seed의 입력이 됩니다.

| 단계 | 일어나는 일 |
| :--- | :--- |
| **Interview** | 소크라테스식 질문으로 숨은 가정 노출 |
| **Seed** | 답변을 불변 명세로 결정화(crystallize) |
| **Execute** | Double Diamond: Discover → Define → Design → Deliver |
| **Evaluate** | 3단계 게이트: Mechanical($0) → Semantic → 다중 모델 Consensus |
| **Evolve** | Wonder("아직 무엇을 모르는가?") → Reflect → 다음 세대 |

- **1세대(Gen 1)**: Seed → Execute → Evaluate
- **2세대 이후(Gen 2+)**: Wonder → Reflect → Seed → Execute → Evaluate 를 **수렴할 때까지** 자율 반복

### 3.2 Interview — 소크라테스식 질문

`socratic-interviewer` 에이전트가 **질문만 하고 절대 만들지 않습니다**. "어떤 플랫폼? 예산 제약은? 성공 기준은?" 같은 질문을 통해 숨은 가정을 드러내며, 용어 혼동을 줄이는 **glossary pack**(예: UI/UX 기본 용어)과 참조 대비(reference contrast) 장치가 함께 작동합니다.

인터뷰는 사용자의 "느낌"이 아니라 **수학이 준비되었다고 판단할 때** 끝납니다 (→ 3.3).

### 3.3 Ambiguity Score — 코드 작성의 관문

모호도는 가중 명확도의 역수로 정량화됩니다:

```
Ambiguity = 1 - Σ(clarity_i × weight_i)
```

각 차원은 LLM이 0.0~1.0으로 채점(재현성을 위해 temperature 0.1)한 뒤 가중합됩니다:

| 차원 | Greenfield | Brownfield |
| :--- | :---: | :---: |
| **Goal Clarity** — 목표가 구체적인가? | 40% | 35% |
| **Constraint Clarity** — 제약이 정의됐는가? | 30% | 25% |
| **Success Criteria** — 결과가 측정 가능한가? | 30% | 25% |
| **Context Clarity** — 기존 코드베이스를 이해했는가? | — | 15% |

**임계값: Ambiguity ≤ 0.2** — 이 조건을 만족해야만 Seed 생성이 허용됩니다.

> 왜 0.2인가? 가중 명확도 80% 수준이면 남은 미지수는 코드 레벨 결정으로 해소할 수 있을 만큼 작습니다. 그 위에서는 아직 아키텍처를 추측하고 있는 것입니다.

### 3.4 Seed — 불변 명세

인터뷰가 관문을 통과하면 답변이 **Seed**(불변 frozen Pydantic 모델, `core/seed.py`)로 결정화됩니다:

```yaml
goal: "SQLite 저장소 기반 개인 재무 트래커 구축"
constraints:
  - "데스크톱 전용 애플리케이션"
  - "카테고리 기반 예산 관리"
acceptance_criteria:
  - "수입/지출 추적"
  - "거래 자동 분류"
  - "월간 리포트 생성"
metadata:
  ambiguity_score: 0.15
  seed_id: "seed_abc123"
```

핵심 규칙:
- **`Seed.direction`(목표·제약·수용 기준)은 불변** — 세대가 바뀌어도 의도는 고정됩니다.
- **온톨로지(ontology)는 진화 가능** — 세대를 거치며 O₁→O₂→…→Oₙ으로 계보(`OntologyLineage`)가 추적됩니다.
- 파워 유저는 [Seed Authoring Guide](./docs/guides/seed-authoring.md)를 따라 YAML을 직접 작성할 수도 있습니다.

### 3.5 Execute — Double Diamond 실행

Seed는 **Double Diamond**(Discover → Define → Design → Deliver) 방식으로 분해되어, 설정된 런타임 백엔드(Claude Code, Codex CLI 등)를 통해 실행됩니다.

내부 동작 (`orchestrator/`):
- **병렬 AC 실행** — `ParallelACExecutor`가 수용 기준(AC)들을 **의존성 레벨** 단위로 병렬 실행. 복잡한 AC는 Sub-AC로 분해되어 각각 별도 에이전트 세션에서 처리됩니다.
- **충돌 조정** — `LevelCoordinator`가 레벨 경계에서 병렬 결과 간 파일 충돌을 감지하고, 필요할 때만 해결 세션을 띄웁니다(충돌 없으면 비용 0).
- **에이전트 프로세스 수명주기** — spawn/pause/resume/cancel/replay 동사를 갖는 `AgentProcess`가 장기 실행을 통합 관리하며, 일시정지 체크포인트는 내구적으로 저장됩니다.
- **PAL Router** — Frugal(1×) → Standard(10×) → Frontier(30×) 3단계 비용 최적화. 실패 시 자동 승급, 성공 시 자동 강등.

### 3.6 Evaluate — 3단계 검증 게이트

비용 계층화된 파이프라인(`evaluation/pipeline.py`)이 실행 결과를 Seed의 수용 기준과 대조합니다:

| 단계 | 구성요소 | 비용 | 내용 |
| :--- | :--- | :--- | :--- |
| **Stage 1** | Mechanical Verifier | $0 | 기계적 검사 (테스트, 린트, 빌드 등) |
| **Stage 2** | Semantic Evaluator | 표준 LLM | 의미론적 평가 |
| **Stage 3** | Consensus Evaluator | 프런티어 다중 모델 | 트리거 조건 충족 시에만: advocate/devil/judge 다중 모델 합의 |

추가 안전장치: **리워드 해킹 탐지**(reward-hacking detector, veto 임계값 존재), 적대적 검토(adversarial), 리뷰어 독립성 보장, ["안전하지만 틀린 출력"](./docs/guides/safe-but-wrong-output.md) 가드.

### 3.7 Evolve — 진화 루프와 수렴

> *"여기가 우로보로스가 자기 꼬리를 먹는 지점이다: 평가의 출력이 다음 세대 Seed 명세의 입력이 된다."* — `reflect.py`

2세대부터 인터뷰 대신 두 엔진이 작동합니다:

- **WonderEngine** — "아직 무엇을 모르는가?" 온톨로지·평가 결과·출력을 검사해 간극과 긴장을 표면화
- **ReflectEngine** — 실행 결과 + 온톨로지 + Wonder 출력을 소비하여 정제된 AC와 온톨로지 변이(mutation)를 생성

**수렴 판정** — 연속 세대의 온톨로지 스키마 유사도로 측정:

```
Similarity = 0.5 × name_overlap + 0.3 × type_match + 0.2 × exact_match
```

**임계값: Similarity ≥ 0.95** 이면 루프가 수렴하고 멈춥니다.

병리적 패턴도 감지합니다:

| 신호 | 조건 | 의미 |
| :--- | :--- | :--- |
| **Stagnation** | 3세대 연속 유사도 ≥ 0.95 | 온톨로지 안정화 |
| **Oscillation** | Gen N ≈ Gen N-2 (주기 2 순환) | 두 설계 사이를 왕복 중 |
| **Repetitive feedback** | 3세대에 걸쳐 질문 70% 이상 중복 | Wonder가 같은 질문 반복 |
| **Hard cap** | 30세대 도달 | 안전 밸브 |

두 개의 수학적 관문, 하나의 철학: **명확해지기 전에는 만들지 말고(Ambiguity ≤ 0.2), 안정되기 전에는 진화를 멈추지 말라(Similarity ≥ 0.95).**

### 3.8 Ralph — 멈추지 않는 루프

`ooo ralph`는 진화 루프를 **세션 경계를 넘어** 수렴까지 지속 실행합니다. 각 스텝은 **무상태(stateless)** 입니다 — EventStore가 전체 계보를 복원하므로 머신이 재시작되어도 뱀은 멈춘 자리에서 다시 시작합니다:

```
Ralph Cycle 1: evolve_step(lineage, seed) → Gen 1 → action=CONTINUE
Ralph Cycle 2: evolve_step(lineage)       → Gen 2 → action=CONTINUE
Ralph Cycle 3: evolve_step(lineage)       → Gen 3 → action=CONVERGED
                                                └── Ralph 종료. 온톨로지 안정화.
```

종료 상태: 성공 = `converged` / 실패 = `failed`, `interrupted`, `exhausted`, `stagnated`. 등급 회귀·진동 윈도우도 감지합니다.

---

## 4. 아키텍처 내부

### 이벤트 소싱 — 모든 것의 척추

Ouroboros의 "재현 가능(replayable)" 주장은 이벤트 소싱으로 실현됩니다:

- **단일 진실 원천** — 모든 상태 변화는 불변 `BaseEvent`(`lineage.created`, `execution.ac.completed` 같은 `점.표기.과거형` 명명)로 SQLite의 append-only 이벤트 테이블에 기록됩니다 (`persistence/event_store.py`).
- **읽기 모델은 항상 재구성** — 계보(lineage), 컨덕터 결정, 워크플로 수명주기 등 모든 상태는 이벤트 스트림을 **fold/replay하여 투영(projection)** 됩니다. 직접 저장되는 상태는 없습니다.
- **단계 경계 원자성** — `UnitOfWork`가 이벤트와 체크포인트를 모아 **단계(phase) 경계에서 원자적으로 커밋**합니다.
- **복구** — 무결성 검증된 체크포인트, 최대 3레벨 롤백, 백그라운드 자동 체크포인트.
- **재개 안전한 워치독** — 벽시계 예산 타이머조차 영속화된 `created_at`에서 유도되므로 별도 직렬화 없이 재개-안전합니다.

이것이 중단된 수 시간짜리 실행이 처음부터가 아니라 **세대 중간에서 재개**될 수 있는 이유입니다.

### 주요 컴포넌트 지도

```
src/ouroboros/
├── bigbang/        인터뷰, 모호도 채점, 브라운필드 탐색기
├── interview_adapters/  글로서리 팩, 참조 대비, 용어 명확화
├── core/           Seed, 온톨로지, 계보, 컨덕터 계약, Result 타입
├── orchestrator/   런타임 추상화 계층 + 병렬 AC 실행 + 제어 평면
├── evaluation/     Mechanical → Semantic → Consensus 파이프라인
├── evolution/      Wonder/Reflect 사이클, 수렴 감지, 되감기(rewind)
├── resilience/     4패턴 정체 감지, 5개 수평사고 페르소나
├── observability/  3요소 드리프트 측정, 자동 회고
├── persistence/    이벤트 소싱(SQLAlchemy + aiosqlite), 체크포인트
├── providers/      LLM 어댑터 (Claude/Codex/Gemini/Copilot/… + LiteLLM 100+ 모델)
├── backends/       백엔드 레지스트리 (LLM 어댑터 × 에이전트 런타임 페어링)
├── mcp/            MCP 서버/클라이언트 (~30개 도구)
├── plugin/         플러그인 시스템 (스킬/에이전트 자동 발견)
├── tui/ + dashboard_web/ + config_tui/   터미널 UI · 웹 대시보드 · 설정 GUI
└── cli/            Typer 기반 CLI (ooo / ouroboros)
```

### 핵심 내부 수치

- **드리프트 측정** — Goal(50%) + Constraint(30%) + Ontology(20%) 가중 측정, 임계값 ≤ 0.3
- **진화** — 최대 30세대, 온톨로지 유사도 ≥ 0.95에서 수렴
- **정체 감지** — spinning(공회전), oscillation(진동), no-drift(무변화), diminishing returns(수확 체감) 4패턴
- **런타임 백엔드** — `orchestrator.runtime_backend` 설정으로 교체 가능. 같은 워크플로 명세, 다른 실행 엔진

### 아홉 개의 마음 (The Nine Minds)

필요할 때만 로드되는 9개의 사고 모드 에이전트:

| 에이전트 | 역할 | 핵심 질문 |
| :--- | :--- | :--- |
| **Socratic Interviewer** | 질문만. 절대 만들지 않음 | "무엇을 가정하고 있는가?" |
| **Ontologist** | 증상이 아닌 본질 탐색 | "이것은 정말 무엇인가?" |
| **Seed Architect** | 대화를 명세로 결정화 | "완전하고 모호하지 않은가?" |
| **Evaluator** | 3단계 검증 | "옳은 것을 만들었는가?" |
| **Contrarian** | 모든 가정에 도전 | "반대가 참이라면?" |
| **Hacker** | 비관습적 경로 탐색 | "어떤 제약이 실제로 진짜인가?" |
| **Simplifier** | 복잡성 제거 | "동작할 수 있는 가장 단순한 것은?" |
| **Researcher** | 코딩을 멈추고 조사 | "실제로 어떤 증거가 있는가?" |
| **Architect** | 구조적 원인 식별 | "처음부터 다시 만든다면 이렇게 만들까?" |

---

## 5. 설치

### 방법 1: Claude Code 플러그인 (권장)

Python 설치 불필요 — Claude Code가 런타임을 처리합니다.

```bash
# 터미널에서
claude plugin marketplace add Q00/ouroboros
claude plugin install ouroboros@ouroboros
```

이후 Claude Code 세션 안에서:
```
ooo setup       # MCP 서버 전역 등록(1회) + 프로젝트 설정
ooo help        # 설치 확인
```

### 방법 2: 원라이너 (런타임 자동 감지)

```bash
curl -fsSL https://raw.githubusercontent.com/Q00/ouroboros/main/scripts/install.sh | bash
```

설치 가능한 런타임을 자동 감지하고, 호스트가 지원하면 MCP 서버를 등록합니다.

### 방법 3: pip / uv / pipx

**Python ≥ 3.12 필요.** (LiteLLM 포함 프로필은 3.12–3.13)

```bash
pip install ouroboros-ai              # 기본 (코어 엔진)
pip install ouroboros-ai[claude]      # + Claude Code 런타임 의존성
pip install ouroboros-ai[litellm]     # + LiteLLM 멀티 프로바이더 (Py 3.12–3.13)
pip install ouroboros-ai[mcp]         # + MCP 서버/클라이언트
pip install ouroboros-ai[tui]         # + Textual 터미널 UI
pip install ouroboros-ai[all]         # 전부 (Py 3.12–3.13)

ouroboros setup                       # 런타임 설정
```

> **어떤 extra가 필요한가?** Claude Code만 런타임으로 쓴다면 `ouroboros-ai[mcp,claude]`. 멀티 모델이 필요하면 `[litellm]` 또는 `[all]`.

### 비-Claude 런타임 설정

```bash
ouroboros setup --runtime codex      # Codex CLI (~/.codex/에 규칙·스킬 설치)
ouroboros setup --runtime opencode   # OpenCode
ouroboros setup --runtime kiro       # Kiro CLI (~/.kiro/settings/mcp.json 등록)
ouroboros setup --runtime copilot    # Copilot CLI (라이브 모델 발견 + 기본 모델 선택)
ouroboros setup --runtime gemini     # Gemini CLI
ouroboros setup --runtime zcode      # Zcode
```

### Windows

WSL 2가 지원 경로입니다. WSL 배포판 안에서 Linux 설치 명령을 실행하세요. 문제 시 [Windows WSL 트러블슈팅](./docs/guides/windows-wsl-troubleshooting.md) 참조.

### 제거

```bash
ouroboros uninstall    # 설정, MCP 등록, 데이터 전부 제거
```

---

## 6. 시작하기

### 가장 빠른 길: `ooo auto`

목표 하나로 인터뷰 → A등급 Seed → 실행 인계까지 자동 진행:

```
ooo setup
ooo auto "태스크 관리 CLI를 만들어줘"
```

`ooo auto`는:
- **경계 있는(bounded)** 소크라테스식 인터뷰 라운드를 실행하고
- **A등급 Seed**를 생성하며 (B/C등급 Seed는 가능하면 자동 수리)
- A등급 게이트 통과 후에만 실행을 시작하고
- `auto_session_id`를 반환하여 중단·차단된 실행을 재개할 수 있게 합니다.

유용한 변형:

```bash
ooo auto "로컬 우선 습관 트래커 CLI" --skip-run    # 실행 없이 Seed까지만
ooo auto --resume auto_abc123                      # 중단된 세션 재개
ouroboros auto "..." --show-ledger                 # (셸 CLI) 수렴 중 포착된 가정·비목표 출력
```

> auto 모드는 **행(hang) 저항적**으로 설계되어 있습니다: 인터뷰·수리 루프는 경계가 있고, 느린 도구 호출은 세션을 `blocked`/`failed`로 전이시키며, 실행 인계는 완료를 무한정 기다리는 대신 job/session ID를 반환합니다.

### 수동 경로: 모든 질문에 직접 답하고 싶을 때

```
ooo interview "태스크 관리 CLI를 만들고 싶어"   # 인터뷰 → 모호도 ≤ 0.2 → Seed 생성
ooo run                                        # Double Diamond 실행
```

### 터미널 독립 실행 (standalone CLI)

Claude Code 밖 터미널에서:

```bash
ouroboros init start "여기에 아이디어"          # 인터뷰 (--llm-backend claude_code|codex|opencode|litellm)
ouroboros run ~/.ouroboros/seeds/seed_abc123.yaml   # Seed 파일 경로가 위치 인자로 필요
ouroboros monitor                               # 별도 터미널에서 TUI 모니터링
```

### 첫 워크플로 4단계

1. **Interview** — `ooo interview "개인 재무 트래커를 만들고 싶어"` → 질문에 답하며 모호도를 0.2 이하로
2. **Execute** — `ooo run` → Double Diamond 분해 후 런타임 백엔드로 실행
3. **Monitor** — 두 번째 터미널에서 `ouroboros monitor` → AC 트리, 비용, 드리프트 실시간 관찰
4. **Review** — 완료 시 QA 평결이 포함된 세션 요약 출력. 후속: `ooo evaluate`, `ooo status`, `ooo evolve`

---

## 7. 명령어 레퍼런스

에이전트 세션 안에서는 `ooo <cmd>` 스킬, 터미널에서는 `ouroboros` CLI를 사용합니다.

| 스킬 (`ooo`) | CLI 대응 | 하는 일 |
| :--- | :--- | :--- |
| `ooo setup` | `ouroboros setup` | 런타임 등록 + 프로젝트 설정 (1회) |
| `ooo interview` | `ouroboros init start` | 소크라테스식 질문 — 숨은 가정 노출 |
| `ooo auto` | `ouroboros auto` | 목표 → A등급 Seed → 실행 인계 (경계 있는 루프) |
| `ooo seed` | *(인터뷰가 생성)* | 불변 명세로 결정화 |
| `ooo run` | `ouroboros run seed.yaml` | Double Diamond 분해 실행 |
| `ooo evaluate` | *(MCP 경유)* | 3단계 검증 게이트 |
| `ooo evolve` | *(MCP 경유)* | 온톨로지 수렴까지 진화 루프 |
| `ooo unstuck` | *(MCP 경유)* | 막혔을 때 5개 수평사고 페르소나 |
| `ooo status` | `ouroboros status executions` | 세션 추적 + (MCP 전용) 드리프트 감지 |
| `ooo resume-session` | `ouroboros resume` | 진행 중 세션 목록 + 재접속 명령 |
| `ooo cancel` | `ouroboros cancel execution [<id>\|--all]` | 멈춘/고아 실행 취소 |
| `ooo ralph` | *(MCP 경유)* | 검증될 때까지 지속 루프 |
| `ooo tutorial` | *(대화형)* | 대화형 실습 학습 |
| `ooo pm` | *(MCP 경유)* | PM 중심 인터뷰 + PRD 생성 |
| `ooo qa` | *(스킬)* | 임의 산출물에 대한 범용 QA 평결 |
| `ooo brownfield` | *(스킬)* | 브라운필드 저장소/워크트리 스캔·관리 |
| `ooo publish` | *(스킬, `gh` CLI 사용)* | Seed를 GitHub Epic/Task 이슈로 발행 |
| `ooo update` | `ouroboros update` | 업데이트 확인 + 최신 버전 업그레이드 |
| `ooo help` | `ouroboros --help` | 전체 레퍼런스 |

CLI 전용 유틸리티:

```bash
ouroboros run seed.yaml --dry-run      # 실행 전 YAML/스키마 검증
ouroboros run seed.yaml --resume <id>  # 세션 재개
ouroboros run seed.yaml --no-qa        # 실행 후 QA 생략
ouroboros run seed.yaml --debug        # 상세 출력
ouroboros cleanup                      # 병합 완료 워크트리, 오래된 락, 완료 세션 정리
ouroboros cleanup --dry-run            # 정리 대상 미리보기
ouroboros config show                  # 설정 요약 확인
ouroboros config backend codex         # 백엔드 전환
ouroboros mcp info / ouroboros mcp serve   # MCP 진단/기동
```

> `/resume`은 Claude Code 내장 세션 선택기용으로 예약되어 있습니다. Ouroboros 진행 세션에는 `ooo resume-session`을 쓰세요.

전체 상세: [CLI Reference](./docs/cli-reference.md)

---

## 8. 설정

### 설정 파일: `~/.ouroboros/config.yaml`

`ouroboros setup`이 합리적 기본값으로 생성합니다. 핵심 키:

```yaml
orchestrator:
  runtime_backend: claude   # claude | codex | opencode | hermes | gemini | copilot | goose | kiro | pi ...

llm:
  backend: claude_code      # claude_code | codex | litellm | copilot | opencode | gemini | goose | kiro | pi

logging:
  level: info

runtime_controls:
  mcp_tool_timeout_seconds: 0                    # 어댑터 벽시계 상한 없음
  generation_idle_timeout_seconds: 7200          # 2시간 무활동 시
  generation_no_progress_timeout_seconds: 14400  # 4시간 실질 진전 없음 시
```

역할별 모델 오버라이드(예: Codex 백엔드에서 GPT-5.4 기준선):

```yaml
llm:
  backend: codex
  qa_model: gpt-5.4
clarification:
  default_model: gpt-5.4
evaluation:
  semantic_model: gpt-5.4
consensus:
  advocate_model: gpt-5.4
  devil_model: gpt-5.4
  judge_model: gpt-5.4
```

### 환경 변수

```bash
export ANTHROPIC_API_KEY="..."          # Claude 기반 플로 (플러그인 사용자는 불필요)
export OPENAI_API_KEY="..."             # Codex 기반 플로
export OUROBOROS_AGENT_RUNTIME=codex    # 런타임 백엔드 오버라이드 (최우선)
```

**해석 우선순위**: `OUROBOROS_AGENT_RUNTIME` 환경변수 > `config.yaml` > `ouroboros setup` 시 자동 감지

기타 설정 영역(전체는 [Configuration Reference](./docs/config-reference.md)):
- `llm_profiles` / `llm_role_profiles` — 역할·프로필별 모델 매핑
- `economics` — 비용 관련 설정
- `resilience` / `drift` / `evaluation` / `consensus` — 각 파이프라인 튜닝
- `~/.ouroboros/backend_limits.yaml` — 백엔드별 동시성·레이트 리밋
- `~/.ouroboros/credentials.yaml` — 자격 증명

---

## 9. 모니터링과 관측

| 도구 | 실행 | 보여주는 것 |
| :--- | :--- | :--- |
| **TUI 대시보드** | `ouroboros monitor` (또는 `ouroboros tui monitor`) | Double Diamond 단계 진행, AC 트리 실시간 상태, 비용·드리프트·에이전트 활동 |
| **웹 대시보드** | MCP `ac_dashboard` 도구 / `ooo config` 경유 | SSE 실시간 스트림, 칸반 보드 |
| **네이티브 Rust TUI** | `ouroboros-tui` (crates/) | SQLite 이벤트 스토어 직접 읽기 — 실행·계보·로그 뷰 |
| **상태 조회** | `ooo status` / `ouroboros status executions` | 세션 상태 + 드리프트 감지 |
| **이벤트 조회** | MCP `query_events` / `query_projection` | 원시 이벤트 스트림·투영 질의 |

**드리프트 모니터링**: Goal(50%) + Constraint(30%) + Ontology(20%) 가중 측정으로 실행이 Seed의 의도에서 얼마나 벗어났는지 정량화합니다 (임계값 ≤ 0.3). Claude Code 플러그인에서는 Write/Edit 시 훅이 자동으로 드리프트 자문을 제공합니다.

---

## 10. 활용 시나리오

### 신규 프로젝트 (Greenfield)

```
ooo interview "블로그용 REST API 만들기"
ooo run
```

### 버그 수정

```
ooo interview "이메일 검증에서 사용자 등록이 실패함"
ooo run
```

인터뷰가 "이것이 근본 원인인가, 증상인가?"를 파고들어 표면 수정 대신 근본 해결로 유도합니다.

### 기능 개선

```
ooo interview "채팅 앱에 실시간 알림 추가"
ooo run
```

### 기존 코드베이스 (Brownfield)

```
ooo brownfield        # 저장소/워크트리 스캔·기본값 관리
```

여러 언어 생태계의 설정 파일을 자동 감지하고, 원격 main에 고정된 지속 스냅샷 워크트리로 탐색합니다. Brownfield 모드에서는 모호도 채점에 **Context Clarity(15%)** 차원이 추가됩니다.

### 완전 자율 실행

```
ooo auto "목표"        # 인터뷰→Seed→실행 인계까지 자동
ooo ralph              # 수렴까지 세대를 넘는 지속 진화 루프
```

### PM/기획 워크플로

```
ooo pm                 # PM 중심 인터뷰 + PRD 생성
ooo publish            # Seed를 GitHub Epic/Task 이슈로 발행 (팀 워크플로)
```

### 막혔을 때

```
ooo unstuck            # 5개 수평사고 페르소나 (hacker, simplifier, contrarian ...)
```

---

## 11. 베스트 프랙티스

### 더 나은 인터뷰를 위해
1. **구체적으로** — "실시간 메시징이 있는 Twitter 클론"이 "소셜 앱"보다 낫습니다
2. **제약을 일찍** — 예산, 일정, 기술 제약을 먼저 말하세요
3. **성공을 정의** — 명확한 수용 기준이 더 좋은 Seed를 만듭니다

### 효과적인 Seed를 위해
1. **비기능 요구사항 포함** — 성능, 보안, 확장성
2. **경계 정의** — 범위 안과 밖을 명시
3. **통합 명시** — API, 데이터베이스, 서드파티 서비스

### 성공적인 실행을 위해
1. **먼저 검증** — `ouroboros run seed.yaml --dry-run`으로 YAML/스키마 사전 확인
2. **TUI로 모니터링** — 긴 워크플로 중 별도 터미널에서 `ouroboros monitor`
3. **QA 유지** — `--no-qa`를 넘기지 않는 한 실행 후 QA가 자동 실행됩니다

---

## 12. 문제 해결

| 증상 | 해결 |
| :--- | :--- |
| 스킬이 로드되지 않음 | `claude plugin install ouroboros@ouroboros --force` |
| CLI를 찾을 수 없음 | `pip install ouroboros-ai` (Python ≥ 3.12 확인) |
| API 오류 | `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` 확인: `env \| grep -E 'ANTHROPIC\|OPENAI'` |
| MCP 서버 문제 | `ouroboros mcp info` → `ouroboros mcp serve` |
| TUI 화면이 비어 있음 | `export TERM=xterm-256color` |
| 비용이 높음 | Seed 범위 축소 또는 낮은 모델 티어 사용 |
| 실행이 멈춤 | `ooo unstuck`, 또는 `ouroboros run seed.yaml --resume <id>` / `ouroboros cancel execution <id>` |
| auto 세션 중단 | 출력된 명령으로 재개: `ooo auto --resume <auto_session_id>` |

---

## 13. 더 읽을 문서

### 개념·설계
- [Architecture](./docs/architecture.md) — 전체 시스템 설계 (Agent OS 커널 용어 포함)
- [Events](./docs/events.md) — 이벤트 스키마
- [Evolution Loop 가이드](./docs/guides/evolution-loop.md) · [Evaluation Pipeline 가이드](./docs/guides/evaluation-pipeline.md)
- [Agent Process Lifecycle](./docs/guides/agent-process-lifecycle.md) · [Execution vs Evaluation](./docs/guides/execution-vs-evaluation.md)

### 사용법
- [Getting Started](./docs/getting-started.md) — 온보딩 단일 진실 원천
- [CLI Reference](./docs/cli-reference.md) · [Configuration Reference](./docs/config-reference.md)
- [Seed Authoring Guide](./docs/guides/seed-authoring.md) · [TUI Usage Guide](./docs/guides/tui-usage.md)
- [Platform Support](./docs/platform-support.md) · [Runtime Capability Matrix](./docs/runtime-capability-matrix.md)

### 런타임별 가이드 (`docs/runtime-guides/`)
Claude Code · Codex · OpenCode · Kiro · Copilot · Gemini · Hermes · Goose · Grok · Pi · Zcode · GJC · Antigravity

---

> *"시작이 곧 끝이고, 끝이 곧 시작이다."*
> **뱀은 반복하지 않는다 — 진화한다.**
