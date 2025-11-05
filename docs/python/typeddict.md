# TypedDict - 구조화된 딕셔너리 타입

## 개요

TypedDict는 딕셔너리의 키와 값 타입을 명시적으로 정의하는 타입입니다. LangGraph에서는 그래프의 상태(State)를 정의할 때 필수적으로 사용됩니다.

```python
from typing_extensions import TypedDict
```

## 기본 사용법

### 클래스 기반 문법 (권장)

```python
from typing_extensions import TypedDict

class User(TypedDict):
    name: str
    age: int
    email: str

# 사용
user: User = {
    "name": "Alice",
    "age": 30,
    "email": "alice@example.com"
}
```

### 함수형 문법

```python
# 클래스 문법과 동일하지만 덜 직관적
User = TypedDict("User", {"name": str, "age": int, "email": str})
```

## 선택적 필드 (Optional Fields)

### total=False 사용

```python
from typing_extensions import TypedDict

# 모든 필드가 선택적
class OptionalUser(TypedDict, total=False):
    name: str
    age: int
    email: str

# 일부 필드만 있어도 OK
user1: OptionalUser = {"name": "Alice"}
user2: OptionalUser = {"name": "Bob", "age": 25}
```

### Required와 NotRequired

```python
from typing_extensions import TypedDict, Required, NotRequired

# 기본적으로 모든 필드 선택적 (total=False)
class User(TypedDict, total=False):
    name: Required[str]      # 필수 필드
    age: Required[int]       # 필수 필드
    email: NotRequired[str]  # 선택적 필드
    phone: str               # 선택적 필드 (total=False이므로)

# 올바른 사용
user: User = {"name": "Alice", "age": 30}

# 오류: name과 age는 필수
user_invalid: User = {"email": "alice@example.com"}  # Type error
```

## LangGraph State 정의

### 기본 State

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph

class State(TypedDict):
    messages: list[str]
    user_input: str
    counter: int

# StateGraph에 State 타입 전달
graph_builder = StateGraph(State)
```

### 복잡한 State

```python
from typing_extensions import TypedDict
from typing import Literal

class Message(TypedDict):
    role: Literal["user", "assistant", "system"]
    content: str

class State(TypedDict):
    messages: list[Message]
    user_id: str
    session_id: str
    metadata: dict[str, str]
    is_complete: bool
```

## 상속

TypedDict는 다른 TypedDict를 상속할 수 있습니다.

```python
from typing_extensions import TypedDict

class BaseState(TypedDict):
    session_id: str
    created_at: str

class ExtendedState(BaseState):
    messages: list[str]
    user_input: str

# ExtendedState는 모든 필드를 포함
state: ExtendedState = {
    "session_id": "abc123",
    "created_at": "2024-01-01",
    "messages": [],
    "user_input": "Hello"
}
```

## LangGraph의 Multiple Schemas 패턴

### Input/Output/Private State 분리

```python
from typing_extensions import TypedDict

# 외부에서 받는 입력 스키마
class InputState(TypedDict):
    user_input: str

# 외부로 반환하는 출력 스키마
class OutputState(TypedDict):
    response: str

# 내부에서만 사용하는 전체 상태
class PrivateState(TypedDict):
    user_input: str
    response: str
    internal_data: list[str]  # 외부에 노출되지 않음
    processing_step: int      # 외부에 노출되지 않음

# StateGraph 설정
from langgraph.graph import StateGraph

graph_builder = StateGraph(
    state_schema=PrivateState,
    input=InputState,
    output=OutputState
)
```

### 작동 방식

```python
# 1. 입력: InputState 스키마만 허용
result = graph.invoke({"user_input": "Hello"})

# 2. 내부: PrivateState로 처리
def process_node(state: PrivateState) -> dict:
    return {
        "response": f"You said: {state['user_input']}",
        "internal_data": ["log1", "log2"],
        "processing_step": 1
    }

# 3. 출력: OutputState 스키마만 반환
print(result)  # {"response": "You said: Hello"}
```

## 중첩된 TypedDict

```python
from typing_extensions import TypedDict

class Address(TypedDict):
    street: str
    city: str
    zip_code: str

class Person(TypedDict):
    name: str
    age: int
    address: Address  # 중첩된 TypedDict

# 사용
person: Person = {
    "name": "Alice",
    "age": 30,
    "address": {
        "street": "123 Main St",
        "city": "Seoul",
        "zip_code": "12345"
    }
}
```

## 제네릭 TypedDict

```python
from typing_extensions import TypedDict
from typing import Generic, TypeVar

T = TypeVar("T")

class Response(TypedDict, Generic[T]):
    status: str
    data: T

class UserData(TypedDict):
    name: str
    age: int

# 사용
response: Response[UserData] = {
    "status": "success",
    "data": {"name": "Alice", "age": 30}
}
```

## 타입 체크와 런타임 검증

### 타입 체커 (정적 분석)

```python
from typing_extensions import TypedDict

class State(TypedDict):
    count: int
    name: str

# 타입 체커는 이 오류를 감지
state: State = {"count": "not a number"}  # Type error

# 런타임에서는 오류가 발생하지 않음 (주의!)
state_runtime = {"count": "not a number"}  # OK at runtime
```

### 런타임 검증 (pydantic 활용)

```python
from pydantic import BaseModel

# TypedDict와 유사하지만 런타임 검증 포함
class State(BaseModel):
    count: int
    name: str

# 런타임 오류 발생
try:
    state = State(count="not a number", name="test")
except ValueError as e:
    print(e)  # validation error
```

## 실전 예제: LangGraph Chatbot State

```python
from typing_extensions import TypedDict, NotRequired
from typing import Literal
from datetime import datetime

class Message(TypedDict):
    role: Literal["user", "assistant"]
    content: str
    timestamp: datetime

class ChatState(TypedDict):
    # 필수 필드
    messages: list[Message]
    session_id: str

    # 선택적 필드
    user_name: NotRequired[str]
    context: NotRequired[dict[str, str]]
    error: NotRequired[str]

# 노드 함수
def chat_node(state: ChatState) -> dict:
    messages = state["messages"]
    user_name = state.get("user_name", "User")

    # 새 메시지 추가
    new_message: Message = {
        "role": "assistant",
        "content": f"Hello {user_name}!",
        "timestamp": datetime.now()
    }

    messages.append(new_message)
    return {"messages": messages}
```

## 베스트 프랙티스

1. **명확한 필드명 사용**: `msg` 대신 `messages`, `usr` 대신 `user_name`
2. **선택적 필드 명시**: `NotRequired` 또는 `total=False` 활용
3. **중첩 구조 분리**: 복잡한 구조는 별도 TypedDict로 정의
4. **Literal로 값 제한**: 문자열 상수는 Literal 사용
5. **문서화**: 복잡한 State는 주석으로 설명 추가

## typing vs typing_extensions

```python
# Python 3.8+
from typing_extensions import TypedDict  # 권장

# Python 3.8 미만
from typing import TypedDict  # 기본 기능만 제공

# typing_extensions 장점
# - 최신 기능 백포트 (Required, NotRequired 등)
# - 여러 Python 버전 호환성
# - LangGraph 공식 예제에서 사용
```

## 참고 자료

- [PEP 589 - TypedDict](https://peps.python.org/pep-0589/)
- [typing_extensions Documentation](https://typing-extensions.readthedocs.io/)
- [LangGraph State Management](https://langchain-ai.github.io/langgraph/concepts/low_level/#state)
