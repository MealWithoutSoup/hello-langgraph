# 노드 캐싱 (Node Caching)

## 개요
LangGraph에서 노드 실행 결과를 캐싱하여 반복 실행 시 성능을 최적화하는 방법을 학습합니다.

## 핵심 개념

### 캐싱의 필요성
- **비용 절감**: API 호출, DB 쿼리 등 비용이 큰 작업 반복 방지
- **성능 향상**: 동일한 입력에 대해 캐시된 결과 재사용
- **응답 시간 단축**: 계산 시간이 긴 작업의 결과 저장

### TTL (Time To Live)
- 캐시된 결과가 유효한 시간(초 단위)
- TTL 내에 동일한 노드를 다시 실행하면 캐시된 결과 반환
- TTL 경과 후에는 노드를 다시 실행하여 새로운 결과 생성

## 캐싱 구현

### 1. 필수 import
```python
from langgraph.types import CachePolicy
from langgraph.cache.memory import InMemoryCache
from datetime import datetime
```

### 2. State 정의
```python
class State(TypedDict):
    time: str
```

### 3. 노드 정의
```python
def node_one(state: State):
    return {}

def node_two(state: State):
    return {"time": f"{datetime.now()}"}

def node_three(state: State):
    return {}
```
- `node_two`는 현재 시간을 상태에 저장
- 캐싱 없이 실행하면 매번 다른 시간이 기록됨

### 4. 캐싱 정책 적용
```python
graph_builder.add_node("node_one", node_one)

graph_builder.add_node(
    "node_two",
    node_two,
    cache_policy=CachePolicy(ttl=20),  # 20초 TTL
)

graph_builder.add_node("node_three", node_three)
```

**CachePolicy 파라미터:**
- `ttl`: 캐시 유효 시간 (초)
- 20초 = 20초 동안 캐시된 결과 사용

### 5. 그래프 컴파일
```python
graph = graph_builder.compile(cache=InMemoryCache())
```

**cache 파라미터:**
- `InMemoryCache()`: 메모리 기반 캐시 백엔드
- 프로세스 종료 시 캐시 소멸
- 프로덕션에서는 Redis, Memcached 등 사용 가능

## 실행 결과 분석

### 테스트 코드
```python
import time

graph = graph_builder.compile(cache=InMemoryCache())

print(graph.invoke({}))
time.sleep(5)
print(graph.invoke({}))
time.sleep(5)
print(graph.invoke({}))
time.sleep(5)
print(graph.invoke({}))
time.sleep(5)
print(graph.invoke({}))  # 20초 경과
time.sleep(5)
print(graph.invoke({}))
```

### 실행 결과
```python
{'time': '2025-10-29 20:28:02.731822'}  # 0초: 첫 실행
{'time': '2025-10-29 20:28:02.731822'}  # 5초: 캐시 사용
{'time': '2025-10-29 20:28:02.731822'}  # 10초: 캐시 사용
{'time': '2025-10-29 20:28:02.731822'}  # 15초: 캐시 사용
{'time': '2025-10-29 20:28:22.749146'}  # 20초: TTL 경과, 재실행
{'time': '2025-10-29 20:28:22.749146'}  # 25초: 새 캐시 사용
```

**관찰:**
- 0~15초: 동일한 시간 출력 (캐시 hit)
- 20초 경과 후: 새로운 시간 출력 (캐시 miss, 재실행)
- 25초: 새로 캐시된 결과 재사용

## 캐시 키 (Cache Key)

### 자동 캐시 키 생성
LangGraph는 다음 요소로 캐시 키를 생성합니다:
```python
cache_key = hash(node_name + input_state)
```

### 동일한 캐시 키
```python
# 두 호출 모두 동일한 캐시 키 생성
graph.invoke({"input": "test"})
graph.invoke({"input": "test"})  # 캐시 hit
```

### 다른 캐시 키
```python
# 다른 입력 = 다른 캐시 키
graph.invoke({"input": "test1"})  # 캐시 miss
graph.invoke({"input": "test2"})  # 캐시 miss (다른 입력)
```

## 캐시 백엔드

### 1. InMemoryCache
```python
from langgraph.cache.memory import InMemoryCache

graph = graph_builder.compile(cache=InMemoryCache())
```

**특징:**
- 메모리에 저장
- 빠른 접근 속도
- 프로세스 종료 시 소멸
- 개발/테스트 환경에 적합

### 2. 영구 캐시 (예: Redis)
```python
# 실제 구현은 LangGraph 버전에 따라 다름
from langgraph.cache.redis import RedisCache

graph = graph_builder.compile(
    cache=RedisCache(url="redis://localhost:6379")
)
```

**특징:**
- 영구 저장
- 여러 프로세스 간 공유
- 네트워크 오버헤드
- 프로덕션 환경에 적합

## 실전 활용 패턴

### 1. API 호출 캐싱
```python
def fetch_weather(state: State):
    # 외부 API 호출 (비용 발생)
    weather = call_weather_api(state["city"])
    return {"weather": weather}

graph_builder.add_node(
    "fetch_weather",
    fetch_weather,
    cache_policy=CachePolicy(ttl=3600),  # 1시간 캐싱
)
```

**효과:**
- 동일 도시 날씨 조회 시 1시간 동안 API 호출 없이 캐시 반환
- API 비용 절감 및 응답 속도 향상

### 2. DB 쿼리 캐싱
```python
def load_user_profile(state: State):
    # 데이터베이스 쿼리
    profile = db.query(state["user_id"])
    return {"profile": profile}

graph_builder.add_node(
    "load_user_profile",
    load_user_profile,
    cache_policy=CachePolicy(ttl=600),  # 10분 캐싱
)
```

### 3. 계산 결과 캐싱
```python
def compute_analytics(state: State):
    # 복잡한 계산 (시간 소요)
    result = complex_analysis(state["data"])
    return {"analytics": result}

graph_builder.add_node(
    "compute_analytics",
    compute_analytics,
    cache_policy=CachePolicy(ttl=1800),  # 30분 캐싱
)
```

### 4. LLM 응답 캐싱
```python
def llm_generate(state: State):
    # LLM API 호출 (비용/시간 소요)
    response = llm.generate(state["prompt"])
    return {"response": response}

graph_builder.add_node(
    "llm_generate",
    llm_generate,
    cache_policy=CachePolicy(ttl=86400),  # 24시간 캐싱
)
```

## TTL 설정 가이드

| 데이터 유형 | 권장 TTL | 사례 |
|-----------|---------|------|
| 정적 데이터 | 24시간+ | 사용자 프로필, 설정 |
| 자주 변경되는 데이터 | 5-15분 | 재고, 가격 |
| 실시간 데이터 | 1-5분 | 날씨, 주식 |
| 계산 결과 | 30-60분 | 분석, 통계 |
| LLM 응답 | 1-24시간 | 챗봇, 생성 결과 |

## 캐싱 주의사항

### 1. 부작용(Side Effects)이 있는 노드
```python
# ❌ 캐싱하면 안 되는 경우
def send_email(state: State):
    email_service.send(state["email"])
    return {"sent": True}

# 캐싱 시 이메일이 한 번만 발송됨!
```

### 2. 상태 의존성
```python
# 입력 상태가 캐시 키에 포함됨
def process(state: State):
    return {"result": compute(state["value"])}

# state["value"]가 다르면 캐시 miss
```

### 3. 메모리 제한
- InMemoryCache는 메모리를 사용하므로 제한 고려
- 큰 결과는 메모리 부족 위험

### 4. 캐시 무효화
- TTL 외에 명시적 캐시 무효화 방법 없음 (InMemoryCache)
- 필요 시 Redis 등에서 수동 삭제

## 주요 학습 포인트

1. **TTL 기반 캐싱**: 시간 기반 캐시 만료 정책
2. **입력 기반 키**: 노드 이름 + 입력 상태로 캐시 키 생성
3. **성능 최적화**: 비용/시간이 큰 작업에 적용
4. **부작용 주의**: 멱등성 없는 작업은 캐싱 금지
5. **백엔드 선택**: 환경에 맞는 캐시 저장소 선택

## 다음 단계
- `06_Conditional_Edge.md`: 조건부 라우팅으로 동적 흐름 제어
- `07_SendAPI.md`: Send API로 병렬 처리 패턴 구현
