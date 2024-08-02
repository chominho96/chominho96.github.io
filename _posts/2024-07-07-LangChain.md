---
title: "LangChain이란?"
excerpt: "LLM 애플리케이션 만들기"

categories:
  - langchain

toc: true
toc_sticky: true

date: 2024-07-07
last_modified_at: 2024-07-07
---

# ⛓️ 1. LangChain이란?

현재 시장에 ChatGPT, Gemini, HyperClovaX 등 수많은 생성형 AI가 출시되어 있는 시점에서 생성형 AI를 한 번도 써보지 않은 사람은 드물 것이라고 생각한다. 나아가 ChatGPT 등에서는 애플리케이션에 개발할 수 있도록 Open API를 제공하고 있어 개발자들은 애플리케이션 개발에 ChatGPT를 도입할 수 있게 되었다.

개인적으로 나도 LangChain을 이용하지 않고 Chat GPT API만을 이용해서 프로젝트를 진행한 적이 있었는데, 생각보다 API 연동이 까다롭고 stream data를 직접 다뤄야 하는 등 신경 써야 할 게 많았던 기억이 있다.

또한, 여러 개의 LLM model이 존재하는 요즘 1개의 애플리케이션이 여러 LLM model을 사용하는 경우도 늘어났다.

이 경우 활용할 수 있는 framework가 [LangChain](https://www.langchain.com/)이다.

LangChain을 이해할 때, JDBC/JPA에 빗대어 설명하던 블로그가 있는데, 이 비유가 되게 적절한 비유인 것 같다. JPA에서는 사용하려는 DB의 종류가 어떤 것이든, 얼마나 많은 DB를 사용하던 간에 DB 연결, CRUD 등에 대해 추상화한 인터페이스를 제공한다.

마치 LLM의 JPA같은 존재가 LangChain이다. <span style='color: red'>LangChain을 이용하면 LLM과의 연결, LLM 질의 등의 과정에 대한 추상화된 인터페이스를 사용</span>할 수 있다.

아래 코드는 내가 이전에 LangChain을 사용하지 않고 ChatGPT API를 사용했을 때의 코드이다.

(물론 지금은 훨씬 간결하게 짤 수 있는 방법이 있을 수도 있지만, 내가 개발하던 당시에는 ChatGPT API가 공개되지 얼마 안된 시점이어서 이 방식이 가장 최신 방식이었다고 알고 있다.) (지금은 Java를 위한 [LangChain4j](https://github.com/langchain4j/langchain4j)도 공개되었다고 한다.)

```java
public Flux<String> ask(List<ChatGptMessageRequest> messages,
    String sessionId, String question, Member member, String chatModel)
throws JsonProcessingException {

    if (question != null) {
        messages.add(new ChatGptMessageRequest(ChatGptConfig.USER_ROLE, question));
    }

    ChatGptRequest chatGptRequest = new ChatGptRequest(
        ChatGptConfig.getChatModel(chatModel),
        ChatGptConfig.MAX_TOKEN,
        ChatGptConfig.TEMPERATURE,
        ChatGptConfig.STREAM,
        messages
    );

    String requestValue = gptObjectMapper.writeValueAsString(chatGptRequest);
    AnswerDto answer = new AnswerDto("");

    return client.post()
        .bodyValue(requestValue)
        .accept(MediaType.TEXT_EVENT_STREAM)
        .exchangeToFlux(response - > {
            if (response.statusCode().is2xxSuccessful()) {
                return response.bodyToFlux(String.class)
                    .mapNotNull(originalResponse - > {
                        ObjectMapper responseObjectMapper = new ObjectMapper();
                        // 정상 응답
                        try {
                            if (originalResponse.equals(ChatGptConfig.DONE_MESSAGE)) {
                                return responseObjectMapper.writeValueAsString(originalResponse);
                            }

                            JsonNode jsonNode = responseObjectMapper.readTree(originalResponse);
                            String content = "";
                            String finishReason = ChatGptConfig.PROCEEDING_RESPONSE;
                            try {
                                JsonNode contentNode = jsonNode.path("choices")
                                    .get(0).path("delta").path("content");
                                JsonNode reasonNode = jsonNode.path("choices")
                                    .get(0).path("finish_reason");
                                content = contentNode.isMissingNode() ? "" : contentNode.asText();
                                answer.addAnswer(content);

                                if (!reasonNode.isNull()) {
                                    finishReason = reasonNode.asText();
                                }
                            } catch (NullPointerException ignored) {}

                            return responseObjectMapper.writeValueAsString(new QuestionResponse(
                                sessionId,
                                content,
                                finishReason));
                        } catch (JsonProcessingException e) {
                            // content 필드가 없는 경우 -> 응답이 끝난 경우
                            log.error("ask 에서 오류 발생: {}", originalResponse);
                            return null;
                        }
                    })
                    .filter(Objects::nonNull);
            } else {
                // Handle non-2xx responses here
                ObjectMapper objectMapper = new ObjectMapper();

                try {
                    log.error("GPT error occurred");
                    log.error("messages: {}", objectMapper.writeValueAsString(messages));
                } catch (JsonProcessingException e) {
                    log.error("=================GPT error occurred when parsing json=================");
                    for (ChatGptMessageRequest request: messages) {

                        if (request.getRole().equals("user")) {
                            log.error("question: {}", request.getContent());
                        } else {
                            log.error("answer: {}", request.getContent());
                        }
                    }
                    log.error("=================GPT error occurred when parsing json=================");
                }

                return Flux.error(new BaseException(CHAT_GPT_EXCEPTION));
            }
        })
        .onErrorResume(WebClientResponseException.class, ex - > {
            log.error("WebClientResponseException 에러 발생", ex);
            return Flux.error(new BaseException(CHAT_GPT_EXCEPTION));
        });
}
```

이 코드가 무언가 대단한걸 수행하는 함수도 아니고 단순히 ChatGPT API를 요청하고, 응답으로 받은 stream data를 반환하기만 하는 함수이다.

이제 이 코드를 LangChain을 이용해서 바꿔보도록 하겠다.

```python
from langchain_openai import ChatOpenAI
import os

os.environ['OPENAI_API_KEY'] = "{YOUR_OPEN_API_KEY}"

llm = ChatOpenAI(model = "gpt-3.5-turbo")
llm.invoke("한국의 수도는 어디야?")
```

위 코드는 gpt-3.5-turbo model로 LLM에게 질의를 하는 코드이다. 굉장히 간단하지 않은가?

개인적으로 LangChain의 여러 유용한 기능도 사용할 만한 이유가 되지만, 이 예시 하나만으로 LangChain을 사용할 이유는 충분하다고 생각한다.

그렇다면 이제 LangChain의 구성 요소에 대해 알아보도록 하자.

# 📚 2. LangChain 구성 요소

![IMG_0247](https://github.com/user-attachments/assets/68483859-3592-41f7-b5ba-f41572acfc14)

위 그림은 LangChain의 전체 구성 요소를 나타내고 있다. 각 요소에 대해 상세히 살펴보자.

## 1) Data Sources

LLM에 적절한 context를 제공하기 위해 PDF, Web, CSV 등 다양한 외부 소스에서 데이터를 액세스해야 할 수 있다.

LangChain은 이러한 소스들에서 데이터에 엑세스할 수 있도록 지원하고, 각 모듈들과의 통합을 지원한다.

## 2) Word Embeddings

각 Data source는 LLM에 그대로 넘겨지는 게 아니라, vector의 형태로 넘겨져야 한다. LangChain은 선택된 LLM을 기반으로 최적의 Embedding model을 선정한다.

## 3) Vector Database

Vector의 저장 및 similarity를 검색하기 위해 vector database를 사용할 수 있다.

이때 LangChain은 여러 vector database들을 지원하고, db 접근 interface를 제공한다.

## 4) LLM

LangChain은 OpenAI, Cohere, AI21에서 제공하는 LLM 이외에도 HuggingFace에서 제공하는 오픈소스 LLM을 지원한다.

갈수록 지원되는 model과 API endpoint 목록은 빠르게 증가하고 있다.

# 👀 3. 정리

이제는 LLM을 이용한 애플리케이션을 개발할 때 LangChain은 선택이 아닌 필수가 되었다.

오픈소스 LLM의 성능이 계속해서 올라가고 있고, 갈수록 애플리케이션을 개발할 때 LLM을 사용하는 일이 많아지는 만큼, LangChain 사용법을 정확하게 숙지해 놓는 것이 중요할 것 같다.

또한, 급변하는 시장인 만큼 LangChain의 API나 함수들도 빠르게 변화하는 것 같다. 당장 LangChain을 공부하기 위해 1년 전 블로그에 적힌 예제 코드를 살펴봤는데, 이미 deprecated된 경우가 많았다. 따라서 코드를 짜는 시점에 공식 문서를 참조하는 습관을 지니는 것이 중요할 것 같다.

> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
