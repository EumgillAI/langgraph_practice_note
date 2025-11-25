# LangGraph 핸즈온 - 2. Checkpointer

## 출처
- [`Teddy note : LangGraph 핸즈온`](https://github.com/teddynote-lab/LangGraph-HandsOn)

## 주의 사항
- ⚠️기존 `Teddynote LangGraph 핸즈온`에서는 `openai` 모델을 활용해서 수행을 하지만, 해당 코드에서는 `upstage` 모델을 활용해서 수행을 합니다.

## 1. Checkpointer
단기 메모리 기능이 없다면 그래프는 이전 대화를 기억하지 못합니다.  
즉, 멀티턴 대화를 지원하지 않는다는 말입니다.  
앞에서 생성했었던 `simple_react_agent`로 한번 수행해봅시다.

```Python
from langchain.agents import create_agent
from langchain_upstage import ChatUpstage
from langchain_tavily import TavilySearch
from dotenv import load_dotenv
import os

load_dotenv()

UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")

model = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-pro2',
)

# 검색 도구
tavily_search = TavilySearch(
    max_results=5,
    topic="general",
)

tools = [tavily_search] # 여러개 리스트로 지정

system_prompt = """
You are a helpful assistant.
Always try to answer the user's question directly.
If the user asks about previous information, answer as if it is new.
"""

# ReAct Agent 생성
simple_react_agent = create_agent(
    model=model,
    tools=[],
    system_prompt=system_prompt
)

# 첫번째 턴 대화
turn1_result = simple_react_agent.invoke({'message': "Hi My name is jinsu."})
print(turn1_result)
print()
turn2_result = simple_react_agent.invoke({'message': "What is My name?"})
print(turn2_result)

```
> ⚠️ solar로 수행시 동문서답을 함, 다른 모델로 변경후 수행해볼것! -> 개별 llm으로 invoke 했을때는 동문서답은 안함 아무래도 solar랑 langgraph의 graph의 호환에서 문제가 있는거 같음  

### 1.1 MemorySaver
LangGraph는 `Checkpointer`를 사용해 각 단계가 끝난 후 그래프 상태를 자동으로 저장합니다. 이 내장된 지속성 계층은 메모리를 공유하여 LangGraph가 마지막 상태 업데이트에서 선택할 수 있도록 합니다. 가장 사용하기 쉬운 체크포인터중 하나는 그래프 상태를 위한 키-값 저장소은 `MemorySaver`입니다. 

```Python
from langchain.agents import create_agent
from langchain_upstage import ChatUpstage
from langchain_tavily import TavilySearch

from langgraph.checkpoint.memory import MemorySaver
from dotenv import load_dotenv
import os

load_dotenv()

UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")

model = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-pro2',
)

#검색 도구
tavily_search = TavilySearch(
    max_results=5,
    topic="general",
)

tools = [tavily_search] 

system_prompt = """당신은 한국어 챗봇입니다. 다음 질문에 답변하세요."""
memory = MemorySaver()
# Agent 생성
agent = create_agent(
    model=model,
    tools=[],
    system_prompt=system_prompt,
    checkpointer=memory
)
config = {"configurable": {
        "thread_id": "thread_1",
        "checkpoint_ns": "practice",
        "checkpoint_id": "user_1"
    }}

turn1_result = agent.invoke(
    {"message": "안녕 내이름은 진수야."},
    config=config
)
print(turn1_result)

turn2_result = agent.invoke(
    {"message": "내 이름이 뭐야?"},
    config=config
)
print(turn2_result)
```
그러면 이전 대화 내용을 잘 기억하는 것을 확인할 수 있다.
