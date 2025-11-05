# 05. Time Travel (시간 여행)

## 개요

LangGraph의 강력한 기능 중 하나인 "시간 여행"을 학습합니다. 과거의 특정 체크포인트로 돌아가 상태를 수정하고, 새로운 타임라인(분기)을 생성할 수 있습니다.

## 핵심 개념

### 1. 상태 히스토리 탐색

```python
state_history = graph.get_state_history(config)

for state_snapshot in state_history:
    print(state_snapshot.next)          # 다음 노드
    print(state_snapshot.values)         # 상태 값
    print(state_snapshot.config)         # 체크포인트 설정
```

**상태 스냅샷 구조:**
- `next`: 다음에 실행될 노드
- `values`: 현재 상태의 모든 값
- `config`: 체크포인트 ID 및 설정
- `metadata`: 메타데이터 (소스, 단계 등)
- `created_at`: 생성 시간
- `parent_config`: 이전 체크포인트 참조

### 2. 특정 체크포인트 선택

```python
state_history = graph.get_state_history(config)
all_states = list(state_history)

# 과거 두 번째 상태 선택
to_fork = all_states[-2]

print(to_fork.config)
# {'configurable': {
#     'thread_id': '11',
#     'checkpoint_id': '1f0b88e8-b8be-66cd-8000-725c36031126'
# }}
```

### 3. 상태 수정 (update_state)

```python
from langchain_core.messages import HumanMessage

new_checkpoint = graph.update_state(
    to_fork.config,
    {
        "messages": [
            HumanMessage(
                content="I live in Europe now. And the city I live in is Seoul.",
                id="bcbdfe8d-0325-4f0a-82ca-07c27698d27f",
            )
        ]
    },
)
```

**`update_state()`의 역할:**
- 특정 체크포인트의 상태를 수정
- 새로운 체크포인트 생성 (원본은 그대로 유지)
- 수정된 체크포인트부터 새로운 분기 생성

## 시간 여행 워크플로우

### 단계 1: 원래 대화 실행

```python
config = {"configurable": {"thread_id": "11"}}

result = graph.invoke(
    {
        "messages": [{
            "role": "user",
            "content": "I live in Europe now. And the city I live in is Valencia.",
        }]
    },
    config=config,
)
```

**결과:**
- LLM이 Valencia(스페인 도시)에 대한 정보 제공
- 체크포인트에 상태 저장

### 단계 2: 과거 시점 찾기

```python
state_history = graph.get_state_history(config)
all_states = list(state_history)

# 히스토리 구조:
# [0] 최종 상태 (대화 완료)
# [1] chatbot 실행 직전
# [2] 초기 상태 (START)

to_fork = all_states[-2]  # chatbot 실행 직전 선택
```

### 단계 3: 상태 수정으로 분기 생성

```python
new_checkpoint = graph.update_state(
    to_fork.config,
    {
        "messages": [
            HumanMessage(
                content="I live in Europe now. And the city I live in is Seoul.",
                id="bcbdfe8d-0325-4f0a-82ca-07c27698d27f",  # 같은 ID 사용!
            )
        ]
    },
)
```

**중요:**
- 같은 메시지 ID를 사용하여 기존 메시지를 교체
- 새로운 ID를 사용하면 메시지 추가됨

### 단계 4: 새 분기에서 실행

```python
result = graph.invoke(
    None,  # 상태 이미 설정되어 있으므로 None
    config={
        "configurable": {
            "thread_id": "11",
            "checkpoint_id": new_checkpoint["checkpoint_id"],
        },
    },
)
```

**결과:**
- LLM이 "Seoul은 유럽이 아니라 아시아에 있다"고 지적
- 원래 Valencia 타임라인은 그대로 유지

## 타임라인 구조 시각화

```
초기 상태 (START)
    ↓
    ├─── [원본 타임라인] ───────────────────┐
    │    "Valencia"                        │
    │    → LLM: Valencia 정보 제공          │
    │                                      │
    └─── [분기된 타임라인] ─────────────────┤
         "Seoul"                           │
         → LLM: 지리적 오류 지적             │
                                          │
         두 타임라인 모두 독립적으로 존재 ─────┘
```

## 실전 활용 사례

### 1. 대화 수정 및 재실행

```python
# 사용자가 잘못된 정보를 제공했다면
# 과거로 돌아가 올바른 정보로 수정
state_history = graph.get_state_history(config)
past_state = list(state_history)[2]

graph.update_state(
    past_state.config,
    {"messages": [HumanMessage(content="corrected information")]}
)

# 수정된 정보로 다시 실행
result = graph.invoke(None, config=past_state.config)
```

### 2. What-If 시나리오 분석

```python
# 다양한 선택지를 시도해보기
base_state = list(graph.get_state_history(config))[-2]

scenarios = [
    "What if I choose option A?",
    "What if I choose option B?",
    "What if I choose option C?",
]

results = []
for scenario in scenarios:
    new_checkpoint = graph.update_state(
        base_state.config,
        {"messages": [HumanMessage(content=scenario)]}
    )

    result = graph.invoke(None, config={
        "configurable": {
            "thread_id": config["thread_id"],
            "checkpoint_id": new_checkpoint["checkpoint_id"],
        }
    })
    results.append(result)

# 모든 시나리오 결과 비교
```

### 3. 디버깅 및 테스트

```python
# 특정 시점의 상태로 돌아가 다른 입력 테스트
def test_from_checkpoint(checkpoint_config, test_input):
    new_checkpoint = graph.update_state(
        checkpoint_config,
        {"messages": [HumanMessage(content=test_input)]}
    )

    return graph.invoke(None, config={
        "configurable": {
            **checkpoint_config["configurable"],
            "checkpoint_id": new_checkpoint["checkpoint_id"],
        }
    })

# 여러 테스트 케이스 실행
test_checkpoint = list(graph.get_state_history(config))[-2]
test_from_checkpoint(test_checkpoint.config, "test case 1")
test_from_checkpoint(test_checkpoint.config, "test case 2")
```

## 주요 학습 포인트

1. **비파괴적 수정**: 원본 타임라인은 유지되고 새 분기 생성
2. **완전한 히스토리**: 모든 체크포인트를 탐색 가능
3. **유연한 실험**: 같은 시점에서 다양한 시나리오 시도
4. **메시지 ID**: 같은 ID 사용 시 교체, 다른 ID 사용 시 추가
5. **체크포인트 관리**: `checkpoint_id`로 특정 분기 선택

## 고급 패턴

### 분기된 타임라인 추적

```python
def get_timeline_tree(config):
    """모든 분기를 트리 구조로 반환"""
    history = graph.get_state_history(config)

    timeline_tree = {}
    for state in history:
        checkpoint_id = state.config["configurable"]["checkpoint_id"]
        parent_id = state.parent_config["configurable"]["checkpoint_id"] if state.parent_config else None

        timeline_tree[checkpoint_id] = {
            "state": state,
            "parent": parent_id,
            "children": []
        }

    return timeline_tree
```

### 특정 조건에서 자동 분기

```python
def auto_fork_on_condition(state, condition_fn):
    """조건이 참일 때 자동으로 분기 생성"""
    if condition_fn(state):
        current_config = graph.get_state(config).config

        # 대안 액션으로 분기 생성
        new_checkpoint = graph.update_state(
            current_config,
            {"alternative_path": True}
        )

        return new_checkpoint
    return None
```

## 제약사항 및 주의사항

### 1. 체크포인터 필수

```python
# 체크포인터 없으면 time travel 불가
graph = graph_builder.compile()  # 체크포인터 없음
graph.get_state_history(config)  # Error!
```

### 2. 메모리 관리

```python
# 많은 분기는 스토리지 사용량 증가
# 주기적으로 오래된 체크포인트 정리 필요
```

### 3. 스레드 격리

```python
# 다른 thread_id의 체크포인트는 접근 불가
# 각 스레드는 독립적인 타임라인 보유
```

## 다음 단계

- [06. DevTools (LangSmith)](./06-devtools-langsmith.md): 개발 도구 및 모니터링

## 관련 코드

- 소스 파일: `agents/05_Time_Travel.ipynb`
