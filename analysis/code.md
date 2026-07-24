# Ouroboros의 코드 분석 — 왜 프롬프트가 아니라 코드인가

> "프롬프트로 된 스킬이나 서브에이전트 말고, 코드도 많더라고요. 왜 이렇게 많은 코드가 있고, 어떻게 동작하나요?"

이 질문은 이 프로젝트의 정체를 정확히 짚습니다. Ouroboros에서 **프롬프트(스킬 21개, 에이전트 페르소나 ~15개)는 얇은 인터페이스 껍질**이고, **본체는 약 22만 줄의 Python 코드**입니다. 이 문서는 (1) 그 코드가 어디에 얼마나 있는지, (2) 왜 프롬프트로는 안 되고 코드여야만 하는지, (3) 실제로 어떤 흐름으로 동작하는지를 분석합니다.

---

## 1. 숫자로 보는 코드 분포

`src/ouroboros/` 하위 패키지별 Python 라인 수 (실측, 2026-07 기준):

| 패키지 | LOC | 역할 한 줄 요약 |
| :--- | ---: | :--- |
| `orchestrator/` | 60,606 | 실행 엔진: AC 병렬 실행, 런타임 어댑터 13종, 제어 평면, 워치독 |
| `mcp/` | 36,605 | MCP 서버: ~30개 도구, 백그라운드 잡 관리, 프로토콜/보안 |
| `auto/` | 21,762 | `ooo auto` 파이프라인: 경계 있는 인터뷰→Seed→실행 인계 상태 기계 |
| `cli/` | 19,314 | Typer 기반 `ooo`/`ouroboros` CLI 명령 ~20종 |
| `providers/` | 11,517 | LLM 어댑터 14종 (Claude/Codex/Gemini/Copilot/… 단일 호출 계층) |
| `plugin/` | 10,119 | 스킬/에이전트 자동 발견, 플러그인 시스템 |
| `core/` | 9,605 | Seed·온톨로지·계보·컨덕터 계약·Result 타입 (도메인 모델) |
| `tui/` | 8,307 | Textual 터미널 대시보드 |
| `bigbang/` | 7,845 | 인터뷰·모호도 채점·브라운필드 탐색 |
| `evaluation/` | 7,382 | 3단계 평가 파이프라인, 리워드 해킹 방어, 배심원 독립성 |
| `evolution/` | 5,276 | Wonder/Reflect, 수렴 판정, 되감기 |
| `harness/` | 4,966 | 실행 하네스 |
| `config/` + `config_tui/` | 4,710 | 설정 스키마·검증·설정 GUI |
| `persistence/` + `events/` | 6,158 | 이벤트 스토어(SQLite), 체크포인트, 이벤트 타입 |
| `observability/` | 1,812 | 드리프트 측정, 자동 회고 |
| `resilience/` | 1,607 | 정체 감지 4패턴, 수평 사고 5페르소나 |
| 기타 (backends, router, dashboard_web, codex/copilot/kiro 정책 등) | ~7,000 | 백엔드 레지스트리, 스킬 디스패치, 웹 대시보드, CLI별 정책 |

첫 인상적인 사실: **상위 4개 패키지(orchestrator, mcp, auto, cli)가 전체의 약 60%**를 차지합니다. 이들은 모두 "AI에게 무엇을 시킬까"(프롬프트의 영역)가 아니라 **"AI를 어떻게 안전하게 구동하고, 관리하고, 검증하고, 복구할까"**(인프라의 영역)를 다룹니다. 반면 철학적으로 가장 유명한 부분 — 정체 감지(1,607줄), 드리프트 측정(1,812줄) — 은 놀랍도록 작습니다. **좋은 원리는 코드가 적게 들고, 그 원리를 현실 세계에서 굴리는 인프라가 코드를 많이 먹습니다.**

---

## 2. 왜 프롬프트로는 안 되는가 — 코드가 존재하는 6가지 이유

### 이유 1: 프롬프트는 지시이고, 코드는 강제다

LLM에게 "테스트를 조작하지 마"라고 프롬프트에 쓰는 것과, 모든 승인 경로가 지나는 단일 관문에서 `if reward_hacking_risk >= 0.7: final_approved = False`를 실행하는 것은 완전히 다른 보증 수준입니다. Ouroboros의 안전 규칙은 전부 후자입니다:

- 검증 명령 허용목록: LLM이 제안한 명령은 셸 메타문자 금지, 실행 파일 allowlist, `make deploy` 같은 파괴적 타겟 거부를 **코드로** 통과해야 실행됨 (`evaluation/detector.py`)
- 만족화 백스톱: Reflect LLM이 "통과한 AC는 건드리지 마"라는 규칙을 어겨도, 결정론적 코드가 보호 AC를 강제 keep (`evolution/reflect.py`)
- 리워드 해킹 거부권: 어떤 평가 분기도 우회할 수 없는 단일 지점 (`evaluation/pipeline.py`)

이것이 이 코드베이스의 서명 패턴입니다: **"LLM은 제안하고, 결정론적 코드가 처분한다."** 프롬프트는 확률적으로 준수되지만, 코드는 결정론적으로 집행됩니다.

### 이유 2: 상태는 프롬프트에 살 수 없다

LLM 세션은 휘발성입니다. 컨텍스트는 압축되고, 세션은 끊기고, 머신은 재시작됩니다. "3세대에 걸친 온톨로지 진화를 이어가라"는 요구는 프롬프트가 아니라 **영속 계층**을 요구합니다:

- `persistence/event_store.py`: SQLite append-only 이벤트 스토어 — 모든 상태 변화가 불변 이벤트
- `persistence/uow.py`: 단계 경계에서 이벤트+체크포인트 원자적 커밋
- `evolution/loop.py`의 `evolve_step()`: 이벤트 재생으로 마지막 세대의 Seed를 재구성해 **세대 중간에서 재개**

"머신이 재시작돼도 뱀은 멈춘 자리에서 다시 시작한다"는 README의 약속은 6,000줄이 넘는 영속성 코드가 지불하는 수표입니다.

### 이유 3: 측정은 결정론적이어야 한다

모호도 0.2, 드리프트 0.3, 유사도 0.95 같은 **관문 수치**가 실행마다 흔들리면 관문이 아닙니다. 그래서:

- 모호도 채점: LLM을 쓰되 temperature 0.1로 고정하고, 가중 합산·클램핑·차원별 하한선 검사는 **순수 Python** (`bigbang/ambiguity.py`)
- 드리프트: LLM 없이 **Jaccard 유사도 + 가중합** (`observability/drift.py`)
- 정체 감지: LLM 없이 **SHA-256 해시 비교와 뺄셈** (`resilience/stagnation.py`)
- 수렴 판정: 스키마 필드의 이름/타입/완전 일치 가중 비교 (`evolution/convergence.py`)

상시 가동되는 감시 지표일수록 싸고 결정론적이어야 하고, 그것은 곧 코드를 의미합니다.

### 이유 4: 프로세스 오케스트레이션은 시스템 프로그래밍이다

Ouroboros는 Claude Code, Codex CLI, Gemini CLI 등 **13종의 외부 CLI를 자식 프로세스로 구동**합니다. 이것은 순수한 시스템 프로그래밍 문제입니다:

- 프로세스 스폰/일시정지/재개/취소/재생 수명주기 (`orchestrator/agent_process.py`)
- CLI별 스트림 파싱과 이벤트 정규화 (예: `providers/gemini_event_normalizer.py`, `codex_cli_stream.py`)
- 협조적 취소: 단일 `task.cancel()` 후 상태 플러시 대기, 결정 이벤트 원자 기록 (`evolution/watchdog.py`)
- 하트비트·타임아웃 3종(총 경과/유휴/무진전) 감시
- AC 의존성 분석 → 레벨 단위 병렬 실행 → 레벨 경계 파일 충돌 감지 (`orchestrator/parallel_executor.py`, `coordinator.py`)

orchestrator가 6만 줄로 가장 큰 이유가 이것입니다. "같은 워크플로를 9개 AI CLI에서 돌린다"는 한 줄의 마케팅 문구 뒤에는 CLI마다 다른 인자 체계·출력 형식·권한 모델·모델 표기(`claude-opus-4-6` ↔ `claude-opus-4.6` 자동 변환까지)를 흡수하는 어댑터 층이 있습니다.

### 이유 5: 프로토콜은 구현해야 한다

스킬(프롬프트)이 실제 엔진을 호출하려면 **다리**가 필요합니다. 그 다리가 MCP 서버(36,605줄)입니다. 여기에는 도구 스키마 정의, 요청 검증, 백그라운드 잡 관리(detached worker, 잡 폴링·복구), 보안 계층, 클라이언트/브리지가 포함됩니다. 특히 `ooo ralph` 같은 장시간 루프는 "MCP 서버가 루프를 소유"하도록 설계되어(issue #528), 클라이언트(프롬프트)가 다세대 루프를 의사코드로 흉내 내지 않게 했습니다 — **루프의 신뢰성이 중요하면 루프를 프롬프트에서 코드로 옮긴다**는 결정의 결과물입니다.

### 이유 6: 보안과 방어는 프롬프트를 신뢰하지 않는 데서 시작한다

- 이벤트 저장 전 소독(`sanitize_event_data_for_persistence`), 시크릿 마스킹 패턴, 로그 전 명령 마스킹 — 코드베이스 전반 246곳
- DoS 상한: 인터뷰 입력 50,000자, LLM 응답 100,000자 절단
- 설정 파일 원자적 쓰기(`.tmp` → `os.replace`) — 마지막 정상 설정 보존
- fail-closed 직렬화: 형식이 이상한 저장 데이터는 관대하게 받지 않고 통째로 거부
- 신뢰할 수 없는 저장소 설정(`.ouroboros/mechanical.toml`)도 allowlist 안에서만 동작 — "저장소를 클론했더니 CI에서 임의 명령이 실행되더라"를 원천 차단

---

## 3. 실제 동작 흐름 — `ooo run`이 실행될 때 일어나는 일

프롬프트 계층과 코드 계층이 실제로 어떻게 맞물리는지, 한 번의 실행을 따라가 봅니다.

### 계층 다이어그램

```
[Claude Code 세션]
  │  사용자: "ooo run seed.yaml"
  ▼
① 스킬 (프롬프트 계층) ─ skills/run/SKILL.md
  │  frontmatter가 다리를 선언:
  │    mcp_tool: ouroboros_execute_seed
  │    mcp_args: { seed_path: "$1", cwd: "$CWD" }
  │  → Claude가 MCP 도구를 호출하도록 지시
  ▼
② MCP 서버 (파이썬 프로세스) ─ src/ouroboros/mcp/
  │  uvx --from ouroboros-ai[mcp,claude] ouroboros mcp serve 로 기동
  │  execution_handlers.py가 요청 검증 → OrchestratorRunner 생성
  │  (장시간 작업이면 detached job으로 분리, job_id 즉시 반환)
  ▼
③ 코어 엔진 ─ core/ + evolution/ + orchestrator/
  │  Seed 파싱·검증 (frozen Pydantic) → AC 의존성 분석
  │  ParallelACExecutor가 AC를 레벨 단위 병렬 실행 계획으로
  ▼
④ 런타임 어댑터 ─ orchestrator/*_runtime.py + providers/
  │  설정된 백엔드(claude/codex/gemini/…)의 CLI를 자식 프로세스로 스폰
  │  또는 Claude Agent SDK 세션 생성 — 여기서 실제 LLM이 코드를 작성
  │  스트림 파싱 → [AC_START]/[AC_COMPLETE] 마커 추적
  ▼
⑤ 이벤트 스토어 ─ persistence/ (SQLite)
  │  모든 단계가 불변 이벤트로 기록 (execution.ac.completed, …)
  │  TUI/웹 대시보드는 이 DB를 읽어 실시간 표시 (별도 프로세스)
  ▼
⑥ 평가 파이프라인 ─ evaluation/
  │  Stage 1: 허용목록 검증된 lint/test 명령 실행 ($0)
  │  Stage 2: 실제 산출물 코드를 수집해 LLM 의미 평가
  │  Stage 3: 트리거 시에만 다중 모델 합의
  │  단일 관문에서 리워드 해킹 거부권 적용
  ▼
결과가 MCP 응답으로 스킬 계층에 반환 → Claude가 사용자에게 요약
```

### 흐름에서 읽어야 할 것

1. **프롬프트는 ①에만 있습니다.** `skills/run/SKILL.md`는 사실상 "이 MCP 도구를 이 인자로 불러라"는 라우팅 선언 + 사용법 문서입니다. 지능적 판단이 아니라 **디스패치**입니다. (비-Claude CLI를 위해 같은 디스패치를 파싱하는 `router/` 패키지가 따로 있습니다 — 1,469줄.)

2. **LLM은 ④에서 두 번째로 등장합니다.** 첫 번째 LLM(Claude Code 세션)은 스킬을 읽고 MCP를 호출하는 운전자이고, 두 번째 LLM(런타임 어댑터가 스폰)은 실제 코드를 작성하는 작업자입니다. 그 사이의 ②③은 전부 결정론적 Python — **두 LLM 사이에 끼어 있는 코드 계층이 계약을 집행합니다.**

3. **왜 MCP 서버가 별도 프로세스인가**: 상태(이벤트 스토어, 잡 큐)가 Claude 세션보다 오래 살아야 하기 때문입니다. Claude 세션이 죽어도 detached job은 계속 돌고, `job_status`/`job_wait`로 다시 붙을 수 있습니다.

4. **관측이 사이드채널이 아니라 DB 읽기인 이유**: TUI(Python), 네이티브 TUI(Rust, `crates/ouroboros-tui`), 웹 대시보드 셋 다 이벤트 스토어 SQLite를 직접 읽습니다. 실행 경로에 관측 코드를 끼워 넣는 대신, "기록이 곧 진실"이므로 화면은 기록의 투영일 뿐입니다.

### 훅: 프롬프트 계층의 자동화

Claude Code 훅 3개(`hooks/hooks.json`)도 실체는 Python 스크립트입니다:

| 훅 | 스크립트 | 하는 일 |
| :--- | :--- | :--- |
| SessionStart | `scripts/session-start.py` | 업데이트 확인 (24시간 캐시) |
| UserPromptSubmit | `scripts/keyword-detector.py` | "ooo ..." 키워드 감지 → 스킬 라우팅 (앞서 이 세션에서 오탐도 목격) |
| PostToolUse (Write\|Edit) | `scripts/drift-monitor.py` | 편집 시마다 목표 드리프트 자문 |

여기서도 같은 원리: 트리거 감지는 코드(결정론적), 대응은 프롬프트(스킬)에 위임.

---

## 4. 패키지별 심층: 코드 질량이 몰린 곳의 사연

### orchestrator/ (60K) — "9개 CLI에서 같은 워크플로"의 실제 비용

- 런타임 1종당 `*_runtime.py` 하나 (claude, codex×2, gemini, copilot, opencode, goose, grok, hermes, gjc, pi, zcode, antigravity) — 각각 스폰 방식, 스트림 형식, 권한 전달이 다름
- `runtime_factory.py`/`model_routing.py`/`effort_routing.py`: 어떤 백엔드·모델·추론 노력을 쓸지의 순수 결정 함수 (frugal→standard→frontier 사다리)
- `control_plane.py`/`control_bus.py`/`heartbeat.py`: 실행 중 제어 지시(CANCEL/RETRY/…) 전달과 생존 확인
- `agent_process.py`: 수명주기 동사(spawn/pause/resume/cancel/replay)의 단일 소유자

### mcp/ (37K) — 프롬프트 세계와 코드 세계의 국경

- `tools/definitions.py` + 핸들러 20여 개: 도구 하나당 스키마 + 검증 + 실행 + 오류 매핑
- `detached_jobs.py`/`detached_worker.py`/`job_manager.py`: 세션과 독립적인 백그라운드 잡 실행·복구 — 최근 커밋("harden job lifecycle polling and recovery")이 보여주듯 가장 손이 많이 가는 부분
- `server/security.py`: 요청 경계에서의 방어

### auto/ (22K) — "한 명령으로 알아서"의 상태 기계

`ooo auto`는 겉보기엔 편의 기능이지만, 내부는 인터뷰 라운드 경계, Seed 등급 판정(A/B/C)과 수리 루프, blocked/failed 전이, 실행 인계, 재개 토큰 관리를 다루는 **행(hang)-저항 상태 기계**입니다. "자동화가 우연히 수렴하면 안 된다"(수렴 계약)는 요구가 코드량으로 치환된 사례입니다.

### 작지만 밀도 높은 패키지들

- `resilience/` (1.6K): 정체 4패턴 감지가 전부 해시 비교·뺄셈이라 작음 — 원리가 좋으면 코드가 적다
- `observability/` (1.8K): Jaccard 드리프트 + 3이터레이션 자동 회고
- `evolution/` (5.3K): Wonder/Reflect 프롬프트 조립 + 만족화 백스톱 + 수렴 판정 — LLM 호출부보다 그 결과를 **검증·처분하는 코드**가 더 큼

---

## 5. 종합: 이 코드베이스가 가르쳐 주는 분업 원리

프롬프트 기반 도구(스킬/에이전트)와 코드의 경계선을 어디에 그어야 하는가에 대한, Ouroboros의 실증적 답변:

| 관심사 | 프롬프트에 | 코드에 |
| :--- | :--- | :--- |
| 창의적 판단 (질문 생성, 코드 작성, 평가 의견) | ✅ | |
| 페르소나·관점 (인터뷰어, 악마의 변호인) | ✅ | |
| 워크플로 안내·라우팅 선언 | ✅ (얇게) | |
| 규칙의 **강제** (거부권, 허용목록, 보호 AC) | | ✅ |
| 상태·영속성·재개 | | ✅ |
| 측정·관문 수치 (모호도, 드리프트, 수렴) | | ✅ |
| 프로세스 관리 (스폰, 취소, 타임아웃, 병렬) | | ✅ |
| 프로토콜 (MCP, CLI, 스트림 파싱) | | ✅ |
| 보안 (소독, 상한, 원자적 쓰기, fail-closed) | | ✅ |
| 예산·비용 라우팅 | | ✅ |

경계선의 판별 질문은 하나입니다: **"이것이 확률적으로 지켜져도 되는가, 결정론적으로 보장되어야 하는가?"** 전자는 프롬프트로, 후자는 코드로. Ouroboros의 22만 줄은 후자의 목록이 생각보다 훨씬 길다는 증거입니다.

이는 우리 자신의 Claude Code 활용에도 그대로 적용됩니다: CLAUDE.md의 지시(프롬프트)는 에이전트가 *따르려 노력*하는 것이고, hooks·권한 설정·CI 게이트(코드)는 *구조적으로 강제*되는 것입니다. 중요한 규칙일수록 왼쪽 열에서 오른쪽 열로 옮겨야 합니다 — 그것이 이 프로젝트가 22만 줄로 쓴 결론입니다.

---

*분석 기준: 2026-07-19, main 브랜치 (커밋 7a064482). LOC는 `wc -l` 실측값.*
