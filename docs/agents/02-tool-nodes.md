# 02. Tool Nodes

## 개요

LangGraph에서 도구(Tool)를 사용하는 방법을 학습합니다. LLM이 외부 함수를 호출하여 정보를 가져오거나 작업을 수행할 수 있게 합니다.

## 핵심 개념

### 1. 도구(Tool) 정의

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str):
    """Gets weather in city"""
    return f"The weather in {city} is sunny."
```

**도구의 구성 요소:**
- `@tool` 데코레이터: 함수를 LangChain 도구로 변환
- 함수 독스트링: LLM이 도구의 목적을 이해하는 데 사용
- 타입 힌트: LLM이 올바른 인자 형식을 알 수 있게 함
- 반환값: 도구 실행 결과 (문자열, 딕셔너리 등)

### 2. LLM에 도구 바인딩

```python
llm_with_tools = llm.bind_tools(tools=[get_weather])
```

- `bind_tools()`: LLM에게 사용 가능한 도구 목록 제공
- LLM은 필요 시 도구 호출 결정을 자율적으로 수행

### 3. ToolNode 생성

```python
from langgraph.prebuilt import ToolNode

tool_node = ToolNode(tools=[get_weather])
```

- `ToolNode`: 도구 호출을 처리하는 사전 구성된 노드
- LLM의 도구 호출 요청을 실제 함수 실행으로 변환
- 실행 결과를 `ToolMessage`로 반환

### 4. 조건부 라우팅 (tools_condition)

```python
from langgraph.prebuilt import tools_condition

graph_builder.add_conditional_edges("chatbot", tools_condition)
```

**`tools_condition`의 역할:**
- LLM 응답을 검사하여 도구 호출이 필요한지 판단
- 도구 호출이 필요하면 → `"tools"` 노드로 라우팅
- 도구 호출이 없으면 → `END`로 라우팅

## 그래프 구조

```python
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", tool_node)

graph_builder.add_edge(START, "chatbot")
graph_builder.add_conditional_edges("chatbot", tools_condition)
graph_builder.add_edge("tools", "chatbot")
```

**실행 흐름:**
```
START → chatbot → [tools_condition]
                      ↓
                   tools → chatbot → END
```

**순환 패턴:**
1. `chatbot` 노드에서 LLM이 도구 호출 결정
2. 도구가 필요하면 `tools` 노드로 이동
3. 도구 실행 결과를 받아 다시 `chatbot`으로 돌아감
4. LLM이 최종 응답 생성 후 종료

## 실행 예제

```python
result = graph.invoke({
    "messages": [
        {"role": "user", "content": "what is the weather in machupichu"},
    ]
})
```

**실행 과정:**
1. 사용자 메시지: "what is the weather in machupichu"
2. LLM이 `get_weather` 도구 호출 결정
3. `tool_node`에서 `get_weather("machupichu")` 실행
4. 결과: "The weather in machupichu is sunny."
5. LLM이 자연스러운 응답으로 변환하여 반환

## 메시지 흐름

```python
[
    HumanMessage(content="what is the weather in machupichu"),
    AIMessage(content="", tool_calls=[...]),  # 도구 호출 요청
    ToolMessage(content="The weather in machupichu is sunny."),  # 도구 결과
    AIMessage(content="The weather in Machu Picchu is sunny!")  # 최종 응답
]
```

## 주요 학습 포인트

1. **도구 정의**: `@tool` 데코레이터와 명확한 독스트링
2. **도구 바인딩**: `bind_tools()`로 LLM에 도구 제공
3. **자동 실행**: `ToolNode`가 도구 호출 자동 처리
4. **조건부 라우팅**: `tools_condition`으로 흐름 제어
5. **순환 패턴**: 도구 → LLM → 도구 반복 가능

## 확장 가능성

### 여러 도구 사용

```python
@tool
def get_weather(city: str):
    """Gets weather in city"""
    return f"The weather in {city} is sunny."

@tool
def get_time(timezone: str):
    """Gets current time in timezone"""
    return f"The time in {timezone} is 3:00 PM."

llm_with_tools = llm.bind_tools(tools=[get_weather, get_time])
tool_node = ToolNode(tools=[get_weather, get_time])
```

### 복잡한 도구

```python
@tool
def search_database(query: str, filters: dict):
    """Searches database with query and filters"""
    # 실제 데이터베이스 검색 로직
    return {"results": [...]}
```

## 다음 단계

- [03. Memory](./03-memory.md): 대화 히스토리 저장
- [04. Human in the Loop](./04-human-in-the-loop.md): 사람 개입 패턴

## 관련 코드

- 소스 파일: `agents/02_Tool_Nodes.ipynb`
