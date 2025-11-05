# Annotated - 타입 메타데이터

## 개요

`Annotated`는 타입에 메타데이터를 추가하는 Python 3.9+ 기능입니다. LangGraph에서는 State 필드에 **Reducer 함수**를 연결하여 상태 업데이트 방식을 제어하는 데 사용됩니다.

```python
from typing import Annotated
# 또는
from typing_extensions import Annotated  # Python 3.8 호환
```

## 기본 문법

```python
from typing import Annotated

# Annotated[타입, 메타데이터1, 메타데이터2, ...]
age: Annotated[int, "User's age in years"]
score: Annotated[float, "Score between 0.0 and 1.0", "nullable"]
```

## LangGraph에서의 Reducer 패턴

### Reducer 함수란?

Reducer는 기존 상태 값과 새 업데이트를 결합하는 방법을 정의합니다.

```python
# Reducer 함수 시그니처
def reducer(existing_value, new_value):
    return combined_value
```

### operator.add를 사용한 리스트 누적

```python
from typing import Annotated
from typing_extensions import TypedDict
import operator

class State(TypedDict):
    # 기본 동작: 새 값으로 덮어쓰기
    counter: int

    # Reducer: 새 값을 기존 리스트에 추가
    messages: Annotated[list[str], operator.add]

# 사용 예제
def node_1(state: State) -> dict:
    return {"messages": ["Hello"]}  # ["Hello"]

def node_2(state: State) -> dict:
    return {"messages": ["World"]}  # ["Hello", "World"]
```

### 작동 원리

```python
import operator

# operator.add의 동작
existing = ["Hello"]
new = ["World"]
result = operator.add(existing, new)  # ["Hello", "World"]

# 이는 다음과 동일
result = existing + new
```

## 다양한 Reducer 함수

### 1. operator.add (리스트 누적)

```python
from typing import Annotated
from typing_extensions import TypedDict
import operator

class State(TypedDict):
    logs: Annotated[list[str], operator.add]

# node_1 실행
{"logs": ["Start"]}
# → logs = [] + ["Start"] = ["Start"]

# node_2 실행
{"logs": ["Processing"]}
# → logs = ["Start"] + ["Processing"] = ["Start", "Processing"]
```

### 2. 커스텀 Reducer: 리스트 최대 길이 제한

```python
def limit_list(existing: list, new: list, max_length: int = 10) -> list:
    """리스트를 최대 길이로 제한"""
    combined = existing + new
    return combined[-max_length:]  # 최근 N개만 유지

class State(TypedDict):
    messages: Annotated[list[str], lambda e, n: limit_list(e, n, max_length=5)]

# 6개 메시지가 추가되면 최근 5개만 유지
```

### 3. 커스텀 Reducer: 딕셔너리 병합

```python
def merge_dicts(existing: dict, new: dict) -> dict:
    """딕셔너리를 병합 (새 값이 기존 값을 덮어씀)"""
    return {**existing, **new}

class State(TypedDict):
    metadata: Annotated[dict[str, str], merge_dicts]

# 사용
# node_1: {"metadata": {"user": "Alice", "session": "123"}}
# node_2: {"metadata": {"session": "456", "new_key": "value"}}
# 결과: {"user": "Alice", "session": "456", "new_key": "value"}
```

### 4. 커스텀 Reducer: 숫자 누적

```python
def accumulate(existing: int, new: int) -> int:
    """숫자를 누적"""
    return existing + new

class State(TypedDict):
    total_score: Annotated[int, accumulate]

# node_1: {"total_score": 10}  → total_score = 0 + 10 = 10
# node_2: {"total_score": 5}   → total_score = 10 + 5 = 15
```

### 5. 커스텀 Reducer: 중복 제거

```python
def unique_list(existing: list[str], new: list[str]) -> list[str]:
    """중복을 제거하며 리스트 누적"""
    return list(set(existing + new))

class State(TypedDict):
    unique_tags: Annotated[list[str], unique_list]

# node_1: {"unique_tags": ["python", "coding"]}
# node_2: {"unique_tags": ["python", "tutorial"]}
# 결과: ["python", "coding", "tutorial"]  (중복 제거)
```

## 실전 예제: Chatbot State

```python
from typing import Annotated
from typing_extensions import TypedDict
import operator
from datetime import datetime

class Message(TypedDict):
    role: str
    content: str
    timestamp: str

def add_messages(existing: list[Message], new: list[Message]) -> list[Message]:
    """메시지 리스트 누적 (최대 100개 제한)"""
    combined = existing + new
    return combined[-100:]  # 최근 100개만 유지

class ChatState(TypedDict):
    # 메시지는 누적되며 최대 100개 제한
    messages: Annotated[list[Message], add_messages]

    # 에러 로그는 무제한 누적
    error_logs: Annotated[list[str], operator.add]

    # 메타데이터는 병합
    metadata: Annotated[dict[str, str], lambda e, n: {**e, **n}]

    # 카운터는 덮어쓰기 (Reducer 없음)
    turn_count: int
```

## Reducer 없이 상태 업데이트

Reducer가 없으면 기본적으로 **덮어쓰기(Replace)** 동작을 합니다.

```python
class State(TypedDict):
    counter: int  # Reducer 없음

# node_1 실행
{"counter": 5}  # counter = 5

# node_2 실행
{"counter": 10}  # counter = 10 (덮어쓰기)
```

## Annotated의 다른 활용 (참고)

### 데이터 검증 메타데이터

```python
from typing import Annotated

# pydantic과 함께 사용
from pydantic import Field, BaseModel

class User(BaseModel):
    age: Annotated[int, Field(ge=0, le=120, description="User age")]
    email: Annotated[str, Field(pattern=r'^[\w\.-]+@[\w\.-]+\.\w+$')]
```

### 단위 표시

```python
from typing import Annotated

# 문서화 목적의 메타데이터
distance: Annotated[float, "meters"]
temperature: Annotated[float, "celsius"]
duration: Annotated[int, "seconds"]
```

## 베스트 프랙티스

### 1. 명시적 Reducer 함수 정의

```python
# ❌ 나쁜 예: 람다 함수 (디버깅 어려움)
messages: Annotated[list, lambda e, n: e + n]

# ✅ 좋은 예: 명명된 함수 (디버깅 쉬움)
def append_messages(existing: list, new: list) -> list:
    return existing + new

messages: Annotated[list[str], append_messages]
```

### 2. 타입 명시

```python
# ❌ 타입 불명확
logs: Annotated[list, operator.add]

# ✅ 타입 명확
logs: Annotated[list[str], operator.add]
```

### 3. Reducer 함수 문서화

```python
def merge_with_priority(existing: dict, new: dict) -> dict:
    """
    딕셔너리를 병합하며 새 값이 기존 값을 덮어씁니다.

    Args:
        existing: 기존 딕셔너리
        new: 새로운 딕셔너리

    Returns:
        병합된 딕셔너리
    """
    return {**existing, **new}

metadata: Annotated[dict[str, str], merge_with_priority]
```

### 4. 안전한 Reducer 작성

```python
def safe_add_messages(
    existing: list[str] | None,
    new: list[str] | None
) -> list[str]:
    """None 값을 안전하게 처리하는 Reducer"""
    existing = existing or []
    new = new or []
    return existing + new
```

## 주의사항

### 1. Reducer는 순수 함수여야 함

```python
# ❌ 나쁜 예: 부작용 발생
def bad_reducer(existing: list, new: list) -> list:
    existing.append("side effect")  # 원본 수정 (부작용)
    return existing + new

# ✅ 좋은 예: 순수 함수
def good_reducer(existing: list, new: list) -> list:
    return existing + new  # 새 리스트 반환
```

### 2. 타입 일관성 유지

```python
# ❌ 타입 불일치
def inconsistent_reducer(existing: list, new: dict) -> str:
    return str(existing) + str(new)

# ✅ 타입 일관성
def consistent_reducer(existing: list[str], new: list[str]) -> list[str]:
    return existing + new
```

## 참고 자료

- [PEP 593 - Annotated](https://peps.python.org/pep-0593/)
- [LangGraph Reducer Functions](https://langchain-ai.github.io/langgraph/how-tos/state-reducers/)
- [Python operator Module](https://docs.python.org/3/library/operator.html)
