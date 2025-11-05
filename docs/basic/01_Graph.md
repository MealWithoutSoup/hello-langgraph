# LangGraph 기본 구조 (Graph)

## 개요
LangGraph의 가장 기본적인 그래프 구조를 정의하고 실행하는 방법을 학습합니다.

## 핵심 개념

### 1. StateGraph
- `StateGraph`는 LangGraph의 핵심 클래스로, 상태 기반 워크플로우를 정의합니다
- 모든 노드는 공통 상태(State)를 공유하며, 이를 통해 데이터를 전달합니다

### 2. State 정의
```python
from typing_extensions import TypedDict

class State(TypedDict):
    hello: str
```
- `TypedDict`를 사용하여 그래프 전체에서 공유할 상태의 구조를 정의합니다
- 각 필드는 타입 힌트를 가지며, 타입 안정성을 제공합니다

### 3. 그래프 구조
```python
graph_builder = StateGraph(State)
```
- `StateGraph` 인스턴스를 생성하여 그래프를 구축합니다
- State 타입을 전달하여 그래프가 다룰 데이터 구조를 지정합니다

## 노드(Node) 정의

### 노드 함수
```python
def node_one(state: State):
    print("node_one")

def node_two(state: State):
    print("node_two")

def node_three(state: State):
    print("node_three")
```

**특징:**
- 각 노드는 `State`를 매개변수로 받는 함수입니다
- 노드 함수는 상태를 읽을 수 있지만, 이 예제에서는 단순히 실행만 됩니다
- 반환값이 없어도 정상 동작합니다 (상태 변경 없음)

## 그래프 구성

### 노드 추가
```python
graph_builder.add_node("node_one", node_one)
graph_builder.add_node("node_two", node_two)
graph_builder.add_node("node_three", node_three)
```
- `add_node(name, function)`: 노드 이름과 실행할 함수를 연결합니다

### 엣지(Edge) 연결
```python
graph_builder.add_edge(START, "node_one")
graph_builder.add_edge("node_one", "node_two")
graph_builder.add_edge("node_two", "node_three")
graph_builder.add_edge("node_three", END)
```

**엣지 타입:**
- `START`: 그래프의 시작점 (진입점)
- `END`: 그래프의 종료점 (출구)
- 노드 간 연결: 문자열 이름으로 순차적 실행 순서를 정의합니다

**실행 흐름:**
```
START → node_one → node_two → node_three → END
```

## 그래프 컴파일 및 실행

### 컴파일
```python
graph = graph_builder.compile()
```
- `compile()`: 그래프 빌더를 실행 가능한 그래프 객체로 변환합니다
- 컴파일 후에는 그래프 구조를 변경할 수 없습니다

### 실행
```python
graph.invoke({"hello": "world"})
```
- `invoke(initial_state)`: 초기 상태와 함께 그래프를 실행합니다
- 그래프는 정의된 순서대로 노드를 실행합니다

## 실행 결과
```
node_one
node_two
node_three
```
- 각 노드가 순차적으로 실행되며 콘솔에 출력됩니다

## 주요 학습 포인트

1. **선형 실행 구조**: 이 예제는 가장 단순한 선형 실행 흐름을 보여줍니다
2. **상태 공유**: 모든 노드가 동일한 State 객체에 접근 가능합니다
3. **명시적 흐름**: START와 END를 사용하여 실행 시작과 종료를 명확히 합니다
4. **빌더 패턴**: `StateGraph`는 빌더 패턴을 사용하여 직관적인 그래프 구성을 지원합니다

## 다음 단계
- `02_State.md`: 노드에서 상태를 수정하고 전달하는 방법 학습
- 조건부 분기, 병렬 실행 등 고급 그래프 패턴 탐색
