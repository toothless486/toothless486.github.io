---
layout: post
title: "임베딩 생성 극대화: 속도·품질·저장 3가지 지표 (BGE-M3 + TEI + Qdrant)"
date: 2026-07-25 00:00:00 +0900
categories: [AI, RAG]
tags: [Embedding, BGE-M3, TEI, Qdrant, RAG, GPU, 최적화, AI]
description: "자체 호스팅 임베딩 생성을 속도·처리량, 품질·정확도, 저장 효율 세 축으로 극대화하는 방법을 공식 문서 기반으로 정리한다."
---

> 대상 구성: **전부 자체 호스팅 · 무료(오픈소스) · NVIDIA GPU 서버 · 한국어 회사 문서**
> 기본 스택: **BGE-M3(모델) + TEI(서빙) + Qdrant(저장)**

임베딩 생성(텍스트 → 벡터)을 "극대화"한다는 것은 단순히 빠르게 만드는 것만이 아니다. 실제로는 **3개 지표**를 함께 끌어올리는 문제다.

| 지표 | 무엇을 극대화하는가 | 왜 중요한가 |
| --- | --- | --- |
| 1. 속도·처리량 | 초당·시간당 벡터 생성량 | 대량 문서 색인 시간·비용 |
| 2. 품질·정확도 | 벡터가 내용을 얼마나 잘 표현하는가 | **검색 만족도를 좌우 — RAG 답변 품질의 근원** |
| 3. 저장 효율 | 벡터당 저장·메모리 비용 | 운영 비용 상한선 |

> 흔한 오해: "임베딩 극대화 = 고속/대량"이라고만 생각하기 쉽다. 그러나 회사 문서 RAG에서는 **2번 품질 축**이 검색 결과의 체감 품질을 실제로 결정한다. 속도는 한 번 색인하면 끝이지만, 품질은 모든 질의에 계속 영향을 준다.

---

## 1. 속도·처리량

### 공식 문서

| 문서 | 링크 |
| --- | --- |
| Sentence Transformers — Speeding up Inference | <https://sbert.net/docs/sentence_transformer/usage/efficiency.html> |
| Hugging Face — Text Embeddings Inference (TEI) | <https://huggingface.co/docs/text-embeddings-inference/en/index> |
| TEI — Quick Tour | <https://huggingface.co/docs/text-embeddings-inference/en/quick_tour> |

### 핵심 원리

처리량의 대부분은 **배치 처리 + 동적 배칭**에서 나온다. 문장을 1개씩 넣으면 GPU가 놀고, 수백 개씩 묶어 넣으면 처리량이 수십 배 오른다.

#### 속도·처리량 기법 정리

| 기법 | 효과 | 속도/처리량 | 품질/정확도 | 저장효율 | 설명 |
| --- | --- | :---: | :---: | :---: | --- |
| 배치 처리(batching) | 가장 큼 | ✅ | — | — | 문장을 모아 한 번에 GPU 투입. 1개씩 vs 32~256개씩은 처리량 수십 배 차이 |
| 길이별 정렬(bucketing) | padding 낭비↓ | ✅ | — | — | 비슷한 길이끼리 묶으면 짧은 문장에 붙는 불필요한 패딩 토큰 계산 제거 |
| max_length 축소 | 연산량↓ | ✅ | — | — | 실제 필요보다 길게 자르지 않기. 어텐션은 길이의 제곱 비용 |
| Mixed precision (FP16/BF16) | 2배↑ | ✅ | — | ✅ | 품질 손실 거의 없이 속도·메모리 절감 |
| 추론 런타임 최적화 | 2~5배 | ✅ | — | — | ONNX Runtime, TensorRT, OpenVINO로 변환 / Flash Attention 적용 |
| int8 양자화 추론 | 추가↑ | ✅ | ⚠️ | ✅ | 가중치를 8비트로. 정확도 소폭 손실 |

> 범례: ✅ 해당 / ⚠️ 부정적 영향(품질 소폭↓) / — 무관
> 6개 모두 주 지표는 **속도·처리량**이며, Mixed precision·int8은 메모리 절감(저장효율)을 함께 얻는다. int8만 품질과 소폭 트레이드오프가 있다.
> 배치 처리·길이별 정렬은 TEI의 토큰 기반 동적 배칭이 내부적으로 자동 처리한다.

#### 백엔드별 속도 향상 (Sentence Transformers 공식 벤치마크)

| 환경 | 권장 방식 | 속도 향상 |
| --- | --- | --- |
| GPU, 짧은 텍스트(<500자) | ONNX-O4 | ~1.2x |
| GPU, 긴 텍스트 | float16 / bf16 | ~1.1x |
| CPU, 정확도 손실 허용 | OpenVINO int8 | ~2.5x |
| CPU, 손실 비허용 | OpenVINO 또는 ONNX | ~1.1x |

→ **본 구성은 GPU이므로 float16 기본**, 짧은 청크 위주면 ONNX-O4 검토.

### 실행 (TEI, GPU)

```bash
docker run --gpus all -p 8080:80 -v $PWD/tei-data:/data \
  ghcr.io/huggingface/text-embeddings-inference:latest \
  --model-id BAAI/bge-m3 \
  --dtype float16 \
  --max-batch-tokens 16384 \
  --auto-truncate
```

- `--dtype float16` — 품질 손실 거의 없이 GPU 속도·VRAM 이득 (공식 권장)
- `--max-batch-tokens` — **처리량의 핵심 손잡이.** VRAM에 맞춰 OOM 안 나는 선까지 올린다
- `--auto-truncate` — 8192 토큰 초과 입력 자동 절단(BGE-M3 최대 길이)
- 요청: `POST /embed` 에 `{"inputs": ["문장1", "문장2", ...]}` — **한 요청에 최대한 많이** 담아 배칭 효과를 살린다

### 처리량 극대화 체크리스트

- [ ] 문장 1개씩 X → 배열로 수백 개씩 전송 (TEI 동적 배칭이 자동 최적화)
- [ ] 클라이언트에서 동시 요청 여러 개(비동기/스레드)로 파이프라인을 채움
- [ ] `--max-batch-tokens`를 VRAM 한계까지 상향
- [ ] **캐싱**: 이미 임베딩한 청크는 해시로 스킵 → 재색인 낭비 방지
- [ ] **증분 색인**: 문서 변경분만 재임베딩

---

## 2. 품질·정확도

### 공식 문서

| 문서 | 링크 |
| --- | --- |
| Sentence Transformers — Computing Embeddings (프롬프트·정규화·encode_query/document) | <https://sbert.net/examples/sentence_transformer/applications/computing-embeddings/README.html> |
| Sentence Transformers — Training with Prompts | <https://sbert.net/examples/sentence_transformer/training/prompts/README.html> |
| Sentence Transformers — Training Overview (파인튜닝) | <https://sbert.net/docs/sentence_transformer/training_overview.html> |
| BAAI/bge-m3 모델 카드 | <https://huggingface.co/BAAI/bge-m3> |

> **가장 중요한 축.** 코드 한두 줄 차이로 검색 정확도가 조용히 무너지거나 살아난다. 색인은 한 번이지만 그 품질은 모든 질의에 영구히 영향을 준다.

### (1) 프롬프트/instruction 접두어 — 가장 흔한 실수 지점

- 일부 모델은 **쿼리와 문서에 서로 다른 접두어**를 요구한다.
  - E5 계열: 쿼리 `"query: "`, 문서 `"passage: "` — 빠뜨리면 성능이 **소리 없이** 하락
- 공식이 `encode_query()` / `encode_document()` 메서드를 별도 제공 → 각각의 프롬프트를 자동 적용
- **BGE-M3**: 일반 검색에서는 instruction 없이 사용 가능하지만, 쿼리 측 instruction 옵션이 있다. **반드시 모델 카드의 규칙을 확인**하고 색인·질의 양쪽에서 동일 규칙을 적용할 것.

### (2) 정규화 — 코사인 검색 시 사실상 필수

```python
model.encode(texts, normalize_embeddings=True)
```

- 벡터를 단위 길이로 정규화 → 코사인 유사도 검색의 전제
- 색인과 질의에서 **동일하게** 적용해야 함

### (3) Pooling 방식

- `mean`(기본)/`cls`/`weightedmean` 등 — 모델이 지정한 pooling을 그대로 따를 것
- 임의로 바꾸면 품질이 저하된다

### (4) 도메인 파인튜닝 — 품질 상한을 올림 (무료)

- 회사 문서의 **(질문 – 정답 청크) 쌍**으로 임베딩 모델을 직접 파인튜닝 → 범용 모델보다 사내 도메인에서 크게 개선
- 전부 오픈소스라 **GPU만 있으면 무료**
- 쌍 데이터가 없으면 LLM으로 각 청크에 대한 합성 질문을 생성해 학습 데이터 구축 가능
- 순서: 인덱스 설계 확정 → 모델·프롬프트 확정 → (필요 시) 파인튜닝 → 색인

---

## 3. 저장 효율

### 공식 문서

| 문서 | 링크 |
| --- | --- |
| Sentence Transformers — Matryoshka Embeddings | <https://sbert.net/examples/sentence_transformer/training/matryoshka/README.html> |
| Qdrant — Quantization | <https://qdrant.tech/documentation/manage-data/quantization/> |

> 품질을 깎지 않고 **저장·메모리 비용**만 줄이는 축. 인덱스 설계가 품질의 상한선이라면, 이 축은 운영 비용의 상한선을 정한다.

### (1) Matryoshka 임베딩 — 차원 축소

- 하나의 벡터를 **앞부분만 잘라(예: 1024 → 256)** 써도 성능이 유지되도록 학습된 방식
- 중요한 정보가 앞쪽 차원에 집중 → "적당한 절단은 성능 손실이 거의 없다"(공식)
- 생성은 한 번, 저장·검색은 짧은 차원으로 → 비용 대폭 절감

### (2) 양자화 (int8 / binary)

- 벡터 성분을 int8·1bit로 저장 → 메모리 대폭 절감
- Qdrant 스칼라(int8) 양자화: 메모리 **75% 절감**, 압축·성능 균형이 좋아 대부분의 경우 우선 선택지
- BGE-M3는 **1024차원** → 이진 양자화 권장 하한선에 걸침. **스칼라 int8부터 시작**이 안전
- 이진 양자화를 쓸 경우 **oversampling + rescoring** 필수 (양자화 벡터로 넓게 뽑고 원본으로 재평가)

### Qdrant 설정 요지

- 컬렉션 생성: `size=1024`, `distance=Cosine` (BGE-M3 dense 차원)
- 스칼라 int8 양자화 활성화
- 원본 벡터를 RAM에 둘 수 있는지 확인 후 rescoring 여부 결정

---

## 요약 표

| 지표 | 핵심 기법 | 공식 문서 | 본 구성(GPU·무료) 액션 | 무료/유료 |
| --- | --- | --- | --- | --- |
| **속도·처리량** | 동적 배칭, fp16, ONNX-O4, 캐싱 | Sentence Transformers Efficiency, TEI | TEI + `float16` + `max-batch-tokens` 상향 + 캐싱 | 무료(인프라 비용만) |
| **품질·정확도** | 프롬프트 접두어, 정규화, pooling 준수, 도메인 파인튜닝 | Computing Embeddings, Training with Prompts, Training Overview | 모델 카드 프롬프트 규칙 준수 + `normalize_embeddings=True` → 필요 시 회사 문서로 파인튜닝 | 무료 |
| **저장 효율** | Matryoshka 차원 축소, int8/binary 양자화 | Matryoshka Embeddings, Qdrant Quantization | Qdrant 스칼라 int8 양자화(메모리 75%↓), 부족 시 차원 축소 | 무료 |

> **한 줄 요약**: 임베딩 생성 극대화 = 속도·처리량(배칭·fp16) **+ 품질(프롬프트·정규화·파인튜닝) + 저장(Matryoshka·양자화)**. 회사 문서 RAG에서는 **품질 축이 검색 만족도를 좌우**하므로, 무료로 얻는 품질(프롬프트 규칙 준수 + 정규화)부터 챙긴 뒤 파인튜닝·양자화로 확장한다.

---

### 부록: 참고한 공식 문서

| 주제 | 출처 |
| --- | --- |
| 추론 속도 최적화(백엔드·양자화 벤치마크) | <https://sbert.net/docs/sentence_transformer/usage/efficiency.html> |
| 고속·대량 임베딩 서빙(TEI) | <https://huggingface.co/docs/text-embeddings-inference/en/index> |
| 프롬프트·정규화·encode_query/document | <https://sbert.net/examples/sentence_transformer/applications/computing-embeddings/README.html> |
| 프롬프트 학습 | <https://sbert.net/examples/sentence_transformer/training/prompts/README.html> |
| 임베딩 모델 파인튜닝 | <https://sbert.net/docs/sentence_transformer/training_overview.html> |
| Matryoshka(차원 효율) | <https://sbert.net/examples/sentence_transformer/training/matryoshka/README.html> |
| 벡터 양자화(저장 효율) | <https://qdrant.tech/documentation/manage-data/quantization/> |
| BGE-M3 모델 카드 | <https://huggingface.co/BAAI/bge-m3> |
