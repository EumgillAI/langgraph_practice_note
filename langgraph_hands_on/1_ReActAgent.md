# LangGraph 핸즈온 - 1. ReAct Agent

## 출처
- [`Teddy note : LangGraph 핸즈온`](https://github.com/teddynote-lab/LangGraph-HandsOn)

## 주의 사항
- ⚠️기존 `Teddynote LangGraph 핸즈온`에서는 `openai` 모델을 활용해서 수행을 하지만, 해당 코드에서는 `upstage` 모델을 활용해서 수행을 합니다.

## 0. 설정한 환경 불러오기
```Python
from dotenv import load_dotenv
import os

load_dotenv()
```

## 1. 기본 ReActAgent 구현 
`LLM모델 생성`
```Python
from langchain_upstage import ChatUpstage

UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")

model = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-mini',
)
```

### 1.1 도구 (Tools) 설정
`도구(Tool)`는 에이전트, 체인 또는 LLM이 외부 세계와 소통하기 위한 인터페이스입니다.  
`LangChain` 에서 제공하는 기본 도구를 사용하여 쉽게 활용할 수 있으며 사용자 정의 도구 또한 구축해서 사용할 수 있습니다.  

- [`LangChain에 통합된 도구`](https://docs.langchain.com/oss/python/integrations/providers/overview)

#### 1.1.1 Tavily
`Tavily`는 검색을 도와주는 도구입니다. 이를 사용하기 위해서는 API키를 발급 받아야합니다.

- [`Tavily  Key 발급`](https://app.tavily.com/)

`주요 매개변수`  
- max_results (int): 반환할 최대 검색 결과 수 (기본값: 5)
- search_depth (str): 검색 깊이 ("basic" 또는 "advanced")
- include_domains (List[str]): 검색 결과에 포함할 도메인 목록
- exclude_domains (List[str]): 검색 결과에서 제외할 도메인 목록
- include_answer (bool): 원본 쿼리에 대한 짧은 답변 포함 여부
- include_raw_content (bool): 각 사이트의 정제된 HTML 콘텐츠 포함 여부
- include_images (bool): 쿼리 관련 이미지 목록 포함 여부

`사용법`
```Python
from langchain_tavily import TavilySearch

# 검색 도구 초기화
tavily_search = TavilySearch(
    max_results=5,
    topic="general",
)

# 이름과 설명 지정
web_search_tool.name = "web_search"
web_search_tool.description = "Use this tool to search on the web"
```

### 1.2 커스텀 도구 Tool 변환
- 커스텀으로 생성한 Tool경우 `LangChain`, `LangGraph` 에서 사용할 수 있는 형태로 Tool를 등록해주어야한다.
- `Tool` 객체로 변환해주어야한다.

- `retriever`의 경우 (`LangChain`의 ``BaseRetriever`를 상속받은 객체의 경우) 다음과 같은 방법으로 만들 수 있다.  
```Python
from langchain_core.vectorstores import VectorStore
from langchain_core.tools.retriever import create_retriever_tool

retriever = vectorstore.as_retriever()

search_tool = create_retriever_tool(
    retriever=retriever,
    name="search",
    description="문서 검색"
)
llm_with_tools = llm.bind_tools([search_tool])
```

- Python 함수나 다른 객체는 다음과 같이 `데코레이터`를 사용하거나 `Tool` 객체로 만들어주면된다.
```Python
from langchain_core.tools import tool
from langchain_core.vectorstores import VectorStore

retriever = vectorstore.as_retriever()

@tool
def search(query: str) -> str:
    """문서 검색"""
    docs = retriever.invoke(query)
    return "\n\n".join([doc.page_content for doc in docs])
```

### 1.3 Tool 등록
#### `LangChain`에 등록
`llm.bind_tools`로 등록
```Python
from langchain_upstage import ChatUpstage
from langchain_core.tools import tool

@tool
def search_documents(query: str) -> str:
    """문서 검색"""
    return "검색 결과"

@tool
def calculator(expression: str) -> str:
    """계산기"""
    return str(eval(expression))

UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")

model = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-mini',
)

# tool 등록
model_with_tools = model.bind_tools([search_documents, calculator])
```

#### `LangGraph`에 등록
`create_react_agent()` 로 등록
```Python
from langgraph.prebuilt import create_react_agent
from langchain_upstage import ChatUpstage
from langchain_core.tools import tool

@tool
def search_documents(query: str) -> str:
    """문서 검색"""
    return "검색 결과"

@tool  
def calculator(expression: str) -> str:
    """계산기"""
    return str(eval(expression))

# Tool 리스트 생성
tools = [search_documents, calculator]

UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")

model = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-mini',
)

# Agent 생성 (tools를 직접 전달)
agent = create_react_agent(model, tools)

```
### 1.4 ReAct Agent 만들기
- 위에서 이미 보았지만 `create_react_agent()`를 활용해서 ReAct Agent를 생성할 수 있다.

> ⭐ 참고로 `craete_react_agent()`는 사라지는 함수입니다.   
최신버전의 `LangGraph`에서는 `from langchain.agents import create_agent` 를 사용합니다.

`create_agent` 사용시 예시
```Python
simple_react_agent = create_agent(
    model=model,
    tools=tools,
    system_prompt="You are a helpful assistant. Answer in Korean."
)
```



`주요 매개변수`
- model : 사욜할 모델
- tools : 도구 목록
- prompts : 시스템 프롬프트

```Python
from langgraph.prebuilt import create_react_agent
from langchain_upstage import ChatUpstage
from langchain_tavily import TavilySearch


UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")

model = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-mini',
)

# 검색 도구
tavily_search = TavilySearch(
    max_results=5,
    topic="general",
)

tools = [tavily_search] # 여러개 리스트로 지정

# ReAct Agent 생성
simple_react_agent = create_react_agent(
    model, tools, prompt="You are a helpful assistant. Answer in Korean."
)
```

### 1.5 ReAct Agent Graph 실행
이렇게 생성한 Agent는 LangGraph의 Graph 형태로 작성되어 있습니다.  
이를 실행하는 방법은 `.run()` 메서드에 input query를 넣어주면됩니다.

```Python
result = simple_react_agent.run("오늘 서울의 날씨는 어떠니?")
```

> ⚠️ Upstage Solar-Pro2 모델은 도구 호출 신호(tool_calls)를 “정확하게 포맷된 JSON” 형태로 항상 내지 않습니다.  
대신 AIMessage 내부에 이미 검색 결과를 반영한 최종 답변을 만들어서 넣어버립니다.
즉, 검색은 정상적으로 수행했지만  
tool_message, tool_calls, args 등이 비어 있거나 깨진 형태로 들어올 수 있습니다.