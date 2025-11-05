# 03. Memory (메모리)

## 개요

LangGraph에서 체크포인터(Checkpointer)를 사용하여 대화 상태를 영속화하고, 여러 세션에 걸쳐 대화를 이어갈 수 있는 방법을 학습합니다.

## 핵심 개념

### 1. Checkpointer 설정

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver

conn = sqlite3.connect("memory.db", check_same_thread=False)
memory = SqliteSaver(conn)

graph = graph_builder.compile(checkpointer=memory)
```

**체크포인터의 역할:**
- 각 노드 실행 후 상태를 자동으로 저장
- 대화 히스토리와 상태를 데이터베이스에 영속화
- 세션 재개 및 상태 복원 가능

### 2. Thread ID를 통한 세션 관리

```python
config = {
    "configurable": {
        "thread_id": "5",
    },
}

result = graph.invoke(
    {"messages": [...]},
    config=config,
)
```

**Thread ID의 개념:**
- 각 대화 세션을 구분하는 고유 식별자
- 같은 `thread_id`를 사용하면 이전 대화 이어가기 가능
- 다른 `thread_id`를 사용하면 완전히 새로운 대화 시작

### 3. 상태 히스토리 조회

```python
for state in graph.get_state_history(config):
    print(state.next)
    print(state.values["messages"])
```

**`get_state_history()`의 기능:**
- 특정 스레드의 모든 체크포인트 조회
- 시간 역순으로 상태 반환 (최신 → 과거)
- 각 체크포인트의 상태와 다음 노드 정보 포함

## 메모리 없이 vs 메모리 사용

### 메모리 없는 경우

```python
graph = graph_builder.compile()  # 체크포인터 없음

# 첫 번째 호출
graph.invoke({"messages": [{"role": "user", "content": "My name is Alice"}]})

# 두 번째 호출 - 이전 대화 기억 못함
graph.invoke({"messages": [{"role": "user", "content": "What's my name?"}]})
# AI: "I don't have information about your name."
```

### 메모리 사용

```python
graph = graph_builder.compile(checkpointer=SqliteSaver(conn))

config = {"configurable": {"thread_id": "1"}}

# 첫 번째 호출
graph.invoke(
    {"messages": [{"role": "user", "content": "My name is Alice"}]},
    config=config
)

# 두 번째 호출 - 이전 대화 기억
graph.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    config=config
)
# AI: "Your name is Alice."
```

## 실행 예제

### 여러 도시 날씨 조회

```python
for event in graph.invoke(
    {
        "messages": [{
            "role": "user",
            "content": "what is the weather in berlin, budapest and bratislava.",
        }]
    },
    stream_mode="updates",
    config={"configurable": {"thread_id": "5"}},
):
    print(event)
```

**실행 과정:**
1. LLM이 3개 도시에 대해 `get_weather` 도구 동시 호출
2. 각 도구 결과가 `ToolMessage`로 반환
3. LLM이 모든 결과를 종합하여 최종 응답 생성
4. 모든 상태가 `thread_id: "5"`로 저장됨

### 상태 히스토리 분석

```python
state_history = graph.get_state_history(config)

for state_snapshot in state_history:
    print(state_snapshot.next)          # 다음에 실행될 노드
    print(state_snapshot.values)         # 현재 상태 값
    print(state_snapshot.config)         # 체크포인트 설정
```

**상태 스냅샷 구조:**
```python
StateSnapshot(
    values={'messages': [...]},      # 현재 메시지들
    next=('chatbot',),               # 다음 실행 노드
    config={...},                    # 체크포인트 ID
    metadata={...},                  # 메타데이터
    created_at='...',                # 생성 시간
)
```

## 스트리밍 모드

```python
async for event in graph.astream(
    {"messages": [...]},
    stream_mode="updates",
    config=config,
):
    print(event)
```

**스트리밍의 장점:**
- 각 노드의 실행 결과를 실시간으로 수신
- 긴 작업의 진행 상황 모니터링 가능
- 비동기 처리로 효율적인 리소스 사용

## 주요 학습 포인트

1. **영속화**: `SqliteSaver`로 대화 상태 저장
2. **세션 관리**: `thread_id`로 독립적인 대화 세션 유지
3. **상태 복원**: 이전 대화 이어가기 가능
4. **히스토리 조회**: 모든 체크포인트 접근 가능
5. **스트리밍**: 실시간 노드 실행 모니터링

## Checkpointer 종류

### SqliteSaver (로컬 개발용)

```python
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

conn = sqlite3.connect("memory.db", check_same_thread=False)
checkpointer = SqliteSaver(conn)
```

### PostgresSaver (프로덕션용)

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:password@localhost/db"
)
```

### MemorySaver (테스트용)

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()  # 메모리에만 저장, 재시작 시 소멸
```

## 실전 활용

### 다중 사용자 채팅

```python
# 사용자 A의 대화
graph.invoke(
    {"messages": [...]},
    config={"configurable": {"thread_id": "user_a"}}
)

# 사용자 B의 대화 (독립적)
graph.invoke(
    {"messages": [...]},
    config={"configurable": {"thread_id": "user_b"}}
)
```

### 대화 재개

```python
# 나중에 같은 thread_id로 재개
result = graph.invoke(
    {"messages": [{"role": "user", "content": "계속 이야기해줘"}]},
    config={"configurable": {"thread_id": "user_a"}}
)
```

## 다음 단계

- [04. Human in the Loop](./04-human-in-the-loop.md): 실행 중 사람 개입
- [05. Time Travel](./05-time-travel.md): 과거 상태로 되돌리기

## 관련 코드

- 소스 파일: `agents/03_Memory.ipynb`
