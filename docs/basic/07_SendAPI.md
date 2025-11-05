# Send API - 팬아웃 패턴 (Fan-out Pattern)

## 개요
LangGraph의 Send API를 사용하여 하나의 노드에서 여러 노드를 병렬로 실행하는 팬아웃(fan-out) 패턴을 구현하는 방법을 학습합니다.

## 핵심 개념

### 팬아웃이란?
- **하나의 입력**을 받아 **여러 개의 병렬 작업**으로 분산
- 각 작업은 독립적으로 실행되고 결과를 누적
- Map-Reduce 패턴의 Map 단계와 유사

### 일반 엣지 vs Send API

**일반 엣지:**
```python
graph_builder.add_edge("node_a", "node_b")
# node_a → node_b (1:1 관계)
```

**Send API:**
```python
from langgraph.types import Send

def dispatcher(state: State):
    return [Send("worker", item) for item in state["items"]]

# node_a → worker(item1), worker(item2), ... (1:N 관계, 병렬)
```

## 기본 구현

### 1. State 정의
```python
from typing import Union
import operator

class State(TypedDict):
    words: list[str]
    output: Annotated[list[dict[str, Union[str, int]]], operator.add]
```

**필드:**
- `words`: 처리할 단어 리스트
- `output`: 누적 결과 (리듀서 함수 사용)

### 2. 노드 정의

**디스패처 노드:**
```python
def node_one(state: State):
    print(f"I want to count {len(state["words"])} words in my state.")
```
- 작업량을 확인하고 로그 출력
- 실제 작업 분산은 조건부 엣지에서 처리

**워커 노드:**
```python
def node_two(word: str):
    return {
        "output": [
            {
                "word": word,
                "letters": len(word),
            }
        ]
    }
```
- **입력**: 개별 단어 (State 전체가 아님!)
- **출력**: 단어와 글자 수를 담은 딕셔너리
- 리듀서에 의해 결과가 누적됨

### 3. Send API 디스패처
```python
def dispatcher(state: State):
    return [Send("node_two", word) for word in state["words"]]
```

**동작:**
- `state["words"]`의 각 단어마다 `Send` 객체 생성
- 각 `Send`는 `node_two`를 해당 단어와 함께 호출
- 모든 `Send`가 **병렬로** 실행됨

### 4. 그래프 구성
```python
graph_builder.add_node("node_one", node_one)
graph_builder.add_node("node_two", node_two)

graph_builder.add_edge(START, "node_one")

graph_builder.add_conditional_edges(
    "node_one",
    dispatcher,
    ["node_two"]
)

graph_builder.add_edge("node_two", END)
```

**핵심:**
- `add_conditional_edges`의 두 번째 인자가 `Send` 리스트를 반환
- 세 번째 인자는 타겟 노드 리스트 (여기서는 `["node_two"]`)

### 5. 실행
```python
graph = graph_builder.compile()

result = graph.invoke({
    "words": ["hello", "world", "how", "are", "you", "doing"],
})
```

**출력 (콘솔):**
```
I want to count 6 words in my state.
```

**결과:**
```python
{
    "words": ["hello", "world", "how", "are", "you", "doing"],
    "output": [
        {"word": "hello", "letters": 5},
        {"word": "world", "letters": 5},
        {"word": "how", "letters": 3},
        {"word": "are", "letters": 3},
        {"word": "you", "letters": 3},
        {"word": "doing", "letters": 5}
    ]
}
```

## Send API 동작 원리

### 1. Send 객체
```python
Send("target_node", data)
```
- `target_node`: 호출할 노드 이름
- `data`: 해당 노드에 전달할 데이터 (State 전체가 아님)

### 2. 병렬 실행
```python
[Send("worker", 1), Send("worker", 2), Send("worker", 3)]
```
- 3개의 `Send` → `worker` 노드가 3번 병렬 실행
- 각 실행은 독립적인 입력 데이터를 받음

### 3. 결과 누적
```python
class State(TypedDict):
    output: Annotated[list[dict], operator.add]

# worker 1 반환: {"output": [{"id": 1}]}
# worker 2 반환: {"output": [{"id": 2}]}
# worker 3 반환: {"output": [{"id": 3}]}

# 최종 상태: {"output": [{"id": 1}, {"id": 2}, {"id": 3}]}
```
- 리듀서 함수(`operator.add`)가 모든 결과를 하나의 리스트로 병합

## 실전 활용 패턴

### 1. 배치 처리
```python
class State(TypedDict):
    user_ids: list[str]
    results: Annotated[list[dict], operator.add]

def process_user(user_id: str):
    user_data = fetch_user_data(user_id)
    return {"results": [user_data]}

def dispatcher(state: State):
    return [Send("process_user", uid) for uid in state["user_ids"]]

graph_builder.add_conditional_edges(
    "start",
    dispatcher,
    ["process_user"]
)
```

**효과:**
- 100명의 사용자를 병렬로 처리
- 순차 처리 대비 100배 속도 향상 가능

### 2. 멀티 모델 추론
```python
class State(TypedDict):
    prompt: str
    models: list[str]
    responses: Annotated[list[dict], operator.add]

def run_model(model_config: dict):
    model_name = model_config["name"]
    prompt = model_config["prompt"]
    response = llm_inference(model_name, prompt)
    return {
        "responses": [{
            "model": model_name,
            "response": response
        }]
    }

def dispatcher(state: State):
    return [
        Send("run_model", {"name": model, "prompt": state["prompt"]})
        for model in state["models"]
    ]
```

**효과:**
- 여러 LLM 모델을 동시에 실행
- 결과를 비교하여 최적 응답 선택

### 3. 파일 처리
```python
class State(TypedDict):
    files: list[str]
    processed: Annotated[list[dict], operator.add]

def process_file(filepath: str):
    data = parse_file(filepath)
    return {
        "processed": [{
            "file": filepath,
            "data": data
        }]
    }

def dispatcher(state: State):
    return [Send("process_file", f) for f in state["files"]]
```

### 4. API 병렬 호출
```python
class State(TypedDict):
    urls: list[str]
    data: Annotated[list[dict], operator.add]

def fetch_url(url: str):
    response = http_get(url)
    return {
        "data": [{
            "url": url,
            "content": response.text
        }]
    }

def dispatcher(state: State):
    return [Send("fetch_url", url) for url in state["urls"]]
```

### 5. 데이터 검증
```python
class State(TypedDict):
    records: list[dict]
    valid: Annotated[list[dict], operator.add]
    invalid: Annotated[list[dict], operator.add]

def validate_record(record: dict):
    if is_valid(record):
        return {"valid": [record]}
    return {"invalid": [record]}

def dispatcher(state: State):
    return [Send("validate_record", rec) for rec in state["records"]]
```

## 고급 패턴

### 1. 조건부 디스패치
```python
def smart_dispatcher(state: State):
    sends = []
    for item in state["items"]:
        if item["priority"] == "high":
            sends.append(Send("fast_processor", item))
        else:
            sends.append(Send("normal_processor", item))
    return sends

graph_builder.add_conditional_edges(
    "triage",
    smart_dispatcher,
    ["fast_processor", "normal_processor"]
)
```

### 2. 청크 단위 처리
```python
def batch_dispatcher(state: State):
    chunk_size = 10
    chunks = [
        state["items"][i:i+chunk_size]
        for i in range(0, len(state["items"]), chunk_size)
    ]
    return [Send("batch_processor", chunk) for chunk in chunks]
```

### 3. 의존성이 있는 병렬 처리
```python
def two_stage_dispatcher(state: State):
    # 1단계: 전처리 병렬 실행
    return [Send("preprocess", item) for item in state["items"]]

def second_dispatcher(state: State):
    # 2단계: 처리 병렬 실행 (전처리 결과 사용)
    return [Send("process", item) for item in state["preprocessed"]]

graph_builder.add_conditional_edges("start", two_stage_dispatcher, ["preprocess"])
graph_builder.add_edge("preprocess", "collect")
graph_builder.add_conditional_edges("collect", second_dispatcher, ["process"])
```

## 제한사항 및 주의사항

### 1. 워커 노드 입력
```python
# ❌ 잘못된 예
def worker(state: State):  # State 전체를 받으면 안 됨
    return {}

# ✅ 올바른 예
def worker(item: str):  # Send로 전달한 개별 데이터
    return {}
```

### 2. 리듀서 필수
```python
# ❌ 리듀서 없으면 결과 병합 안 됨
class State(TypedDict):
    results: list[dict]  # 마지막 결과만 남음

# ✅ 리듀서로 모든 결과 누적
class State(TypedDict):
    results: Annotated[list[dict], operator.add]
```

### 3. 실행 순서 보장 없음
```python
# 병렬 실행이므로 순서가 보장되지 않음
# 순서가 중요하면 순차 처리 사용
```

### 4. 에러 처리
```python
# 하나의 Send가 실패하면 전체 그래프 실패
# 개별 에러 처리가 필요한 경우 워커 내부에서 try-catch
def worker(item: str):
    try:
        result = process(item)
        return {"results": [{"success": True, "data": result}]}
    except Exception as e:
        return {"results": [{"success": False, "error": str(e)}]}
```

## 주요 학습 포인트

1. **팬아웃 패턴**: 하나의 입력을 여러 병렬 작업으로 분산
2. **Send API**: `Send("node", data)` 객체로 병렬 실행 제어
3. **리듀서 필수**: `Annotated[list, operator.add]`로 결과 누적
4. **워커 입력**: State 전체가 아닌 개별 데이터 수신
5. **성능 향상**: 독립적 작업의 병렬 처리로 속도 개선

## Map-Reduce 패턴
Send API는 Map-Reduce의 Map 단계에 해당:
- **Map (Send API)**: 데이터를 분산하여 병렬 처리
- **Reduce (Reducer Function)**: 결과를 하나로 병합

## 다음 단계
- `08_Command.md`: Command API로 명시적 노드 타겟팅 및 상태 업데이트
- 실전 프로젝트: Send API로 대규모 배치 처리 구현
