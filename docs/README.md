# LangGraph 학습 문서

이 디렉토리는 `basic/` 폴더의 LangGraph 개념 노트북을 한국어로 정리한 문서입니다.

## 📚 문서 목록

### 기초 개념
1. **[01_Graph.md](01_Graph.md)** - LangGraph 기본 구조
   - StateGraph 기본 사용법
   - 노드와 엣지 정의
   - 선형 실행 흐름

2. **[02_State.md](02_State.md)** - 상태 관리
   - 상태 업데이트 메커니즘
   - 부분 업데이트 패턴
   - 상태 전파 방식

3. **[03_Multiple_Schemas.md](03_Multiple_Schemas.md)** - 다중 스키마
   - Input/Output/Private State 분리
   - 데이터 캡슐화
   - 인터페이스 제어

### 고급 패턴
4. **[04_Reducer_Functions.md](04_Reducer_Functions.md)** - 리듀서 함수
   - Annotated 타입 사용법
   - 상태 누적 패턴
   - operator.add와 커스텀 리듀서

5. **[05_Node_Caching.md](05_Node_Caching.md)** - 노드 캐싱
   - CachePolicy와 TTL
   - InMemoryCache 사용
   - 성능 최적화 전략

### 제어 흐름
6. **[06_Conditional_Edge.md](06_Conditional_Edge.md)** - 조건부 엣지
   - 동적 라우팅
   - 결정 함수 패턴
   - 다중 분기 구현

7. **[07_SendAPI.md](07_SendAPI.md)** - Send API
   - 팬아웃 패턴 (Fan-out)
   - 병렬 실행
   - Map-Reduce 구조

8. **[08_Command.md](08_Command.md)** - Command API
   - 명시적 라우팅
   - 상태 업데이트와 라우팅 통합
   - 노드 내부 제어

## 🎯 학습 순서

### 초보자 추천 경로
```
01_Graph → 02_State → 04_Reducer_Functions → 06_Conditional_Edge
```

### 중급자 추천 경로
```
03_Multiple_Schemas → 05_Node_Caching → 07_SendAPI → 08_Command
```

## 📖 개념 간 관계

```
기초
├── Graph (01) - 그래프 구조의 기본
└── State (02) - 상태 관리의 기본

상태 고급
├── Multiple Schemas (03) - 상태 분리 및 캡슐화
└── Reducer Functions (04) - 상태 누적 패턴

제어 흐름
├── Conditional Edge (06) - 조건부 분기
├── Send API (07) - 병렬 실행
└── Command (08) - 명시적 라우팅

최적화
└── Node Caching (05) - 성능 향상
```

## 🔑 핵심 개념 요약

| 개념 | 핵심 키워드 | 사용 시점 |
|------|-----------|----------|
| Graph | StateGraph, 노드, 엣지 | 모든 LangGraph 프로젝트 |
| State | TypedDict, 상태 업데이트 | 데이터 전달 필요 시 |
| Multiple Schemas | Input/Output/Private | API 인터페이스 설계 |
| Reducer Functions | Annotated, operator.add | 리스트/값 누적 필요 시 |
| Node Caching | CachePolicy, TTL | 비용/시간 절감 필요 시 |
| Conditional Edge | 조건부 라우팅, 결정 함수 | 동적 흐름 제어 |
| Send API | 팬아웃, 병렬 실행 | 배치 처리, 병렬화 |
| Command | 명시적 라우팅, goto | 노드 내부 제어 |

## 💡 실전 활용 팁

### 언제 어떤 패턴을 사용할까?

**데이터 파이프라인**
- State (02) + Reducer Functions (04)
- 데이터 변환과 누적

**고객 서비스 봇**
- Conditional Edge (06) + Command (08)
- 사용자 의도 분류 및 라우팅

**배치 처리 시스템**
- Send API (07) + Reducer Functions (04)
- 대량 데이터 병렬 처리

**API 서비스**
- Multiple Schemas (03) + Node Caching (05)
- 깔끔한 인터페이스와 성능 최적화

## 🔗 참고 자료

- [LangGraph 공식 문서](https://langchain-ai.github.io/langgraph/)
- [LangGraph GitHub](https://github.com/langchain-ai/langgraph)
- [프로젝트 CLAUDE.md](../CLAUDE.md) - 프로젝트 전체 가이드

## 📝 기여 및 피드백

문서 개선 제안이나 오류 발견 시 이슈를 남겨주세요.

---

**생성일**: 2025-11-03
**기반 노트북**: `basic/` 디렉토리
