# Ouroboros(ooo) Harness/Agent OS 활용 가이드 — Claude Code 사용자 기준

> Claude Code(주력)와 Codex CLI(병행)를 쓰는 개발자가 Ouroboros를 **개발 워크플로 엔진**으로
> 적극 활용하기 위해 알아야 할 개념, 명령, 기억할 것, 팁을 정리한 문서.
> 구체적 프로젝트 적용 예시는 [`migration-scenario-react-spring-kotlin.md`](./migration-scenario-react-spring-kotlin.md) 참고.

---

## 1. Ouroboros는 무엇인가 — "프롬프트가 아니라 스펙"

Ouroboros는 AI 코딩의 실패 지점이 **출력이 아니라 입력(사람의 불명확함)** 이라는 전제에서 출발한다.
그래서 에이전트에게 바로 일을 시키는 대신, 다음 루프를 강제한다:

```
Interview → Seed → Execute → Evaluate
    ↑                          ↓
    └──── Evolutionary Loop ───┘
```

| 단계 | 하는 일 | 게이트 |
|---|---|---|
| **Interview** | 소크라테스식 질문으로 숨은 가정을 드러냄 | 모호도(Ambiguity) ≤ **0.2** 이전엔 Seed 생성 불가 |
| **Seed** | 답변을 불변(immutable) 스펙 YAML로 결정화 | Seed QA 통과선 **0.90** |
| **Execute** | Double Diamond 분해로 백그라운드 실행 | AC(수용 기준) 트리 단위 |
| **Evaluate** | 3단계 게이트: Mechanical($0) → Semantic → Consensus | Stage 2 점수 ≥ **0.8** |
| **Evolve** | Wonder→Reflect로 다음 세대 스펙 진화 | 온톨로지 유사도 ≥ **0.95** 에서 수렴, 최대 30세대 |

핵심 설계 원칙: **Frugal First**(싼 검증 먼저), **Immutable Direction**(Seed는 못 바꾸고 경로만 바꾼다 — 바꾸려면 새 세대), **Event-Sourced**(모든 것이 append-only 이벤트 저널에 기록되어 재생/재개 가능).

### Agent OS 스택

| 레이어 | 리포 | 역할 |
|---|---|---|
| Shell | `Q00/ourocode` | 터미널 TUI 통합 콕핏 |
| Apps | `Q00/ouroboros-plugins` | 도메인 워크플로 플러그인 |
| **OS (이 리포)** | `Q00/ouroboros` | Seed·Ledger·Runtime·MCP 커널 |

"OS"라는 표현이 과장이 아닌 이유: 어떤 LLM/런타임(Claude Code, Codex, Gemini, …)이 실행하든 모든 행동이 **Seed에 묶이고, 이벤트 저널에 기록되고, 재생 가능**한 실행 계약이 되기 때문이다. Claude Code와 Codex는 같은 Seed 파일과 같은 EventStore(`~/.ouroboros/data/ouroboros.db`)를 공유한다.

---

## 2. 설치와 초기 설정 (Claude Code 기준)

```bash
# 터미널에서 1회
claude plugin marketplace add Q00/ouroboros
claude plugin install ouroboros@ouroboros
```

```
# Claude Code 세션 안에서
> ooo setup        # MCP 서버 등록(전역 1회), CLAUDE.md 블록, 모델 설정, 브라운필드 스캔까지 안내
> ooo              # 첫 온보딩 / ooo tutorial 로 5분 체험
```

- MCP 서버는 `uvx --from 'ouroboros-ai[mcp]' ouroboros mcp serve` 형태의 **격리 프로세스**로 등록된다. `pipx` 또는 `uv` 가 필요하다.
- 사전 요구: **Python ≥ 3.12, Git ≥ 2.36** (워크트리 명령 사용). Windows는 WSL2만 지원.
- 모델 설정은 나중에 언제든 `ooo config` 로.

### ⚠️ 반드시 기억: `[claude]` 와 `[mcp]` 는 같은 환경에 설치 금지

Claude Agent SDK는 MCP **1.x**를, Ouroboros MCP 서버는 MCP **2**를 쓴다. 두 extras를 한 Python 환경에 함께 설치하면 깨진다. `ooo setup` / `ooo update` 가 알아서 분리하지만, 수동으로 pip 을 만질 때 `pip install 'ouroboros-ai[mcp,claude]'` 는 절대 하지 말 것. 같은 이유로 Ouroboros는 `~/.claude/mcp.json` 을 직접 수정하지 않는다.

### Codex 병행 설정

```bash
codex plugin marketplace add Q00/ouroboros && codex plugin add ouroboros@ouroboros
# 또는 표준 경로:
ouroboros setup --runtime codex
```

`setup --runtime codex` 는 `~/.ouroboros/config.yaml` 에 `runtime_backend: codex` 를 쓰고, `~/.codex/rules/`·`~/.codex/skills/ouroboros-*`·`~/.codex/config.toml` 의 MCP 항목을 관리한다. 업그레이드 후 rules/skills만 갱신하려면 `ouroboros codex refresh`.

---

## 3. 명령 레퍼런스 — 언제 무엇을 쓰나

Claude Code 세션 안에서는 `ooo <cmd>`, 터미널에서는 `ouroboros <cmd>`. 주요 명령의 선택 기준:

| 상황 | 명령 |
|---|---|
| 요구사항이 아직 모호하다 | `ooo interview "<주제>"` |
| 인터뷰 끝났고 스펙으로 굳히고 싶다 | `ooo seed` |
| 검증된 Seed를 실행 | `ooo run [seed.yaml]` |
| 실행 결과를 공식 검증 | `ooo evaluate <session_id>` |
| 아무 산출물이나 빠른 품질 판정 | `ooo qa <파일/텍스트>` |
| 목표 하나로 전 과정 자동 (인터뷰→A급 Seed→실행) | `ooo auto "<목표>"` |
| 스펙을 세대 진화시키며 수렴 | `ooo evolve "<목표>"` |
| 통과할 때까지 무인 반복 | `ooo ralph --lineage-id <id>` |
| 막혔다, 관점을 바꾸고 싶다 | `ooo unstuck` / `ooo lateral debate <p1> <p2>` |
| 기존 리포를 인터뷰 컨텍스트로 등록 | `ooo brownfield` |
| Stage 1 검증 명령 계약 생성 | `ooo brownfield detect [path]` |
| 목표 대비 이탈(드리프트) 확인 | `ooo status` |
| PRD가 필요하다 (PM 관점) | `ooo pm` |
| Seed를 GitHub Epic/Task 이슈로 발행 | `ooo publish [seed_path]` |
| 세션 끊김 후 재접속 | `ooo resume-session` |
| 스테이지별 모델/런타임 설정 | `ooo config` |
| 멈춘 실행 정리 | `ooo cancel [--all]` |

주의: Claude Code는 `/run`, `/status`, `/help`, `/config`, `/resume` 을 **내장 명령으로 예약**하고 있다. 슬래시로 직접 부르려면 `/ouroboros:ouroboros-run` 처럼 풀네임을 쓰고, 평소에는 그냥 채팅에 `ooo run` 이라고 치는 게 가장 간단하다.

### 3.1 interview — 답은 코드가 먼저 한다

인터뷰는 MCP가 질문을 만들고, **메인 세션(Claude)이 라우터** 역할을 한다:

- 코드/매니페스트에서 확인 가능한 사실 → 자동 확인(`[auto-confirmed]`, `[from-code]`)
- 웹 리서치로 답할 것 → `[from-research]`
- **사람의 판단이 필요한 것만 나에게 온다** → `[from-user]`

단, 3연속 비(非)사용자 답변 후에는 반드시 사용자 질문이 오도록 리듬 가드가 있다. 브라운필드 기본 리포를 등록해 두면 이 자동 답변의 품질이 크게 올라간다(§5). 인터뷰 상태는 `~/.ouroboros/data/` 에 저장되어 중단 후 재개 가능하다.

### 3.2 seed — 생성은 1회, 수정은 QA 루프에서

`ooo seed` 는 Seed를 **한 번만 생성**하고, 이후 통과선 0.90의 QA 루프(최대 5회)에서 Wonder→Reflect→Refine 흐름으로 **YAML을 제자리 수정**한다. 수정 후보는 전부 사용자 채택 게이트를 거친다 — 아무것도 자동 채택되지 않는다. 산출물은 `~/.ouroboros/seeds/<seed_id>.yaml`.

Seed 스키마 핵심 (필수: `goal`, `ontology_schema`, `metadata.ambiguity_score`):

```yaml
goal: "..."
task_type: code          # code(기본) | research | analysis — 문서 산출물이면 꼭 명시
constraints: [...]        # 협상 불가 제약
acceptance_criteria:      # AC 하나 = 실행 트리 노드 하나 = 독립 평가 단위
  - "구체적 파일/함수/엔드포인트를 명시한, 검증 가능한 기준"
ontology_schema: {...}    # 도메인 개념 구조 (entity/action/relation...)
exit_conditions: [...]    # "끝났다"의 정의
metadata: { ambiguity_score: 0.15 }
```

AC 작성 요령: 관심사 하나씩, 검증 가능하게, 의존 순서대로, 산출물 이름 명시. 의존성을 문장에 쓰면("AC1의 config 모듈에 의존") 병렬 스케줄러가 활용한다.

### 3.3 run — 백그라운드 잡 + 관찰자 모델

`ooo run` 은 실행을 **백그라운드 잡**으로 시작하고 `job_id` 를 즉시 돌려준다. 이때:

- 메인 대화는 자유로워지고, **읽기 전용 관찰자(observer) 자식 에이전트 하나**가 잡 이벤트를 독점 구독해 요약을 릴레이한다 (메인 세션이 직접 폴링하지 않는 것이 계약).
- `dashboard_url` 이 표시되면 브라우저로, 아니면 별도 터미널에서 `ouroboros tui monitor` 로 관찰.
- main 브랜치에서 시작하면 `ooo/run/<session_id>` 브랜치/워크트리로 분리되어 실행된다.
- 실행 시작 시 효율 정책을 묻는다: Efficient(`adaptive`) vs Quality-first — 고민되면 처음엔 Quality-first.
- 완료 후 `chained_evaluate_job_id` 가 오면 평가가 자동 체인된 것이니 따라가면 된다.

**중요**: 러너 주도 Seed 실행은 해당 런타임의 `bypassPermissions` 상당 모드로 강제된다. 워크트리 분리가 안전망이므로, 실행 전 Seed의 constraints 에 건드리면 안 되는 영역을 명시해 둘 것.

### 3.4 evaluate — 3단계 게이트와 mechanical.toml

```
Stage 1 Mechanical ($0)   lint/build/test/coverage 를 실제 셸 실행, exit code 판정
Stage 2 Semantic  ($)     LLM 1회: AC 준수, 목표 정렬, 드리프트, 불확실성
Stage 3 Consensus ($$$)   6개 트리거 조건 중 하나 발동 시에만: 다중 모델 2/3 다수결
```

- Stage 1의 명령은 프로젝트 루트의 **`.ouroboros/mechanical.toml`** 이 최우선 계약. 없으면 마커 파일(락파일 등)로 자동 감지, 그것도 없으면 **전부 스킵 = 통과 처리**된다(위험!). `ooo brownfield detect` 로 미리 만들어 두자.
- mechanical.toml 의 명령은 실행 파일 allowlist(pytest, npm, make, cargo, go 등) 검사를 받고, 목록 밖 실행 파일은 **조용히 차단·스킵**된다. Gradle 등은 Makefile 로 감싸 `make` 진입점을 쓰는 게 안전.
- `working_dir` 가 Stage 1 실행 위치와 Stage 2 파일 가시성을 결정한다. 브라운필드 기본 리포 → Seed 메타데이터 → MCP cwd 순으로 폴백.
- 평가 전 "acting verification" 단계가 있다: 소스를 읽는 것에 그치지 않고 **실제로 실행해 관찰한 동작**을 증거로 쓴다.
- 원칙: **실행 완료 ≠ 평가 승인**. 워커가 "task 완료"라 해도 공식 AC 판정은 evaluate 가 내린다.

### 3.5 auto — 급할 때의 원버튼, 대신 한계를 알고 쓰기

`ooo auto "<목표>"` 는 인터뷰→Seed→A급 수리→실행을 한 번에 묶는다. 알아야 할 것:

- 인터뷰가 **바운디드**다: 자동으로 답할 수 있는 건 출처 태그를 달아 자동 답변하고, 사람 권한이 필요한 것(자격증명, 프로덕션 배포, 파괴적 데이터 작업 등)은 절대 자동으로 메꾸지 않는다. 모든 결정이 **Seed Draft Ledger** 에 기록된다 (`ouroboros auto --show-ledger`).
- `--skip-run`(Seed까지만), `--resume <auto_session_id>`, `--complete-product`(Ralph 체인) 플래그.
- 중요한 프로젝트라면 auto 보다 interview→seed→run 수동 루프가 낫다. auto 는 탐색적/소규모 작업용.

### 3.6 evolve / ralph — 세대 진화와 무인 루프

- `ooo evolve "<목표>"` 는 lineage(계보)를 만들고 **한 번에 한 세대**씩 진화시킨다. PASS 된 AC 노드는 동결되고 실패/재개 노드만 진화한다.
- `ontology_stable` 은 성공이 아니다 — 온톨로지가 안정됐을 뿐, 같은 lineage 로 실행을 계속해야 한다.
- 종료 신호: `converged`(수렴) / `stagnated`(3세대 무진전 → `ooo unstuck` 권장) / `exhausted`(30세대 상한) / `failed`.
- `ooo ralph --lineage-id <id>` 는 서버가 소유한 루프로 evolve_step 을 QA 통과 또는 한계까지 반복한다(기본 10세대). **자연어 요구를 ralph 에 직접 넣지 말 것** — interview→seed 를 거친 입력만 받는다.

### 3.7 qa / unstuck / status / publish

- `ooo qa`: 단건 판정. ≥0.80 PASS / 0.40~0.79 REVISE / <0.40 FAIL. quality bar 는 Seed AC 에서 가져오는 게 정석.
- `ooo unstuck`: 5개 페르소나(hacker, researcher, simplifier, architect, contrarian). `ooo lateral debate architect contrarian` 처럼 골라 토론시키고 **최종 선택은 항상 사용자**.
- `ooo status`: 드리프트 = Goal 50% + Constraint 30% + Ontology 20%. **0.3 초과 시 경고** — 스펙이 틀렸으면 인터뷰로 스펙 갱신, 구현이 틀렸으면 롤백.
- `ooo publish`: MCP 불필요, `gh` CLI만 필요. Seed → `[Epic]` 이슈 1개 + AC별 `[Task]` 이슈, `ouroboros`/`epic`/`task` 라벨. 팀 공유·TODO 추적에 유용.

---

## 4. ID 체계 — 헷갈리면 여기부터

| ID | 발급처 | 용도 |
|---|---|---|
| `session_id` (interview) | `ooo interview` | 인터뷰 재개, `ooo seed` 입력 |
| `session_id` / `execution_id` (실행) | `ooo run` | `ooo evaluate`, `ooo status`, `ouroboros cancel execution` |
| `job_id` | 모든 `start_*` 백그라운드 잡 | `job status/wait/result`, `ouroboros_cancel_job` |
| `lineage_id` | `ooo evolve` | `ooo ralph`, `ooo evolve --status` |
| `auto_session_id` | `ooo auto` | `ooo auto --resume` |

자주 하는 실수 두 가지:
1. **ralph 의 lineage_id 로 `ooo evaluate` 호출** — evaluate 는 실행 session_id 를 받는다.
2. **Ralph 잡을 `ouroboros cancel execution` 으로 취소 시도** — 잡은 `ouroboros_cancel_job(job_id)` 로만 취소된다.

### 세션이 끊겼을 때

백그라운드 잡은 세션과 독립적으로 살아있다. 새 Claude Code 세션에서:

```
> ooo resume-session          # EventStore에서 in-flight 세션 나열 (읽기 전용, MCP 불필요)
# 재접속 경로 3가지:
#   ouroboros status execution <exec_id> --events
#   ouroboros tui monitor
#   ouroboros run workflow --orchestrator --resume <session_id> seed.yaml
```

터미널 결과는 지속화된 잡 이벤트에서 읽으므로, 시간이 오래 지나도 `ouroboros job result <job_id>` 로 회수할 수 있다.

---

## 5. 브라운필드 — 기존 코드베이스를 1급 시민으로

```
> ooo brownfield              # $HOME 기준 2-depth 스캔 → 번호 선택으로 기본 리포 등록
> ooo brownfield defaults     # 확인
> ooo brownfield detect .     # .ouroboros/mechanical.toml 생성 (AI 1회 호출)
```

- 등록된 기본 리포는 이후 모든 인터뷰/PM 흐름의 자동 컨텍스트가 된다. 브라운필드 모드에서는 모호도 공식에 **Context Clarity 15%** 가 추가되어, "기존 코드를 이해했는가"도 게이트에 들어간다.
- 스캔 경계: 2단계 깊이, dot 디렉터리와 `node_modules` 제외, 리모트 없는 로컬 리포도 등록 가능. 더 깊은 리포는 안 잡히므로 위치를 조정하거나 CLI 로 `ouroboros setup scan <루트>` 를 쓴다.

---

## 6. 설정 체계 — `~/.ouroboros/config.yaml`

`ooo config` (GUI/TUI) 또는 `ouroboros config show|set|validate`. 꼭 알아둘 키:

```yaml
orchestrator:
  runtime_backend: claude        # claude | codex | gemini | opencode | ...
  max_parallel_workers: 3
  runtime_profile:
    stages: { interview: claude, execute: claude, evaluate: ..., reflect: ... }
clarification:
  ambiguity_threshold: 0.2
  default_model: claude-opus-4-6   # 인터뷰/Seed 단계 모델
execution:
  default_model: null              # 런타임 기본 모델 상속 (권장)
  auto_evaluate: true
  auto_evolve: true
evaluation:
  semantic_model: claude-opus-4-6
  satisfaction_threshold: 0.8
consensus:
  min_models: 3
  diversity_required: true         # 생성 모델 ≠ 검증 모델 (벤더 다양성)
drift:
  warning_threshold: 0.3
```

- **스테이지별로 런타임/모델을 다르게** 둘 수 있다. 프리셋 예: "Claude+Verify"(실행은 Claude, 평가는 타 벤더) — 만든 모델이 스스로 채점하지 않게 하는 설계 철학(diversity)이 깔려 있다.
- 모델 미지정(`default`) 은 "모델 핀을 보내지 않음" — Claude Code/Codex 앱에서 고른 최신 모델을 그대로 상속한다. **특별한 이유가 없으면 핀 고정보다 이 방식이 낫다.**
- 팀 규칙은 `execution.project_guidance: [team]` + `<프로젝트>/.ouroboros/guidance/team/GUIDANCE.md` 로 주입 (16KiB 제한, 평가 우회 불가, resume 시 파일 변경되면 fail-closed).
- 외부 MCP 서버(Context7, Tavily 등)는 `~/.ouroboros/mcp_servers.yaml` 에 등록해 실행 자식 에이전트에게 붙일 수 있다 — 작고 이름 있는 도구 셋을 워크플로별로.

---

## 7. 기억해야 할 것 (함정 목록)

1. **개발 모드 vs 플러그인 모드**: 이 리포를 clone 해서 쓰면 CLAUDE.md 가 `ooo` 명령을 SKILL.md 직독으로 라우팅한다(Skill 도구 사용 금지). 플러그인으로 설치하면 네이티브 스킬로 동작한다. 두 모드를 섞지 말 것.
2. **"Invalid tool parameters" 에러**: MCP 도구 스키마는 deferred 라서 턴 사이에 언로드될 수 있다. 스킬들이 매 호출 전 재-discovery 하도록 설계되어 있으니, 이 에러가 보이면 재시도가 정상 동작이다.
3. **Stage 1 전부 스킵 = 통과 처리**: mechanical.toml 도 마커 파일도 없으면 기계 검증이 무력화된다. 새 프로젝트(특히 Gradle/Kotlin 처럼 자동 감지 밖의 스택)는 반드시 수동 작성.
4. **`--dry-run` 은 현재 동작하지 않는다** (orchestrator 모드에서 무시됨). Seed 검증은 그냥 실행하면 스키마 에러가 실행 전에 출력되는 것으로 대신한다.
5. **실행은 bypassPermissions 상당으로 돈다**: 워크트리 분리와 Seed constraints 가 안전망. 지저분한 워킹트리에서 시작하지 말 것. 실행 잔여물은 `ouroboros cleanup` 으로 정리(라이브/더티 워크트리는 건드리지 않음).
6. **`ooo auto --runtime` 은 실행 단계에만 적용**: 인터뷰·Seed 생성·수리는 항상 MCP 서버 인프로세스로 돈다.
7. **Seed 는 불변**: 스펙을 바꾸고 싶으면 evolve 로 다음 세대를 만들거나 인터뷰를 다시 연다. 실행 중 스펙을 "슬쩍" 바꾸는 경로는 없다 — 그게 기능이다.
8. **MCP 서버 시작 시 1시간 이상 RUNNING/PAUSED 방치 세션은 자동 취소**된다. 오래 걸리는 작업은 잡 이벤트로 상태가 남으니 걱정할 것 없지만, "왜 세션이 취소됐지?"의 흔한 원인.
9. **인터뷰 입력 제한**: 초기 컨텍스트 50,000자, 답변당 10,000자. 큰 문서는 요약해 넣고 세부는 인터뷰가 끌어내게 한다.
10. **이벤트 저널이 진실의 원천**: 뭔가 이상하면 `ouroboros status execution <id> --events` 또는 `ouroboros job events <job_id>` 로 이벤트를 직접 본다. `evaluation.stage1.completed` 페이로드에 체크별 결과가 다 있다.

---

## 8. 실전 팁

- **인터뷰를 아끼지 말 것**: Ouroboros 의 가치 대부분이 인터뷰에서 나온다. "빨리 Seed 뽑자"는 마음으로 강제 생성(옵션 2)을 누르면 도구 전체의 의미가 없어진다. 모호도가 안 떨어지면 구체적 산출물 이름·제약·측정 가능한 성공 기준을 답에 넣어라.
- **AC 는 곧 테스트 계획**: AC 하나가 실행 노드이자 평가 단위다. AC 를 잘 쪼개면 병렬 실행·부분 재시도·정확한 실패 지점 파악이 공짜로 따라온다.
- **Seed 는 작게 여러 개**: 큰 목표는 Seed 여러 개로 나누고 순서대로 실행. 1MB 스키마 제한 이전에, 평가·수렴 가능성이 먼저 무너진다.
- **드리프트 체크를 습관으로**: 긴 작업 중간중간 `ooo status`. 0.3 을 넘기 전에 잡는 게 싸다.
- **막히면 페르소나 토론**: 같은 접근을 3번 반복하고 있다면 그게 stagnation 이다. `ooo lateral debate contrarian simplifier` 는 생각보다 자주 정답을 준다.
- **관찰은 대시보드/TUI 로, 대화는 자유롭게**: 백그라운드 잡 실행 중에 메인 세션에서 다른 질문을 해도 된다 — 그러라고 만든 구조다. 단, 실행 중인 워크스페이스에 직접 쓰기 개입은 충돌 확인 후에.
- **버전 관리**: `ooo update` 가 PyPI 체크 + 원래 설치 방식 재사용 + CLAUDE.md 버전 마커 갱신까지 해준다. 업데이트 후 세션 재시작 필요.
- **Claude Code 자체 기능과의 조합**: Ouroboros 는 워크플로 계약을 담당하고, 세부 코드 리뷰는 Claude Code 의 `/code-review`, 브라우저 검증은 Claude in Chrome 을 병용하면 된다. 서로 배타적이지 않다.

---

## 9. Codex 병행 사용 — 차이점 요약

| 항목 | Claude Code | Codex CLI |
|---|---|---|
| 인증/과금 | Claude 구독(Max 권장), API 키 불필요 | OpenAI 로그인 (`codex login`) |
| 모델 지정 | `ooo config` 스테이지별, 미지정 시 세션 모델 상속 | **Codex 기본 모델 상속 권장** — Ouroboros 는 역할별 reasoning effort(low/medium/high/xhigh)만 전달 |
| 스킬 위치 | 플러그인 (`/ouroboros:*`) | `~/.codex/skills/ouroboros-*` + `~/.codex/rules/` |
| MCP 등록 | `ooo setup` 이 전역 등록 | `~/.codex/config.toml` 의 `[mcp_servers.ouroboros]` |
| 서브에이전트 | 네이티브 관찰자/fan-out 사용 | `spawn_agent` 부재 시 명시적 위임으로 대체 (unstuck 토론 등) |
| Windows | 실험적 | 미지원 (WSL2) |
| 모델 핀 위치 | `~/.ouroboros/config.yaml` | 동일 — `~/.codex/config.toml` 에 스테이지 모델 핀을 두지 말 것 |

같은 Seed 파일이 양쪽에서 그대로 돌지만 실행 경로와 결과는 다를 수 있다 — 이 점을 역이용해 **한쪽에서 생성, 다른 쪽에서 평가**(벤더 다양성)하거나, 컨텍스트별 Seed 를 두 세션에 분산 실행할 수 있다. 두 런타임이 EventStore 를 공유하므로 `ooo status` / `ouroboros resume` 은 어느 쪽에서든 전체 상황을 본다. 단, 하나의 잡/lineage 는 한 세션이 소유하게 할 것.

---

## 10. 최소 치트시트

```
# 설정 (1회)
claude plugin marketplace add Q00/ouroboros && claude plugin install ouroboros@ouroboros
> ooo setup && ooo brownfield && ooo brownfield detect .

# 기본 루프
> ooo interview "<하고 싶은 것>"     # 모호도 ≤ 0.2 까지
> ooo seed                           # QA 0.90 통과 Seed
> ooo run                            # 백그라운드 실행 (+dashboard/TUI)
> ooo evaluate <session_id>          # 3단계 게이트

# 반복/수렴
> ooo evolve "<목표>" → ooo ralph --lineage-id <id>

# 상시
> ooo status        # 드리프트
> ooo unstuck       # 막힘
> ooo resume-session # 세션 끊김
> ooo publish       # 팀 공유 (GitHub Issues)
```
