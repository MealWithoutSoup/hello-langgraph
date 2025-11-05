# 조건부 엣지 (Conditional Edge)

## 개요
LangGraph에서 상태에 따라 동적으로 다음 노드를 선택하는 조건부 라우팅을 구현하는 방법을 학습합니다.

## 핵심 개념

### 일반 엣지 vs 조건부 엣지

**일반 엣지 (Static Edge):**
```python
graph_builder.add_edge("node_a", "node_b")
# node_a는 항상 node_b로 이동
```

**조건부 엣지 (Conditional Edge):**
```python
graph_builder.add_conditional_edges(
    "node_a",
    decision_function,
    {True: "node_b", False: "node_c"}
)
# node_a 다음 노드가 상태에 따라 결정됨
```

## 기본 구현

### 1. State 정의
```python
class State(TypedDict):
    seed: int
```

### 2. 결정 함수 (Decision Function)
```python
def decide_path(state: State):
    return state["seed"] % 2 == 0
```

**반환값:**
- Boolean, 문자열, 또는 매핑에 정의된 키
- 반환값을 기반으로 다음 노드 선택

### 3. 조건부 엣지 추가
```python
graph_builder.add_conditional_edges(
    START,
    decide_path,
    {
        True: "node_one",
        False: "node_two",
        "hello": END,
    },
)
```

**파라미터:**
- 첫 번째: 소스 노드 (START 또는 노드 이름)
- 두 번째: 결정 함수
- 세 번째: 반환값 → 타겟 노드 매핑 딕셔너리

## 결정 함수 패턴

### 1. Boolean 반환
```python
def decide_path(state: State):
    return state["seed"] % 2 == 0

graph_builder.add_conditional_edges(
    "source",
    decide_path,
    {
        True: "even_node",
        False: "odd_node",
    },
)
```

**실행 흐름:**
```
seed=4 → decide_path → True → even_node
seed=5 → decide_path → False → odd_node
```

### 2. Literal 문자열 반환
```python
from typing import Literal

def decide_path(state: State) -> Literal["node_three", "node_four"]:
    if state["seed"] % 2 == 0:
        return "node_three"
    else:
        return "node_four"

graph_builder.add_conditional_edges(
    "source",
    decide_path,
    {
        "node_three": "node_three",
        "node_four": "node_four",
    },
)
```

**장점:**
- 타입 안정성 (IDE 자동완성)
- 명시적인 라우팅 경로

### 3. 다중 조건 분기
```python
def multi_decide(state: State):
    value = state["value"]
    if value < 0:
        return "negative"
    elif value == 0:
        return "zero"
    else:
        return "positive"

graph_builder.add_conditional_edges(
    "check_value",
    multi_decide,
    {
        "negative": "handle_negative",
        "zero": "handle_zero",
        "positive": "handle_positive",
    },
)
```

## 복잡한 그래프 예제

### 노드 정의
```python
def node_one(state: State):
    print("node_one ->", state)
    return {}

def node_two(state: State):
    print("node_two ->", state)
    return {}

def node_three(state: State):
    print("node_three ->", state)
    return {}

def node_four(state: State):
    print("node_four ->", state)
    return {}
```

### 그래프 구성
```python
graph_builder.add_node("node_one", node_one)
graph_builder.add_node("node_two", node_two)
graph_builder.add_node("node_three", node_three)
graph_builder.add_node("node_four", node_four)

# START에서 조건부 분기
graph_builder.add_conditional_edges(
    START,
    decide_path,
    {
        True: "node_one",
        False: "node_two",
        "hello": END,
    },
)

# node_one → node_two (일반 엣지)
graph_builder.add_edge("node_one", "node_two")

# node_two에서 조건부 분기
graph_builder.add_conditional_edges(
    "node_two",
    decide_path,
    {
        True: "node_three",
        False: "node_four",
        "hello": END,
    },
)

# 종료 엣지
graph_builder.add_edge("node_four", END)
graph_builder.add_edge("node_three", END)
```

### 실행 흐름

**seed=4 (짝수):**
```
START → decide_path(4) → True → node_one
node_one → node_two
node_two → decide_path(4) → True → node_three
node_three → END
```

**seed=5 (홀수):**
```
START → decide_path(5) → False → node_two
node_two → decide_path(5) → False → node_four
node_four → END
```

## 실전 활용 패턴

### 1. 사용자 권한 체크
```python
def check_auth(state: State) -> Literal["authorized", "unauthorized"]:
    if state["user"]["role"] == "admin":
        return "authorized"
    return "unauthorized"

graph_builder.add_conditional_edges(
    "auth_check",
    check_auth,
    {
        "authorized": "protected_operation",
        "unauthorized": "access_denied",
    },
)
```

### 2. 데이터 검증
```python
def validate_data(state: State) -> Literal["valid", "invalid"]:
    if all(state["data"].values()):
        return "valid"
    return "invalid"

graph_builder.add_conditional_edges(
    "validation",
    validate_data,
    {
        "valid": "process_data",
        "invalid": "error_handler",
    },
)
```

### 3. 재시도 로직
```python
def check_retry(state: State) -> Literal["retry", "fail", "success"]:
    if state["status"] == "success":
        return "success"
    elif state["retry_count"] < 3:
        return "retry"
    return "fail"

graph_builder.add_conditional_edges(
    "operation",
    check_retry,
    {
        "retry": "operation",  # 자기 자신으로 다시
        "fail": "error_handler",
        "success": "next_step",
    },
)
```

### 4. 우선순위 라우팅
```python
def route_by_priority(state: State):
    priority = state["priority"]
    if priority == "urgent":
        return "fast_track"
    elif priority == "high":
        return "priority_queue"
    else:
        return "normal_queue"

graph_builder.add_conditional_edges(
    "triage",
    route_by_priority,
    {
        "fast_track": "immediate_process",
        "priority_queue": "scheduled_process",
        "normal_queue": "batch_process",
    },
)
```

### 5. 멀티 모델 라우팅
```python
def select_model(state: State):
    complexity = analyze_complexity(state["query"])
    if complexity > 0.8:
        return "advanced_model"
    elif complexity > 0.5:
        return "standard_model"
    return "basic_model"

graph_builder.add_conditional_edges(
    "analyze_query",
    select_model,
    {
        "advanced_model": "gpt4_node",
        "standard_model": "gpt3_5_node",
        "basic_model": "simple_llm_node",
    },
)
```

## 특별한 타겟: END

### END로의 조건부 라우팅
```python
def check_complete(state: State):
    if state["is_complete"]:
        return "end"
    return "continue"

graph_builder.add_conditional_edges(
    "check_status",
    check_complete,
    {
        "end": END,
        "continue": "next_step",
    },
)
```

**사용 사례:**
- 조기 종료 조건
- 성공/실패 시 즉시 종료
- 루프 탈출 조건

## 주의사항

### 1. 매핑에 모든 케이스 포함
```python
# ❌ 잘못된 예
def decide(state: State):
    return state["value"]  # "a", "b", "c" 반환 가능

graph_builder.add_conditional_edges(
    "source",
    decide,
    {"a": "node_a", "b": "node_b"},  # "c" 케이스 누락!
)
```

**결과:** "c" 반환 시 런타임 에러

### 2. 결정 함수의 순수성
```python
# ⚠️ 부작용이 있는 결정 함수
def decide_with_side_effect(state: State):
    send_log("Decision made")  # 부작용!
    return state["value"] > 10

# ✅ 순수 함수 권장
def pure_decide(state: State):
    return state["value"] > 10
```

### 3. None 반환 방지
```python
# ❌ None 반환
def decide(state: State):
    if state["value"] > 10:
        return "node_a"
    # else 케이스 없음 → None 반환

# ✅ 명시적 반환
def decide(state: State):
    if state["value"] > 10:
        return "node_a"
    return "node_b"
```

## 고급 패턴

### 1. 상태 기반 루프
```python
graph_builder.add_conditional_edges(
    "process",
    lambda s: "continue" if s["count"] < 10 else "done",
    {
        "continue": "process",  # 자기 자신으로 루프
        "done": END,
    },
)
```

### 2. 다단계 조건 체크
```python
def multi_stage_check(state: State):
    if not state["validated"]:
        return "validate"
    if not state["processed"]:
        return "process"
    if not state["saved"]:
        return "save"
    return "complete"

graph_builder.add_conditional_edges(
    "pipeline",
    multi_stage_check,
    {
        "validate": "validation_node",
        "process": "processing_node",
        "save": "save_node",
        "complete": END,
    },
)
```

## 주요 학습 포인트

1. **동적 라우팅**: 상태 기반 실행 흐름 제어
2. **결정 함수**: 상태를 읽고 다음 노드 키 반환
3. **매핑 딕셔너리**: 반환값 → 타겟 노드 연결
4. **타입 안정성**: Literal 타입으로 명시적 경로 정의
5. **유연한 제어**: 루프, 조기 종료, 다중 분기 구현

## 다음 단계
- `07_SendAPI.md`: Send API로 병렬 실행 및 팬아웃 패턴
- `08_Command.md`: Command API로 명시적 노드 타겟팅
