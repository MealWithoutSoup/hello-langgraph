# Python 모듈과 패키지

## 개요

Python의 모듈화 시스템은 코드를 논리적 단위로 구성하여 재사용성과 유지보수성을 높입니다. LangGraph 프로젝트에서는 노드, 상태, 그래프를 모듈로 분리하여 관리합니다.

## 모듈 (Module)

모듈은 Python 코드가 포함된 단일 `.py` 파일입니다.

### 모듈 생성

```python
# math_utils.py
def add(a: int, b: int) -> int:
    return a + b

def multiply(a: int, b: int) -> int:
    return a * b

PI = 3.14159
```

### 모듈 import

```python
# main.py
import math_utils

result = math_utils.add(5, 3)
print(math_utils.PI)
```

### 특정 항목만 import

```python
from math_utils import add, PI

result = add(5, 3)
print(PI)
```

### 별칭(alias) 사용

```python
import math_utils as mu
from math_utils import add as addition

result = mu.multiply(4, 5)
result2 = addition(1, 2)
```

## 패키지 (Package)

패키지는 `__init__.py` 파일을 포함하는 디렉토리로, 여러 모듈을 그룹화합니다.

### 기본 패키지 구조

```
my_package/
├── __init__.py
├── module_a.py
└── module_b.py
```

### `__init__.py`의 역할

#### 1. 패키지 표시 (Python 3.3+에서는 선택사항)

```python
# my_package/__init__.py
# 빈 파일이어도 패키지로 인식됨
```

#### 2. 패키지 초기화

```python
# my_package/__init__.py
print("Package initialized")

# 패키지 수준 변수
VERSION = "1.0.0"
```

#### 3. 편리한 import 제공

```python
# my_package/__init__.py
from .module_a import function_a
from .module_b import function_b

__all__ = ["function_a", "function_b"]
```

```python
# main.py
# 이제 이렇게 간단하게 사용 가능
from my_package import function_a, function_b

# 대신에
# from my_package.module_a import function_a
# from my_package.module_b import function_b
```

## LangGraph 프로젝트 구조 예제

### 디렉토리 구조

```
langgraph_project/
├── __init__.py
├── state.py           # State 정의
├── nodes/
│   ├── __init__.py
│   ├── chat.py        # Chat 노드
│   └── tools.py       # Tool 노드
├── graphs/
│   ├── __init__.py
│   └── main_graph.py  # 메인 그래프
└── utils/
    ├── __init__.py
    └── helpers.py     # 헬퍼 함수
```

### state.py

```python
# state.py
from typing_extensions import TypedDict
from typing import Annotated
import operator

class ChatState(TypedDict):
    messages: Annotated[list[str], operator.add]
    user_input: str
    turn_count: int
```

### nodes/chat.py

```python
# nodes/chat.py
from ..state import ChatState

def chat_node(state: ChatState) -> dict:
    """채팅 응답 생성"""
    messages = state["messages"]
    user_input = state["user_input"]

    response = f"You said: {user_input}"
    messages.append(response)

    return {
        "messages": messages,
        "turn_count": state.get("turn_count", 0) + 1
    }
```

### nodes/\_\_init\_\_.py

```python
# nodes/__init__.py
from .chat import chat_node
from .tools import tool_node

__all__ = ["chat_node", "tool_node"]
```

### graphs/main_graph.py

```python
# graphs/main_graph.py
from langgraph.graph import StateGraph, START, END
from ..state import ChatState
from ..nodes import chat_node, tool_node

def create_graph():
    graph_builder = StateGraph(ChatState)

    graph_builder.add_node("chat", chat_node)
    graph_builder.add_node("tools", tool_node)

    graph_builder.add_edge(START, "chat")
    graph_builder.add_edge("chat", "tools")
    graph_builder.add_edge("tools", END)

    return graph_builder.compile()
```

### 메인 파일에서 사용

```python
# main.py
from langgraph_project.graphs.main_graph import create_graph
from langgraph_project.state import ChatState

graph = create_graph()

initial_state: ChatState = {
    "messages": [],
    "user_input": "Hello",
    "turn_count": 0
}

result = graph.invoke(initial_state)
print(result)
```

## 상대 import vs 절대 import

### 상대 import (Relative Import)

패키지 내부에서 사용합니다.

```python
# nodes/chat.py 안에서
from ..state import ChatState        # 상위 디렉토리
from .tools import tool_node         # 같은 디렉토리
from ..utils.helpers import format_message  # 상위/다른 디렉토리
```

### 절대 import (Absolute Import)

프로젝트 루트부터 전체 경로를 명시합니다.

```python
# 어디서든 사용 가능
from langgraph_project.state import ChatState
from langgraph_project.nodes.chat import chat_node
from langgraph_project.utils.helpers import format_message
```

### 권장사항

- **패키지 내부**: 상대 import 사용 (`.`, `..`)
- **외부에서 패키지 사용**: 절대 import 사용
- **명확성 우선**: 애매한 경우 절대 import

## `__all__` 리스트

public API를 명시적으로 정의합니다.

```python
# nodes/__init__.py
from .chat import chat_node, internal_helper
from .tools import tool_node

# public API만 노출
__all__ = ["chat_node", "tool_node"]
# internal_helper는 from nodes import *에서 제외됨
```

```python
# main.py
from nodes import *  # chat_node, tool_node만 import됨

# internal_helper는 import되지 않음
# 명시적으로 import 필요:
from nodes.chat import internal_helper
```

## 모듈 실행 vs Import

### `__name__` 변수

```python
# module.py
def main():
    print("Running main function")

if __name__ == "__main__":
    # 이 파일이 직접 실행될 때만 실행됨
    main()
else:
    # 다른 파일에서 import될 때는 실행되지 않음
    pass
```

```bash
# 직접 실행
python module.py  # "Running main function" 출력

# import로 사용
# other_file.py
import module  # main() 실행되지 않음
```

### LangGraph 그래프 테스트 예제

```python
# graphs/chat_graph.py
from langgraph.graph import StateGraph, START, END
from ..state import ChatState
from ..nodes import chat_node

def create_chat_graph():
    graph_builder = StateGraph(ChatState)
    graph_builder.add_node("chat", chat_node)
    graph_builder.add_edge(START, "chat")
    graph_builder.add_edge("chat", END)
    return graph_builder.compile()

if __name__ == "__main__":
    # 테스트 코드: 이 파일을 직접 실행할 때만 동작
    graph = create_chat_graph()
    test_state: ChatState = {
        "messages": [],
        "user_input": "Test message",
        "turn_count": 0
    }
    result = graph.invoke(test_state)
    print(f"Test result: {result}")
```

## 순환 import 방지

### 문제 상황

```python
# module_a.py
from module_b import function_b

def function_a():
    return function_b()

# module_b.py
from module_a import function_a  # 순환 import!

def function_b():
    return function_a()
```

### 해결 방법 1: Import 위치 변경

```python
# module_a.py
def function_a():
    from module_b import function_b  # 함수 내부로 이동
    return function_b()
```

### 해결 방법 2: 구조 재설계

```python
# common.py - 공통 기능 분리
def shared_function():
    return "shared"

# module_a.py
from common import shared_function

# module_b.py
from common import shared_function
```

## 네임스페이스 패키지

`__init__.py` 없이 여러 위치의 패키지를 하나로 통합합니다 (PEP 420).

```
# 디렉토리 1
project/
└── my_package/
    └── module_a.py

# 디렉토리 2
plugins/
└── my_package/
    └── module_b.py

# 둘 다 import 가능
from my_package import module_a
from my_package import module_b
```

## 베스트 프랙티스

### 1. 명확한 디렉토리 구조

```
project/
├── state/          # 상태 정의
├── nodes/          # 노드 함수들
├── graphs/         # 그래프 구성
├── tools/          # 유틸리티 도구
└── tests/          # 테스트 코드
```

### 2. `__init__.py`로 간단한 API 제공

```python
# my_package/__init__.py
from .core import main_function
from .utils import helper_function

__version__ = "1.0.0"
__all__ = ["main_function", "helper_function"]
```

### 3. 타입 힌트와 함께 사용

```python
# nodes/__init__.py
from .chat import chat_node
from .tools import tool_node
from typing import Callable

NodeFunction = Callable[[dict], dict]

__all__ = ["chat_node", "tool_node", "NodeFunction"]
```

### 4. 문서화

```python
# my_package/__init__.py
"""
LangGraph Chat Package

이 패키지는 채팅 기능을 제공합니다.

Modules:
    - state: ChatState 정의
    - nodes: Chat 노드 함수들
    - graphs: 그래프 구성
"""

from .state import ChatState
from .nodes import chat_node
from .graphs import create_graph

__all__ = ["ChatState", "chat_node", "create_graph"]
__version__ = "1.0.0"
```

## 실전 예제: LangGraph 모듈화

### 완전한 프로젝트 구조

```
langgraph_chatbot/
├── __init__.py
├── state/
│   ├── __init__.py
│   └── chat_state.py
├── nodes/
│   ├── __init__.py
│   ├── chat.py
│   ├── tools.py
│   └── router.py
├── graphs/
│   ├── __init__.py
│   └── chat_graph.py
├── utils/
│   ├── __init__.py
│   └── formatters.py
└── main.py
```

### state/chat_state.py

```python
from typing_extensions import TypedDict
from typing import Annotated
import operator

class ChatState(TypedDict):
    messages: Annotated[list[str], operator.add]
    user_input: str
    turn_count: int
```

### state/\_\_init\_\_.py

```python
from .chat_state import ChatState

__all__ = ["ChatState"]
```

### 사용

```python
# main.py
from langgraph_chatbot.state import ChatState
from langgraph_chatbot.graphs import create_chat_graph

graph = create_chat_graph()
state: ChatState = {"messages": [], "user_input": "Hi", "turn_count": 0}
result = graph.invoke(state)
```

## 참고 자료

- [Python Modules Documentation](https://docs.python.org/3/tutorial/modules.html)
- [PEP 420 - Namespace Packages](https://peps.python.org/pep-0420/)
- [Python Packaging Guide](https://packaging.python.org/)
