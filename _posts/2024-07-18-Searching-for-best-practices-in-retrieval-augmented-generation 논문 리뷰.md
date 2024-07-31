---
title: "Searching for Best Practices in Retrieval-Augmented Generation 논문 리뷰"
excerpt: "RAG 기법과 성능 비교"

categories:
  - papers

toc: true
toc_sticky: true

date: 2024-07-18
last_modified_at: 2024-07-18
---

# 🐤 0. 배경지식

[RAG에 대해 정리한 포스트](https://chominho96.github.io/nlp/RAG/)를 참조하자

# 🖼️ 1. 배경

RAG (Retrieval-Augmented Generation) 기법은 LLM에 최신 정보를 반영할 수 있다는 점에서 효과적이라고 알려져 있다. RAG는 hallucination을 줄이고, 특정 도메인 관련 응답 quality를 높인다. 많은 RAG가 LLM의 query 기반 retrieval 성능을 높이기 위해 제안되었지만, 이 접근들은 여전히 복잡한 구현, 긴 응답 시간이라는 단점을 가지고 있다.

전형적인 RAG workflow에서는 다수의 processing step을 유발하는데, 각 step들은 다양한 방식으로 수행된다. 이에 대해 존재하는 RAG 접근법들에 대해 조사하였고, 최적의 RAG를 위한 조합을 구상하였다. 이를 위한 광범위한 실험을 진행하였고, 좋은 성능과 효율의 균형을 맞춘 RAG 전략을 제안하려 한다.

나아가 multimodal retrieval 기술이 시각적 input 관련 Q&A 능력을 향상시키고, “retrieval as generation” 전략을 사용한 multimodal content 시대를 가속화한다는 것을 증명하려 한다.

이 연구의 주요 목적은 3가지이다.

1. 광범위한 실험을 통해 존재하는 RAG 접근법을 조사하고 최적의 RAG를 위한 그들 간의 조합을 규명한다.
2. 포괄적이고 일반적인 domain-specific한 RAG model의 성능을 평가하기 위한 metric과 이를 위한 framewor를 소개한다.
3. Multimodal retrieval 기술의 통합이 visual input에서의 Q&A 능력을 상당 부분 개선하고 “retrieval as generation” 전략으로써 multimodal content 생성을 가속화시킨다는 점을 증명한다.

# 📌 2. 기존 기술과의 차별점

전형적인 기존의 RAG workflow는 다음과 같은 단계를 따른다.

1. Query classification (최신 정보 반영을 위한 retrieval이 필수적인지에 대한 판단)
2. Reranking (retrieve된 문서의 중요도를 제공하기 위해 문서의 rank 재정렬)
3. Repacking (query에 기반하여 retrieve된 문서를 정리)
4. Summarization (repacking된 문서로부터 중요 정보를 추출 및 불필요하거나 중복된 부분 제거)

이러한 방식은 대부분의 RAG에 공통적으로 적용되는 것은 맞지만, 수행하고자 하는 task에 따라 추가해야 하거나 생략되어도 되는 작업들이 있다. 예를 들어, 번역 task의 경우 번역을 위한 원문이 이미 사용자에 의해 제공되므로, 추가적으로 검색을 하는 과정이 필요하지 않다.

이러한 <span style="color:red">task 별로 Retrieval의 필요 여부를 다음과 같이 정리</span>하였다.
![IMG_0230](https://github.com/user-attachments/assets/d7ca42cf-0984-48dc-8c4f-4cb7f0521ae2)

또한, RAG의 각 구성 요소를 모둘별로 나누어 각 모듈의 대표적인 방식과 해당 방식들의 성능 비교를 진행하였다.

각 모듈과 그에 대한 설명은 다음과 같다.

1. Query classification: 검색의 필요 여부 판단
2. Retrieval: 관련 문서 검색
3. Reranking: 검색된 문서들의 중요도를 재정렬
4. Repacking: 검색된 문서를 구조화된 형태로 정리
5. Summarization: 정리된 문서에서 핵심 정보를 추출하여 답변 생성
6. Chunking: 문서를 작은 조각으로 분할
7. Embedding: 문서 조각을 의미적으로 표현
8. Vector database: 문서의 특징을 효율적으로 저장

# 📖 3. 기술 요약

앞서 설명한 RAG의 모듈 별로 필요한 경우, 평가 지표, 각 model별 성능 비교, 성능 향상을 위한 방법을 정리하면 다음과 같다.

## 1) Query classification

<span style="color:red">Query classification은 model의 paramete을 넘어서는 지식이 필요한 경우 사용</span>한다.

Query classification을 평가하기 위한 지표는 MRR (Mean Reciprocal Rank)가 있다. MRR에서는 사용자 쿼리에 대한 검색 결과에서 정답이 몇 번째 순위에 나타나는지를 측정한다. 이를 위해 모든 쿼리에 대해 정답의 역순위의 평균을 계산한다.

## 2) Chunking

Chunking은 검색 정확성을 높이고 LLM의 길이 문제를 피하기 위해 사용된다.

Chunking에서 중요한 요소는 chunk size, advanced chunking, embedding model이다. 다음 그림은 chunk size별 LLM의 성능 지표를 평가한 결과이다.
![IMG_0231](https://github.com/user-attachments/assets/7ad43d91-fc15-4ddc-b680-3665df9c7698)

Advance chunking 기법으로는 small-to-big 기법과 sliding window 기법이 있는데, 해당 기법들을 사용하여 검색 품질을 향상시킬 수 있다.

다음 그림에서 각 기법 별 성능을 비교한 그림을 나타내고 있다.
![IMG_0232](https://github.com/user-attachments/assets/7b4cbcc7-348d-4ef4-a8b9-e82d90a29ef3)

마지막으로는 embedding model 선택이다. 각 chunk의 효과적인 의미 매칭을 위해 적절한 embedding model을 선택한는 것이 중요하다. 각 embedding model에 대한 성능 비교의 결과는 다음과 같다.
![IMG_0233](https://github.com/user-attachments/assets/500dcb38-c64a-44f2-87f0-3ffa1453006b)

## 3) Vector databases

vector database는 embedding vector와 metadata를 저장하여 쿼리에 대한 문서를 효율적으로 검색할 수 있게 한다.

vector database를 선택하는 기준은 1) 다양한 인덱스 유형 2) 대규모 벡터 지원 3) 하이브리드 검색 4) 클라우드 네이티브 가 있다. 각 vector database 별 해당 기능의 유무를 정리한 바는 다음과 같다.
![IMG_0229](https://github.com/user-attachments/assets/a86f7128-7aff-41f6-adcf-e05cf1a7db03)

Milvus database가 모든 필수 기준을 충족하는 것으로 나타났다.

## 4) Retrieval

검색 기법은 크게 다음과 같이 분류된다.

1. Query rewriting: query가 더 잘 매칭되도록 재작성한다.
2. Query decomposition: 원본 query에서 파생된 하위 질문을 기반으로 문서를 검색한다.
3. Pseudo-documents generation: 사용자 쿼리를 기반으로 가상 문서를 생성하고, 이 가상 응답 임베딩을 사용해 유사한 문서를 검색한다.
4. 혼합 검색 방법: 어휘 기반 검색과 벡터 검색을 결합하여 성능을 향상시킨다.
   각 기법 별 검색 방법의 성능은 다음과 같다.
   ![IMG_0234](https://github.com/user-attachments/assets/425212c8-d17d-4087-b211-f4c51cf6ad8c)

기본적으로 supervised method가 unsupervised method보다 성능이 훨씬 우수하며, LLM-Embedder를 HyDE와 혼합 검색하는 방식이 최고점을 받았다. 이에, Hybrid search와 HyDE를 검색 방법으로 채택하는 것이 권장된다.

## 5) Reranking

Reranking에는 DLM reranking, TILDE reranking 방식이 존재한다. DLM reranking은 deep language model을 사용하여 문서의 관련성을 true 또는 false로 분류한다. 이후 쿼리의 문서를 결합하여 fine-tuning 뒤 추론 과정에서 true token의 확률을 기반으로 문서를 재정렬한다. TIDLE reranking은 각 쿼리 용어의 확률을 독립적으로 예측하여 문서의 점수를 계산한다. 각 방식 별 성능 비교는 다음과 같다.
![IMG_0235](https://github.com/user-attachments/assets/a2e6accc-a8af-4d85-8544-1975f6f1bb7c)

## 6) Repacking

Repacking 기법으로는 Forward(reranking 결과의 내림차순으로 문서를 repacking), Reverse(reranking 단계의 오름차순으로 문서를 repacking), Sides(중요한 정보를 입력의 머리나 꼬리에 배치)가 있으며, Sides 방식이 가장 높은 성능을 보였다.

## 7) Summarization

Summarization 기법에는 extractive (추출적) 요약, abstractive (추상적) 요약 기법이 존재한다. 추출적 요약의 경우 텍스트를 문장 단위로 나누고 중요도에 따라 점수화하여 순위를 매기고, 추상적 요약의 경우 여러 문서에서 정보를 종합하여 새로운 문장으로 재구성한다. 각 요약 방식에 대한 성능 비교는 다음과 같다.

![IMG_0236](https://github.com/user-attachments/assets/3ff6fdc4-3323-48c3-a934-64aeeb40d804)

# 🗝️ 4. 기술의 기대효과

앞서 설명한 RAG의 각 module 별 성능 지표와, 해당 성능 지표에 따른 성능 비교를 종합해보면 다음과 같다.
![IMG_0239](https://github.com/user-attachments/assets/0305d50a-f4f7-4a05-b442-f64e2a449651)

해당 표에 나온 내용을 정리해보자면

1. 쿼리 분류 모듈 사용
2. 검색의 경우 HyDE와 함께 Hybrid retrieval 사용
3. Reranking의 경우 monoT5 사용
4. Repacking의 경우 Reverse 사용
5. Summarization의 경우 Recomp 사용
   이 가장 좋은 RAG 성능을 보여준다는 것을 알 수 있다.

이러한 성능 지표의 제시와 RAG 구현 방법 식별로 인해 각 module에 대한 다양한 솔루션을 체계적으로 평가하고, 가장 효과적인 접근 방식을 추천했다는 점에서 RAG 성능을 높일 수 있을 것으로 예상된다.

또한, RAG와 관련된 새로운 module (새로운 vector database 등)이 등장하더라도 해당 논문에서 제시된 benchmark 방법과 툴을 이용해 성능 평가를 객관적으로 진행할 수 있을 것이다.

결론적으로, 각 task별, 그리고 RAG의 각 module별 최적의 성능을 낼 수 있는 조합과 그러한 조합을 선택할 수 있는 기준을 제시했다는 점에서 RAG 적용에 도움을 줄 것으로 기대된다.

> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
