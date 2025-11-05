# Python 타입 힌팅 기초 (Typing Basics)

## 개요

Python 3.5부터 도입된 타입 힌팅(Type Hinting)은 변수, 함수 매개변수, 반환값의 타입을 명시하는 문법입니다. LangGraph에서는 타입 힌팅이 필수적이며, 특히 상태(State) 정의와 노드 함수 시그니처에서 중요합니다.

## typing 모듈

```python
from typing import List, Dict, Optional, Union, Any, Callable
```

### 기본 타입 힌트

```python
# 변수 타입 힌팅
name: str = "Alice"
age: int = 30
score: float = 95.5
is_active: bool = True

# 함수 타입 힌팅
def greet(name: str) -> str:
    return f"Hello, {name}!"

def add(a: int, b: int) -> int:
    return a + b
```

## 컨테이너 타입

### List (리스트)

```python
from typing import List

# Python 3.9 이전
numbers: List[int] = [1, 2, 3, 4, 5]
names: List[str] = ["Alice", "Bob", "Charlie"]

# Python 3.9 이상 (권장)
numbers: list[int] = [1, 2, 3, 4, 5]
names: list[str] = ["Alice", "Bob", "Charlie"]
```

### Dict (딕셔너리)

```python
from typing import Dict

# Python 3.9 이전
user: Dict[str, int] = {"age": 30, "score": 100}
config: Dict[str, Any] = {"timeout": 30, "enabled": True}

# Python 3.9 이상 (권장)
user: dict[str, int] = {"age": 30, "score": 100}
config: dict[str, Any] = {"timeout": 30, "enabled": True}
```

## Optional과 Union

### Optional (선택적 값)

```python
from typing import Optional

# Optional[T]는 Union[T, None]과 동일
def find_user(user_id: int) -> Optional[str]:
    if user_id == 1:
        return "Alice"
    return None  # None을 반환할 수 있음

# Python 3.10 이상 (권장)
def find_user(user_id: int) -> str | None:
    if user_id == 1:
        return "Alice"
    return None
```

### Union (여러 타입 중 하나)

```python
from typing import Union

# 여러 타입을 허용
def process(value: Union[int, str]) -> str:
    if isinstance(value, int):
        return str(value)
    return value

# Python 3.10 이상 (권장)
def process(value: int | str) -> str:
    if isinstance(value, int):
        return str(value)
    return value
```

## Literal (리터럴 타입)

특정 값만 허용하는 타입입니다. LangGraph의 Command 반환값이나 조건부 라우팅에서 자주 사용됩니다.

```python
from typing import Literal

# 특정 문자열만 허용
def set_mode(mode: Literal["fast", "slow", "auto"]) -> None:
    print(f"Mode set to: {mode}")

set_mode("fast")  # OK
set_mode("medium")  # Type checker 오류

# LangGraph 예제: 조건부 라우팅
def decide_path(state: dict) -> Literal["path_a", "path_b"]:
    if state["condition"]:
        return "path_a"
    return "path_b"
```

## Callable (호출 가능 객체)

함수를 타입으로 명시할 때 사용합니다.

```python
from typing import Callable

# Callable[[입력타입들], 반환타입]
def apply_operation(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

def add(x: int, y: int) -> int:
    return x + y

def multiply(x: int, y: int) -> int:
    return x * y

result1 = apply_operation(add, 5, 3)       # 8
result2 = apply_operation(multiply, 5, 3)  # 15
```

## Any (모든 타입 허용)

타입 체커를 우회하고 모든 타입을 허용합니다. 가급적 사용을 피하고 구체적인 타입을 명시하는 것이 좋습니다.

```python
from typing import Any

def process_data(data: Any) -> Any:
    # 어떤 타입이든 받을 수 있음
    return data
```

## LangGraph에서의 활용 예제

### 노드 함수 타입 힌팅

```python
from typing_extensions import TypedDict
from typing import Literal

class State(TypedDict):
    messages: list[str]
    user_input: str

def chat_node(state: State) -> dict[str, list[str]]:
    """노드 함수는 State를 받고 부분 상태 업데이트를 반환"""
    messages = state["messages"]
    user_input = state["user_input"]

    response = f"You said: {user_input}"
    messages.append(response)

    return {"messages": messages}
```

### 조건부 라우팅 타입 힌팅

```python
def router(state: State) -> Literal["continue", "end"]:
    """조건부 엣지 함수는 Literal 타입으로 경로를 반환"""
    if len(state["messages"]) > 10:
        return "end"
    return "continue"
```

### Command 타입 힌팅

```python
from langgraph.types import Command

def smart_node(state: State) -> Command[Literal["node_a", "node_b"]]:
    """Command를 사용한 명시적 라우팅"""
    if state["condition"]:
        return Command(goto="node_a", update={"status": "processing"})
    return Command(goto="node_b")
```

## 타입 체킹 도구

### mypy

```bash
# 설치
pip install mypy

# 타입 체크 실행
mypy your_script.py
```

### pyright (VS Code 기본)

```json
// settings.json
{
    "python.analysis.typeCheckingMode": "basic"  // or "strict"
}
```

## 베스트 프랙티스

1. **구체적인 타입 사용**: `Any` 대신 구체적인 타입 명시
2. **Python 3.9+ 문법 선호**: `list[str]` > `List[str]`
3. **Python 3.10+ 문법 선호**: `str | None` > `Optional[str]`
4. **Literal로 명시적 값 제한**: 오타 방지와 IDE 자동완성
5. **함수 반환 타입 명시**: 코드 가독성과 유지보수성 향상

## 참고 자료

- [Python Typing Documentation](https://docs.python.org/3/library/typing.html)
- [PEP 484 - Type Hints](https://peps.python.org/pep-0484/)
- [PEP 604 - Union Operator](https://peps.python.org/pep-0604/)
- [mypy Documentation](https://mypy.readthedocs.io/)
