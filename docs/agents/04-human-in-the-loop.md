# 04. Human in the Loop (사람 개입)

## 개요

LangGraph 실행 중 사람의 피드백이나 승인을 받아야 하는 시나리오를 구현하는 방법을 학습합니다. `interrupt()`를 사용하여 그래프 실행을 일시 중지하고 사용자 입력을 대기할 수 있습니다.

## 핵심 개념

### 1. interrupt()를 사용한 실행 중단

```python
from langgraph.types import interrupt

@tool
def get_human_feedback(poem: str):
    """
    Asks the user for feedback on the poem.
    Use this before returning the final response.
    """
    feedback = interrupt(f"Here is the poem, tell me what you think\n{poem}")
    return feedback
```

**`interrupt()`의 동작:**
- 그래프 실행을 즉시 중단
- 사용자에게 메시지 전달
- 사용자 응답을 기다림
- 사용자 입력을 함수의 반환값으로 제공

### 2. Command를 사용한 실행 재개

```python
from langgraph.types import Command

# 중단된 지점에서 재개
response = Command(resume="It looks great!")

result = graph.invoke(response, config=config)
```

**`Command.resume`의 역할:**
- 중단된 `interrupt()` 호출에 값 전달
- 그래프 실행을 이어서 진행
- 동일한 `thread_id`와 `checkpoint_id` 사용 필요

### 3. 체크포인터 필수

```python
graph = graph_builder.compile(
    checkpointer=SqliteSaver(conn),  # 체크포인터 필수!
)
```

**주의사항:**
- Human-in-the-loop은 체크포인터 없이 작동하지 않음
- 중단된 상태를 저장하고 복원하기 위해 필요

## 실행 흐름

### 단계 1: 초기 실행 (중단까지)

```python
config = {"configurable": {"thread_id": "3"}}

result = graph.invoke(
    {"messages": [
        {"role": "user", "content": "Please make a poem about Python code."}
    ]},
    config=config,
)
```

**실행 과정:**
1. LLM이 시를 작성
2. `get_human_feedback` 도구 호출
3. `interrupt()`에서 실행 중단
4. 사용자에게 시를 보여주고 피드백 대기

### 단계 2: 피드백 제공 및 재개

```python
response = Command(resume="It looks great!")

result = graph.invoke(response, config=config)
```

**재개 과정:**
1. 사용자 피드백("It looks great!")을 `interrupt()`가 반환
2. 도구가 피드백을 LLM에 전달
3. LLM이 피드백을 기반으로 최종 응답 생성
4. 그래프 실행 완료

## 상태 확인

```python
snapshot = graph.get_state(config)

print(snapshot.next)     # 다음 실행될 노드
print(snapshot.values)   # 현재 상태
```

**중단된 상태:**
```python
snapshot.next = ('tools',)  # tools 노드에서 대기 중
```

**완료된 상태:**
```python
snapshot.next = ()  # 더 이상 실행할 노드 없음
```

## 실전 예제: 시 작성 봇

### 프롬프트 설계

```python
def chatbot(state: State):
    response = llm_with_tools.invoke(
        f"""
        You are an expert in making poems.

        Use the `get_human_feedback` tool to get feedback on your poem.

        Only after you receive positive feedback you can return the final poem.

        ALWAYS ASK FOR FEEDBACK FIRST.

        Here is the conversation history:
        {state["messages"]}
        """
    )
    return {"messages": [response]}
```

**중요 포인트:**
- LLM에게 항상 피드백을 먼저 요청하도록 지시
- 긍정적 피드백을 받은 후에만 완료하도록 명시

### 실행 결과

```
=================================== AI Message ===================================
Tool Calls:
  get_human_feedback (call_ht5DDUaBcRL5vSfN1bV8MzXQ)
  Args:
    poem: Python whispers in the shell-light dawn,
          Indentation serves as a patient guide...

[실행 중단 - 사용자 피드백 대기]

=================================== Tool Message ==================================
Name: get_human_feedback
It looks great!

=================================== AI Message ===================================
Python whispers in the shell-light dawn,
Indentation serves as a patient guide...
```

## 다양한 활용 사례

### 1. 승인 워크플로우

```python
@tool
def request_approval(action: str, details: dict):
    """Request approval before executing action"""
    approval = interrupt({
        "action": action,
        "details": details,
        "message": "Do you approve this action? (yes/no)"
    })
    return approval
```

### 2. 정보 수집

```python
@tool
def collect_user_info(question: str):
    """Collect additional information from user"""
    answer = interrupt(question)
    return answer
```

### 3. 선택지 제공

```python
@tool
def present_options(options: list):
    """Present options and wait for user selection"""
    selection = interrupt({
        "options": options,
        "message": "Please select an option:"
    })
    return selection
```

## 주요 학습 포인트

1. **실행 중단**: `interrupt()`로 언제든지 그래프 실행 멈춤
2. **상태 보존**: 체크포인터가 중단된 상태 저장
3. **유연한 재개**: `Command.resume`으로 값 전달 및 재개
4. **도구 통합**: `interrupt()`를 도구 안에서 사용
5. **사용자 경험**: 자연스러운 대화형 워크플로우 구현

## 고급 패턴

### 여러 번 피드백 받기

```python
@tool
def iterative_feedback(draft: str):
    """Get feedback and iterate until approved"""
    while True:
        feedback = interrupt({
            "draft": draft,
            "message": "Feedback? (Type 'approve' to finish)"
        })

        if feedback.lower() == "approve":
            return draft
        else:
            # LLM이 피드백 기반으로 수정하도록 유도
            return f"User feedback: {feedback}. Please revise."
```

### 조건부 중단

```python
@tool
def smart_approval(confidence: float, action: str):
    """Only ask for approval if confidence is low"""
    if confidence < 0.7:
        approval = interrupt(f"Low confidence ({confidence}). Approve {action}?")
        return approval
    return "auto-approved"
```

## 에러 처리

### 중단 없이 실행 시도

```python
# 체크포인터 없이 컴파일
graph = graph_builder.compile()  # 체크포인터 없음

# interrupt() 사용 시 에러 발생
result = graph.invoke({"messages": [...]})
# GraphInterrupt: Cannot interrupt without checkpointer
```

### 잘못된 재개 시도

```python
# 다른 thread_id로 재개 시도
response = Command(resume="feedback")
result = graph.invoke(
    response,
    config={"configurable": {"thread_id": "wrong_id"}}
)
# Error: No interrupted state found
```

## 다음 단계

- [05. Time Travel](./05-time-travel.md): 과거 상태 수정 및 분기
- [06. DevTools (LangSmith)](./06-devtools-langsmith.md): 개발 도구 및 모니터링

## 관련 코드

- 소스 파일: `agents/04_Human_in_the_loop.ipynb`
