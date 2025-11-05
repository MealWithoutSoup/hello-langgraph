# 리듀서 함수 (Reducer Functions)

## 개요
LangGraph에서 상태 필드를 누적하거나 커스텀 방식으로 업데이트하는 리듀서 함수를 사용하는 방법을 학습합니다.

## 핵심 개념

### 기본 상태 업데이트의 한계
일반적인 상태 업데이트는 **덮어쓰기(overwrite)** 방식입니다:
```python
# 현재 상태
{"messages": ["Hello"]}

# 노드 반환
{"messages": ["World"]}

# 결과
{"messages": ["World"]}  # 이전 메시지 손실!
```

### 리듀서 함수의 필요성
메시지 누적, 값 합산 등 **누적 업데이트**가 필요한 경우:
```python
{"messages": ["Hello"]} + ["World"] = {"messages": ["Hello", "World"]}
```

## Annotated를 사용한 리듀서 정의

### 1. 기본 문법
```python
from typing import Annotated
import operator

class State(TypedDict):
    messages: Annotated[list[str], operator.add]
```

**구조:**
- `Annotated[타입, 리듀서함수]`
- 첫 번째 인자: 필드의 실제 타입
- 두 번째 인자: 업데이트 시 사용할 리듀서 함수

### 2. operator.add 리듀서
```python
messages: Annotated[list[str], operator.add]
```

**동작:**
- `old_value + new_value` 연산 수행
- 리스트의 경우: 요소를 이어붙임 (concatenation)
- 숫자의 경우: 값을 더함 (addition)

## 리듀서 동작 방식

### 실행 흐름
```python
def node_one(state: State):
    return {
        "messages": ["Hello, nice to meet you!"],
    }

def node_two(state: State):
    return {}  # 업데이트 없음

def node_three(state: State):
    return {}  # 업데이트 없음
```

### 그래프 실행
```python
graph = graph_builder.compile()

result = graph.invoke({
    "messages": ["Hello!"]
})
```

**상태 변화 추적:**
```
초기 상태: {"messages": ["Hello!"]}

node_one 실행:
  반환: {"messages": ["Hello, nice to meet you!"]}
  리듀서 적용: ["Hello!"] + ["Hello, nice to meet you!"]
  결과: {"messages": ["Hello!", "Hello, nice to meet you!"]}

node_two 실행:
  반환: {}
  상태: {"messages": ["Hello!", "Hello, nice to meet you!"]}

node_three 실행:
  반환: {}
  상태: {"messages": ["Hello!", "Hello, nice to meet you!"]}
```

**최종 결과:**
```python
{'messages': ['Hello!', 'Hello, nice to meet you!']}
```

## 커스텀 리듀서 함수

### 1. 사용자 정의 리듀서
```python
def update_function(old, new):
    return old + new

class State(TypedDict):
    messages: Annotated[list[str], update_function]
```

**시그니처:**
- `reducer(old_value, new_value) -> updated_value`
- `old_value`: 현재 상태의 값
- `new_value`: 노드가 반환한 새 값
- 반환: 업데이트된 값

### 2. 고급 리듀서 예제

**고유 값만 유지:**
```python
def unique_reducer(old: list, new: list) -> list:
    combined = old + new
    return list(set(combined))

class State(TypedDict):
    tags: Annotated[list[str], unique_reducer]
```

**최대 길이 제한:**
```python
def max_length_reducer(old: list, new: list) -> list:
    combined = old + new
    return combined[-10:]  # 최근 10개만 유지

class State(TypedDict):
    history: Annotated[list[str], max_length_reducer]
```

**숫자 누적:**
```python
def sum_reducer(old: int, new: int) -> int:
    return old + new

class State(TypedDict):
    total_count: Annotated[int, sum_reducer]
```

## 리듀서 vs 일반 업데이트

### 일반 업데이트 (덮어쓰기)
```python
class State(TypedDict):
    status: str  # 리듀서 없음

# 노드 1 반환: {"status": "processing"}
# 노드 2 반환: {"status": "completed"}
# 결과: {"status": "completed"}  # 덮어쓰기
```

### 리듀서 업데이트 (누적)
```python
class State(TypedDict):
    events: Annotated[list[str], operator.add]

# 노드 1 반환: {"events": ["start"]}
# 노드 2 반환: {"events": ["process"]}
# 결과: {"events": ["start", "process"]}  # 누적
```

## 실전 활용 패턴

### 1. 대화 히스토리 관리
```python
class ChatState(TypedDict):
    messages: Annotated[list[dict], operator.add]

def user_input(state: ChatState):
    return {"messages": [{"role": "user", "content": "Hello"}]}

def assistant_reply(state: ChatState):
    return {"messages": [{"role": "assistant", "content": "Hi!"}]}
```

**결과:**
```python
{
    "messages": [
        {"role": "user", "content": "Hello"},
        {"role": "assistant", "content": "Hi!"}
    ]
}
```

### 2. 로그 수집
```python
class LogState(TypedDict):
    logs: Annotated[list[str], operator.add]

def step_one(state: LogState):
    # 작업 수행
    return {"logs": ["Step 1 completed"]}

def step_two(state: LogState):
    # 작업 수행
    return {"logs": ["Step 2 completed"]}
```

### 3. 점수 누적
```python
def max_reducer(old: int, new: int) -> int:
    return max(old, new)

class ScoreState(TypedDict):
    total_score: Annotated[int, operator.add]
    high_score: Annotated[int, max_reducer]
```

### 4. 에러 수집
```python
class ErrorState(TypedDict):
    errors: Annotated[list[dict], operator.add]

def validation_node(state: ErrorState):
    errors_found = validate_data(state["data"])
    return {"errors": errors_found}
```

## 리듀서 함수 선택 가이드

| 사용 사례 | 리듀서 함수 | 예제 |
|----------|-----------|------|
| 리스트 누적 | `operator.add` | 메시지, 로그, 이벤트 |
| 숫자 합산 | `operator.add` | 카운터, 점수 |
| 최대값 유지 | `max` | 최고 점수, 피크 값 |
| 최소값 유지 | `min` | 최저 비용, 최소 시간 |
| 고유 값 | 커스텀 `set()` | 태그, 카테고리 |
| 최신 N개 | 커스텀 슬라이싱 | 최근 히스토리 |

## 주요 학습 포인트

1. **누적 업데이트**: 덮어쓰기 대신 값을 누적하는 패턴
2. **Annotated 타입**: 타입과 리듀서를 함께 정의
3. **operator.add**: 가장 일반적인 리스트/숫자 누적 리듀서
4. **커스텀 리듀서**: 특수한 업데이트 로직 구현 가능
5. **상태 보존**: 이전 상태를 기반으로 새로운 상태 생성

## 주의사항

### 초기값 필수
```python
# ❌ 초기값 없이 리듀서 사용
graph.invoke({})  # KeyError 발생 가능

# ✅ 초기값 제공
graph.invoke({"messages": []})
```

### 타입 일관성
```python
# ❌ 타입 불일치
messages: Annotated[list[str], operator.add]
return {"messages": "string"}  # 오류!

# ✅ 타입 일치
return {"messages": ["string"]}
```

## 다음 단계
- `05_Node_Caching.md`: 노드 실행 결과 캐싱으로 성능 최적화
- `07_SendAPI.md`: Send API로 병렬 처리 및 팬아웃 패턴
