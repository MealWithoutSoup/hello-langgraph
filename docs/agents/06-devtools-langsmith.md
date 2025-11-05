# 06. DevTools - LangSmith

## 개요

LangSmith는 LangChain과 LangGraph 애플리케이션을 위한 통합 개발 도구 플랫폼입니다. 그래프 실행을 추적, 디버깅, 모니터링하고 성능을 분석할 수 있습니다.

## LangSmith란?

LangSmith는 다음과 같은 기능을 제공하는 개발자 플랫폼입니다:

- **실행 추적**: 모든 노드 실행과 상태 변화를 실시간으로 추적
- **디버깅**: 각 단계의 입력/출력 및 중간 상태 검사
- **성능 모니터링**: 실행 시간, 토큰 사용량, 비용 분석
- **에러 추적**: 예외 및 실패 지점 자동 캡처
- **테스트 관리**: 다양한 입력에 대한 테스트 케이스 관리
- **협업**: 팀원과 추적 데이터 공유

## 설정 방법

### 1. LangSmith 계정 생성

1. [smith.langchain.com](https://smith.langchain.com/)에서 계정 생성
2. API 키 발급 (Settings → API Keys)

### 2. 환경 변수 설정

```bash
# .env 파일 또는 환경 변수에 추가
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your_api_key_here
LANGCHAIN_PROJECT=my-project-name
```

**환경 변수 설명:**
- `LANGCHAIN_TRACING_V2`: LangSmith 추적 활성화
- `LANGCHAIN_API_KEY`: 인증을 위한 API 키
- `LANGCHAIN_PROJECT`: 프로젝트 이름 (추적 그룹화)

### 3. Python 코드에서 설정

```python
import os

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your_api_key"
os.environ["LANGCHAIN_PROJECT"] = "my-project"
```

## 예제 코드 분석

### 기본 그래프 구조

```python
import sqlite3
from langgraph.graph import StateGraph, START, END
from langgraph.graph import MessagesState
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_core.tools import tool
from langchain.chat_models import init_chat_model
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.types import interrupt
```

### 도구 정의

```python
@tool
def get_human_feedback(poem: str):
    """
    Get human feedback on a poem.
    Use this to get feedback on a poem.
    The user will tell you if the poem is ready or if it needs more work.
    """
    response = interrupt({"poem": poem})
    return response["feedback"]
```

**LangSmith에서 보이는 정보:**
- 도구 이름: `get_human_feedback`
- 입력 파라미터: `poem` (string)
- 실행 시간 및 결과
- `interrupt()` 호출 지점 표시

### LLM 및 그래프 설정

```python
tools = [get_human_feedback]

llm = init_chat_model("openai:gpt-5-nano")
llm_with_tools = llm.bind_tools(tools)

class State(MessagesState):
    pass

def chatbot(state: State) -> State:
    response = llm_with_tools.invoke(f"""
    You are an expert at making poems.

    You are given a topic and need to write a poem about it.

    Use the `get_human_feedback` tool to get feedback on your poem.

    Only after the user says the poem is ready, you should return the poem.

    Here is the conversation history:
    {state['messages']}
    """)
    return {"messages": [response]}
```

### 그래프 컴파일

```python
tool_node = ToolNode(tools=tools)

graph_builder = StateGraph(State)
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", tool_node)

graph_builder.add_edge(START, "chatbot")
graph_builder.add_conditional_edges("chatbot", tools_condition)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge("chatbot", END)

conn = sqlite3.connect("memory.db", check_same_thread=False)
memory = SqliteSaver(conn)

graph = graph_builder.compile(name="mr_poet")
```

**`name` 파라미터:**
- LangSmith에서 그래프를 식별하는 이름
- 여러 그래프를 구분하여 추적 가능

## LangSmith 대시보드 기능

### 1. 추적(Traces) 뷰

**실행 추적 정보:**
```
Run: mr_poet
├── Node: chatbot
│   ├── Input: {"messages": [...]}
│   ├── LLM Call: gpt-5-nano
│   │   ├── Prompt tokens: 150
│   │   ├── Completion tokens: 80
│   │   └── Latency: 1.2s
│   └── Output: {"messages": [...], "tool_calls": [...]}
│
├── Node: tools
│   ├── Tool: get_human_feedback
│   ├── Interrupt: waiting for user input
│   └── Resume: user feedback received
│
└── Node: chatbot (재실행)
    ├── Input: {"messages": [...], "tool_results": [...]}
    └── Output: {"messages": [...]}
```

### 2. 성능 메트릭

**자동 수집되는 메트릭:**
- **실행 시간**: 각 노드 및 전체 그래프
- **토큰 사용량**: LLM 호출별 프롬프트/완료 토큰
- **비용 추정**: 토큰 사용량 기반 비용 계산
- **에러율**: 실패한 실행 비율
- **레이턴시 분포**: p50, p95, p99 레이턴시

### 3. 상태 검사

**각 단계의 상태 확인:**
```python
# LangSmith에서 볼 수 있는 정보
{
    "messages": [
        {"role": "user", "content": "Write a poem about Python"},
        {"role": "assistant", "content": "", "tool_calls": [...]},
        {"role": "tool", "content": "feedback: looks great!"},
        {"role": "assistant", "content": "final poem..."}
    ],
    "custom_stuff": "additional state data"
}
```

### 4. 디버깅 도구

**에러 발생 시:**
- 스택 트레이스 자동 캡처
- 실패 지점의 입력 상태 저장
- 재실행 가능한 테스트 케이스 생성

### 5. 비교 및 평가

**A/B 테스트:**
```python
# 다양한 설정으로 실행 비교
os.environ["LANGCHAIN_PROJECT"] = "experiment-a"
result_a = graph.invoke(...)

os.environ["LANGCHAIN_PROJECT"] = "experiment-b"
result_b = graph.invoke(...)

# LangSmith에서 두 실행 비교
```

## 고급 추적 설정

### 1. 커스텀 메타데이터 추가

```python
from langsmith import traceable

@traceable(
    run_type="llm",
    metadata={"version": "1.0", "model": "gpt-5-nano"}
)
def chatbot(state: State):
    response = llm_with_tools.invoke(...)
    return {"messages": [response]}
```

### 2. 태그 및 필터링

```python
# 실행에 태그 추가
result = graph.invoke(
    {"messages": [...]},
    config={
        "tags": ["production", "high-priority"],
        "metadata": {"user_id": "12345"}
    }
)
```

### 3. 샘플링

```python
import random

# 10% 실행만 추적 (프로덕션 부하 감소)
if random.random() < 0.1:
    os.environ["LANGCHAIN_TRACING_V2"] = "true"
else:
    os.environ["LANGCHAIN_TRACING_V2"] = "false"

result = graph.invoke(...)
```

## 실전 활용 팁

### 1. 개발 환경별 프로젝트 분리

```python
import os

environment = os.getenv("ENVIRONMENT", "development")
os.environ["LANGCHAIN_PROJECT"] = f"my-app-{environment}"
```

### 2. 민감 정보 마스킹

```python
# LangSmith에 민감 정보가 전송되지 않도록 주의
# 필요시 데이터 마스킹 처리
def mask_sensitive_data(state):
    if "password" in state:
        state["password"] = "***"
    return state
```

### 3. 성능 모니터링 알림

LangSmith에서 설정:
- 에러율 임계값 초과 시 알림
- 레이턴시 급증 시 알림
- 비용 예산 초과 시 알림

### 4. 팀 협업

```python
# 특정 실행을 팀원과 공유
# LangSmith UI에서 "Share" 버튼 클릭
# 또는 URL 공유
```

## 문제 해결

### 추적이 표시되지 않는 경우

```python
# 1. 환경 변수 확인
print(os.environ.get("LANGCHAIN_TRACING_V2"))
print(os.environ.get("LANGCHAIN_API_KEY"))

# 2. API 키 유효성 확인
# LangSmith 웹사이트에서 키 상태 확인

# 3. 네트워크 연결 확인
# 방화벽이나 프록시 설정 확인
```

### 느린 추적 전송

```python
# 비동기 추적 전송 활성화 (기본값)
os.environ["LANGCHAIN_TRACING_ASYNC"] = "true"

# 배치 크기 조정
os.environ["LANGCHAIN_TRACING_BATCH_SIZE"] = "20"
```

## LangSmith CLI

### 설치

```bash
pip install langsmith
```

### 유용한 명령어

```bash
# 프로젝트 목록 조회
langsmith projects list

# 최근 실행 조회
langsmith runs list --project my-project --limit 10

# 실행 상세 정보
langsmith runs get <run-id>

# 데이터셋 관리
langsmith datasets create my-dataset
langsmith datasets list
```

## 프로덕션 배포 고려사항

### 1. 샘플링 전략

```python
# 트래픽이 많은 프로덕션에서는 샘플링 필수
sampling_rate = 0.01  # 1%만 추적

if random.random() < sampling_rate:
    os.environ["LANGCHAIN_TRACING_V2"] = "true"
```

### 2. 비용 관리

- LangSmith는 추적 데이터 저장량에 따라 요금 부과
- 불필요한 추적 비활성화
- 오래된 추적 데이터 자동 삭제 설정

### 3. 보안

- API 키를 환경 변수나 시크릿 매니저에 안전하게 저장
- 민감한 데이터가 추적에 포함되지 않도록 주의
- 필요시 프라이빗 배포 옵션 고려

## 주요 학습 포인트

1. **자동 추적**: 환경 변수만 설정하면 모든 실행 자동 추적
2. **상세한 인사이트**: 노드별 실행 시간, 상태, 입출력 확인
3. **디버깅 편의성**: 실패 지점과 원인을 빠르게 파악
4. **성능 최적화**: 병목 지점 식별 및 개선
5. **협업 도구**: 팀원과 추적 데이터 공유 및 분석

## 추가 리소스

- **공식 문서**: [docs.smith.langchain.com](https://docs.smith.langchain.com/)
- **LangSmith 대시보드**: [smith.langchain.com](https://smith.langchain.com/)
- **튜토리얼**: [python.langchain.com/docs/langsmith](https://python.langchain.com/docs/langsmith/)
- **API 레퍼런스**: [api.smith.langchain.com](https://api.smith.langchain.com/docs)

## 관련 코드

- 소스 파일: `agents/06_DevTools.py`
