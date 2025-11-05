# 상태 관리 (State Management)

## 개요
LangGraph에서 노드 간 데이터를 전달하고 상태를 업데이트하는 방법을 학습합니다.

## 핵심 개념

### State 정의
```python
class State(TypedDict):
    hello: str
    a: bool
```
- 여러 필드를 가진 상태 구조 정의
- 각 노드는 이 상태를 읽고 수정할 수 있습니다

## 상태 업데이트 메커니즘

### 1. 노드에서 상태 반환
```python
def node_one(state: State):
    print("node_one", state)
    return {
        "hello": "from node one.",
        "a": True,
    }
```

**동작 방식:**
- 노드 함수는 딕셔너리를 반환하여 상태를 업데이트합니다
- 반환된 딕셔너리의 키-값 쌍이 현재 상태에 **병합(merge)**됩니다
- 반환하지 않은 필드는 기존 값을 유지합니다

### 2. 부분 업데이트 (Partial Update)
```python
def node_two(state: State):
    print("node_two", state)
    return {"hello": "from node two."}
    # 'a' 필드는 반환하지 않아도 기존 값 유지
```

**특징:**
- 모든 필드를 반환할 필요가 없습니다
- 명시적으로 반환한 필드만 업데이트됩니다
- 업데이트하지 않은 필드는 이전 값을 보존합니다

### 3. 상태 불변 (No Update)
```python
def node_three(state: State):
    print("node_three", state)
    return {}  # 빈 딕셔너리 반환 = 상태 변경 없음
```

## 실행 흐름 예제

### 초기 실행
```python
graph = graph_builder.compile()

result = graph.invoke({
    "hello": "world",
})
```

### 실행 과정 추적
```
node_one {'hello': 'world'}
→ 반환: {"hello": "from node one.", "a": True}
→ 현재 상태: {'hello': 'from node one.', 'a': True}

node_two {'hello': 'from node one.', 'a': True}
→ 반환: {"hello": "from node two."}
→ 현재 상태: {'hello': 'from node two.', 'a': True}

node_three {'hello': 'from node two.', 'a': True}
→ 반환: {}
→ 현재 상태: {'hello': 'from node three.', 'a': True}
```

### 최종 결과
```python
result
# {'hello': 'from node three.', 'a': True}
```

## 상태 업데이트 규칙

### 1. 병합 (Merge) 방식
```python
# 현재 상태
current = {"hello": "world", "a": False}

# 노드 반환값
update = {"hello": "new value"}

# 결과 (병합)
result = {"hello": "new value", "a": False}
```
- 기본적으로 **얕은 병합(shallow merge)** 수행
- 반환된 키만 덮어쓰고, 나머지는 유지

### 2. 타입 안정성
```python
class State(TypedDict):
    hello: str
    a: bool

# ✅ 올바른 반환
return {"hello": "text", "a": True}

# ⚠️ 타입 불일치 (런타임 경고 가능)
return {"hello": 123, "a": "not a bool"}
```

### 3. 존재하지 않는 키
```python
# State 정의에 없는 키 반환
return {"undefined_key": "value"}  # 무시됨
```
- State에 정의되지 않은 키는 무시됩니다
- TypedDict 정의와 일치해야 합니다

## 그래프 구성
```python
graph_builder.add_node("node_one", node_one)
graph_builder.add_node("node_two", node_two)
graph_builder.add_node("node_three", node_three)

graph_builder.add_edge(START, "node_one")
graph_builder.add_edge("node_one", "node_two")
graph_builder.add_edge("node_two", "node_three")
graph_builder.add_edge("node_three", END)
```

## 주요 학습 포인트

1. **부분 업데이트**: 필요한 필드만 반환하여 상태를 효율적으로 관리
2. **상태 보존**: 명시적으로 업데이트하지 않은 필드는 자동 유지
3. **타입 힌트**: TypedDict를 통해 상태 구조를 명확히 정의
4. **순차적 전파**: 각 노드의 업데이트가 다음 노드로 전파됨

## 실전 활용 시나리오

### 1. 데이터 파이프라인
```python
def fetch_data(state: State):
    return {"data": fetch_from_api()}

def process_data(state: State):
    processed = transform(state["data"])
    return {"data": processed}

def save_data(state: State):
    save_to_db(state["data"])
    return {"status": "saved"}
```

### 2. 조건부 처리
```python
def check_condition(state: State):
    if state["value"] > 10:
        return {"status": "high"}
    return {"status": "low"}
```

## 다음 단계
- `03_Multiple_Schemas.md`: Input/Output/Private State 분리 학습
- `04_Reducer_Functions.md`: 리스트 누적 등 고급 상태 업데이트 패턴
