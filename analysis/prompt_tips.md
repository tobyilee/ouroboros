# Ouroboros 프롬프트에서 배우는 AI 코딩 프롬프트 작성 팁

> 인터뷰/실행/평가 파이프라인 분석(`interview_prompts_params.md`, `run_agent_flow.md`,
> `evaluate_agent_flow.md`)에서 수집한 실제 프롬프트 원문들로부터 추출한,
> **일반 개발자가 자기 CLAUDE.md와 프롬프트에 바로 옮겨 쓸 수 있는** 작성 기법들.
> 모든 인용은 실제 코드/페르소나 파일 원문이며 출처를 표기함.

이 프로젝트의 프롬프트들이 특별한 이유: 대부분의 조항이 **실패를 겪은 뒤 추가된 흔적**이
뚜렷하다. "hardcoded outputs를 찾아라"는 조항은 하드코딩으로 테스트를 통과시킨 에이전트를
겪었다는 뜻이고, "heredoc 금지"는 멀티라인 verify 명령이 깨졌다는 뜻이다. 즉 이것은
이론이 아니라 **부검 보고서의 축적**이다.

---

## A. 출력 형식은 계약이다

### 팁 1. 형식은 설명하지 말고 예시 그대로 박아라

모든 채점/판정 프롬프트가 JSON을 "설명"하지 않고 **모양 그대로** 보여준다:

```
You must respond ONLY with a valid JSON object in the following exact format:
{
    "score": <float between 0.0 and 1.0>,
    "ac_compliance": <boolean>,
    ...
}
```
— `semantic-evaluator.md`, `qa-judge.md`, `consensus-reviewer.md` 등 전부 동일 패턴

그리고 반드시 짝으로 붙는 문장: **"No explanation, no preamble, no markdown fences."**
(semantic.py:197) — 형식 명시와 형식-외-출력 금지는 항상 세트다.

**적용**: 구조화된 출력이 필요하면 스키마 설명 대신 채워진 예시 + "ONLY" + 금지 형식
열거(마크다운 펜스, 서두 문장)를 함께 써라.

### 팁 2. null 케이스에 값싼 탈출구 단어를 줘라

분해 판정 프롬프트 (parallel_executor.py:3737):
```
If the AC is one focused outcome, respond with: ATOMIC
```
복잡한 JSON 구조와 나란히, "해당 없음"을 위한 **한 단어 응답**이 준비되어 있다. 같은 패턴:
- `expect: NONE` (Seed 추출 — 검증 불가능한 AC용)
- "If a category (e.g. coverage) has no clear command, **OMIT it** from the JSON — do not guess" (detector.py:360)
- 결측 데이터는 `insufficient data` / `unchecked`로 (safe-but-wrong 가드)

**왜**: 탈출구가 없으면 모델은 형식을 채우기 위해 **내용을 지어낸다**. "없으면 없다고
말할 수 있는 형식"이 환각의 가장 싼 백신이다.

**적용**: 리스트/구조 출력을 요구할 때 "해당 없으면 X라고만 답해"를 항상 포함하라.

### 팁 3. 금지는 추상적으로 말고 구체적 형태를 열거하라

Seed 추출 프롬프트 (seed_generator.py:673):
```
`verify` must be one complete single-line shell command. Never use heredoc or
multiline syntax (`<<`, `<<'PY'`, `cat <<EOF`, line-continuation scripts);
use `python -c "..."`, `python3 -c "..."`, or `python -m pytest -q` instead.
```

"간단한 명령을 써라"가 아니라: ① 금지 형태를 **문법 토큰 단위로 열거**하고
② **허용되는 대안을 바로 옆에** 제시한다. QA 워커 프롬프트도 동일:
```
omit exploratory discovery commands such as rg, grep, sed, cat, ls, find, or pwd
```

**적용**: "~하지 마라"를 쓸 때마다 자문하라 — 모델이 이 금지를 우회할 구체적 형태를
3개 이상 열거했는가? 대안을 제시했는가?

### 팁 4. 개수와 품질 기준을 같이 묶어라

```
produce 3-7 outcome-level criteria             (seed-architect 규칙)
Five concrete differences beat twenty vague ones   (qa-judge.md)
Keep questions focused (1-2 sentences)          (socratic-interviewer.md)
```

개수 상한만 주면 모델은 상한을 채운다. **"다섯 개의 구체가 스무 개의 모호를 이긴다"**처럼
개수 경계에 품질 비교를 붙이면 "채우기" 행동이 억제된다.

---

## B. 역할 경계는 금지 문구의 예시로 그어라

### 팁 5. "하지 말 것"을 모델이 실제로 뱉을 문장으로 보여줘라

`socratic-interviewer.md`의 역할 경계:
```
- NEVER say "I will implement X", "Let me build", "I'll create" - you gather requirements only
- NEVER promise to build demos, write code, or execute anything
```

"구현하지 마라"(개념 금지)가 아니라 **"I will implement X"라고 말하지 마라**(문구 금지)다.
LLM의 역할 이탈은 특정 관용구로 시작되므로, 그 관용구 자체를 차단하는 것이 훨씬 잘 듣는다.

첫 응답 모양도 같은 기법 (interview.py:1160):
```
CRITICAL: Start your FIRST response with a DIRECT QUESTION about the project.
Do NOT introduce yourself. Do NOT say "I'll conduct" or "Let me ask".
```
그리고 GOOD/BAD 쌍 예시:
```
- GOOD: "Given that JWT auth exists, should the new module extend it or use a different approach?"
- BAD: "What authentication method do you use?" (the caller already told you)
```

**적용**: 역할을 제한할 때 ① 금지 개념이 아니라 금지 **문구**를, ② GOOD/BAD 실물 예시
한 쌍을 써라. "코드 리뷰만 해, 수정하지 마" 대신 → "'수정해드리겠습니다'라고 말하지 마라.
리뷰 결과만 보고하고 멈춰라."

### 팁 6. 다관점 리뷰는 이름만 바꾸지 말고 구조를 다르게 하라

Ouroboros의 3인 합의는 페르소나 이름만 다른 게 아니다:
- **Advocate**: 일반 투표 프롬프트, JSON 예시에 `"approved": true`가 박혀 있고 "반대는
  드물어야 한다"고 명시 — 구조적 승인 편향
- **Devil**: 자유 비판이 아니라 4대 온톨로지 질문(ESSENCE/ROOT_CAUSE)을 **강제로 통과**
  — 비판의 절차화
- **Judge**: 양측 입장 **전문을 프롬프트로** 받고 approved/rejected/conditional 3값 평결

같은 모델을 3번 부르더라도 관점 프롬프트가 실제로 다르면 다른 평가가 나온다
(consensus.py의 single-model fallback이 정확히 이 설계).

**적용**: "다른 시각으로 검토해줘"를 3번 반복하지 말고, 각 호출에 **다른 질문 체계**를
부여하라. 예: 1회차는 "강점만", 2회차는 "이 코드가 증상 치료인지 근본 해결인지만",
3회차는 "앞 두 결과를 보고 판정만".

---

## C. 증거를 형식으로 강제하라

### 팁 7. "근거 필드"를 필수로 만들고, 비어 있으면 실패라고 선언하라

Semantic 평가 프롬프트의 백미 (semantic.py:191):
```
## Evaluation Transparency (anti-reward-hacking)
You MUST show your work so the user can audit the verdict:
- Populate `questions_used` with the concrete Socratic ... questions you asked
- Populate `evidence` with concrete references (file paths, snippets, observed behavior)
- An empty `questions_used` or `evidence` is treated as a verification failure —
  the evaluator is claiming success without showing proof.
```

"근거를 대라"까지는 누구나 쓴다. 결정적 차이는 마지막 줄 — **빈 근거의 의미론을 미리
선언**한 것. "비어 있음 = 검증 실패"라고 규정하면 모델이 근거 생략을 선택지로 여기지 않는다.

**적용**: CLAUDE.md에 "완료 보고에는 실행한 명령과 출력 원문을 포함하라. 이 항목이 비어
있는 보고는 미완료로 간주한다"처럼 **누락의 해석**까지 정의하라.

### 팁 8. 정보에 출처 태그를 붙여 계약 승격을 통제하라

브라운필드 인터뷰의 3색 태그 (socratic-interviewer.md):
```
- [from-code]: 기존 시스템 상태 (사실)
- [from-user]: 인간의 결정/판단
- [from-research]: 외부 조사 정보
```
그리고 승격 정책: **사용자가 말했거나 확인한 것만 요구사항이 될 수 있다** — 코드에서
발견한 사실이나 모델의 추측은 확인 없이 계약이 되지 못한다.

**적용**: 에이전트에게 긴 컨텍스트를 줄 때 "이건 참고용 사실 / 이건 내 결정 / 이건 아직
추측"을 구분해서 표기하라. 모델은 태그가 없으면 전부 같은 무게의 진실로 취급한다.

### 팁 9. 자기보고 대신 실물을 프롬프트에 넣어라

Semantic 평가는 에이전트의 "구현했습니다" 요약이 아니라 **ArtifactCollector가 모은 실제
소스 파일**(총 150K자 상한)을 프롬프트에 넣는다. 실행 워커 프롬프트도 시작이 **실제
`os.listdir` 결과**다:
```
## Working Directory
`{cwd}`
Files present:
- {실제 파일 목록}
**Important**: Use Glob to discover files. Never guess absolute paths.
```

**적용**: "아까 만든 코드 검토해줘" 대신 파일을 다시 읽혀라. 리뷰 서브에이전트에게는
대화 요약이 아니라 diff와 스펙 원문을 줘라. **모델의 기억은 증거가 아니다.**

---

## D. 판정자 프롬프트의 특수 기법

### 팁 10. 채점 기준 수치를 채점자에게 공개하라

`semantic-evaluator.md` 말미:
```
A passing artifact should have:
- ac_compliance = true / score >= 0.8 / goal_alignment >= 0.7
- drift_score <= 0.3 / uncertainty <= 0.3
```
`qa-judge.md`도 verdict 경계(0.80/0.40)를 직접 안다. 기준을 숨기면 점수 분포가 임의로
떠다니고, 공개하면 점수가 **의미를 갖는 눈금**에 정렬된다.

### 팁 11. "감점하지 말 것"을 명시해 정직한 불확실성을 보호하라

모호도 채점 프롬프트 (ambiguity.py:608):
```
IMPORTANT: If the additional context lists "decide-later" or "deferred" items,
these are INTENTIONAL deferrals — the team has deliberately chosen to postpone
those decisions. Do NOT penalise the clarity score for intentionally deferred
items. Score only what is present and answerable.
```

이 조항이 없으면 어떻게 되나? 평가자가 유보 항목을 감점하고 → 작업자는 감점을 피하려고
**모르는 것을 아는 척**하게 된다. "무엇을 벌하지 않는지"의 명시가 정직성을 만든다.
PM Seed 추출의 같은 패턴: "Preserve uncertainty explicitly: do not turn uncertain ...
answers into confirmed requirements."

**적용**: 평가/리뷰를 시킬 때 "TODO나 '미확인' 표기는 결함이 아니다. 미확인을 확인된
것처럼 쓴 곳만 지적하라"를 추가하라.

### 팁 12. 게이밍 수법을 이름 붙여 열거하라

Anti-Gaming 블록 (semantic.py:184):
```
- Look for hardcoded outputs, test-only branches, placeholder logic, or narrow
  implementations that only fit obvious examples.
```
"속임수를 찾아라"가 아니라 **구체적 수법 4종의 카탈로그**다. QA judge의 적대적 프로브
9종(malformed input, stale state, misleading output...)도 같은 설계 — 각각 이름 + 발동
조건(trigger)이 붙어 있다.

**적용**: 리뷰 요청 시 "버그 찾아줘" 대신 점검 항목을 명명하라: "하드코딩된 기대값,
테스트에서만 타는 분기, 예외를 삼키는 곳, 테스트 자체 수정 — 이 네 가지를 우선 확인."

### 팁 13. 판정자에게 "네 판정 기준이 산출물 유형마다 다르다"를 가르쳐라

`qa-judge.md`:
```
... it differs for executable artifacts (missing evidence for an applicable probe
is an evidence gap) versus documents/specifications (unrunnable is never a defect —
apply the classes only as a completeness lens over the document's substance).
```
문서에 "실행 불가"를 결함으로 찍는 오판 — 평가자가 흔히 하는 범주 오류 — 을 프롬프트가
선제 차단한다.

---

## E. 실패와 재시도의 프롬프트

### 팁 14. 재시도 프롬프트에는 실패의 물증을 실어라

AC 재시도 시 주입되는 것 (parallel_executor.py:5592):
```
### Prior failure classification
{실패 분류}
### Last error (tail)
{에러 마지막 500자}
```
그리고 **최종 시도에만** 접근 전환 지시(lateral change-of-approach directive)가 추가된다:
"이전 접근이 실패했다 — 같은 방법을 다시 시도하지 말고 접근을 바꿔라."

**적용**: "다시 해봐"는 최악의 재시도 프롬프트다. ① 무엇이 어떻게 실패했는지(에러 원문),
② 몇 번째 시도인지, ③ 마지막 기회라면 "전략을 바꿔라"까지 실어라.

### 팁 15. 에이전트에게 자기 실패 패턴의 분류학을 미리 줘라

모든 실행 워커의 시스템 프롬프트에 포함되는 Self-Recovery Protocol (recovery.py:180):
```
If you notice that the run is stalled, repeating the same failed edit, or making
no acceptance-criterion progress, switch strategy before continuing:
- spinning: stop retrying the same fix; isolate or bypass the blocker.
- no_drift: gather the missing fact or inspect the source of truth.
- diminishing_returns: simplify the task and remove unnecessary moving parts.
- oscillation: choose one architecture and make the smallest coherent step.
```

정체 감지는 외부 코드(해시 비교)가 하지만, **같은 분류학을 에이전트 자신에게도 줘서**
외부 개입 전에 자가 교정할 기회를 만든다. 각 패턴에 처방이 한 줄씩 붙어 있는 것이 핵심.

**적용**: CLAUDE.md에 넣어라 — "같은 파일을 3번 이상 고치고 있다면 멈추고 접근을
재검토하라. 두 설계 사이를 왕복 중이라면 하나를 선택하고 최소 단위로 진행하라."

### 팁 16. 환경의 진실을 미리 고지해 유령 행동을 막아라

도구 없는 세션에서 모델이 도구 호출을 "흉내"내는 문제의 대응 (claude_code_adapter.py:66):
```
CRITICAL: You have NO tools in this session. Tool calls are impossible and will
be discarded unexecuted. Do NOT emit any tool call or function-call markup.
Respond with plain text only.
```
"도구를 쓰지 마라"(규범)가 아니라 **"호출해도 실행되지 않고 버려진다"(사실)**를 말한다.
모델은 금지보다 **무의미함**에 더 잘 반응한다.

**적용**: "인터넷 접근이 없으니 검색하는 척하지 마라. 모르면 모른다고 써라"처럼 환경
제약을 사실 서술로 알려라.

---

## F. 컨텍스트에는 용도 라벨을 붙여라

### 팁 17. 참고용 컨텍스트는 "행동 금지" 라벨과 함께 줘라

병렬 실행에서 형제 AC 정보를 주되 (atomic_prompt_builder.py:184):
```
## Current AC Scope Boundary
Sibling/future ACs are listed only to define work that is outside the current
dispatch. Do not satisfy those criteria now, and do not pre-create their files,
tests, docs, or evidence.
```

컨텍스트를 주면 모델은 그것을 **할 일로 해석**하는 경향이 있다. "이 목록은 네 스코프
밖을 정의하기 위해서만 존재한다"는 라벨이 과잉 구현을 막는다.

**적용**: 관련 코드를 보여줄 때 "이건 참고용이다 — 수정 대상은 X 파일뿐"을 명시하라.

### 팁 18. 채점 기준을 작업자에게 미리 공개하라 — 단, 채점은 따로 하라

워커 프롬프트의 SUCCESS CONTRACT (atomic_prompt_builder.py:33):
```
SUCCESS CONTRACT for this AC:
- Run locally before completion: {verify_command}. The verify gate re-runs it
  and records authoritative evidence.
```
그리고 완료 지시의 마지막:
```
the harness decides success from typed evidence plus the verifier PASS
```

작업자는 ① 채점 명령을 미리 알고, ② 자기 완료 선언이 판정에 쓰이지 않음을 미리 안다.
채점 명령 공개가 게이밍이 되지 않는 이유는 **하네스가 같은 명령을 독립적으로 재실행**하기
때문 — 투명성과 검증 분리가 세트일 때만 안전하다.

**적용**: 작업 요청에 완료 판정 명령을 포함하라("완료 기준: `pytest tests/test_x.py`
통과"). 그리고 완료 보고를 받으면 그 명령을 **직접 실행해서** 확인하라.

---

## G. 메타 팁: 프롬프트의 한계를 인정한 설계

### 팁 19. 중요한 규칙은 프롬프트에 쓰고 + 코드로도 강제하라

이 코드베이스의 서명 패턴 (code.md 분석 참조): 모든 중요한 프롬프트 규칙에는 **결정론적
백스톱이 짝으로** 존재한다.

| 프롬프트 규칙 | 결정론적 백스톱 |
| :--- | :--- |
| "테스트를 게이밍하지 마라" (Anti-Gaming) | `reward_hacking_risk >= 0.7` 단일 거부권 |
| "verify는 한 줄 셸 명령" | 허용목록 + 셸 메타문자 검사 (detector) |
| "통과한 AC는 건드리지 마라" (Reflect) | 보호 AC 강제 keep 코드 |
| "JSON만 응답하라" | extract_json_payload + 재시도 루프 |
| "3-7개 AC" | 파서의 개수 검증 |

프롬프트는 1차 방어(확률적 준수), 코드는 2차 방어(결정론적 집행)다. 개인 워크플로에서의
번역: **CLAUDE.md 지시(프롬프트)로 시작하되, 반복해서 어겨지는 규칙은 hooks·CI·린트
(코드)로 승격하라.**

### 팁 20. 프롬프트 예산도 우선순위를 설계하라

인터뷰 시스템 프롬프트 조립 (interview.py:1212)은 문자수 초과 시 **잘리는 순서**가
정해져 있다: 동적 헤더(라운드/컨텍스트) 최우선 보존 → 관점 패널 → 베이스 페르소나 순서로
희생. "무엇이 먼저 잘려도 되는가"를 설계한 것.

**적용**: 긴 프롬프트를 쓸 때 가장 중요한 지시를 앞에, 잘려도 되는 참고 자료를 뒤에.
컨텍스트가 압축될 때 살아남아야 하는 것(스펙, 제약)은 별도 파일로 앵커링하라.

---

## 요약: 옮겨 쓸 수 있는 체크리스트

프롬프트를 쓸 때 자문할 것:

1. [ ] 출력 형식을 예시 그대로 보여주고 "ONLY"를 붙였는가? (팁1)
2. [ ] "해당 없음"의 탈출구 단어가 있는가? (팁2)
3. [ ] 금지를 구체적 형태로 열거하고 대안을 제시했는가? (팁3)
4. [ ] 개수 경계에 품질 기준을 묶었는가? (팁4)
5. [ ] 역할 이탈을 금지 문구·GOOD/BAD 예시로 차단했는가? (팁5)
6. [ ] 근거 필드를 필수화하고 "비면 실패"를 선언했는가? (팁7)
7. [ ] 컨텍스트에 출처/용도 라벨을 붙였는가? (팁8, 17)
8. [ ] 리뷰어에게 요약이 아니라 실물을 줬는가? (팁9)
9. [ ] 채점자가 통과 기준 수치를 아는가? (팁10)
10. [ ] 정직한 유보를 감점하지 않도록 보호했는가? (팁11)
11. [ ] 점검 항목(게이밍 수법·프로브)을 이름 붙여 열거했는가? (팁12)
12. [ ] 재시도에 실패 물증과 "전략 전환" 지시를 실었는가? (팁14)
13. [ ] 자가 교정용 실패 분류학을 줬는가? (팁15)
14. [ ] 환경 제약을 사실로 고지했는가? (팁16)
15. [ ] 반복해서 어겨지는 규칙을 코드(훅/CI)로 승격했는가? (팁19)

---

**Created**: 2026-07-23
**출처 분석**: `interview_prompts_params.md`, `run_agent_flow.md`, `evaluate_agent_flow.md`
