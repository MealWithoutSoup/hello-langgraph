# Python 문법 가이드 for LangGraph

LangGraph를 효과적으로 사용하기 위해 필요한 Python 문법과 개념들을 정리한 가이드입니다.

## 📚 문서 목록

### 1. [타입 힌팅 기초 (typing_basics.md)](./typing_basics.md)

Python의 타입 힌팅 시스템과 `typing` 모듈의 기본 사용법을 다룹니다.

**주요 내용:**
- 기본 타입 힌트 (`str`, `int`, `float`, `bool`)
- 컨테이너 타입 (`List`, `Dict`, `list`, `dict`)
- `Optional`과 `Union` 타입
- `Literal` 타입 (LangGraph 라우팅에 필수)
- `Callable` 타입
- 타입 체킹 도구 (mypy, pyright)

**LangGraph 연관성:**
- 노드 함수 시그니처
- 조건부 라우팅 반환 타입
- Command 타입 힌팅

---

### 2. [TypedDict (typeddict.md)](./typeddict.md)

구조화된 딕셔너리 타입 정의 방법을 다룹니다. LangGraph의 State 정의에 핵심입니다.

**주요 내용:**
- TypedDict 기본 사용법
- 선택적 필드 (`total=False`, `Required`, `NotRequired`)
- TypedDict 상속
- Multiple Schemas 패턴 (Input/Output/Private State)
- 중첩된 TypedDict
- 타입 체크 vs 런타임 검증

**LangGraph 연관성:**
- State 정의
- Input/Output 스키마 분리
- 노드 함수 파라미터 타입

---

### 3. [Annotated (annotated.md)](./annotated.md)

타입에 메타데이터를 추가하는 `Annotated`와 Reducer 함수 패턴을 다룹니다.

**주요 내용:**
- Annotated 기본 문법
- Reducer 함수 개념
- `operator.add`를 사용한 리스트 누적
- 커스텀 Reducer 함수 작성
- 안전한 Reducer 작성 베스트 프랙티스

**LangGraph 연관성:**
- State 필드의 업데이트 동작 제어
- 메시지 리스트 누적
- 상태 병합 로직

---

### 4. [모듈과 패키지 (modules.md)](./modules.md)

Python의 모듈화 시스템과 `__init__.py` 사용법을 다룹니다.

**주요 내용:**
- 모듈과 패키지 개념
- `__init__.py`의 역할
- 상대 import vs 절대 import
- `__all__` 리스트
- `__name__` 변수와 스크립트 실행
- 순환 import 방지
- 프로젝트 구조 베스트 프랙티스

**LangGraph 연관성:**
- 노드, State, 그래프를 모듈로 분리
- 재사용 가능한 컴포넌트 구성
- 대규모 LangGraph 프로젝트 구조화

---

### 5. [operator 모듈 (operators.md)](./operators.md)

Python의 `operator` 모듈과 LangGraph Reducer로 활용하는 방법을 다룹니다.

**주요 내용:**
- `operator.add` 기본 사용법
- `+` 연산자와의 관계
- LangGraph Reducer로 사용
- 다양한 operator 함수들
- operator vs 커스텀 Reducer

**LangGraph 연관성:**
- State 필드의 리스트 누적
- 메시지 히스토리 관리
- 로그 수집

---

## 🎯 학습 순서 추천

### 초급 (LangGraph 시작 단계)

1. **typing_basics.md** - 타입 힌팅의 기초를 이해합니다
2. **typeddict.md** - State 정의 방법을 배웁니다
3. **operators.md** - 기본 Reducer 사용법을 익힙니다

### 중급 (LangGraph 활용 단계)

4. **annotated.md** - 커스텀 Reducer를 작성합니다
5. **modules.md** - 프로젝트를 모듈화합니다

### 실전 프로젝트 구조

```
my_langgraph_project/
├── state/
│   ├── __init__.py
│   └── chat_state.py      # TypedDict 정의
├── nodes/
│   ├── __init__.py
│   ├── chat.py             # 노드 함수들
│   └── tools.py
├── graphs/
│   ├── __init__.py
│   └── main_graph.py       # 그래프 구성
└── main.py                 # 실행 진입점
```

---

## 🔗 각 문법의 LangGraph 적용 예제

### State 정의 (TypedDict + Annotated + operator)

```python
from typing_extensions import TypedDict
from typing import Annotated
import operator

class ChatState(TypedDict):
    # 메시지는 누적 (operator.add)
    messages: Annotated[list[str], operator.add]

    # 사용자 입력은 덮어쓰기
    user_input: str

    # 턴 카운트는 숫자 누적
    turn_count: Annotated[int, operator.add]
```

### 노드 함수 (typing + TypedDict)

```python
from typing import Literal

def chat_node(state: ChatState) -> dict[str, list[str] | int]:
    """타입 힌팅을 통한 명확한 함수 시그니처"""
    messages = state["messages"]
    user_input = state["user_input"]

    response = f"You said: {user_input}"
    return {
        "messages": [response],
        "turn_count": 1
    }

def router(state: ChatState) -> Literal["continue", "end"]:
    """Literal 타입으로 명시적 라우팅"""
    if state["turn_count"] > 10:
        return "end"
    return "continue"
```

### 프로젝트 모듈화 (modules + \_\_init\_\_)

```python
# state/__init__.py
from .chat_state import ChatState
__all__ = ["ChatState"]

# nodes/__init__.py
from .chat import chat_node
from .tools import tool_node
__all__ = ["chat_node", "tool_node"]

# main.py
from state import ChatState
from nodes import chat_node, tool_node
from graphs import create_graph

graph = create_graph()
```

---

## 🛠️ 개발 환경 설정

### 타입 체크 설정

#### mypy 설정 (mypy.ini)

```ini
[mypy]
python_version = 3.13
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
```

#### VS Code 설정 (settings.json)

```json
{
    "python.analysis.typeCheckingMode": "basic",
    "python.linting.mypyEnabled": true,
    "python.linting.enabled": true
}
```

---

## 📖 추가 학습 자료

### 공식 문서
- [Python Typing Documentation](https://docs.python.org/3/library/typing.html)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [typing_extensions](https://typing-extensions.readthedocs.io/)

### PEP (Python Enhancement Proposals)
- [PEP 484 - Type Hints](https://peps.python.org/pep-0484/)
- [PEP 589 - TypedDict](https://peps.python.org/pep-0589/)
- [PEP 593 - Annotated](https://peps.python.org/pep-0593/)

### LangGraph 관련
- [LangGraph State Management](https://langchain-ai.github.io/langgraph/concepts/low_level/#state)
- [LangGraph Reducer Functions](https://langchain-ai.github.io/langgraph/how-tos/state-reducers/)

---

## 💡 팁

1. **타입 체커 활용**: mypy나 pyright를 사용하여 코드 작성 중 타입 오류를 조기에 발견하세요.

2. **단계적 타입 적용**: 처음부터 완벽한 타입을 작성하려 하지 말고, 점진적으로 타입을 추가하세요.

3. **문서화**: 복잡한 State나 Reducer는 docstring으로 설명을 추가하세요.

4. **일관성 유지**: 프로젝트 전체에서 일관된 타입 힌팅 스타일을 사용하세요.

5. **실험**: Jupyter Notebook에서 타입 힌팅을 실험하며 익히세요.

---

## 🤝 기여

오타나 개선 사항을 발견하면 이슈나 PR을 남겨주세요!
