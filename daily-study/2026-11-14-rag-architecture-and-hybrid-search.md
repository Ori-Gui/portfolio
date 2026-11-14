# [2026-11-14] RAG(검색 증강 생성) 아키텍처와 Hybrid Search·Reranking 전략

> **한 줄 핵심 요약**: RAG는 LLM의 학습 데이터 한계(최신성 부족, 도메인 지식 부재, 환각)를 외부 벡터 DB 검색으로 보완하는 아키텍처이며, 실무에서는 단순 벡터 유사도 검색의 한계를 넘기 위해 **청킹(Chunking) 최적화 + 키워드(BM25)·벡터 하이브리드 검색(RRF) + Reranker** 파이프라인을 조합합니다.

---

## 1. 개념 및 동작 원리

### 1.1 왜 RAG(Retrieval-Augmented Generation)인가?
LLM은 파인튜닝(Fine-tuning)만으로는 실시간으로 변하는 사내 문서, 정책, 상품 재고 등을 반영하기 어렵고 비용도 큽니다. RAG는 **질문 시점에 관련 문서를 검색(Retrieval)해 프롬프트 컨텍스트에 주입(Augmentation)한 뒤 답변을 생성(Generation)**함으로써 다음 세 가지를 해결합니다.

1. **최신성(Freshness)**: 모델 재학습 없이 DB 색인만 갱신하면 최신 정보 반영 가능
2. **근거 기반 응답(Grounding)**: 출처 문서를 함께 반환해 환각(Hallucination) 검증 가능
3. **접근 제어(Access Control)**: 사용자 권한에 따라 검색 대상 문서를 필터링(Metadata Filtering) 가능

### 1.2 전체 파이프라인 구조
RAG는 크게 **오프라인 색인(Indexing)** 과 **온라인 질의(Query)** 두 단계로 나뉩니다.

```text
[오프라인 색인 파이프라인]
원본 문서 (PDF/MD/DB)
  └─▶ 텍스트 정제 & 청킹 (Chunking + Overlap)
        └─▶ 임베딩 모델 (Embedding Model -> 고차원 벡터)
              └─▶ Vector DB 저장 (벡터 + 원문 텍스트 + 메타데이터)

[온라인 질의 파이프라인]
사용자 질문
  ├─▶ (경로 A) Dense Search: 질문 임베딩 -> 코사인 유사도 Top-K
  ├─▶ (경로 B) Sparse Search: 형태소/키워드(BM25) 매칭 Top-K
  └─▶ RRF(Reciprocal Rank Fusion) 병합
        └─▶ Cross-Encoder Reranker 재정렬 (Top-N 축소)
              └─▶ LLM 프롬프트 주입 -> 근거 포함 답변 생성
```

---

## 2. 실무 구현: FastAPI 기반 Hybrid Search & RRF 예시

실무에서 벡터 검색(Dense)만 쓰면 고유명사·에러 코드·정확한 법령 조항("제14조 2항") 같은 **정확 일치(Exact Match)** 에 약하고, 키워드 검색(BM25)만 쓰면 **유의어·문맥적 의미**를 놓칩니다. 두 결과를 **RRF(Reciprocal Rank Fusion)** 로 합치면 튜닝 부담 없이 검색 정확도를 안정적으로 끌어올릴 수 있습니다.

```python
# app/services/rag_retriever.py
# 의존성: fastapi==0.115.0, pydantic==2.9.2, qdrant-client==1.12.0, rank-bm25==0.2.2

from dataclasses import dataclass
from typing import Dict, List


@dataclass(frozen=True)
class DocumentChunk:
    chunk_id: str
    content: str
    source: str
    page: int


def reciprocal_rank_fusion(
    dense_results: List[DocumentChunk],
    sparse_results: List[DocumentChunk],
    k: int = 60,
    top_n: int = 5,
) -> List[DocumentChunk]:
    """
    Dense(벡터) 검색 순위와 Sparse(BM25) 검색 순위를 RRF 공식으로 결합합니다.
    score(d) = sum(1 / (k + rank_i(d)))
    서로 다른 점수 스케일(코사인 유사도 0~1 vs BM25 0~∞)을 정규화 없이 공정하게 합산할 수 있습니다.
    """
    scores: Dict[str, float] = {}
    doc_map: Dict[str, DocumentChunk] = {}

    for rank, doc in enumerate(dense_results, start=1):
        doc_map[doc.chunk_id] = doc
        scores[doc.chunk_id] = scores.get(doc.chunk_id, 0.0) + 1.0 / (k + rank)

    for rank, doc in enumerate(sparse_results, start=1):
        doc_map[doc.chunk_id] = doc
        scores[doc.chunk_id] = scores.get(doc.chunk_id, 0.0) + 1.0 / (k + rank)

    sorted_ids = sorted(scores.keys(), key=lambda cid: scores[cid], reverse=True)
    return [doc_map[cid] for cid in sorted_ids[:top_n]]
```

### 2.1 Bi-Encoder vs Cross-Encoder (Reranker)
| 구분 | Bi-Encoder (1차 검색) | Cross-Encoder (2차 Reranking) |
| --- | --- | --- |
| **입력 방식** | 질문과 문서를 각각 독립 벡터로 변환 | `[질문 + 문서]` 쌍을 함께 모델에 입력 |
| **연산 비용** | 매우 빠름 (문서 벡터 사전 계산 가능) | 느림 (실시간 추론 필요) |
| **정확도** | 토큰 간 세부 상호작용 반영 한계 | 질문-문서 간 Attention 직접 계산 → 매우 높음 |
| **역할** | 수만 건 중 후보 Top-30~50 빠르게 추출 | 후보 30건 중 진짜 관련 있는 Top-3~5만 정밀 선별 |

---

## 3. 실무 적용 및 주의사항 (Edge Cases)

1. **청킹(Chunking) 크기와 문맥 단절**
   - 청크가 너무 작으면(예: 100자) 앞뒤 주어나 전제 조건이 잘려 의미가 왜곡되고, 너무 크면(예: 2,000자) 노이즈가 섞여 임베딩 벡터의 변별력이 떨어집니다.
   - 일반적으로 **300~600 토큰 + 10~20% 오버랩(Overlap)** 을 시작점으로 삼고, 마크다운 헤더(`#`, `##`)나 문단 경계를 우선 보존하는 Recursive Character Splitting을 사용합니다.

2. **Lost in the Middle 현상**
   - LLM은 컨텍스트 창이 길어질수록 프롬프트의 **맨 앞과 맨 뒤에 있는 문서**는 잘 참고하지만, 중간에 끼어든 문서는 무시하는 경향이 있습니다.
   - 따라서 Top-K 문서를 무작정 많이(예: 20개) 넣기보다, **Reranker로 3~5개로 압축**하고 관련도가 가장 높은 문서를 맨 앞·맨 뒤에 배치하는 전략이 효과적입니다.

3. **메타데이터 필터링(Pre-filtering vs Post-filtering)**
   - 예: "2026년 11월 공고만 검색" 조건을 벡터 검색 **이후(Post)** 에 걸면, Top-10을 뽑은 뒤 전부 탈락해 결과가 0건이 될 수 있습니다.
   - Qdrant, pgvector, Elasticsearch 등에서는 검색 **이전(Pre-filtering)** 에 메타데이터 조건으로 후보군을 좁힌 뒤 ANN(근사 최근접 이웃) 탐색을 수행하도록 설계해야 합니다.

4. **원본 문서 수정·삭제 시 고아 벡터(Orphan Vector) 정합성**
   - RDB에서 게시글이나 정책 문서를 수정·삭제했는데 Vector DB의 기존 청크 벡터를 지우지 않으면, 삭제된 구버전 내용이 답변에 계속 인용됩니다.
   - 반드시 `document_id`를 메타데이터에 기록해 두고, 갱신 시 **기존 `document_id` 벡터 일괄 삭제 후 재색인(또는 Outbox 패턴 기반 비동기 동기화)** 을 보장해야 합니다.

5. **프롬프트 인젝션 및 권한 우회**
   - 검색된 외부 문서 안에 악의적인 지시문("이전 지시를 무시하고...")이 포함될 수 있으므로, 컨텍스트 영역을 명확한 구분자(`<context>...</context>`)로 감싸고 시스템 프롬프트에서 데이터로만 취급하도록 제한해야 합니다.

---

## 4. 기술 면접 단골 꼬리 질문 & 답변 팁

- **Q. RAG에서 검색 품질이 기대보다 낮을 때, LLM 모델 교체 외에 어떤 순서로 개선하나요?**
  - **A.** "먼저 **검색(Retrieval) 단계와 생성(Generation) 단계를 분리해서 평가**합니다. 검색된 청크 자체에 정답이 없다면 LLM을 바꿔도 소용없으므로, 1) 청킹 크기 및 문단 경계 조정, 2) 키워드(BM25) + 벡터 하이브리드 검색(RRF) 도입, 3) Cross-Encoder Reranker 추가 순으로 Recall과 Precision을 먼저 끌어올린 뒤 프롬프트를 튜닝합니다."

- **Q. Vector DB의 HNSW 인덱스는 무엇이고, 왜 RDB LIKE 검색 대신 사용하나요?**
  - **A.** "고차원 임베딩 벡터 간 코사인 유사도를 전수 조사(Brute-force)하면 데이터 증가 시 $O(N)$ 으로 느려집니다. HNSW(Hierarchical Navigable Small World)는 계층적 그래프 구조를 만들어 근사 최근접 이웃(ANN)을 $O(\log N)$ 수준으로 빠르게 찾아주므로 실시간 RAG 응답 속도를 확보할 수 있습니다."

- **Q. RDB와 Vector DB 간 데이터 정합성은 어떻게 유지하나요?**
  - **A.** "원본 문서 CRUD가 일어나는 트랜잭션 내에서 Transactional Outbox 패턴으로 변경 이벤트를 기록하고, 워커가 이를 소비해 `document_id` 단위로 기존 벡터를 삭제 후 재임베딩·업서트(Upsert)하도록 구성해 부분 실패 시에도 재시도가 가능하게 설계합니다."
