# 시나리오: React/Node.js 서비스 → React/Spring-Kotlin DDD 마이그레이션에 ooo 활용하기

> 이 문서는 기존 React/Node.js 서비스를 **분석 → 스펙화 → 테스트 세트 구축 → 리뷰/개선 → DDD 스타일 React/Spring-Kotlin 마이그레이션**까지 진행할 때,
> 각 단계에서 Ouroboros(`ooo`) 를 어떻게 끼워 넣는지를 구체적 명령과 함께 정리한 실행 시나리오다.
> 메인 에이전트는 **Claude Code** 기준이며, Codex 병행 팁은 각 단계 말미에 덧붙였다.
> 전체 개념/명령 레퍼런스는 [`ooo-harness-guide.md`](./ooo-harness-guide.md) 참고.

---

## 전체 그림

```
Phase 0  준비          ooo setup → ooo brownfield → ooo brownfield detect
Phase 1  분석/이해      ooo interview → ooo seed (task_type: analysis) → ooo run → ooo evaluate
Phase 2  스펙 생성      ooo interview → ooo seed  (기능 명세 + 온톨로지 = 도메인 모델 초안)
Phase 3  테스트 세트    ooo seed (task_type: code, 특성화 테스트) → ooo run → Stage 1 로컬 검증
Phase 4  리뷰/TODO     ooo qa + /code-review + ooo unstuck → ooo publish (GitHub Issues)
Phase 5  마이그레이션    도메인 모듈별 Seed 분할 → ooo run / ooo ralph → mechanical.toml (gradle)
Phase 6  수렴/검증      ooo evaluate → ooo status(drift) → ooo evolve 수렴
```

Ouroboros의 원칙 하나만 기억하면 전체 시나리오가 자연스럽다:
**"코드를 만지기 전에 모호함을 수치로 줄이고(Ambiguity ≤ 0.2), 결과는 기계적 게이트로 검증한다."**
마이그레이션이야말로 "무엇을 옮기는지"가 애매하면 실패하는 작업이라, 이 프레임과 궁합이 가장 좋다.

---

## Phase 0 — 준비: 브라운필드 등록과 검증 계약 만들기

### 0-1. 설치·설정

```
> ooo setup          # Claude Code 세션 안에서. MCP 프로필/CLAUDE.md 블록/모델 설정까지 안내
```

### 0-2. 기존 리포를 브라운필드 기본 컨텍스트로 등록

```
> ooo brownfield             # 홈 기준 2-depth 스캔 → 번호 목록 표시
  (목록에서 기존 서비스 리포 번호를 채팅으로 답변: 예 "3, 7")
> ooo brownfield defaults    # 등록 확인
```

이후 모든 `ooo interview` 는 등록된 리포를 **자동으로 컨텍스트로 사용**한다.
인터뷰 중 "현재 인증은 어떻게 되어 있나?" 같은 질문은 사람이 아니라 **코드에서 답을 찾아**(`[from-code]` 태그) 진행되므로, 브라운필드 등록 여부가 인터뷰 품질을 좌우한다.

> 주의: 스캔은 기본적으로 홈 디렉터리에서 최대 2단계 깊이만 본다. 리포가 더 깊이 있으면 목록에 안 나온다 — 터미널에서 `ouroboros setup scan <스캔루트>` 로 루트를 직접 지정하면 된다.

### 0-3. 기존 서비스의 검증 계약(mechanical.toml) 생성

```
> ooo brownfield detect ~/work/my-service     # AI 1회 호출로 .ouroboros/mechanical.toml 작성
```

Node.js 프로젝트는 락파일(`package-lock.json`/`pnpm-lock.yaml`/`yarn.lock`)로 자동 감지도 되지만, `detect` 로 미리 만들어 두고 **직접 열어서 lint/test/coverage 명령을 확인·수정**해 두는 것을 권장한다. 이 파일이 이후 모든 `ooo evaluate` Stage 1(무료 기계 검증)의 **유일한 계약**이 된다.

```toml
# my-service/.ouroboros/mechanical.toml (예시)
lint = "npm run lint"
test = "npm test"
coverage = "npm run test:coverage"
coverage_threshold = 0.6      # 레거시라면 처음엔 낮게 시작
```

---

## Phase 1 — 기존 서비스 분석·이해

목표: "이 서비스가 실제로 무엇을 하는가"를 사람이 리뷰할 수 있는 **분석 문서**로 만든다.

### 1-1. 인터뷰로 분석 범위 확정

```
> ooo interview "기존 React/Node.js 서비스의 도메인·아키텍처 분석 문서 작성.
  마이그레이션 준비가 목적이며, 도메인 개념 맵 / API 인벤토리 / 데이터 모델 /
  외부 의존성 / 암묵적 비즈니스 규칙을 산출물로 원한다"
```

인터뷰는 브라운필드 컨텍스트를 읽으며 진행되고, 코드로 확인 가능한 사실은 자동 확인(`[auto-confirmed]`, `[from-code]`)되며 **사람 판단이 필요한 질문만 나에게 온다**(예: "관리자 전용 API도 분석 범위에 포함하나?"). 모호도 0.2 이하가 되면 Seed 생성이 제안된다.

### 1-2. Seed 생성 — `task_type: analysis`

```
> ooo seed                    # 인터뷰 session_id를 이어받아 생성 + QA 루프(통과선 0.90)
```

생성된 Seed에서 확인할 포인트:

```yaml
goal: "마이그레이션 준비를 위한 기존 서비스 도메인/아키텍처 분석 문서 작성"
task_type: analysis           # 코드가 아니라 마크다운 문서가 산출물
acceptance_criteria:
  - "도메인 개념 맵: 엔티티/값객체 후보와 관계를 mermaid로 정리"
  - "API 인벤토리: 전체 엔드포인트를 메서드/경로/요청·응답 스키마/호출처와 함께 표로"
  - "데이터 모델: DB 스키마와 코드상 모델의 불일치 지점 명시"
  - "암묵적 비즈니스 규칙: 코드에만 존재하고 문서에 없는 규칙 목록"
  - "결합도 분석: 서비스 경계 후보 3-5개와 의존성 근거"
ontology_schema:              # ← 이 온톨로지가 Phase 5의 DDD 도메인 모델 초안이 된다
  name: "MyServiceDomain"
  fields:
    - { name: "order",  field_type: "entity",   description: "..." }
    - { name: "settle", field_type: "action",   description: "..." }
```

### 1-3. 실행과 평가

```
> ooo run                     # 백그라운드 잡으로 실행, 대시보드 URL 표시
> ooo evaluate <session_id>   # 문서 산출물이므로 artifact_type: docs 로 평가
```

**Codex 병행 팁**: 분석처럼 read-heavy 한 작업은 Codex 세션에서 같은 Seed로 돌려 교차 검증하기 좋다. Seed YAML은 런타임 중립이므로 `~/.ouroboros/seeds/` 의 같은 파일을 그대로 쓴다.

---

## Phase 2 — 스펙(명세) 생성

목표: 분석 문서를 근거로 "현재 서비스가 보장해야 하는 동작"을 **불변 Seed 스펙**으로 고정한다. 이 스펙이 곧 마이그레이션의 동등성(parity) 기준이 된다.

```
> ooo interview "Phase 1 분석 문서(analysis/current-service-analysis.md)를 기준으로,
  현행 서비스의 기능 명세를 acceptance criteria 로 고정하고 싶다.
  마이그레이션 후에도 반드시 유지돼야 하는 동작 계약이 목적"
> ooo seed
```

팁:

- **AC 하나 = 검증 가능한 동작 하나**로 쪼갠다. "주문 기능이 잘 동작한다"(나쁨) → "POST /orders 는 재고 부족 시 409와 `INSUFFICIENT_STOCK` 코드를 반환한다"(좋음).
- 규모가 크면 **도메인 영역별로 Seed를 나눈다**(주문 Seed, 정산 Seed, …). Seed는 1MB 제한도 있고, 작을수록 실행·평가·수렴이 빠르다.
- Seed QA 루프에서 REVISE가 나오면 Wonder→Reflect→Refine 흐름을 따라가되, **모든 수정 후보는 내가 채택 여부를 결정**한다(자동 채택 없음).
- 팀 공유가 필요하면 이 시점에 `ooo publish` 로 스펙을 GitHub Epic/Task 이슈로 발행해 리뷰받을 수 있다.

---

## Phase 3 — 테스트 세트 생성과 로컬 실행

목표: 마이그레이션의 안전망이 될 **특성화 테스트(characterization tests)** 를 기존 서비스에 만든다. "현행 동작을 그대로 캡처"하는 것이지, 이상적인 동작을 새로 정의하는 것이 아니라는 점이 핵심이다.

```
> ooo interview "Phase 2 스펙 Seed의 AC를 커버하는 특성화 테스트 스위트를
  기존 Node.js 서비스에 추가. supertest 기반 API 레벨 테스트 우선,
  현행 동작(버그 포함)을 그대로 캡처하는 것이 원칙"
> ooo seed
> ooo run
```

Seed 작성 요령:

```yaml
task_type: code
constraints:
  - "기존 프로덕션 코드는 수정하지 않는다 (테스트 코드만 추가)"
  - "현행 동작을 있는 그대로 캡처한다 — '올바른' 동작으로 고치지 않는다"
  - "외부 의존성(결제/메일 등)은 기록된 응답으로 스텁"
acceptance_criteria:
  - "tests/characterization/orders.spec.ts: 주문 API 전체 경로의 응답 스냅샷 테스트"
  - "npm test 가 로컬에서 그린으로 통과"
  - "API 인벤토리의 전 엔드포인트에 최소 1개 테스트 (커버리지 리포트로 증명)"
```

로컬 검증 흐름:

1. `ooo run` 이 끝나면 `ooo evaluate <session_id>` — Stage 1이 `mechanical.toml` 의 `npm test` 를 **실제로 실행**해 exit code 로 판정한다. "테스트를 만들었다"가 아니라 "테스트가 돈다"가 게이트다.
2. 커버리지가 목표에 못 미치면 evaluate 가 REJECTED 를 주고, 그 결과가 다음 사이클 입력이 된다 — 수동으로 고치거나 `ooo ralph` 로 "통과할 때까지" 루프를 돌린다.
3. 이 테스트 스위트는 Phase 5에서 **신규 Spring 백엔드에 같은 요청을 쏘는 동등성 테스트로 재사용**할 것이므로, 테스트가 HTTP 레벨(포트/베이스URL 주입 가능)로 작성되도록 constraints 에 명시해 두면 좋다.

---

## Phase 4 — 기본 리뷰와 개선 TODO 목록화

목표: 당장 고칠 것과 마이그레이션 때 설계로 풀 것을 구분해 **추적 가능한 TODO** 로 만든다.

### 4-1. 리뷰 3종 세트

| 도구 | 용도 |
|---|---|
| Claude Code `/code-review` | 브랜치/디프 단위 정통 코드 리뷰 (버그·정리 후보) |
| `ooo qa <파일/디렉터리>` | Seed의 AC를 quality bar 로 한 단건 판정 (0.80 PASS / 0.40–0.79 REVISE / <0.40 FAIL) |
| `ooo unstuck architect` / `ooo lateral debate architect contrarian simplifier` | "구조적으로 무엇이 문제인가"를 페르소나 관점으로 도출 — 마이그레이션 경계 설계 재료 |

`ooo qa` 는 소스 텍스트보다 **관찰된 동작**을 우선하므로, 서버를 실제로 띄운 상태에서 돌리면 판정 품질이 올라간다.

### 4-2. TODO 목록화 — `ooo publish`

개선 항목을 Seed(또는 PM Seed)로 정리한 뒤:

```
> ooo publish ~/.ouroboros/seeds/<improvement_seed>.yaml
```

Epic 이슈 1개 + AC별 Task 이슈들이 `ouroboros`/`epic`/`task` 라벨과 함께 생성된다. `gh auth login` 이 되어 있어야 하고, 대상 리포 확인 후 발행된다. "당장 개선"(기존 코드에서 바로 수리)과 "마이그레이션에서 해소"(신규 설계에 반영) 를 **라벨이나 별도 Epic 으로 구분**해 두면 Phase 5 인터뷰의 입력으로 깔끔하게 이어진다.

간단히 가려면 GitHub 이슈 대신 `analysis/improvement-todo.md` 마크다운 체크리스트로도 충분하다 — 핵심은 각 항목이 Phase 2 스펙의 AC 를 참조하게 하는 것.

---

## Phase 5 — React/Spring-Kotlin DDD 마이그레이션

### 5-1. 마이그레이션 인터뷰 — 온톨로지가 곧 도메인 모델

```
> ooo interview "기존 Node.js 백엔드를 Spring Boot + Kotlin, DDD 스타일로 재구축.
  프론트는 React 유지하되 API 계약은 Phase 2 스펙 Seed 를 따른다.
  Phase 1 분석의 서비스 경계 후보와 Phase 4 TODO 중 '설계로 해소' 항목을 입력으로 사용"
```

여기서 Ouroboros 의 온톨로지 개념이 DDD 와 정확히 맞물린다:

- `ontology_schema` 의 `entity` → 애그리거트 루트/엔티티 후보
- `field_type: action` → 도메인 이벤트/커맨드 후보
- 인터뷰의 "What IS a task?" 식 질문 → 유비쿼터스 언어 정련 과정 그 자체
- `ooo evolve` 의 온톨로지 수렴(유사도 ≥ 0.95) → "도메인 모델이 더 이상 흔들리지 않는 시점"의 수치화

### 5-2. Seed 분할 전략

빅뱅 Seed 하나가 아니라 **바운디드 컨텍스트 단위로 Seed 를 나누고 순서대로 실행**한다:

```
seed-01-walking-skeleton.yaml   # Gradle 멀티모듈 + hexagonal 골격 + CI + 헬스체크
seed-02-orders-context.yaml     # 주문 컨텍스트: 도메인 → 애플리케이션 → 어댑터
seed-03-settlement-context.yaml # 정산 컨텍스트 (주문에 의존 → AC 순서로 표현)
seed-04-parity-gate.yaml        # Phase 3 특성화 테스트를 신규 백엔드로 돌리는 동등성 게이트
```

constraints 에 DDD 규칙을 명시하면 실행 전반에 걸쳐 강제된다. Seed 하나에 담기 애매한 팀 공통 규칙(패키지 구조, 네이밍, 계층 규칙 등)은 신규 리포에 `.ouroboros/guidance/team/GUIDANCE.md` 를 만들고 `~/.ouroboros/config.yaml` 에 `execution.project_guidance: [team]` 을 켜서 **모든 실행에 자동 주입**할 수도 있다 (16KiB 제한, 평가 게이트 우회는 불가):

```yaml
constraints:
  - "Kotlin + Spring Boot, JDK 21"
  - "hexagonal: domain 모듈은 Spring/JPA 의존 금지 (순수 Kotlin)"
  - "애그리거트 간 참조는 ID로만, 트랜잭션은 애그리거트 단위"
  - "API 응답 계약은 Phase 2 스펙 Seed 의 AC 와 바이트 수준 호환"
```

### 5-3. 신규 프로젝트의 mechanical.toml — 반드시 수동 작성

**중요**: Stage 1 자동 감지 목록에 Gradle/Kotlin 은 없다. 신규 리포에는 `.ouroboros/mechanical.toml` 을 직접 만든다:

```toml
lint  = "make lint"      # ktlintCheck + detekt 를 Makefile 로 래핑
build = "make build"     # ./gradlew build -x test
test  = "make test"      # ./gradlew test
```

mechanical.toml 의 명령은 **실행 파일 allowlist**(pytest, npm, make, cargo, go 등) 검사를 받는다. `./gradlew` 가 조용히 차단되어 체크가 스킵될 수 있으므로(스킵 = 통과 처리라서 위험), **Makefile 로 감싸 `make` 를 진입점으로 쓰는 방식이 가장 안전**하다. 설정 후 `ooo evaluate` 이벤트에서 `evaluation.stage1.completed` 에 체크 결과가 실제로 기록되는지 한 번 확인할 것.

### 5-4. 실행 — run, 그리고 ralph

```
> ooo run seed-01-walking-skeleton.yaml      # 컨텍스트별 순차 실행
> ooo evaluate <session_id>
```

반복 수렴이 필요한 컨텍스트(동등성 테스트가 계속 깨지는 경우)는 Ralph 루프로 넘긴다:

```
> ooo evolve "orders 컨텍스트를 동등성 게이트 통과까지 진화"   # lineage 생성
> ooo ralph --lineage-id <lineage_id>                        # QA 통과/한계까지 세대 반복
```

주의: `ooo ralph` 에 자연어 요구사항을 바로 넣지 말 것 — 반드시 interview → seed 를 거친 검증된 Seed/lineage 로 시작해야 한다. 새 백엔드는 별도 워크트리/브랜치(`ooo/run/<session_id>`)에서 진행되므로 기존 서비스와 충돌하지 않는다.

**Codex 병행 팁**: 구현 단계에서 컨텍스트별 Seed 를 Claude Code 와 Codex 세션에 나눠 돌릴 수 있다(같은 EventStore 를 공유하므로 `ooo status` / `ouroboros resume` 로 통합 관찰). 단, 하나의 lineage/잡은 하나의 세션이 소유하게 하고, 교차 개입은 `ooo status` 확인 후에.

### 5-5. 프론트엔드(React) 쪽

React 는 유지가 기본이므로 Seed 는 "백엔드 교체에 따른 어댑터 작업"으로 좁힌다: API 클라이언트 베이스URL/인증 헤더 전환, 에러 코드 매핑, (계약이 바뀌는 지점이 있다면) BFF 어댑터. Phase 3 테스트가 HTTP 레벨이면 프론트 스모크는 Claude in Chrome 브라우저 자동화로 보완할 수 있다.

---

## Phase 6 — 수렴 판정과 마무리

```
> ooo evaluate <session_id>        # 컨텍스트별 3단계 게이트 (동등성 테스트가 Stage 1)
> ooo status                       # drift 측정: 목표 대비 이탈도 ≤ 0.3 유지 확인
> ooo evolve --status <lineage_id> # 온톨로지 수렴 여부 (converged / stagnated)
```

- **drift > 0.3** 경고가 뜨면 구현이 스펙에서 이탈한 것 — 스펙이 틀렸으면 `ooo interview` 로 스펙을 갱신(새 Seed 세대)하고, 구현이 틀렸으면 되돌린다. "조용히 스펙과 코드가 갈라지는" 마이그레이션 최대 리스크를 수치로 감시하는 장치다.
- `stagnated` (3세대 무진전) 이 나오면 `ooo unstuck` 으로 관점을 바꾼다. 마이그레이션 후반에 자주 오는 신호다.
- 전환 완료 기준을 Seed 의 `exit_conditions` 로 미리 박아두면 "끝났다"의 정의가 흔들리지 않는다: 예) "동등성 스위트 100% 통과 + 신규 스택 커버리지 ≥ 70% + 구 백엔드 트래픽 0".

---

## 자주 하는 실수 요약 (이 시나리오 한정)

1. **브라운필드 등록 없이 인터뷰 시작** → 코드로 답할 질문이 전부 나에게 옴. Phase 0-2 먼저.
2. **분석·스펙 Seed 에 `task_type` 누락** → 기본값 `code` 로 실행되어 문서 대신 코드를 만들려 함. `analysis`/`research` 명시.
3. **특성화 테스트에서 "올바른 동작"으로 교정** → 동등성 기준 자체가 오염됨. constraints 로 금지 명시.
4. **Spring 리포에 mechanical.toml 없이 evaluate** → Stage 1 전체 스킵(=통과 처리)으로 게이트가 무력화됨. gradle 은 자동 감지 안 됨.
5. **거대 단일 마이그레이션 Seed** → 평가·수렴 불가능. 바운디드 컨텍스트 단위 분할.
6. **ralph 에 자연어 목표 직접 투입** → 구조화 입력 요구로 반려됨. interview → seed 경유.
7. **ralph 의 lineage_id 로 `ooo evaluate` 시도** → lineage_id 는 실행 session_id 가 아님. 평가는 실행 세션 기준.
