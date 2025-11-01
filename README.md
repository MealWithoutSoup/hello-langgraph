# LangGraph Basic Concepts Repository

LangGraph의 핵심 개념과 패턴을 학습하기 위한 Jupyter 노트북 모음입니다.

## 의존성

- **langgraph 0.6.6**: 그래프 워크플로우 프레임워크
- **langchain[openai] 0.3.27**: OpenAI 통합
- **langgraph-checkpoint-sqlite 2.0.11**: 상태 영속화
- **python-dotenv 1.1.1**: 환경 변수 관리

## 각 노트북 설명

### 1. Command.ipynb
**핵심 개념**: Command API를 사용한 명시적 노드 라우팅

노드 함수 내에서 `Command` 객체를 반환하여 다음에 실행할 노드를 명시적으로 지정합니다.

```python
def triage_node(state: State) -> Command[Literal["account_support", "tech_support"]]:
    return Command(
        goto="account_support",  # 명시적으로 다음 노드 지정
        update={"transfer_reason": "비밀번호 변경 요청"}
    )
```

**활용**: 복잡한 라우팅 로직을 노드 내부에서 처리할 때 유용

---

### 2. Conditional Edge.ipynb
**핵심 개념**: 조건부 엣지를 사용한 동적 라우팅

상태 값에 따라 다음 노드를 동적으로 결정하는 decision 함수를 정의합니다.

```python
def decide_path(state: State):
    return state["seed"] % 2 == 0  # True/False 반환

graph_builder.add_conditional_edges(
    "node_two",
    decide_path,
    {
        True: "node_three",   # True면 node_three로
        False: "node_four",   # False면 node_four로
    }
)
```

**활용**: 상태 기반 분기 로직, if-else 패턴 구현

---

### 3. Multiple Schemas.ipynb
**핵심 개념**: Input/Output/Private 스키마 분리

그래프의 입출력 인터페이스를 제어하고 내부 상태를 캡슐화합니다.

```python
class PrivateState(TypedDict):  # 내부 전용
    a: int
    b: int

class InputState(TypedDict):    # 입력 인터페이스
    hello: str

class OutputState(TypedDict):   # 출력 인터페이스
    bye: str

graph_builder = StateGraph(
    PrivateState,
    input_schema=InputState,
    output_schema=OutputState
)
```

**주요 특징**:
- `InputState`에 없는 필드는 그래프 입력 시 무시됨
- `OutputState`에 없는 필드는 최종 결과에 포함되지 않음
- 노드 간 상태 전달은 `PrivateState` 기준

**활용**: API 인터페이스 설계, 내부 상태 은닉

---

### 4. Node Caching.ipynb
**핵심 개념**: 노드 결과 캐싱으로 성능 최적화

동일한 입력에 대해 TTL(Time To Live) 내에서는 노드 재실행을 생략합니다.

```python
graph_builder.add_node(
    "node_two",
    node_two,
    cache_policy=CachePolicy(ttl=20)  # 20초 캐싱
)

graph = graph_builder.compile(cache=InMemoryCache())
```

**실행 결과**:
```
{'time': '2025-10-29 20:28:02.731822'}  # 최초 실행
{'time': '2025-10-29 20:28:02.731822'}  # 캐시 사용 (동일 시간)
{'time': '2025-10-29 20:28:02.731822'}  # 캐시 사용
{'time': '2025-10-29 20:28:22.749146'}  # TTL 만료 후 재실행
```

**활용**: 비용이 큰 API 호출, 데이터베이스 쿼리 최적화

---

### 5. Reducer Functions.ipynb
**핵심 개념**: Reducer 함수를 사용한 상태 누적

`Annotated` 타입과 reducer 함수로 상태 업데이트 방식을 제어합니다.

```python
class State(TypedDict):
    messages: Annotated[list[str], operator.add]  # 리스트 병합

def node_one(state: State):
    return {"messages": ["Hello, nice to meet you!"]}

# 실행 결과
graph.invoke({"messages": ["Hello!"]})
# → {"messages": ["Hello!", "Hello, nice to meet you!"]}
```

**기본 동작 vs Reducer**:
- **기본**: 새 값으로 덮어쓰기 (`{"messages": [1, 2]}` → `[1, 2]`)
- **operator.add**: 기존 값에 추가 (`["a"] + ["b"]` → `["a", "b"]`)

**활용**: 메시지 히스토리 누적, 이벤트 로그 수집

---

### 6. SendAPI.ipynb
**핵심 개념**: Send API를 사용한 병렬 실행 (fan-out 패턴)

하나의 노드에서 여러 개의 병렬 작업을 생성합니다.

```python
def dispatcher(state: State):
    # 각 단어마다 node_two를 병렬 실행
    return [Send("node_two", word) for word in state["words"]]

graph_builder.add_conditional_edges(
    "node_one",
    dispatcher,
    ["node_two"]  # 병렬 실행 대상 노드
)
```

**실행 흐름**:
```
node_one
   ├─→ node_two("hello")   ┐
   ├─→ node_two("world")   ├─ 병렬 실행
   ├─→ node_two("how")     │
   ├─→ node_two("are")     │
   ├─→ node_two("you")     │
   └─→ node_two("doing")   ┘
         ↓
   결과 병합 (reducer 사용)
```

**활용**:
- 배치 처리 (각 아이템에 대해 동일한 작업)
- 맵리듀스 패턴
- 병렬 API 호출

---

## LangGraph 기본 구조

모든 노트북은 다음 패턴을 따릅니다:

```python
# 1. State 정의
class State(TypedDict):
    field: type

# 2. Graph Builder 생성
graph_builder = StateGraph(State)

# 3. 노드 정의 및 추가
def node_function(state: State):
    return {}  # 상태 업데이트

graph_builder.add_node("node_name", node_function)

# 4. 엣지 연결
graph_builder.add_edge(START, "node_name")
graph_builder.add_edge("node_name", END)

# 5. 컴파일 및 실행
graph = graph_builder.compile()
result = graph.invoke({"initial": "state"})
```
