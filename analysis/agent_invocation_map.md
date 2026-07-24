# Ouroboros Agent 호출 지점 맵핑 (호출점 1: Execution Flow)

> **목표**: Ouroboros 코드에서 Claude Code 같은 에이전트가 **어디서**, **몇 번**, **어떤 조건에서** 호출되는지 추적하기

---

## 📍 호출 지점 맵핑 (Invocation Callstack)

### 핵심 호출 경로: Seed 실행

```
┌─────────────────────────────────────────────────────────────┐
│ User Input: `ooo run seed.yaml`                             │
└────────────────────────┬────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ orchestrator/runner.py :: execute_seed()                    │
│ • Seed 검증, session 생성                                    │
│ • externally_satisfied_acs 처리 (이미 만족된 AC 스킵)       │
└────────────────────────┬────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ runner.py :: execute_precreated_session()                   │
│ • execution_contract 구축                                   │
│ • System prompt + tool catalog 준비                         │
│ parallel=True 시: → _execute_parallel()                     │
└────────────────────────┬────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ runner.py :: _execute_parallel()                            │
│ • AC 의존성 분석 (DependencyAnalyzer)                       │
│ • ParallelACExecutor 생성                                   │
│ • 의존성 그래프 → StagedExecutionPlan                       │
└────────────────────────┬────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ ParallelACExecutor :: execute_parallel()                    │
│ (orchestrator/parallel_executor.py:1614)                    │
│                                                              │
│ • 체크포인트에서 복구 시도 (RC3)                            │
│ • Stage 루프: for stage in execution_plan.stages:           │
│   └→ for each level in stage:                              │
│      └→ semaphore 제한 하에 AC 병렬 실행                    │
│         └→ _handle_ac() 호출                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔍 ParallelACExecutor 내부: AC별 에이전트 호출

### Stage 루프 구조 (병렬 처리)

**파일**: `orchestrator/parallel_executor.py:1614~`

```python
async def execute_parallel(...):
    for level_idx, stage in enumerate(execution_plan.stages):
        # ← Stage 별 순차 처리 (단계마다 이전 단계 대기)
        
        level_context = ...  # 현재 stage의 공유 상태
        failed_in_level = set()
        
        async with anyio.create_task_group() as tg:
            # ← 이 stage 내의 AC들은 동시 실행
            for ac_idx in stage.ac_indices:
                
                # Semaphore로 동시 실행 제한
                async with self._semaphore:
                    if self._should_skip(ac_idx):
                        # ← externally_satisfied_acs 또는 의존성 실패
                        continue
                    
                    tg.start_soon(
                        self._handle_ac,
                        ac_idx,
                        level_idx,
                        seed,
                        level_context,
                        ...
                    )
```

### 개별 AC 실행: `_handle_ac()` (에이전트 호출 지점)

**파일**: `parallel_executor.py` (정확한 라인은 아직 미확인)

```python
async def _handle_ac(
    self,
    ac_idx: int,
    level_idx: int,
    seed: Seed,
    level_context: LevelContext,
    ...
):
    """한 개 AC를 실행하는 개별 에이전트 세션.
    
    이것이 **실제 Claude Code 호출**이 일어나는 지점.
    """
    
    ac = seed.acceptance_criteria[ac_idx]
    
    # 1️⃣ Decomposition (선택적)
    if self._enable_decomposition:
        decomposed = await self._decompose_ac(ac)
        # ← LLM 호출: AC를 Sub-AC로 분해 (프롬프트 생성 + 호출)
    
    # 2️⃣ Execution (핵심)
    result = await self._adapter.execute_task(
        ac_content=ac.content,
        system_prompt=system_prompt,
        tools=tools,
        model=resolved_model,  # Model router로부터 선택된 모델
        effort=reasoning_effort,  # Effort routing (low/standard/frontier)
        # ← 이 호출이 실제 Claude Code CLI를 스폰
    )
    
    # 3️⃣ Verification (자체 검증)
    if self._run_verify_commands:
        verify_result = await self._run_verify_command(ac.verify_command)
        # ← 셸 명령 실행 (LLM 아님)
    
    # 4️⃣ Evaluation (3단계 평가 게이트)
    eval_result = await self._evaluation_pipeline.evaluate(
        ac=ac,
        output=result,
        # ← 별도 트리거로 LLM 호출 (evaluation/pipeline.py)
    )
    
    # 5️⃣ Recording
    await self._event_emitter.emit_ac_completed(...)
```

---

## 📊 에이전트 호출 통계

### Seed당 몇 개 에이전트 호출?

**AC 3개짜리 Seed 실행 예시:**

```
Seed: goal="...", acceptance_criteria=[AC1, AC2, AC3]

Stage 1 (의존성 레벨 1):
  ├─ AC1 실행
  │  ├─ Decomposition? (if enabled) → 1× LLM 호출
  │  ├─ Execution (adapter.execute_task) → 1× Claude Code 호출
  │  ├─ Verify command → 셸 명령 (LLM 아님)
  │  └─ Evaluation (stage 2, 3 조건부) → 조건부 LLM 호출
  │
  └─ AC2도 병렬로 (위와 같음)

Stage 2 (의존성 레벨 2):
  └─ AC3 (AC1, AC2에 의존)
     └─ (위와 같음)

─────────────────────────────────
총계 (최소): 3× Execution (필수)
총계 (with decomposition): 3 + 3 = 6× LLM
총계 (with evaluation stage 2/3): 6 + 0~3 = 6~9× LLM
```

### 호출되지 않는 단계들

❌ **Decomposition 비활성화 가능**:
```python
ParallelACExecutor(enable_decomposition=False)
# → _decompose_ac() 스킵
```

❌ **Evaluation 게이트**:
- Stage 1 (Mechanical): LLM 아님 (lint, test, build)
- Stage 2 (Semantic): LLM 호출 (조건: `score < 0.8` 또는 `uncertainty > 0.3`)
- Stage 3 (Consensus): LLM 호출 (트리거 조건)
  - Seed 수정 시도
  - 온톨로지 진화
  - 드리프트 > 0.3
  - Stage 2 불확실성 > 0.3

---

## 🎯 호출 지점별 상세 정보

### 1️⃣ Execution (핵심: adapter.execute_task)

**파일**: `orchestrator/adapter.py` (ClaudeAgentAdapter / CodexCliRuntime 등)

**호출 조건**: 모든 AC마다 필수 (1회)

**하는 일**:
- 에이전트 CLI (Claude Code, Codex, OpenCode 등) 스폰
- Prompt 작성 → 스트림 파싱 → 이벤트 발행
- Runtime 어댑터별로 다름 (claude_worker_runtime.py, codex_cli_runtime.py 등)

**스폰되는 프로세스**:
```
Claude Code:  $ claude ... (에이전트 세션)
Codex CLI:    $ codex ... (외부 CLI)
OpenCode:     $ opencode ...
(이하 11개 런타임)
```

### 2️⃣ Decomposition (선택적: _decompose_ac)

**파일**: `orchestrator/decomposition_policy.py`

**호출 조건**:
```python
if enable_decomposition and (
    decomposition_mode == "preflight" or
    (decomposition_mode == "bounce_only" and ac_failed_before)
):
```

**하는 일**:
- 복잡한 AC를 Sub-AC로 분해 (Claude 호출)
- Depth 제한 (기본 5)
- 분해 결과 캐시 (_decomposition_decisions)

### 3️⃣ Evaluation (조건부)

**파일**: `evaluation/pipeline.py`

**호출 조건**:
- Stage 2: `ac_compliance != True or score < 0.8`
- Stage 3: 트리거 조건 (위 참고)

**호출 주체**:
- SemanticEvaluator (Stage 2 LLM)
- ConsensusEvaluator (Stage 3 다중 모델)

---

## 🔗 Runtime Backend별 에이전트 호출

### Claude Code (기본값)

**파일**: `orchestrator/claude_worker_runtime.py` (또는 `claude_agent_adapter.py`)

```python
async def execute_task(...):
    # Claude Code 세션 내 MCP 도구 호출
    # → ouroboros_execute_ac MCP 도구
    # → 실제 코드 작성은 Claude 에이전트가 함
```

### Codex CLI

**파일**: `orchestrator/codex_cli_runtime.py`

```python
async def execute_task(...):
    # $ codex ... 프로세스 스폰
    # 스트림 파싱 → codex_event_normalizer.py로 정규화
```

### OpenCode, Hermes, Gemini, ... (9개 더)

각각 `*_runtime.py` 파일에서 정의:
- `opencode_runtime.py`
- `hermes_runtime.py`
- `gemini_cli_runtime.py`
- `gjc_runtime.py`
- `pi_runtime.py`
- `kiro_adapter.py`
- `grok_cli_runtime.py`
- `copilot_cli_runtime.py`
- `goose_runtime.py`
- `zcode_cli_runtime.py`
- `worker_runtime.py` (일반형)
- `antigravity_cli_runtime.py`

---

## 🛠️ 다음 단계 (아직 탐색 안 함)

### Q2: 워크플로우 흐름 추적
- `ooo run` 한 번 → 에이전트 몇 번 호출?
- Interview → Seed → Execute → Evaluate 각 단계의 호출 수

### Q3: 비용 분석
- 어떤 단계(interview/execute/evaluate/evolve)가 가장 비용이 많이?
- Decomposition 활성화 시 vs 비활성화 시 비용 비교

### Q4: 패턴 발굴
- 병렬 vs 순차
- 에이전트 간 의존성 구조
- 에러 재시도 정책

---

## 📝 핵심 코드 경로 요약

| 파일 | 함수 | 줄 번호 | 역할 |
| :--- | :--- | ---: | :--- |
| runner.py | execute_seed | 3892 | 진입점 |
| runner.py | execute_precreated_session | 4023 | Session 준비 |
| runner.py | _execute_parallel | 4690 | 병렬 실행 준비 |
| parallel_executor.py | ParallelACExecutor.__init__ | 877 | Executor 생성 |
| parallel_executor.py | execute_parallel | 1614 | Stage 루프 |
| (미확인) | _handle_ac | ? | AC별 에이전트 호출 |
| adapter.py | execute_task | 1043+ | 실제 런타임 호출 |
| *_runtime.py | execute_task | 각각 | 각 런타임별 구현 |

---

**Created**: 2026-07-23 (호출점 맵핑 파트 1)
**Status**: In Progress (Q2~Q4 탐색 예정)
