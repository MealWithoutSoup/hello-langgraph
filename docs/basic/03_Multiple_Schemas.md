# 다중 스키마 (Multiple Schemas)

## 개요
LangGraph에서 Input, Output, Private State를 분리하여 그래프의 입출력을 제어하고 내부 상태를 캡슐화하는 방법을 학습합니다.

## 핵심 개념

### 상태 스키마 분리의 필요성
- **Input Schema**: 그래프 외부에서 받을 수 있는 데이터 정의
- **Output Schema**: 그래프 외부로 반환할 데이터 정의
- **Private State**: 그래프 내부에서만 사용하는 데이터 정의

이를 통해 **캡슐화**와 **인터페이스 제어**가 가능합니다.

## 스키마 정의

### 1. Private State (내부 상태)
```python
class PrivateState(TypedDict):
    a: int
    b: int
```
- 그래프 내부 노드들이 실제로 사용하는 상태
- 외부에서는 직접 접근할 수 없습니다

### 2. Input Schema (입력 스키마)
```python
class InputState(TypedDict):
    hello: str
```
- 그래프 실행 시 외부에서 전달할 수 있는 데이터
- `graph.invoke()`에 전달 가능한 필드

### 3. Output Schema (출력 스키마)
```python
class OutputState(TypedDict):
    bye: str
```
- 그래프 실행 완료 후 외부로 반환되는 데이터
- 최종 `invoke()` 결과에 포함되는 필드

### 4. 그래프 생성
```python
graph_builder = StateGraph(
    PrivateState,
    input_schema=InputState,
    output_schema=OutputState,
)
```
- 첫 번째 인자: 내부적으로 사용할 상태 타입
- `input_schema`: 외부 입력 제한
- `output_schema`: 외부 출력 제한

## 노드별 상태 변환

### 1. Input → Private 변환
```python
def node_one(state: InputState) -> InputState:
    print("node_one ->", state)
    return {"hello": "world"}
```
- `InputState`를 받아 처리
- 반환값은 무시됨 (PrivateState로 자동 전환)

### 2. Private State 처리
```python
def node_two(state: PrivateState) -> PrivateState:
    print("node_two ->", state)
    return {"a": 1}
```
- `PrivateState`에 접근하여 내부 필드 업데이트
- 외부 스키마와 무관하게 자유롭게 작업

```python
def node_three(state: PrivateState) -> PrivateState:
    print("node_three ->", state)
    return {"b": 1}
```

### 3. Private → Output 변환
```python
def node_four(state: PrivateState) -> OutputState:
    print("node_four ->", state)
    return {"bye": "world"}
```
- `PrivateState`를 읽고 `OutputState`로 변환
- 외부로 노출할 데이터만 선택적으로 반환

### 4. Output State 후처리
```python
def node_five(state: OutputState):
    return {"secret": True}
```
- `OutputState`에 없는 필드를 반환해도 무시됨
- 출력 스키마에 정의되지 않은 데이터는 외부로 전달되지 않음

### 5. 스키마 외부 접근
```python
def node_six(state: MegaPrivate):
    print(state)
```
- 정의되지 않은 스키마 타입 사용 가능
- 하지만 실제 상태에는 영향을 주지 않음

## 실행 흐름

### 그래프 구성
```python
graph_builder.add_edge(START, "node_one")
graph_builder.add_edge("node_one", "node_two")
graph_builder.add_edge("node_two", "node_three")
graph_builder.add_edge("node_three", "node_four")
graph_builder.add_edge("node_four", "node_five")
graph_builder.add_edge("node_five", "node_six")
graph_builder.add_edge("node_six", END)
```

### 실행 및 결과
```python
graph = graph_builder.compile()

result = graph.invoke({"hello": "world"})
print(result)
```

**실행 로그:**
```
node_one -> {'hello': 'world'}
node_two -> {}
node_three -> {'a': 1}
node_four -> {'a': 1, 'b': 1}
{'secret': True}
{'bye': 'world'}
```

**최종 반환값:**
```python
{'bye': 'world'}  # OutputState만 반환
```

## 주요 특징

### 1. 자동 상태 변환
- 노드 간 이동 시 상태가 자동으로 변환됩니다
- 입력 → 내부 → 출력으로 단방향 흐름

### 2. 데이터 캡슐화
```python
# PrivateState의 'a', 'b' 필드는 외부로 노출되지 않음
result = graph.invoke({"hello": "world"})
# {'bye': 'world'}  # 'a', 'b' 필드 없음
```

### 3. 인터페이스 제어
```python
# ❌ InputState에 없는 필드는 전달 불가
graph.invoke({"hello": "world", "a": 100})  # 'a'는 무시됨

# ✅ InputState 필드만 유효
graph.invoke({"hello": "world"})
```

### 4. 출력 필터링
```python
def node_five(state: OutputState):
    return {"secret": True}  # OutputState에 없는 필드

# 최종 결과에서 'secret'은 제외됨
```

## 실전 활용 패턴

### 1. API 워크플로우
```python
class APIInput(TypedDict):
    user_id: str

class InternalState(TypedDict):
    user_id: str
    user_data: dict
    processed_data: dict

class APIOutput(TypedDict):
    result: str
    status: int
```

**장점:**
- 민감한 내부 데이터 숨김
- 깔끔한 공개 API 제공

### 2. 데이터 파이프라인
```python
class PipelineInput(TypedDict):
    raw_data: str

class PipelineInternal(TypedDict):
    raw_data: str
    cleaned_data: str
    features: list
    model_output: float

class PipelineOutput(TypedDict):
    prediction: float
```

**장점:**
- 중간 처리 단계 숨김
- 최종 결과만 외부 노출

### 3. 보안/권한 제어
```python
class PublicInput(TypedDict):
    query: str

class SecureInternal(TypedDict):
    query: str
    api_key: str
    decrypted_data: str

class PublicOutput(TypedDict):
    response: str
```

**장점:**
- 인증 정보를 내부에만 보관
- 외부에 민감 데이터 노출 방지

## 주요 학습 포인트

1. **3단계 분리**: Input → Private → Output 명확한 경계
2. **자동 변환**: 스키마 간 전환이 자동으로 처리됨
3. **캡슐화**: 내부 구현 세부사항 숨김
4. **타입 안정성**: 각 스키마가 독립적인 타입 체크 제공

## 다음 단계
- `04_Reducer_Functions.md`: 상태 누적 및 리듀서 함수 활용
- `06_Conditional_Edge.md`: 조건부 라우팅으로 동적 흐름 제어
