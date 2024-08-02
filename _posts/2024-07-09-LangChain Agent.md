---
title: "LangChain Agent란?"
excerpt: "나만의 AI Agent 만들기"

categories:
  - langchain

toc: true
toc_sticky: true

date: 2024-07-09
last_modified_at: 2024-07-09
---

# 🐤 0. 배경지식

LangChain Agent를 이해하기 위해서는 LangChain에 대한 기초적인 지식이 필요하다.
LangChain에 대해서는[LangChain에 대해 정리한 포스트](https://chominho96.github.io/langchain/LangChain/)를 참조하자

또한 글 도중에 RAG와의 비교를 서술하였는데, RAG에 대해서는 [RAG에 대해 정리한 포스트](https://chominho96.github.io/nlp/RAG/)를 참조하자

# 👨‍🔧 1. LangChain Agent란?

LangChain Agent는 사용자로부터 주어진 프롬프트에 대해 <span style='color: red'>LLM이 적절한 의사 결정을 내린 뒤, 특정 행동을 수행할 수 있도록 하는 것</span>을 의미한다.

![제목 없음](https://github.com/user-attachments/assets/951fa60a-4fd8-4080-9b95-ac473dc4ecf7)

위의 예시를 통해 살펴보자.

사용자가 "한국의 수도가 어디야?"라는 질문을 했을 때, LangChain Agent는 이 질문을 바로 처리할 수 있는지, 아니면 외부의 도움을 받아야 하는지 판단한다.

이 경우에는 LLM이 한국의 수도가 어디인지를 모르는 상황이라 해보자. 그러면 한국의 수도 정보를 알아오기 위해 사용할 수 있는 적절한 Tool을 선정한다. 이 예시에서는 위키피디아 Tool을 선정했다고 가정하자.

최종적으로 위키피디아 Tool을 이용해 한국의 수도는 서울이라는 정보를 얻어온 뒤, 이 정보를 이용해서 사용자에게 적절한 응답을 내려줄 수 있다.

앞서 Tool이라는 단어를 사용하였는데, <span style='color: red'>Tool이란 LangChain Agent가 목적을 달성하기 위해 사용할 수 있는 도구</span>와 같은 개념이다. 앞선 예제에서 살펴보면 LangChain에게 위키피디아 Tool을 제공한다면, 필요하다고 판단되는 상황에서 Agent가 해당 Tool을 사용하는 것이다.

기존의 LLM만 있는 구조에서는 단순히 사용자의 질문에 대해 LLM이 응답을 내려주는 역할만 담당했다. 하지만 LangChain Agent라는 개념이 추가되면서, 단순히 LLM이 응답만 하는 역할이 아닌, <span style='color: red'>LLM이 이 질문을 어떻게 처리할지 결정하고, 추가 행동을 수행하는 구조</span>로 바뀐 것이다.

지금까지가 LangChain Agent에 대한 추상적인 설명이었다. 다음으로 더 구체적으로 LangChain Agent에 대해 알아보자.

# 🤔 2. LangChain Agent vs RAG

RAG에 대해 알고 있다면, 이런 의문이 들 수 있다. RAG도 LLM의 부족한 지식을 보완하기 위해 외부 지식을 LLM에게 제공하는 형식인데, 그렇다면 Agent와 RAG의 차이점이 무엇일까?

결론부터 말하자면, LangChain Agent는 RAG와 달리 Tool을 이용한 <span style='color : red'>복잡한 Action을 수행</span>할 수 있다.

![제목 없음](https://github.com/user-attachments/assets/15742587-eb8c-4b7a-9e62-f345b7a09e19)

LangChain Tool은 위키피디아, Google 캘린더, CSV Tool 등 굉장히 다양한 분야의 여러 Tool들을 지원한다. 이 Tool들을 이용해서 수행할 수 있는 action의 예시들은 다음과 같다.

1. 위키피디아 Tool을 이용해 필요한 정보 검색
2. Google 캘린더 Tool을 이용해 특정 날짜의 일정 조회 및 새로운 일정 추가
3. CSV Tool을 이용해 원하는 데이터를 CSV 형식으로 내보내기

즉, 단순히 프롬프트와 관련된 정보를 검색하여 LLM에게 넘겨줌으로써 추가 정보를 제공할 수 있는 RAG와 달리 LLM이 특정 행동까지 취할 수 있는 것이다.

또한, vector database가 미리 구축되어 있어야 사용할 수 있는 RAG와 달리, LangChain Agent에서는 원하는 Tool을 Agent에 넘겨주기만 하면 사용할 수 있다.

# 🖼️ 3. LangChain Agent 전체 구조

<p align="center"><img src="https://github.com/user-attachments/assets/e5e22d5e-3ff9-4100-9c2b-bf1cf432e598"></p>

위 그림은 LangChain Agent의 전체 구조를 표현한 것이다.

Prompt (질문)이 들어오고, 답변이 생성될 때까지의 순서대로 각 요소를 살펴보자.

## 1) Planning

먼저 사용자로부터 들어온 프롬프트에 대해 LLM이 해당 질문을 어떻게 처리해야 하는지에 대한 계획을 세운다. Planning의 주요 성질은 다음과 같다.

1. Subgoal decomposition: 주어진 프롬프트를 작은 sub-task들로 분할한다. 사용자의 프롬프트는 하나의 목적만을 가지고 있을 수도 있지만 여러 개의 질문/명령의 조합일 수도 있다. 따라서 각 task를 나눠서 task 별 action을 구분할 필요가 있다.
2. Reflection and Refinement: Planning 과정에서 LLM은 이전 프롬프트들에 대해 수행했던 action에 대해 self-reflection을 통해 더 개선된 planning을 진행한다.

이 과정을 통해 Agent는 프롬프트를 어떻게 처리할지 결정하고, 이 과정에서 Tool이 필요한지 여부를 결정하게 된다. (<span style='color:red'>그리고 이 Tool들 중 하나로써 RAG가 쓰일수도 있는 것이다. 즉, RAG는 LangChain Tool에 포함될 수 있는 개념이다.</span>)

## 2) 각 task에 대해 chain 수행

Planning 과정에서 규정된 sub-task들과 사용할 tool을 바탕으로 각 sub-task들을 수행하는 과정이다.

## 3) 전체 답변 생성

모든 task들에 대한 chain 수행이 완료되었다면, 최종 답변을 생성하고, 클라이언트에게 반환하게 된다.

## Memory

전체 구조 그림에서 묘사하진 않았지만 LangChain Agent에는 memory가 존재한다.

LangChain Agent는 일련의 과정을 수행하는 과정에서 memory를 두어 각 context를 기억하고, 이를 사용하여 답변을 생성한다.

Memory의 종류는 다음과 같다.

1. Short-term memory: Agent가 현재 진행 중인 task와 관련된 정보를 기억한다.
2. Long-term memory: 과거에 수행한 task와 관련된 정보를 기억한다. Vector database를 사용하여 저장하므로 원하는 만큼 저장이 가능하다.

# 💻 4. 예제

다음 코드는 LangChain Agent를 이용한 간단한 예시이다.

```python
from langchain.agents import initialize_agent
from langchain.agents import AgentType
from langchain.agents import Tool
from langchain_core.prompts import PromptTemplate
from langchain_community.llms.ollama import Ollama
from langchain_community.agent_toolkits import FileManagementToolkit

from tempfile import TemporaryDirectory

llm = Ollama(base_url="{OLLAMA_BASE_URL}", model="{LLM_MODEL}", num_ctx=32768, temperature=0.0)

# local file system에 접근할 수 있게 하는 tool
tools = FileManagementToolkit(
    root_dir=str("{YOUR_ROOT_DIRECTORY}"),
).get_tools()

agent = initialize_agent(tools, llm=llm, agent=AgentType.STRUCTURED_CHAT_ZERO_SHOT_REACT_DESCRIPTION, verbose=True) # verbose : 중간 과정 출력

PROMPT = """
Please create create a text file named sample.txt in the current directory and fill in the content with Hello.
And then print all files in current directory.
"""

print(agent.run(PROMPT))
```

llm의 경우 Ollama를 이용하였는데, OpenAI에서 제공하는 llm 등을 사용해도 무방하다.

예시에서는 FileManagementToolkit을 사용하였는데, 이 tool을 이용하면 실행되는 컴퓨터의 local file system에 대한 접근(파일 읽기, 파일 쓰기 등)을 할 수 있다.

이를 이용해 다음 2가지 작업을 지시했다.

1. 현재 디렉토리에 sample.txt 파일을 만들고, 내용을 "Hello"라고 채워넣기
2. 현재 디렉토리의 모든 파일을 출력하기

그 결과는 다음과 같다.

<p align="center"><img src="https://github.com/user-attachments/assets/4aeba991-bc61-4ccc-bac4-5f7a63e997ee"></p>

먼저 task가 2개로 나눠서 실행되었음을 확인할 수 있었다. 첫 번째 task는 sample.txt 파일을 생성하도록 한 task이다.

<p align="center"><img src="https://github.com/user-attachments/assets/b6e5ebfb-5db3-4b6b-ad9b-0754e6116540"></p>

두 번째 task는 현재 디렉토리의 파일들을 조회하는 task이다.

<p align="center"><img src="https://github.com/user-attachments/assets/67493113-551a-4584-9878-ca077045ab91"></p>

마지막으로 최종 답변까지 정상적으로 생성된 모습을 확인할 수 있었다.

<p align="center"><img src="https://github.com/user-attachments/assets/7b7dd6f6-26ac-419e-9be3-ccfe1a03be2f"></p>

그리고 파일도 정상적으로 생성되었음을 확인했다.

# 👀 5. 정리

LangChain은 단순히 여러 LLM model을 사용할 수 있게 할 뿐만 아니라, Agent 기능을 지원하여 여러 Tool을 이용한 action들을 수행할 수 있게 한다. 그리고 지원하는 Tool들이 굉장히 많아서, 웬만한 작업들은 다 수행할 수 있는 것 같다.

LangChain Agent로 인해 쉽게 개인이 나만의 AI agent를 만들 수 있는 시대가 오고 있다고 생각한다.

> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
