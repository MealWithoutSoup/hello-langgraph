# operator 모듈 - 연산자 함수

## 개요

Python의 `operator` 모듈은 내장 연산자들을 함수로 제공합니다. LangGraph에서는 주로 `operator.add`를 Reducer 함수로 사용하여 리스트를 누적합니다.

```python
import operator
```

## operator.add

### 기본 사용법

```python
import operator

# 숫자 덧셈
result = operator.add(5, 3)  # 8

# 문자열 연결
result = operator.add("Hello", " World")  # "Hello World"

# 리스트 연결
result = operator.add([1, 2], [3, 4])  # [1, 2, 3, 4]

# 튜플 연결
result = operator.add((1, 2), (3, 4))  # (1, 2, 3, 4)
```

### + 연산자와의 관계

```python
import operator

# 다음 두 줄은 동일합니다
result1 = 5 + 3
result2 = operator.add(5, 3)

# 다음 두 줄도 동일합니다
result3 = [1, 2] + [3, 4]
result4 = operator.add([1, 2], [3, 4])
```

## LangGraph에서의 operator.add

### Reducer로 사용

```python
from typing import Annotated
from typing_extensions import TypedDict
import operator

class State(TypedDict):
    # operator.add를 Reducer로 사용
    messages: Annotated[list[str], operator.add]
    logs: Annotated[list[str], operator.add]

# 작동 방식
def node_1(state: State) -> dict:
    return {"messages": ["Hello"]}
    # messages = [] + ["Hello"] = ["Hello"]

def node_2(state: State) -> dict:
    return {"messages": ["World"]}
    # messages = ["Hello"] + ["World"] = ["Hello", "World"]
```

### 내부 동작 원리

```python
import operator

# LangGraph가 내부적으로 하는 일
existing_messages = ["Hello"]
new_messages = ["World"]

# Reducer 함수 호출
combined = operator.add(existing_messages, new_messages)
# combined = ["Hello", "World"]
```

## 다른 유용한 operator 함수들

### 산술 연산

```python
import operator

# 뺄셈
operator.sub(10, 3)      # 7

# 곱셈
operator.mul(4, 5)       # 20

# 나눗셈
operator.truediv(10, 3)  # 3.333...
operator.floordiv(10, 3) # 3

# 나머지
operator.mod(10, 3)      # 1

# 거듭제곱
operator.pow(2, 3)       # 8
```

### 비교 연산

```python
import operator

# 같음
operator.eq(5, 5)        # True

# 같지 않음
operator.ne(5, 3)        # True

# 크기 비교
operator.lt(3, 5)        # True (less than)
operator.le(5, 5)        # True (less than or equal)
operator.gt(5, 3)        # True (greater than)
operator.ge(5, 5)        # True (greater than or equal)
```

### 논리 연산

```python
import operator

# AND
operator.and_(True, False)   # False

# OR
operator.or_(True, False)    # True

# NOT
operator.not_(True)          # False
```

## LangGraph Reducer 함수로 활용

### operator.add 외 다른 연산자 활용

```python
from typing import Annotated
from typing_extensions import TypedDict
import operator

class CounterState(TypedDict):
    # 덧셈: 값을 누적
    total: Annotated[int, operator.add]

    # 곱셈: 값을 곱함
    multiplier: Annotated[int, operator.mul]

# 사용 예
def node_1(state: CounterState) -> dict:
    return {"total": 10, "multiplier": 2}
    # total = 0 + 10 = 10
    # multiplier = 1 * 2 = 2

def node_2(state: CounterState) -> dict:
    return {"total": 5, "multiplier": 3}
    # total = 10 + 5 = 15
    # multiplier = 2 * 3 = 6
```

### 주의: 초기값 설정

```python
# operator.add는 초기값이 필요할 수 있음
class State(TypedDict):
    count: Annotated[int, operator.add]

# 첫 노드에서 초기값 없이 사용하면 오류
def first_node(state: State) -> dict:
    # count가 없으면 오류 발생!
    return {"count": 5}

# 해결: 초기 상태에 값 포함
initial_state = {"count": 0}  # 초기값 0

# 또는 get() 사용
def safe_node(state: State) -> dict:
    current = state.get("count", 0)
    return {"count": current + 5}
```

## 함수형 프로그래밍과의 조합

### functools.reduce와 함께

```python
import operator
from functools import reduce

numbers = [1, 2, 3, 4, 5]

# 모든 숫자의 합
total = reduce(operator.add, numbers)  # 15

# 모든 숫자의 곱
product = reduce(operator.mul, numbers)  # 120
```

### map과 함께

```python
import operator

pairs = [(1, 2), (3, 4), (5, 6)]

# 각 쌍의 합 계산
sums = list(map(lambda pair: operator.add(*pair), pairs))
# [3, 7, 11]
```

## 커스텀 Reducer vs operator

### operator.add 사용 (간단한 경우)

```python
import operator
from typing import Annotated

# 간단한 리스트 누적
messages: Annotated[list[str], operator.add]
```

### 커스텀 Reducer (복잡한 로직)

```python
def custom_add_with_limit(existing: list, new: list) -> list:
    """리스트를 누적하되 최대 100개로 제한"""
    combined = existing + new
    return combined[-100:]  # 최근 100개만 유지

messages: Annotated[list[str], custom_add_with_limit]
```

## 실전 예제: 다양한 Reducer 패턴

```python
from typing import Annotated
from typing_extensions import TypedDict
import operator

class ComplexState(TypedDict):
    # 1. 리스트 누적 (operator.add)
    messages: Annotated[list[str], operator.add]

    # 2. 숫자 누적 (operator.add)
    total_score: Annotated[int, operator.add]

    # 3. 커스텀: 리스트 병합 + 중복 제거
    unique_tags: Annotated[list[str], lambda e, n: list(set(e + n))]

    # 4. 커스텀: 딕셔너리 병합
    metadata: Annotated[dict[str, str], lambda e, n: {**e, **n}]

    # 5. Reducer 없음: 덮어쓰기
    current_user: str

# 사용 예
def process_node(state: ComplexState) -> dict:
    return {
        "messages": ["New message"],        # 누적
        "total_score": 10,                  # 누적
        "unique_tags": ["python", "ai"],    # 병합 + 중복 제거
        "metadata": {"key": "value"},       # 병합
        "current_user": "Alice"             # 덮어쓰기
    }
```

## operator 모듈 완전 참조

### 산술 연산자

| 함수 | 연산자 | 설명 |
|------|-------|------|
| `add(a, b)` | `a + b` | 덧셈 |
| `sub(a, b)` | `a - b` | 뺄셈 |
| `mul(a, b)` | `a * b` | 곱셈 |
| `truediv(a, b)` | `a / b` | 나눗셈 (실수) |
| `floordiv(a, b)` | `a // b` | 나눗셈 (정수) |
| `mod(a, b)` | `a % b` | 나머지 |
| `pow(a, b)` | `a ** b` | 거듭제곱 |

### 비교 연산자

| 함수 | 연산자 | 설명 |
|------|-------|------|
| `eq(a, b)` | `a == b` | 같음 |
| `ne(a, b)` | `a != b` | 같지 않음 |
| `lt(a, b)` | `a < b` | 작음 |
| `le(a, b)` | `a <= b` | 작거나 같음 |
| `gt(a, b)` | `a > b` | 큼 |
| `ge(a, b)` | `a >= b` | 크거나 같음 |

### 논리 연산자

| 함수 | 연산자 | 설명 |
|------|-------|------|
| `and_(a, b)` | `a & b` | AND |
| `or_(a, b)` | `a \| b` | OR |
| `not_(a)` | `not a` | NOT |

### 시퀀스 연산

| 함수 | 연산자 | 설명 |
|------|-------|------|
| `concat(a, b)` | `a + b` | 연결 |
| `contains(a, b)` | `b in a` | 포함 여부 |
| `getitem(a, b)` | `a[b]` | 인덱스 접근 |
| `setitem(a, b, c)` | `a[b] = c` | 인덱스 설정 |
| `delitem(a, b)` | `del a[b]` | 인덱스 삭제 |

## 베스트 프랙티스

### 1. operator.add는 불변 타입에만 사용

```python
# ✅ 좋은 예: 불변 타입 (리스트, 문자열, 튜플)
messages: Annotated[list[str], operator.add]
text: Annotated[str, operator.add]

# ❌ 나쁜 예: 가변 타입은 커스텀 Reducer 필요
# (operator.add가 제대로 작동하지 않을 수 있음)
```

### 2. 초기값 명시

```python
# ✅ 좋은 예
initial_state = {
    "messages": [],      # 빈 리스트로 초기화
    "total_score": 0     # 0으로 초기화
}

# ❌ 나쁜 예: 초기값 없음
initial_state = {}  # operator.add 사용 시 오류 가능
```

### 3. 타입 명시

```python
# ✅ 좋은 예
messages: Annotated[list[str], operator.add]

# ❌ 나쁜 예: 타입 불명확
messages: Annotated[list, operator.add]
```

### 4. 복잡한 로직은 커스텀 Reducer

```python
# operator.add로 충분한 경우
simple_list: Annotated[list[str], operator.add]

# 추가 로직이 필요한 경우 커스텀 Reducer
def add_with_validation(existing: list, new: list) -> list:
    # 검증, 필터링, 변환 등
    validated = [item for item in new if item]
    return existing + validated

complex_list: Annotated[list[str], add_with_validation]
```

## 참고 자료

- [Python operator Module](https://docs.python.org/3/library/operator.html)
- [LangGraph Reducers](https://langchain-ai.github.io/langgraph/how-tos/state-reducers/)
- [functools.reduce Documentation](https://docs.python.org/3/library/functools.html#functools.reduce)
