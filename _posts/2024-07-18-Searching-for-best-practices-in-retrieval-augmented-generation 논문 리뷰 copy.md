---
title: "Searching for Best Practices in Retrieval-Augmented Generation 논문 리뷰"
excerpt: "rag"

categories:
  - nlp

toc: true
toc_sticky: true

date: 2024-07-18
last_modified_at: 2024-07-18
---

# 🐤 0. 배경지식

## 1) RAG

() 포스트를 참조하자.

# 🖼️ 1. 배경

RNN, LSTM, gated recurrent는 sequence modeling과 transduction 문제의 해결책으로 확고히 자리잡았다. 이로 인해 이 두 기술은 기존 NLP task에서 가장 많이 쓰이는 model이 되었다.

그러나, recurrent model은 각 계산 시점의 단계에 따라 위치를 정렬하여 t-1번째 hidden state와 t번째 input position을 바탕으로 다음 hidden state를 생성한다. 이러한 특성으로 인해 recurrent model은 순차적으로 계산되어야 하는 특성을 가지며 병렬화가 불가능하다.

또한, 문장의 길이가 길어지면 학습 능력이 현저하게 떨어지는 Long-Term Dependency Problem이 존재한다. 이 문제를 해결하기 위해 Attention mechanism이 등장한다. Attention mechanism은 sequence에서 특정 요소가 input, output과 얼마나 떨어져 있는지와 관계없이 높은 수준의 modeling을 하기 위한 필수적인 요소이다.

이 논문에서는 Transformer model을 제안한다. Transformer은 recurrence를 사용하지 않고 Attention mechanism만을 사용하여 input과 output 사이의 dependencies들이 모두 고루 분배되는 것을 유도한다. 이로 인해 높은 수준의 병렬화를 가능하게 해주고 더 낮은 사양의 GPU를 사용하여 높은 수준의 NLP task를 처리할 수 있게 한다.

# 📌 2. 기존 기술과의 차별점

Attention mechanism이 나오기 전까지 기계번역 영역에서 가장 많이 쓰이던 model은 Seq2Seq model이었다. Seq2Seq model은 LSTM 기반의 model로써 sequence 정보를 modeling하는데 사용되었다.

그런데 Seq2Seq model은 고정된 크기의 context vector을 사용하고 있기 때문에 주어진 문장을 전부 고정된 크기의 하나의 벡터에 압축을 해야 하는 특성을 가진다. 이로 인해 Long-Term dependency 문제가 발생할 수 있다.

또한, LSTM은 recurrent 기반 model로써, 이전 state의 정보가 다음 state의 input으로 들어가야 하는 특성을 가진다. 이로 인해 순차적으로 처리되어야 하므로, 각 연산을 최적화한다고 해도 성능 최적화의 한계가 존재한다.

앞서 설명한 두 가지 문제점을 Transformer model은 Attention mechanism과 recurrence를 사용하지 않는 특성을 이용해 해결하였다.

먼저 Attention mechanism은 encoder의 모든 출력 값들을 별도의 배열에 기록해 놓은 후 decoder에서 출력 단어를 생성할 때 context vector과 더불어 해당 값들을 참고하는 방식이다. 이때 각 encoder의 출력 값들을 반영할 때 모두 동일한 비율로 참고하는 것이 아닌 예측할 단어와 관련이 있는 단어에 더 큰 가중치를 두는 방식을 사용한다. 따라서, 단순히 context vector만 참고하는 것이 아닌 다음에 예측할 단어와 관련된 hidden state 값들을 더 잘 반영할 수 있으므로, Long-Term dependency 문제를 완화할 수 있다.

두번째로 recurrence를 없앴다는 점인데, 순차적으로 처리되어야 하는 recurrence를 제거함으로써 task가 병렬적으로 수행될 수 있게 하였다.

# 📖 3. 기술 요약

Transformer model을 구성하는 핵심 개념에 대해서 먼저 살펴본 후, Transformer model의 전체 동작 구조를 살펴보고자 한다.

## 1) Self-attention

Self-attention에는 Query, Key, Value라는 3가지 변수가 존재한다. 이때 Query에 대한 Key를 얻고, 해당 Key를 이용해 Value를 얻는다고 생각하면 된다.

Self-attention에서는 Query, Key, Value의 초기값이 모두 동일하다. 하지만 중간의 학습 weight W에 의해 최종적인 Query, Key, Value는 서로 다르게 된다.

이 Attention 값을 구하는 공식은 다음과 같다.
![IMG_0226](https://github.com/user-attachments/assets/da0c2900-f5bf-4706-b51f-3cd7772c26b3)

먼저 Query와 Key를 내적하는데, 이는 둘 사이의 연관성을 계산하기 위함이다. 이 내적된 값이 Attention score이다. 여기서 Query와 Key의 차원이 커지면 내적 값, 즉 Attention score가 커지게 되어 모델 학습에 어려움이 생긴다. 이 문제를 해결하기 위해 차원 d_k의 루트만큼을 나누어주는 scaling 작업을 진행한다. 여기까지의 과정을 <span style="color:red">Scaled dot-producted Attention</span>이라 한다.

이후, 값들을 정규화하기 위해 softmax activation 함수를 거치고, 보정을 위해 score 행렬과 value 행렬을 내적하면 최종적인 Attention value 행렬을 얻을 수 있다.

이때 실제로는 다음과 같이 문장의 각 단어들을 병렬적으로 계산한다.

## 2) Multi-Head Attention

Self-attention(Scaled Dot-Product Attention)을 병렬적으로 여러 번 수행하는 개념이다. 이때 head의 수만큼 Attention을 각각 병렬로 나누어 계산하고, 도출된 Attention value들은 마지막에 concatenate를 통해 하나로 합쳐진다. 이 과정을 거치면 Attention을 한번 사용할 때와 같은 크기의 결과가 도출된다.

이 방식을 사용하면 여러 부분에서 도출된 결과를 통해 정보를 상호 보완하기 때문에 Attention을 한번에 계산하는 것보다 성능이 더 우수하게 나오게 된다.

## 3) Masked Multi-Head Attention

Decoder 파트에서는 attention을 수행할 때 앞쪽 단어만 참고한다. 아직 출력되지 않은 미래 단어에 대한 attention을 적용하면 안되기 때문이다.

따라서 이를 위해 mask matrix를 두어 특정 단어를 무시할 수 있도록 한다.

이때 mask 값으로 -∞ 값을 넣어 softmax 함수의 출력이 0에 가까워지도록 한다.

## 4) Transformer model의 동작 방식

![IMG_0227](https://github.com/user-attachments/assets/7844fbc2-a5a1-4366-b68c-264d70e599cd)

Transformer model은 위와 같은 Encoder-Decoder 구조를 지닌다. 여기서 마지막 Encoder layer의 출력이 모든 Decoder layer에 입력되는 구조를 가진다. 또한 앞서 살펴보았듯이 Multi-Head Attention을 구성하는 Self-attention을 병렬적으로 수행할 수 있기 때문에 전체 동작의 병렬화가 가능하다.

또한 병렬적으로 각 Attention을 구하기 때문에 별도로 문장 내의 단어 순서 정보를 전달하기 위해 Positional Encoding을 사용한다. 이때 sin, cos 함수를 사용하게 된다.

# 🗝️ 4. 기술의 기대효과

Transformer model은 현재 Generative AI에 사용되는 LLM model의 중추가 되는 BERT, GPT-3 등의 기반이 되고 있다. 일례로 ChatGPT의 경우 GPT-3 model을 기반으로 fine-tuning한 모델이다.

또한, transformer model은 NLP 분야뿐만 아니라 컴퓨터 비전 분야에서도 활용성이 높아지고 있다. 텍스트를 input으로 넣으면 그에 맞는 이미지를 생성하는 model인 DALLE2나 stable diffusion과 같은 기술에도 transformer model이 사용된다.

이와 같이 NLP 분야에서 CNN 기반 모델에서 transformer 기반 모델이 대세가 되어가고 있는 추세이다. 또한, 원래 목적인 NLP 분야 이외에도 다양한 분야에서의 활용 범위가 넓어지고 있다.

> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
