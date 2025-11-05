# 01. LangGraph Chatbot

## 개요

LangGraph의 가장 기본적인 챗봇 구현 예제입니다. 상태 관리와 그래프 구조를 사용하여 LLM과의 대화를 처리하는 방법을 학습합니다.

## 핵심 개념

### 1. MessagesState

```python
from langgraph.graph.message import MessagesState

class State(MessagesState):
    custom_stuff: str
```

- `MessagesState`: 메시지 기반 상태 관리를 위한 기본 상태 클래스
- 대화 메시지를 자동으로 관리하는 `messages` 필드 제공
- 커스텀 필드 추가 가능 (예: `custom_stuff`)

### 2. StateGraph 생성

```python
from langgraph.graph import StateGraph, START, END

graph_builder = StateGraph(State)
```

- `StateGraph`: 상태 기반 워크플로우를 구축하는 그래프 빌더
- `State` 타입으로 그래프 전체에서 사용할 상태 구조 정의

### 3. 챗봇 노드 정의

```python
def chatbot(state: State):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}
```

**노드 함수의 특징:**
- 입력: 현재 `State` 객체
- 출력: 상태 업데이트를 위한 딕셔너리 (부분 업데이트 또는 빈 딕셔너리 가능)
- `state["messages"]`로 대화 히스토리 접근
- LLM 응답을 메시지 리스트에 추가하여 반환

### 4. 그래프 구성

```python
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)

graph = graph_builder.compile()
```

**그래프 구조:**
```
START → chatbot → END
```

- `add_node()`: 노드 추가 (이름, 함수)
- `add_edge()`: 노드 간 연결 정의
- `compile()`: 실행 가능한 그래프로 컴파일

## 실행 방법

```python
result = graph.invoke({
    "messages": [
        {"role": "user", "content": "how are you?"},
    ]
})
```

**실행 결과:**
- 입력 메시지와 LLM 응답이 포함된 상태 객체 반환
- `result["messages"]`에서 전체 대화 히스토리 확인 가능

## 주요 학습 포인트

1. **상태 관리**: `MessagesState`를 통한 대화 상태 자동 관리
2. **그래프 구조**: 단순한 선형 흐름 (START → 노드 → END)
3. **노드 함수**: 상태를 받아 상태 업데이트를 반환하는 패턴
4. **LLM 통합**: `llm.invoke()`로 언어 모델 호출

## 다음 단계

- [02. Tool Nodes](./02-tool-nodes.md): 도구 사용 기능 추가
- [03. Memory](./03-memory.md): 대화 상태 영속화

## 관련 코드

- 소스 파일: `agents/01_LangGraph Chatbot.ipynb`
