---
title: "Corrective Retrieval Augmented Generation 논문 리뷰"
excerpt: "rag"

categories:
  - papers

toc: true
toc_sticky: true

date: 2024-07-24
last_modified_at: 2024-07-24
---

# 🐤 0. 배경지식

[RAG에 대해 정리한 포스트](https://chominho96.github.io/nlp/RAG/)를 참조하자

# 🖼️ 1. 배경

RAG (Retrieval-Augmented Genration) 기법은 LLM에게 최신 정보 또는 domain-specific knowledge를 제공한다는 측면에서 LLM의 hallucination 현상을 보완할 수 있다. LLM은 특정 시점까지의 정보만 학습되어 있기 때문에 최신 정보 또는 학습되지 않은 domain-specific knowledge를 제공할 수 없다는 단점이 있는데, 이를 RAG가 어느 정도 보완할 수 있다는 측면에서 RAG는 각광을 받았고, 실제로 Fine-tuning과 더불어 domain-specific knowledge를 제공하는데 가장 많이 쓰이고 있다.

하지만 RAG는 Retriever 성능에 전체 성능이 좌우된다는 약점을 가지고 있다. 결국 사용자의 질문과 유사한 document의 embedding을 LLM에 같이 넘겨주는 방식으로 RAG가 동작하는데, 이때 제공되는 embedding은 retriever에 의해 관련 문서를 찾아서 진행되기 때문이다. 따라서 RAG를 적용하더라도 관련 없는 문서에 대한 embedding이 LLM에 제공된다면 성능이 향상되지 않을 뿐만 아니라 오히려 단순히 LLM만을 사용하여 답변을 생성하는 것보다도 성능이 하락할 수 있다.

따라서 해당 논문에서는 <span style='color:red'>Retrieval 과정에서 Retrieval evaluator을 도입</span>해 검색된 문서의 관련성과 신뢰도를 평가하고자 한다. 이후, 검색된 문서에서 RAG에 도움이 되지 않는 context를 제거하여 RAG workflow 전반에 걸쳐 분해 후 재구성하고자 한다. 이러한 일련의 과정을 통해 더 향상된 성능을 가진 RAG를 구현하고자 한다.

# 📌 2. 기존 기술과의 차별점

다음 그림에서도 볼 수 있듯이 기존 RAG 방식에서는 전체 RAG 성능이 Retriever의 성능에 좌우된다.
![IMG_0240](https://github.com/user-attachments/assets/5ebb19ac-78b4-4d5f-b1f9-2b6a4a7ca3f7)

이는 단순히 답변의 질이 떨어지는 문제 뿐만 아니라 잘못된 domain-specific knowledge를 제공한다는 점에서 치명적일 수 있다. 이런 현상이 나타나는 원인은 고정된 대량 지식 문서에서 관련 문서를 검색하기 때문이다.

CRAG (Corrective Retrieval Augmented Generation)에서는 Retrieval evaluator을 도입하여 검색된 문서와 입력 쿼리의 관련성을 평가하는 방식으로 이러한 문제를 해결한다. 관련성은 3개의 단계로 평가되는데, (정답, 부정확, 모호) 단계로 나뉘어 평가된다. 이후, 평가 결과에 따라 각각 다른 방식으로 Retrieval을 진행한다.

1. Correct (정답): 검색된 문서를 더 정확한 지식으로 정제하는 과정을 거친다. (Knowledge refinement)
2. Incorrect (부정확): 검색된 문서 제거 후 웹에서 관련 지식을 검색하여 augmentation을 진행한다. (Web search)
3. Ambiguous (모호): Knowledge refinement와 Web search를 모두 진행한다.

즉, 결론적으로 가장 정확하다고 판단되는 문서를 검색하여 해당 문서를 기반으로 RAG를 진행하는 방식으로 구성되어 있다.

또한, CRAG는 Plug-and-Play 방식으로 구현되기 때문에 특정 model에 종속적으로 동작하는 것이 아닌, 다양한 RAG에 쉽게 결합할 수 있다는 차별점도 존재한다.

정리하자면, 기존 RAG에 비해 <span style='color:red'>Retrieval을 1) 문서 검색 2) 검색된 문서가 적절한 문서인지 판단 3) 문서 정제 및 웹 검색을 통한 개선된 검색</span>을 통해 개선하였다.

# 📖 3. 기술 요약

CRAG의 전체적인 구조는 다음과 같다.
![IMG_0241](https://github.com/user-attachments/assets/6d994ee1-733a-46bb-a7c3-a46cf21f830d)

CRAG는 기본적인 RAG의 Retrieval 과정에서 차이점을 보인다.
먼저 사용자의 프롬프트가 들어올 경우 해당 프롬프트에 연관된 document들을 vector database 등에서 찾는다. (Retrieval)

이후, 해당 검색 결과가 정말 프롬프트와 관련이 있는지를 파악하기 위해 Retrieval evaluator을 사용한다. 이때 Retrieval evaluator은 PopQA 등의 dataaset을 이용해 T5-large를 fine-tuning하여 모델을 구성하였다.

검색 결과를 판단하는 과정에 대한 알고리즘은 다음과 같다.
![IMG_0242](https://github.com/user-attachments/assets/847af03d-71ce-4914-96e0-f2fd0b7ce743)

1. input으로는 input question (프롬프트), 검색된 문서가 들어간다.
2. output으로는 최종 결과 문서가 나오게 된다.
3. evaluate 실행 순서는 다음과 같다.

   1. 검색된 문서와 질문과의 유사도 score을 계산한다.
   2. 해당 score에 따라 [CORRECT], [INCORRECT], [AMBIGUOUS] 3가지 confidence로 분류한다.
   3. [CORRECT]일 경우 knowledge refine을 진행한다.
   4. [INCORRECT]일 경우 web search를 진행한다.
   5. [AMBIGUOUS]일 경우 knowledge refine가 web search를 모두 진행한다.

Knowledge Refinement의 경우 다음과 같은 순서로 진행된다.

1. 검색된 관련 문서가 주어지면, 몇 문장 수준의 작은 단위로 다시 나눈다.
2. 나눠진 단위는 Retreival evaluator로 관련성을 평가하며, 관련성이 낮은 지식을 제거하여 더 낮은 지식으로 정제한다.

Web search의 경우 다음과 같은 순서로 진행한다.

1. 검색된 결과가 모두 관련성이 없는 경우 웹 검색을 실행한다.
2. 웹 검색은 Google Search API를 사용하며, 위키피디아를 우선적으로 검색한다.
3. GPT 3.5 model을 이용하여 query 내용을 웹 검색을 위한 검색 쿼리로 재작성한다. 이에 대한 예시는 다음과 같다.
   ![IMG_0244](https://github.com/user-attachments/assets/15a7905d-4aa7-4e41-9e46-080844150a0a)
4. 키워드로 검색된 웹 페이지를 탐색하며 knowledge refinement 방법으로 관련된 웹 지식을 도출한다.

앞서 설명한 방식을 토대로 테스트를 진행하였는데, 테스트 환경은 다음과 같다.

1. PopQA, Biography, PubHealth, Arc-Challenge 데이터셋으로 검증
2. RAG와 Self-RAG에 CRAG 적용 전과 후를 비교하는 방식으로 테스트

이와 같이 테스트한 결과 다음과 같은 결과를 얻을 수 있었다.

1. CRAG 적용시 성능이 크게 향상되었는데, 자세한 결과는 다음과 같다.
   ![IMG_0243](https://github.com/user-attachments/assets/bd86c1bc-163a-4c29-9f53-a64252c1908a)
2. Retrieval evaluator의 각 action 효과에 대해 검증하기 위해 각 action을 제거한 후 성능 테스트를 했을 때 하나의 action이라고 제거했을 때 성능이 저하되는 것을 확인하였다.
   ![IMG_0245](https://github.com/user-attachments/assets/b7156258-27c0-419e-b231-b80713093f70)
3. Knowledge refinement, rewriting, web search의 작업을 개별적으로 제거하면서 성능을 테스트하였고, 각 지식 활용 작업이 지식의 활용도를 높이는 데 기여했음을 확인하였다.
   ![IMG_0246](https://github.com/user-attachments/assets/d7ae128f-a0b6-4943-8010-35854f4ebb9a)

# 🗝️ 4. 기술의 기대효과

기본적으로 RAG는 정의되어 있는 문서들에서만 Retrieval을 할 수 있다는 단점이 있었다. 이로 인해 RAG로 성능이 올라가는 경우도 있지만, 오히려 정의된 문서 안에 존재하지 않는 정보일 경우 잘못된 정보를 Retrieve하여 성능이 하락할 수 있다는 한계가 존재했다.

이때 CRAG를 적용하면 정의된 문서에 있지 않은 정보일 경우 Web search를 통해 정보를 보완할 수 있고, Retrieval evaluator을 통해 검색된 문서에 대해 평가를 할 수 있기 때문에 더 양질의 추가 context로 RAG를 수행할 수 있다.

따라서 추가 document들이 잘 정제되어 있지 않거나, document들이 수가 부족해도 높은 성능의 RAG를 수행할 수 있을 것으로 기대된다.

또한, Web에서의 데이터는 계속해서 늘어나고, 최신 정보가 꾸준히 갱신되기 때문에 최신 정보를 기존 document의 갱신 없이도 어느 정도 가져올 수 있을 것으로 예상되며, 나아가 CRAG 기술을 발전시켜 Web search만으로도 RAG를 수행할 수 있을 것으로 기대된다.

물론 Web search의 경우 외부 API를 사용해야 하므로 network overhead 등이 발생하여 성능이 저하될 가능성이 있지만 Network 기술의 발전, data caching을 이용해 적절히 해결할 수 있을 것으로 예상된다.

결론적으로, RAG에서의 문제점인 <span style='color:red'>1) RAG를 적용하더라도 정확하지 않거나 정제되지 않은 정보로 인해 오히려 성능이 떨어지는 현상 2) document의 수가 부족해 적절한 외부 context를 찾을 수 없는 경우를 해결</span>한다는 점에서 CRAG는 RAG의 성능을 크게 향상시킬 것으로 예상된다.

> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
