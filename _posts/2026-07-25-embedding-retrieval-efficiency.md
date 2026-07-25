---
layout: post
title: "임베딩 검색 효율 높이기: BM25 하이브리드, RRF, 리랭킹, 양자화"
date: 2026-07-25 00:00:00 +0900
categories: [AI, RAG]
tags: [Embedding, BM25, RRF, HybridSearch, Reranker, Qdrant, 양자화, AI]
description: "임베딩 단일 검색의 기하학적 한계(딥마인드 LIMIT)와 BM25 하이브리드, RRF 융합, 크로스 인코더 리랭킹, 벡터 양자화까지 한국어 기준으로 정리한다."
---

## 1. 배경: 왜 임베딩만으로는 부족한가

### 딥마인드 LIMIT 논문 (2025)

**On the Theoretical Limitations of Embedding-Based Retrieval**

- 논문: <https://arxiv.org/pdf/2508.21038>
- 코드·데이터셋: <https://github.com/google-deepmind/limit>

핵심 내용:

- 단일 벡터 임베딩은 차원 수 d에 의해 표현 가능한 top-k 문서 조합의 수가 수학적으로 제한됨 (기하학적 한계)
- 512차원은 약 50만 문서 규모에서 검색 성능 붕괴
- 단 46개 문서짜리 LIMIT small에서도 최신 임베딩 모델이 완전한 recall 실패
- BM25(희소 표현)는 이 한계를 받지 않음
- 대안: 하이브리드 검색, 크로스 인코더, 멀티벡터(ColBERT)

### BM25의 반대편 약점

- 원저자 공식 문헌: Robertson & Zaragoza (2009), *The Probabilistic Relevance Framework: BM25 and Beyond* — <https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf>
- Elastic 공식 설명: <https://www.elastic.co/search-labs/blog/improving-information-retrieval-elastic-stack-search-relevance>

핵심 한계:

- **어휘 불일치(vocabulary mismatch)**: 쿼리와 문서의 단어가 다르면 매칭 실패
- 동의어·문맥 등 의미론적 이해 없음 (순수 통계 기반 희소 검색)
- 멀티벡터와는 무관한 별개 기술

### 결론

두 약점이 상보적이므로 **BM25 + 임베딩 하이브리드가 표준**. 그리고 양쪽 모두 품질의 대부분은 **색인 시점(인덱스 설계)** 에 결정된다.

> 이는 벤더 문서에서도 동일하게 확인된다. BGE-M3 공식 모델 카드는 RAG 검색 파이프라인으로 "하이브리드 검색 + 리랭킹"을 명시적으로 권장하며, 그 근거로 여러 방식의 강점을 합쳐 정확도와 일반화 성능을 높인다는 점을 든다. Elastic 공식 문서 역시 하이브리드 검색을 RRF로 구현할 것을 권장한다.
> — <https://huggingface.co/BAAI/bge-m3> , <https://www.elastic.co/docs/solutions/search/hybrid-search>

---

## 2. 인덱스 설계 잘하는 법

### 공통 원칙 (스택 무관)

1. **파라미터 튜닝보다 인덱스/쿼리 설계 먼저** — Elastic 공식 결론: k1=1.2, b=0.75 기본값은 대부분 충분. 부스팅, 동의어, 어간 추출이 우선
2. **필드 구조 설계** — 문서를 title / body / category 등으로 분리, 필드별 가중치 부여 (BM25F 원리)
3. **애널라이저 설계** — 토크나이저(한국어는 형태소 분석 필수), 어간 추출(효과 있음), 불용어 제거(효과 없음 — Trotman 2014), 동의어 사전
4. **튜닝은 마지막에, 검증 세트 지표(recall/MAP) 기반으로** — 그리드 서치 범위: b = 0~1, k1 = 0~3

### Elasticsearch

공식 문서: **Practical BM25 3부작**

| Part | 주제 | 링크 |
| --- | --- | --- |
| 1 | 샤드가 점수에 미치는 영향 (샤드별 IDF 왜곡) | <https://www.elastic.co/blog/practical-bm25-part-1-how-shards-affect-relevance-scoring-in-elasticsearch> |
| 2 | BM25 공식과 변수(k1, b) 동작 원리 | <https://www.elastic.co/blog/practical-bm25-part-2-the-bm25-algorithm-and-its-variables> |
| 3 | k1, b 선택 기준 — "튜닝은 마지막 수단" | <https://www.elastic.co/blog/practical-bm25-part-3-considerations-for-picking-b-and-k1-in-elasticsearch> |

설계 체크리스트:

- 매핑: text(전문 검색) vs keyword(정확 일치) 구분
- 애널라이저: 한국어는 Nori 형태소 분석기
- 부스팅/동의어/퍼지 매칭을 bool 쿼리로 먼저 해결
- 샤드 수 최소화 또는 샤드별 IDF 통계 왜곡 인지

### Neo4j

공식 문서 3종:

| 문서 | 용도 | 링크 |
| --- | --- | --- |
| Cypher Manual — Full-text indexes | 인덱스 생성·질의·애널라이저 설정 | <https://neo4j.com/docs/cypher-manual/current/indexes/semantic-indexes/full-text-indexes/> |
| Operations Manual — Index configuration | 인덱스 유형 비교, 성능 설정 | <https://neo4j.com/docs/operations-manual/current/performance/index-configuration/> |
| Java Reference — Analyzer provider | 커스텀 애널라이저 확장 | <https://neo4j.com/docs/java-reference/current/extending-neo4j/full-text-analyzer-provider/> |

핵심 포인트:

- **내부가 Apache Lucene** → Elasticsearch와 같은 원리(BM25 스코어링, 애널라이저) 적용
- 인덱스 유형: range / text(단순 문자열 일치) / **full-text**(애널라이저 기반 관련도 검색) / **vector**(임베딩)
- 기본 애널라이저는 `standard-no-stop-words`, `dbms.index.fulltext.default_analyzer`로 변경 (인덱스 생성 시점에만 적용)
- 한국어: `db.index.fulltext.listAvailableAnalyzers()`로 nori 지원 확인 → 없으면 AnalyzerProvider 직접 구현
- 프로퍼티별 다른 애널라이저 필요 시 Composite 애널라이저

### 검색 엔진 밖에서 BM25를 돌릴 때 (신규)

Elasticsearch나 Neo4j를 쓰지 않고 애플리케이션 레벨에서 sparse 검색을 구성하는 경우, 토크나이저가 sparse 품질 전체를 좌우한다. 공백 분리만으로는 조사·어미 때문에 매칭이 그대로 깨진다.

- 형태소 분석기 + BM25 조합: Kiwi, Kkma, Okt (LangChain 한국어 튜토리얼이 별도 항목으로 다룸 — <https://wikidocs.net/253434>)
- 대안: BM25 인덱스를 따로 두지 않고 **BGE-M3의 lexical weights**를 사용. 공식 모델 카드에 따르면 dense 임베딩 생성 시 추가 비용 없이 BM25 유사한 토큰 가중치를 함께 얻을 수 있다. 인덱스를 2개 운영할 여력이 없을 때 유효.

---

## 3. 임베딩 모델 선택 — 한국어 기준 (신규)

인덱스 설계가 상한선을 정한다면, 임베딩 모델은 그 상한선까지 도달할 수 있는지를 정한다. 한국어에서는 글로벌 리더보드 순위와 실제 성능이 어긋나는 구간이 뚜렷하다.

### MTEB 한국어 retrieval 벤치마크

고려대 NLP&AI 연구실 KURE 공식 저장소가 MTEB에 등록된 한국어 retrieval 벤치마크 전체(Ko-StrategyQA, AutoRAGRetrieval, MIRACL, PublicHealthQA, Belebele, Mr.TyDi, MultiLongDocRetrieval, XPQA)로 평가한 결과. Top-k 5 기준 평균 NDCG:

| 모델 | Avg NDCG@5 |
| --- | --- |
| nlpai-lab/KURE-v1 | 0.67479 |
| dragonkue/BGE-m3-ko | 0.66692 |
| BAAI/bge-m3 | 0.66615 |
| Snowflake/snowflake-arctic-embed-l-v2.0 | 0.66236 |
| intfloat/multilingual-e5-large | 0.64402 |
| BAAI/bge-multilingual-gemma2 | 0.63338 |
| openai/text-embedding-3-large | 0.59389 |

출처: <https://github.com/nlpai-lab/KURE>

**읽어야 할 지점**: OpenAI text-embedding-3-large가 하위권이다. 한국어 코퍼스에서 상용 API를 검증 없이 기본값으로 두는 판단은 근거가 약하다.

### 선택지별 성격

| 모델 | 차원 | 성격 | 언제 쓰나 |
| --- | --- | --- | --- |
| nlpai-lab/KURE-v1 | 1024 | 한국어 전용 dense, MIT | 한국어 단일 언어, dense 중심 |
| BAAI/bge-m3 | 1024 | dense+sparse+multi-vector, 8192 토큰, 100+ 언어 | 하이브리드를 한 모델로 처리 |
| Qwen3-Embedding (0.6B/4B/8B) | 가변 | 인스트럭션 인식, 100+ 언어 | 다국어 확장 예정, 차원 조절 필요 |

- **KURE-v1**: BAAI/bge-m3를 기반으로 파인튜닝. 한국어 query-document-hard negative(5개) 쌍 약 200만 건으로 학습. 라이선스 MIT. 단, **dense 전용**이므로 sparse가 필요하면 BM25 인덱스를 별도로 붙여야 한다.
- **BGE-M3**: 공식 카드 기준 dense retrieval, multi-vector(ColBERT), sparse(lexical matching)를 동시에 수행하고, 짧은 문장부터 8192 토큰 장문까지 처리. `compute_score`의 `weights_for_different_modes`로 dense·sparse·colbert 점수의 가중합을 지정할 수 있으며 문서 예시 가중치는 `[0.4, 0.2, 0.4]`. → LIMIT 논문이 제시한 대안 3개(하이브리드, 멀티벡터, 크로스 인코더) 중 앞의 둘을 한 모델로 커버한다.
- **Qwen3-Embedding**: 8B가 2025-06-05 기준 MTEB 다국어 리더보드 1위(70.58). 인스트럭션 사용 시 대부분의 downstream 태스크에서 1~5% 개선(자체 평가). 출력 차원을 지정할 수 있어 저장 비용 조절이 가능. 단 위 한국어 표에는 Qwen3 계열이 포함되어 있지 않으므로 도메인 데이터로 직접 재측정 필요.

### 주의사항

- **KoE5는 prefix 필수**: `query: {query}` / `passage: {document}`. 누락 시 성능이 조용히 떨어진다.
- 위 표는 공개 벤치마크 평균값이다. KURE 저장소는 `eval/evaluate.py`에 모델을 추가해 직접 평가하는 코드를 제공하므로, 실제 도메인 데이터로 재측정하는 편이 신뢰할 만하다.
- 모델 교체는 **재색인**을 의미한다. 인덱스 설계 확정 → 모델 확정 → 색인 순서를 지킬 것.

---

## 4. 최종 아키텍처

```text
문서 → [인덱스 설계: 필드 분리 + 형태소 분석 + 동의어]
     → full-text 인덱스 (BM25, 어휘 정확성 담당)
     → vector 인덱스 (임베딩, 의미 이해 담당)
쿼리 → 두 인덱스 병렬 검색 → 점수 융합(RRF) → (선택) 리랭커
```

### 4-1. 점수 융합 — 왜 필요한가

BM25와 벡터 검색은 **점수의 단위가 다르다.**

- BM25 점수: 상한 없는 값 (3.2, 15.7, 42.1...) — 코퍼스/쿼리에 따라 스케일 변동
- 벡터 유사도: 보통 0~1 사이 코사인 유사도

단순 합산하면 BM25가 압도하고, min-max 정규화도 쿼리마다 분포가 달라 불안정하다.

### 4-2. RRF (Reciprocal Rank Fusion)

**점수를 버리고 각 목록에서의 등수만 사용**하는 융합 방식.

```text
RRF(d) = Σ 1 / (k + rank_i(d))     (k = 60)
```

예시 (k=60):

| 문서 | BM25 등수 | 벡터 등수 | RRF 점수 |
| --- | --- | --- | --- |
| A | 1위 | 5위 | 1/61 + 1/65 = 0.0318 |
| B | 3위 | 2위 | 1/63 + 1/62 = 0.0320 |
| C | 2위 | 없음 | 1/62 = 0.0161 |

→ 최종 순위 **B > A > C**: 한쪽 1위(A)보다 양쪽 고른 상위권(B)이 이긴다.

RRF가 표준인 이유:

- 등수만 쓰므로 점수 단위 문제가 사라짐 (몇 개 목록이든 공평하게 융합)
- 별도 튜닝이 필요 없고, 결합하는 관련성 지표들이 서로 연관되어 있지 않아도 동작
- k가 클수록 하위 순위 문서의 영향력이 커져 한쪽 목록의 독식 방지
- Elasticsearch, OpenSearch, Azure AI Search 등의 하이브리드 기본 융합 방식

#### 공식 파라미터 (Elasticsearch RRF retriever)

출처: <https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion>

| 파라미터 | 의미 | 기본값 | 제약 |
| --- | --- | --- | --- |
| `rank_constant` | 위 공식의 k. 클수록 하위 순위 문서 영향력 증가 | **60** | ≥ 1 |
| `rank_window_size` | 개별 결과 집합의 크기 | **10** | ≥ size, ≥ 1 |
| `weight` (하위 검색기별) | 최종 랭킹에 대한 각 검색기의 영향력 | 균등 | — |

**운영상 함정**: `rank_window_size` 기본값 10을 그대로 두고 리랭커를 붙이면 후보가 10개밖에 안 남는다. 이 값은 개별 결과 집합의 크기를 결정하고 최종 결과는 size로 잘려나가므로, 리랭커에 넣을 후보 수(50~100)만큼 반드시 올려 잡아야 한다. 값이 클수록 관련성은 좋아지지만 성능 비용이 발생한다.

대안 — 가중 선형 결합: 정규화 후 `α·BM25 + (1-α)·벡터`. α 튜닝 필요, 점수 분포에 민감. 도메인상 의도적 가중이 필요할 때 사용. Elasticsearch에서는 RRF를 버리지 않고 하위 검색기에 `weight`를 주는 방식(weighted RRF)으로도 같은 목적을 달성할 수 있다.

### 4-3. (선택) 리랭커

RRF 상위 결과(top 50~100)를 **크로스 인코더**로 재채점하는 단계.

- **바이 인코더(임베딩)**: 쿼리·문서를 따로 벡터화 후 유사도 계산 → 빠르지만 LIMIT의 기하학적 한계 보유
- **크로스 인코더**: 쿼리+문서를 한 쌍으로 모델에 입력, 토큰 단위 상호작용 전부 반영 → 훨씬 정확하지만 쌍마다 추론 필요 (전체 코퍼스 적용 불가)

따라서 **깔때기(funnel) 구조**로 사용:

```text
1차 검색 (빠르고 넓게): BM25 + 벡터 → RRF → top 100
2차 리랭킹 (느리고 정밀하게): 크로스 인코더로 top 100만 재채점 → top 10
```

"(선택)"인 이유: 지연 시간·비용 추가. 단, RAG처럼 top 5 정밀도가 답변 품질에 직결되는 경우 가치가 크다.

대표 선택지: Cohere Rerank, BGE-reranker(오픈소스), Jina Reranker, ColBERT 계열(멀티벡터 late interaction — 속도/정확도의 중간 지점).

**한국어 기본값**: BGE-M3 공식 카드가 후속 필터링용으로 직접 지목하는 것은 bge-reranker 및 bge-reranker-v2 계열이다. 한국어 전용 리랭커는 임베딩만큼 선택지가 넓지 않으므로 `BAAI/bge-reranker-v2-m3`(다국어)가 현실적인 출발점.

---

## 5. 벡터 인덱스 최적화 — 양자화 (신규)

인덱스 설계가 *품질*의 상한선이라면, 양자화는 *운영 비용*의 상한선을 정한다. 품질을 깎지 않고 메모리·지연시간을 줄이는 단계.

출처: <https://qdrant.tech/documentation/manage-data/quantization/> , <https://qdrant.tech/articles/vector-search-resource-optimization/>

### 방식별 성격

| 방식 | 압축 | 특징 |
| --- | --- | --- |
| 스칼라 (int8) | 메모리 75% 절감 | float32 성분을 uint8로. 압축·성능 균형이 좋아 대부분의 경우 우선 선택지 |
| 이진 (1bit) | 최대 압축 | rescoring 없이는 품질 저하 큼. 저차원에서 정보 손실 심각 |
| 이진 (2bit) | 16배 | 0 근처 값을 명시적으로 표현해 저차원 벡터 성능 개선 |
| 이진 (1.5bit) | 24배 | 중간 정확도 |

### 정확도 회복: oversampling + rescoring

양자화 벡터로 후보를 넓게 뽑고(oversampling), 원본 벡터로 재평가(rescoring)해 최종 결과를 다듬는 2단계 구조. **4-3의 리랭킹 깔때기와 정확히 같은 발상이 벡터 인덱스 레벨에서 반복된다.**

공식 문서상 주의점:

- 이진 양자화는 **반드시 rescoring과 함께** 쓸 것이 권장된다. 다만 원본 벡터가 디스크에 있으면 rescoring이 검색 속도를 크게 떨어뜨린다.
- 기본적으로 rescoring이 켜져 있는 것은 이진 양자화와 TurboQuant 1/1.5/2비트뿐이다. 나머지 방식은 기본적으로 rescore하지 않는다.
- 1비트 압축은 **1000차원 미만 벡터에서 정보 손실과 정확도 저하가 크다.**

### 한국어 스택에 대한 함의

KURE-v1과 BGE-M3는 모두 **1024차원**으로, 이진 양자화 권장 하한선에 딱 걸친다. 따라서:

1. 스칼라(int8)부터 시작 — 메모리 75% 절감을 안전하게 확보
2. 그래도 부족하면 이진 + rescoring, 단 원본 벡터를 RAM에 둘 수 있는지 먼저 확인
3. Qwen3-Embedding처럼 출력 차원을 줄일 수 있는 모델을 쓰면서 동시에 이진 양자화를 적용하는 조합은 피할 것 (차원 축소 + 1비트 압축은 손실이 중첩된다)

---

## 6. 역할 분담 요약

| 단계 | 담당 | 강점 | 약점 보완 |
| --- | --- | --- | --- |
| 인덱스 설계 | 상한선 결정 | 필드 분리·형태소 분석·동의어 | 이후 모든 단계의 천장 |
| BM25 | 어휘 정확성 | LIMIT 한계 없음 | 의미 이해 없음 → 벡터가 보완 |
| 벡터 검색 | 의미 이해 | 동의어·의도 매칭 | 기하학적 한계 → BM25가 보완 |
| 임베딩 모델 선택 | 벡터 품질 | 한국어는 KURE-v1 / BGE-M3 | 글로벌 리더보드 ≠ 한국어 성능 |
| RRF | 공평한 통합 | 단위 문제 해소, 무튜닝(k=60) | rank_window_size 기본 10 함정 |
| 리랭커 | 최종 정밀도 | 토큰 단위 상호작용 | 느림 → 상위 후보에만 적용 |
| 양자화 | 운영 비용 | 스칼라 int8로 메모리 75%↓ | 정확도 손실 → oversampling+rescore |

> **한 줄 요약**: 임베딩의 기하학적 한계는 BM25가, BM25의 어휘 불일치는 임베딩이 보완하며, 인덱스 설계가 양쪽의 상한선을 결정한다. 그 위에서 모델 선택이 도달 가능한 품질을, 양자화가 감당 가능한 비용을 정한다.

---

## 부록: 참고한 공식 문서

| 주제 | 출처 |
| --- | --- |
| 한국어 임베딩 벤치마크·KURE | <https://github.com/nlpai-lab/KURE> |
| BGE-M3 모델 카드 (하이브리드+리랭킹 권장) | <https://huggingface.co/BAAI/bge-m3> |
| Qwen3-Embedding | <https://github.com/QwenLM/Qwen3-Embedding> |
| RRF 공식·파라미터 | <https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion> |
| RRF retriever 스펙 | <https://www.elastic.co/docs/reference/elasticsearch/rest-apis/retrievers/rrf-retriever> |
| 하이브리드 검색 권장 | <https://www.elastic.co/docs/solutions/search/hybrid-search> |
| 벡터 양자화 | <https://qdrant.tech/documentation/manage-data/quantization/> |
