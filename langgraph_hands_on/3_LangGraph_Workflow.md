# LangGraph 핸즈온 - 2. Checkpointer

## 출처
- [`Teddy note : LangGraph 핸즈온`](https://github.com/teddynote-lab/LangGraph-HandsOn)

## 주의 사항
- ⚠️기존 `Teddynote LangGraph 핸즈온`에서는 `openai` 모델을 활용해서 수행을 하지만, 해당 코드에서는 `upstage` 모델을 활용해서 수행을 합니다.

## 3. LangGraph 워크 플로우 구현
이전까지는 `create_react_agent`, `create_agent` 함수를 사용해서 에이전트를 구성했습니다. 하지만 이전의 에이전트는 단일 에이전트 형태이기 때문에 복잡한 워크플로우를 구현하기가 어렵습니다.  
> 또한 `upstage` 모델을 활용하는 경우 출력형식이 ReAct에 맞게 나오지 않는 경우가 있기 때문에 호환성의 문제가 있을 수 있습니다. 

이를 해결하기 위해서는 워크플로우를 직접 구현해야합니다.  

`LangGraph WorkFlow 구현 Step`
1. `State` 정의 (TypeDict 형식으로 정의 or Pydantic 형식으로 정의)
2. 노드 정의 (함수로 구현)
3. 그래프 생성 (StageGraph 클래스 사용)
4. 컴파일 (checkpointer 설정)
5. 실행 
---

### 3.1 State 정의
`State` : Graph의 노드와 노드 간 공유하는 상태를 정의합니다.  
일반적으로 `TypeDict` 형식을 사용합니다.  
```Python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

# 기본 Graph State 정의
class GraphState(TypedDict):
    question: Annotated[str, "User's Question"]
    documents: Annotated[str, "Retrieved Documents"]
    answer: Annotated[str, "LLM Generated Answer"]
    messages: Annotated[list, add_messages]
```

### 3.2 Node 정의
`Node`: 각 단계를 처리하는 노드입니다. 보통은 Python 함수로 구현합니다. 입력과 출력이 상태 (State) 값 입니다.
```Python
import os
from dotenv import load_dotenv
from langchain_upstage import ChatUpstage
from langchain_tavily import TavilySearch

load_dotenv()

UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")
TAVILY_API_KEY= os.getenv("TAVILY_API_KEY")

llm = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-mini',
)

web_search = TavilySearch(
    api_key=TAVILY_API_KEY,
    max_results=5
)

# 기본 Graph State 정의
class GraphState(TypedDict):
    question: Annotated[str, "User's Question"]
    documents: Annotated[str, "Retrieved Documents"]
    answer: Annotated[str, "LLM Generated Answer"]
    messages: Annotated[list, add_messages]

# 검색 노드
def retrieve_document(state: GraphState) -> GraphState:
    question = state["question"]
    search_results = web_search.run(question)
    state["documents"] = "\n".join(search_results)
    return state

# LLM 답변 노드
def llm_answer(state: GraphState) -> GraphState:
    prompt = f"질문: {state['question']}\n검색 결과:\n{state['documents']}\n답변:"
    
    # LLM 호출
    result = llm.invoke(prompt)
    
    # 메시지 추가
    return {
        "messages": [
            HumanMessage(content=state["question"]),
            AIMessage(content=result.content)
        ]
    }
```

### 3.3 그래프 생성
이제 만든 노드들을 연결해서 그래프를 만들어 줎시다.
- `StateGraph` : `State`를 입력으로 받아서 `Node`를 실행하고 `State`를 업데이트하는 그래프 생성 클래스
- `Edges` : 현재 `State`를 기반으로 다음에 실행할 `Node`를 결정
- `set_entry_point` : 그래프 진입점 설정
- `compile` : 그래프 컴파일

```Python
from langgraph.graph import END, StateGraph
from langgraph.checkpoint.memory import MemorySaver

workflow = StateGraph(GraphState)

# 노드 추가
workflow.add_node("retrieve", retrieve_document)
workflow.add_node("llm_answer", llm_answer)

# 엣지 추가
workflow.add_edge("retrieve", "llm_answer")  # 검색 -> 답변
workflow.add_edge("llm_answer", END)         # 답변 -> 종료

# 진입점 설정
workflow.set_entry_point("retrieve")

# 체크포인터 설정
memory = MemorySaver()

# 그래프 컴파일
app = workflow.compile(checkpointer=memory)
```

### 3.4 그래프 실행
이제 그래프는 완성되었습니다. 해당 그래프를 실행해봅시다.

```Python
initial_state: GraphState = {
        "question": "이번주 프리미어리그 일정 알려줘",
        "documents": "",
        "answer": "",
        "messages": []
    }

config = {"configurable": {"thread_id": "1"}}
result_state = app.invoke(initial_state, config=config)
print("=== ANSWER ===")
print(result_state["messages"][-1].content)
```
해당 그래프는 `invoke` 메서드를 통해서 실행시키면됩니다. 이때 주의할점은 `MemorySaver()`를 사용하는 경우 `config` 인자로 `thread_id` 정보가 포함된 `configurable`에 맵핑되는 dict를 넘겨줘야합니다.

> 📗해당 방법의 경우 한계점은 검색어를 질문으로 넣다보니 검색 결과가의 품질이 떨어질 수 있습니다. 이를 개선해서 질문에서 검색어를 먼저 추출하고 검색하는 방법을 적용해볼 수 있습니다.

```Python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages
from langchain_upstage import ChatUpstage
from langchain_tavily import TavilySearch
from langchain_core.messages import HumanMessage, AIMessage
from langgraph.graph import END, StateGraph
from langgraph.checkpoint.memory import MemorySaver
from dotenv import load_dotenv
import os

load_dotenv()

UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")
TAVILY_API_KEY= os.getenv("TAVILY_API_KEY")

llm = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-mini',
)

web_search = TavilySearch(
    api_key=TAVILY_API_KEY,
    max_results=5
)

# 기본 Graph State 정의
class GraphState(TypedDict):
    question: Annotated[str, "User's Question"]
    retrieve_query: Annotated[str, "Retrieve_query"]
    documents: Annotated[str, "Retrieved Documents"]
    answer: Annotated[str, "LLM Generated Answer"]
    messages: Annotated[list, add_messages]

# 검색어 생성 노드
def question2retrieve_query(state: GraphState) -> GraphState:
    prompt = f"질문: {state['question']}\n해당 질문에 대한 정보를 얻을 수 있는 검색어로 바꿔서 검색어만 출력해줘\n 검색어:"

    result = llm.invoke(prompt)

    print(f"만들어진 검색어: {result.content}")
    return {
        "messages": [
            AIMessage(content=result.content)
        ],
        "retrieve_query": result.content
    }


# 검색 노드
def retrieve_document(state: GraphState) -> GraphState:
    question = state["retrieve_query"]
    search_results = web_search.run(question)
    print(f"검색된 결과: {search_results}")
    state["documents"] = "\n".join(search_results)
    return state

# LLM 답변 노드
def llm_answer(state: GraphState) -> GraphState:
    prompt = f"질문: {state['question']}\n검색 결과:\n{state['documents']}\n답변:"
    
    # LLM 호출
    result = llm.invoke(prompt)
    
    # 메시지 추가
    return {
        "messages": [
            HumanMessage(content=state["question"]),
            AIMessage(content=result.content)
        ]
    }

workflow = StateGraph(GraphState)

# 노드 추가
workflow.add_node("make_retrieve_query", question2retrieve_query)
workflow.add_node("retrieve", retrieve_document)
workflow.add_node("llm_answer", llm_answer)

# 엣지 추가
workflow.add_edge("make_retrieve_query", "retrieve") # 질문 -> 검색어 만들기
workflow.add_edge("retrieve", "llm_answer")  # 검색 -> 답변
workflow.add_edge("llm_answer", END)         # 답변 -> 종료

# 진입점 설정
workflow.set_entry_point("make_retrieve_query")
# 체크포인터 설정
memory = MemorySaver()

# 그래프 컴파일
app = workflow.compile(checkpointer=memory)


if __name__ == "__main__":
    initial_state: GraphState = {
        "question": "세계에서 가장 높은 빌딩의 이름과 높이 알려줘",
        "documents": "",
        "answer": "",
        "messages": []
    }

    config = {"configurable": {"thread_id": "1"}}
    result_state = app.invoke(initial_state, config=config)
    print("=== ANSWER ===")
    print(result_state["messages"][-1].content)
```
이런식으로 다양한 조합을 통해서 그래프를 구성해 `Agent`의 성능을 개선해볼 수 있습니다.