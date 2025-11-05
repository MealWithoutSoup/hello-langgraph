# Command API - 명시적 라우팅

## 개요
LangGraph의 Command API를 사용하여 노드 내부에서 다음 실행할 노드를 명시적으로 지정하고 동시에 상태를 업데이트하는 방법을 학습합니다.

## 핵심 개념

### 기존 라우팅 방식의 한계

**조건부 엣지 (Conditional Edge):**
```python
def decide(state: State):
    return "node_a" if state["value"] > 10 else "node_b"

graph_builder.add_conditional_edges(
    "source",
    decide,
    {"node_a": "node_a", "node_b": "node_b"}
)
```
- 라우팅 로직과 노드 로직이 분리됨
- 상태 업데이트와 라우팅이 별도 단계

**Command API:**
```python
def node(state: State) -> Command[Literal["node_a", "node_b"]]:
    # 라우팅과 상태 업데이트를 동시에
    return Command(
        goto="node_a",
        update={"processed": True}
    )
```
- 노드 내부에서 라우팅 결정
- 상태 업데이트와 라우팅이 하나의 반환값

## Command 기본 구조

### 1. Import
```python
from langgraph.types import Command
from typing import Literal
```

### 2. Command 객체
```python
Command(
    goto="target_node",
    update={"field": "value"}
)
```

**파라미터:**
- `goto`: 다음 실행할 노드 이름
- `update`: 상태에 반영할 업데이트 (딕셔너리)

### 3. 타입 힌트
```python
def node(state: State) -> Command[Literal["node_a", "node_b"]]:
    return Command(goto="node_a", update={})
```
- `Command[Literal[...]]`: 가능한 타겟 노드 명시
- 타입 안정성과 IDE 자동완성 제공

## 기본 구현 예제

### 1. State 정의
```python
class State(TypedDict):
    transfer_reason: str
```

### 2. 라우팅 노드
```python
def triage_node(state: State) -> Command[Literal["account_support", "tech_support"]]:
    return Command(
        goto="account_support",
        update={
            "transfer_reason": "The user wants to change password.",
        },
    )
```

**특징:**
- 반환 타입이 `Command[Literal[...]]`
- `goto`로 타겟 노드 지정
- `update`로 상태 업데이트

### 3. 타겟 노드
```python
def tech_support(state: State):
    return {}

def account_support(state: State):
    print("account_support running")
    return {}
```

### 4. 그래프 구성
```python
graph_builder.add_node("triage_node", triage_node)
graph_builder.add_node("tech_support", tech_support)
graph_builder.add_node("account_support", account_support)

graph_builder.add_edge(START, "triage_node")
graph_builder.add_edge("tech_support", END)
graph_builder.add_edge("account_support", END)
```

**주의:**
- Command를 사용하는 노드에는 **조건부 엣지를 추가하지 않음**
- Command가 라우팅을 자동으로 처리
- 타겟 노드들에는 일반 엣지 추가 가능

### 5. 실행
```python
graph = graph_builder.compile()

result = graph.invoke({})
```

**출력:**
```
account_support running
```

**실행 흐름:**
```
START → triage_node → Command(goto="account_support") → account_support → END
```

## Command vs 조건부 엣지 비교

### 조건부 엣지 방식
```python
def process_node(state: State):
    # 처리 로직
    result = process(state["data"])
    return {"result": result}

def routing_function(state: State):
    if state["result"] == "success":
        return "success_node"
    return "failure_node"

graph_builder.add_node("process_node", process_node)
graph_builder.add_conditional_edges(
    "process_node",
    routing_function,
    {"success_node": "success_node", "failure_node": "failure_node"}
)
```

**단점:**
- 라우팅 로직이 별도 함수
- 상태를 다시 읽어야 함
- 그래프 구성이 복잡

### Command 방식
```python
def process_node(state: State) -> Command[Literal["success_node", "failure_node"]]:
    # 처리 로직
    result = process(state["data"])

    # 결과에 따라 라우팅과 업데이트를 동시에
    if result == "success":
        return Command(
            goto="success_node",
            update={"result": result, "status": "completed"}
        )
    return Command(
        goto="failure_node",
        update={"result": result, "status": "failed"}
    )

graph_builder.add_node("process_node", process_node)
# 조건부 엣지 불필요!
```

**장점:**
- 라우팅 로직이 노드 내부에 통합
- 한 번의 상태 읽기로 결정
- 그래프 구성이 단순

## 실전 활용 패턴

### 1. 고객 서비스 라우팅
```python
def customer_service_triage(state: State) -> Command[Literal[
    "billing_support",
    "technical_support",
    "account_support",
    "general_inquiry"
]]:
    query = state["customer_query"].lower()

    if "bill" in query or "payment" in query:
        return Command(
            goto="billing_support",
            update={"department": "billing", "priority": "high"}
        )
    elif "error" in query or "bug" in query:
        return Command(
            goto="technical_support",
            update={"department": "tech", "priority": "urgent"}
        )
    elif "password" in query or "login" in query:
        return Command(
            goto="account_support",
            update={"department": "account", "priority": "medium"}
        )
    else:
        return Command(
            goto="general_inquiry",
            update={"department": "general", "priority": "low"}
        )
```

### 2. 재시도 로직
```python
def api_call_node(state: State) -> Command[Literal["retry", "success", "failure"]]:
    try:
        result = call_external_api(state["request"])
        return Command(
            goto="success",
            update={"result": result, "attempts": state["attempts"] + 1}
        )
    except TemporaryError:
        if state["attempts"] < 3:
            return Command(
                goto="retry",
                update={"attempts": state["attempts"] + 1}
            )
        return Command(
            goto="failure",
            update={"error": "Max retries exceeded", "attempts": state["attempts"] + 1}
        )
```

### 3. 워크플로우 상태 머신
```python
def workflow_controller(state: State) -> Command[Literal[
    "validate",
    "process",
    "approve",
    "reject",
    "complete"
]]:
    if state["stage"] == "new":
        return Command(
            goto="validate",
            update={"stage": "validating"}
        )
    elif state["stage"] == "validated":
        if state["validation_passed"]:
            return Command(
                goto="process",
                update={"stage": "processing"}
            )
        return Command(
            goto="reject",
            update={"stage": "rejected", "reason": "Validation failed"}
        )
    elif state["stage"] == "processed":
        return Command(
            goto="approve",
            update={"stage": "approving"}
        )
    else:
        return Command(
            goto="complete",
            update={"stage": "completed"}
        )
```

### 4. 인증 및 권한 체크
```python
def auth_gate(state: State) -> Command[Literal[
    "authenticated_flow",
    "login_required",
    "insufficient_permissions"
]]:
    user = state.get("user")

    if not user:
        return Command(
            goto="login_required",
            update={"auth_status": "unauthenticated"}
        )

    if not has_permission(user, state["required_permission"]):
        return Command(
            goto="insufficient_permissions",
            update={"auth_status": "unauthorized"}
        )

    return Command(
        goto="authenticated_flow",
        update={"auth_status": "authorized", "user_id": user["id"]}
    )
```

### 5. 데이터 처리 파이프라인
```python
def data_processor(state: State) -> Command[Literal[
    "clean_data",
    "enrich_data",
    "save_data",
    "error_handler"
]]:
    data = state["raw_data"]

    if not is_valid_format(data):
        return Command(
            goto="error_handler",
            update={"error": "Invalid data format"}
        )

    if needs_cleaning(data):
        return Command(
            goto="clean_data",
            update={"processing_step": "cleaning"}
        )

    if needs_enrichment(data):
        return Command(
            goto="enrich_data",
            update={"processing_step": "enrichment"}
        )

    return Command(
        goto="save_data",
        update={"processing_step": "saving", "clean_data": data}
    )
```

## 고급 패턴

### 1. 동적 타겟 선택
```python
def dynamic_router(state: State) -> Command:
    # 런타임에 타겟 결정
    target = state["config"]["next_step"]

    return Command(
        goto=target,
        update={"last_step": state["current_step"]}
    )
```

### 2. 조건부 업데이트
```python
def conditional_update(state: State) -> Command[Literal["next"]]:
    updates = {"processed": True}

    if state.get("needs_logging"):
        updates["log"] = create_log_entry(state)

    if state.get("send_notification"):
        updates["notification_sent"] = send_notification(state)

    return Command(goto="next", update=updates)
```

### 3. 에러 핸들링 통합
```python
def safe_processor(state: State) -> Command[Literal["success", "error"]]:
    try:
        result = risky_operation(state["data"])
        return Command(
            goto="success",
            update={"result": result, "status": "success"}
        )
    except Exception as e:
        return Command(
            goto="error",
            update={"error": str(e), "status": "failed", "stack_trace": traceback.format_exc()}
        )
```

## 제한사항 및 주의사항

### 1. Command와 조건부 엣지 혼용 불가
```python
# ❌ 잘못된 예
def node(state: State) -> Command[Literal["target"]]:
    return Command(goto="target", update={})

graph_builder.add_node("node", node)
graph_builder.add_conditional_edges("node", ...)  # 불필요하고 충돌 가능
```

### 2. 타겟 노드 존재 확인
```python
# ❌ 정의되지 않은 노드로 라우팅
return Command(goto="nonexistent_node", update={})  # 런타임 에러

# ✅ 먼저 노드 추가
graph_builder.add_node("target_node", target_node)
```

### 3. 타입 힌트 일치
```python
# ❌ 타입과 실제 반환이 불일치
def node(state: State) -> Command[Literal["node_a", "node_b"]]:
    return Command(goto="node_c", update={})  # node_c는 타입에 없음!

# ✅ 타입과 일치
def node(state: State) -> Command[Literal["node_a", "node_b"]]:
    return Command(goto="node_a", update={})
```

### 4. 업데이트는 선택적
```python
# ✅ 업데이트 없이 라우팅만 가능
return Command(goto="next_node", update={})

# ✅ update 생략도 가능
return Command(goto="next_node")
```

## Command vs Send vs Conditional Edge

| 특징 | Command | Send | Conditional Edge |
|------|---------|------|------------------|
| 라우팅 위치 | 노드 내부 | 노드 내부 | 그래프 레벨 |
| 병렬 실행 | ❌ | ✅ | ❌ |
| 상태 업데이트 | ✅ | ✅ | 별도 노드 |
| 타입 안정성 | ✅ | ✅ | ✅ |
| 복잡도 | 낮음 | 중간 | 높음 |
| 사용 사례 | 단일 라우팅 | 팬아웃 | 복잡한 조건 |

## 주요 학습 포인트

1. **통합 라우팅**: 노드 내부에서 라우팅과 상태 업데이트를 동시에 처리
2. **타입 안정성**: `Command[Literal[...]]`로 가능한 타겟 명시
3. **단순한 그래프**: 조건부 엣지 불필요, 그래프 구성이 간결
4. **유연한 로직**: 노드 내부에서 복잡한 라우팅 로직 구현 가능
5. **명시적 제어**: 다음 노드를 명확하게 지정

## 선택 가이드

### Command 사용이 적합한 경우:
- 노드 내부 로직과 라우팅이 밀접하게 연관
- 처리 결과에 따라 즉시 라우팅 결정
- 상태 업데이트와 라우팅을 원자적으로 처리
- 그래프를 단순하게 유지하고 싶을 때

### 조건부 엣지가 적합한 경우:
- 라우팅 로직을 재사용하고 싶을 때
- 여러 노드에서 동일한 라우팅 패턴 사용
- 그래프 구조를 시각적으로 명확히 표현
- 라우팅 로직을 노드와 분리하고 싶을 때

## 다음 단계
- 실전 프로젝트: Command API로 복잡한 워크플로우 구현
- 여러 패턴 조합: Command, Send, Conditional Edge 혼용
- 에러 처리: Command를 활용한 강건한 에러 핸들링
