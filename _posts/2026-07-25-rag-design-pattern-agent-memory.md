---
layout: post
title: "RAG 디자인 패턴과 Agent Memory: Self-RAG부터 GraphRAG, Neo4j 메모리까지"
date: 2026-07-25 00:00:00 +0900
categories: [AI, RAG]
description: "LLM과 Advanced RAG의 한계, 이를 보완하는 Self/CRAG/Adaptive RAG 패턴, GraphRAG와 Knowledge Graph, 그리고 Neo4j 기반 Agent Memory 설계까지 정리한다."
---

## LLM의 한계점

긴 문장 처리와 학습 범위 외 정보 생성에 한계를 가짐

1. **환각 현상**
   1. 딥러닝 모델은 학습된 데이터 이외의 정보에 취약
   2. 답을 하기 위해, 학습하지 못한 내용에 대해서 거짓말이나 엉뚱한 대답을 함
2. **기억 불가**
   1. 사전학습 시에 받아들인 정보 외의 것은 배우지 못함
   2. 따라서 오늘 나와 대화하고 있는 LLM은 어제 나와 대화했던 내용을 기억하지 못함
3. **토큰 제한**
   1. 입력값의 길이가 길어지면 계산량이 크게 늘어나, 대부분의 모델이 길이를 제한하고 있음

---

## Advanced RAG의 한계점

사용자의 질문에 올바른 응답을 유도하기 위해서는 검색 단계를 개선해야 함.

- 사용자의 질문이 DB에 있는 정보들과 비교가 잘 되는 문장인지?
- 사용자의 질문이 자세하면 자세할수록 사용자의 질문에 유사한 힌트 문장을 가져올 수 있음
- 사용자에게 어떤 질문 형태를 강제할 수가 없음. 사용자는 자유로운 프롬프트를 입력할 수 있어야 함.

---

## RAG의 한계점을 보완하기 위한 Pattern

### Self-RAG

일단 답변을 생성하고 문서에 근거하는지 판단

참고: <https://www.baihezi.com/mirrors/langgraph/tutorials/rag/langgraph_self_rag/index.html>

1. **Retrieve (노드)**: 질문으로 관련 문서를 검색 (DB 검색)
2. **Grade (노드)**: 검색된 문서들이 질문과 관련 있는지 평가한 후, 관련 문서만 남김 (필터링)
3. **Docs relevant? (엣지)**: 관련 문서가 하나라도 있으면 Generate로, 없으면 Re-write question으로 보냄
4. **Generate (노드)**: 필터링된 문서를 컨텍스트로 답변을 생성
5. **Hallucinations? (엣지)**: 답변이 문서에 근거하지 않으면 Generate로 돌려보내 재생성
6. **Answers question? (엣지)**: 답변이 질문을 해결하면 최종 출력(Answer), 아니면 Re-write question으로 보냄
7. **Re-write question (노드)**: 질문을 검색에 유리한 형태로 재작성한 뒤 1번 Retrieve로 재귀

### CRAG (Corrective RAG)

무관한 문서가 검색되는 경우, 웹 검색을 통해 보정 (폴백)

참고: <https://www.baihezi.com/mirrors/langgraph/tutorials/rag/langgraph_crag/index.html>

### Adaptive RAG

VectorDB와 웹 검색 중 어떤 Data source를 참고해야할지 결정 + Self RAG + CRAG

참고: <https://www.baihezi.com/mirrors/langgraph/tutorials/rag/langgraph_adaptive_rag/index.html>

---

## RAG using VectorDB 의 한계점

### 청킹의 한계

청킹 과정에서 청크 사이즈에 따라서 문맥이 끊길 수 있음

**예시:** 질문 — "2025년 4월 가입한 VIP 고객의 해외 송금 수수료는 면제돼?"

```
CHUNK 1: "VIP 고객은 수수료 면제 대상이다."
CHUNK 2: "단, 해외 송금 수수료는 면제 대상이 아니다."
CHUNK 3: "2025년 3월 이후 가입자는 정책 적용 대상이다."
```

1. 1번 혹은 2번 혹은 3번 중 하나가 검색이 될 것임.
2. 1번이 검색이 되었다면 LLM의 답변은 "면제가 됩니다."
3. 전체 청크 문맥을 봤을 때, VIP 고객은 수수료 면제가 맞지만, 해외 송금 수수료는 면제 대상이 아니기 때문에, 4월에 가입하더라도 면제 대상이 아니다.
4. 답변의 오류 발생

### TOP-K 검색의 한계

- TOP-K: 가장 유사한 문서 K만큼 검색해서 LLM 이 답변 생성
- 두 개의 문서가 답변을 하는데 필요한, 힌트가 될만한 문장이 검색될거라는 보장이 없음

**예시:** K=2, 질문 — "김민수와 보안팀은 어떤 관계야?"

```
문서 1: "김민수는 결제 시스템 리팩터링을 담당했다."
문서 2: "결제 시스템 리팩터링은 장애율을 낮추기 위한 프로젝트 였다."
문서 3: "장애율 개선 프로젝트는 보안팀과 플랫폼팀이 공동으로 진행했다."
```

1. 문서 1, 3이 검색되었다고 가정
2. "장애율을 낮추기 위한 프로젝트인 결제 시스템 리팩터링"이란 문맥 정보를 LLM이 놓치게 됨. 질문에 대한 답변을 하기 어려워짐.

---

## GraphRAG

- Knowledge graph라는 엔티티간의 관계를 바탕으로 검색하는 시스템
- 데이터들 간의 관계를 구축

```
문서 1: "김민수는 결제 시스템 리팩터링을 담당했다."
문서 2: "결제 시스템 리팩터링은 장애율을 낮추기 위한 프로젝트 였다."
문서 3: "장애율 개선 프로젝트는 보안팀과 플랫폼팀이 공동으로 진행했다."
```

질문: "김민수와 보안팀은 어떤 관계야?"

LLM은 김민수와 보안팀과의 관계를 이 그래프를 보고 순회하며 찾아갈 수 있음

### Knowledge Graph (지식 그래프)

- **Node**: 대상, 개념 (명사)
- **Edge**: 관계 (동사)
- **Property**: 노드나 관계의 상세정보

놀랍게도 구글은 2012년부터 지식 그래프를 사용하고 있었음.
참고: <https://ko.wikipedia.org/wiki/구글_지식_그래프>

#### 벡터 vs 지식 그래프

- Vector 검색: 데이터 규모가 증가할 수록 검색하는 속도도 떨어짐
- 그래프 탐색: 데이터 규모가 증가하더라도 검색 속도는 상대적으로 빠름
- 그렇다고해서 벡터 db 기반의 RAG 시스템을 완전히 배척할 수는 없음. graph, vector를 상호 보완 관계로 RAG 시스템을 구축할 수 있음.

### Hybrid GraphRAG

- 1편 Getting Started: <https://neo4j.com/blog/developer/get-started-graphrag-python-package/>
- 2편 벡터 + 그래프 순회: <https://neo4j.com/blog/developer/graph-traversal-graphrag-python-package/>
- 3편 Hybrid Retrieval (검색·병합 다이어그램): <https://neo4j.com/blog/developer/hybrid-retrieval-graphrag-python-package/>
- 4편 Hybrid + 그래프 순회: <https://neo4j.com/blog/developer/enhancing-hybrid-retrieval-graphrag-python-package/>

| 리트리버 | 검색 방식 | "1375년 영화의 배우는?" 질문 결과 |
| --- | --- | --- |
| `VectorRetriever` (1편) | 벡터 | ❌ 영화도 배우도 못 찾음 |
| `VectorCypherRetriever` (2편) | 벡터 + 그래프 탐색 | ❌ 배우 쿼리는 있으나 영화를 못 찾음 |
| `HybridRetriever` (3편) | 벡터 + 전문 검색 | △ 영화는 찾지만 배우 정보 없음 |
| `HybridCypherRetriever` (4편) | 벡터 + 전문 + 그래프 탐색 | ✅ 영화와 배우 모두 반환 |

**전문 검색: 정확한 문자열 매칭**에 강하지만, 글자만 비슷한 문장을 잘못 매칭할 수 있음

| 비교 쌍 | 어휘적 유사성 | 의미적 유사성 | 유리한 검색 |
| --- | --- | --- | --- |
| "배가 아프다" / "배가 떠났다" | 높음 | 낮음 | 전문 인덱스의 오탐 가능 사례 |
| "값이 저렴하다" / "가격이 싸다" | 낮음 | 높음 | 벡터 검색 |
| "김민수" / "결제 시스템" | 정확 일치 필요 | | 전문 인덱스 |

### 그래프 스키마(데이터 모델링) 작성 가이드

- LLM으로 추출할 경우, 타입을 열어두면 `Person`, `HumanBeing`, `Individual`이 뒤섞인 지저분한 그래프가 나옴
- **허용 타입과 관계를 명시적으로 제한**하는 게 중요
- 스키마에 없는 유용한 정보는 버려지는 트레이드오프 존재

#### 가이드 출처

1. **Neo4j 공식 문서 — 데이터 모델링 허브** (출발점): <https://neo4j.com/docs/model/>
   - **기초 개념**: <https://neo4j.com/docs/getting-started/data-modeling/> — 데이터 모델과 노드, 레이블, 관계, 속성의 역할을 정의하는 입문 문서
   - **실습 튜토리얼**: <https://neo4j.com/docs/getting-started/data-modeling/tutorial-data-modeling/> — 실제 시나리오로 모델을 만드는 단계별 가이드
   - **모델링 설계 패턴**: <https://neo4j.com/docs/getting-started/data-modeling/modeling-designs/> — 중간 노드, 시간 트리 등 성능을 고려한 설계 전략. "속성 vs 노드 승격", "슈퍼노드 분산" 판단이 여기에 포함됨
   - **모델링 팁**: <https://neo4j.com/docs/getting-started/data-modeling/modeling-tips> — 모델 개선을 위한 실무 팁 모음
   - **관계형 → 그래프 전환**: <https://neo4j.com/docs/getting-started/data-modeling/relational-to-graph-modeling/> — 기존 RDB 스키마가 있을 때 유용
2. **문서 적재(KG 구축) 시 추출 스키마 정의 — neo4j-graphrag 공식 가이드**: <https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_kg_builder.html>
3. **GraphAcademy — Graph Data Modeling Fundamentals (무료 공식 강의)**: <https://graphacademy.neo4j.com/courses/modeling-fundamentals/>
   도메인 질문 정의, 노드/관계 도출, 리팩토링까지 모델링 전 과정을 실습 진행

### GraphRAG의 경량화

GraphRAG의 최대 약점인 인덱싱 비용 낮추기 위한 기술

인덱싱 비용: GraphRAG는 인덱싱 단계에서 LLM을 대량으로 호출

1. **엔티티·관계 추출**: 모든 청크를 LLM에 넣어 엔티티(인물, 조직, 개념 등)와 엔티티 간 관계를 추출하고, 각각에 대한 설명문까지 생성. 청크 하나당 최소 1회 이상의 LLM 호출이 발생
2. **엔티티 요약**: 여러 청크에 흩어져 등장한 동일 엔티티의 설명들을 LLM으로 통합 요약
3. **커뮤니티 탐지 및 요약**: 관련 노드들의 그룹을 만듬, **각 그룹마다 LLM으로 요약문을 생성**

- 오픈소스: HippoRAG 2, LightRAG
- Microsoft: LazyGraphRAG
- Microsoft의 LazyGraphRAG 기법을 LangChain에서 재현
  - 튜토리얼 문서: [LazyGraphRAG in LangChain](https://datastax.github.io/graph-rag/examples/lazy-graph-rag/)
  - GitHub 저장소: [datastax/graph-rag](https://github.com/datastax/graph-rag)

---

## 비교

| 구분 | Vector RAG | GraphRAG (Neo4j) | DataStax (LangChain) |
| --- | --- | --- | --- |
| **그래프 구조** | 없음 (청크 + 임베딩) | 타입 있는 엔티티·관계 지식그래프 | 문서 노드 + 메타데이터 엣지 (mentions/entities) |
| **인덱싱 LLM 사용** | X (임베딩만) | O — 엔티티·관계 추출 | X (spaCy NER / KeyBERT) |
| **인덱싱 비용** | 낮음 | **매우 높음** | 낮음 (벡터 RAG 수준) |
| **저장소** | 벡터 DB | Neo4j (영속 그래프 DB) | 기존 벡터 스토어 그대로 |
| **문서 갱신 대응** | 쉬움 (upsert) | 부담 큼 (증분 갱신 설계 필요) | 쉬움 (문서 단위 upsert) |
| **멀티홉 능력** | X | **의미 있는 관계 경로 추적 가능** | 문서 링크·공통 엔티티 순회 수준 |
| **구축 난이도** | 낮음 | **높음** (스키마 설계 + 파이프라인 + 운영) | 낮음~중간 (기존 스토어에 얹기) |

---

## Neo4j Agent Memory

**동일한 질문에는 RAG 파이프라인 전체를 실행하고 싶지 않다.**

- LLM이 자체적으로 기억할 수 있는 컨텍스트의 양(context window)
- 현재 대화중인 세션, 다른 세션과의 대화내용 중 핵심에 되는 내용
- 사용자의 지식 상태, 선호도, 특징 보존
- 사용자와 나눈 대화 순서, 어떤 핵심 엔티티가 언급되었나, 어떤 대화와 연결되는가
- 에이전트가 어떤 요청에서 어떤 선택을 했고, 어떤 작업을 했고, 어떤 도구를 호출했는지 또한 기억 가능

### Context Graph

- 지식 그래프를 통해 이미 대량의 데이터를 그래프 형태로 이미 저장했다고 했을 때, 그걸 전부 LLM으로 줄 순 없음. 지금 당장 AI 시스템이 어떤 정보를 필요로 하는지를 동적으로 탐색하여 전달해 줄 수 있어야 함.
- 현재 상황에 필요한 서브 그래프를 동적으로 구성하여 AI 시스템에 전달 가능
- 장기 기억, 대화를 위한 단기 기억, 의사 결정 추적을 위한 추론 기억의 세 가지 구성 요소를 포함하는 맥락 그래프
- 하나의 그래프에 세 개의 메모리 레이어 — 대화, 엔터티, 추론 — 이 있음

### Short-Long-Reasoning

**Short:**

- message 노드를 생성, Conversation 체인 연결
- 메시지 내용에서 엔티티 추출
- 엔티티를 장기 기억 계층으로 승격 및 저장
- 추출된 엔티티와 메시지를 MENTIONS/RELATED_TO 관계로 저장

**Long:**

- 사용자의 선호도, 사실정보와 같은 내용을 추출
- 주어, 서술어, 목적어는 팩트 노드로 저장
- 여기서 추출한 노드 + short 에서 승격된 노드

**Reasoning:**

- 에이전트가 어떤 요청에서 어떤 선택을 했고, 어떤 작업을 했고, 어떤 도구를 호출했는지 또한 기억 가능
- 왜 그런 결론에 도달했는지의 과정을 그래프로 구축
- 메시지에 대한 답변을 찾기위해 사용한 도구/쿼리 등을 reasoning 노드로 생성
- 에이전트가 잘 실행했는지 평가하기 위해 활용 가능

**엔티티 추출:**

- POLE+O 내장 모델로 엔티티 추출
- 여러 도메인 전용 따로 저장 가능: <https://create-context-graph.dev/docs/reference/domain-catalog>

---

## 원본 문서가 변경된 경우, 저장한 메모리와 다름 이슈

**핵심 설계: 출처 메타데이터를 함께 저장**

메모리 추출 로직이 문서 기반 답변에서 기억을 저장할 때, value에 출처를 함께 기록. 이게 있어야 나중에 "어떤 메모리가 어떤 문서에 의존하는지" 역추적이 가능

```python
store.put(
    (user_id, "memories"),
    memory_id,
    {
        "content": "A 기능 요금은 월 5만원으로 안내",
        "source_doc_id": "pricing-policy",   # 어느 문서에서 왔는지
        "source_version": "v3",              # 또는 콘텐츠 해시
    }
)
```

**무효화 실행: 문서 변경 이벤트에 핸들러 연결**

```
[동기화 잡: 스케줄 or Confluence 웹훅]
변경 감지 (첨부파일 해시 비교 / 페이지 버전 비교)
   ↓ 변경 확인된 문서에 대해서만
① 벡터 스토어 재인덱싱 (변경: 재임베딩 / 삭제: 청크 제거)
② 해당 문서에서 파생된 memory 제거 (provenance 필터로 조회)
```

**보조 수단들**

- **TTL**: 문서 파생 메모리에 짧은 TTL을 걸어두면 업뎃 이벤트를 놓쳐도 자연 소멸

**오픈 소스**

| 시스템 | 메타데이터 저장 | 비고 |
| --- | --- | --- |
| **LangGraph Store (+LangMem)** | value 필드에 자유 저장 | 지금까지 논의한 그대로. 문서별 namespace로 통삭제도 가능 |
| **Mem0 OSS** | `add(metadata={...})` | 백엔드 벡터스토어(pgvector, Qdrant 등) 선택 가능 |
| **Graphiti** | 구조적으로 내장 | provenance를 따로 설계할 필요가 없다는 게 차별점. temporal invalidation(삭제 대신 "이 사실은 X시점까지 유효" 마킹)도 선택 가능 |
| **Memori** | SQL 컬럼 | SQL 네이티브라 이 구조에서는 가장 단순무식하게 동작 |
