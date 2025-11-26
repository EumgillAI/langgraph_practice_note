# LangGraph 핸즈온 - 2. Checkpointer

## 출처
- [`Teddy note : LangGraph 핸즈온`](https://github.com/teddynote-lab/LangGraph-HandsOn)

## 주의 사항
- ⚠️기존 `Teddynote LangGraph 핸즈온`에서는 `openai` 모델을 활용해서 수행을 하지만, 해당 코드에서는 `upstage` 모델을 활용해서 수행을 합니다.

## 4. Routing
LLM 애플리케이션에서 라우팅은 입력 쿼리나 상태에 따라 적절한 처리 경로나 구성 요소로 요청을 전달하는 메커니즘입니다.  
LangChain / LangGraph에서 라우팅은 특정 작업에 가장 적합한 모델이나 도구를 선택하고, 복잡한 워크플로우를 관리하여, 비용과 성능 균형을 최적화 하는데 필수적입니다.  

### 4.1 Agent
도구 선택을 하는 방식으로 라우팅  
따라서, 도우에 대한 description 이 상세하게 작성되어야 합니다.

### 4.2 LLM.with_structured_output
Function Calling 을 사용하는 방식으로 라우팅  

```Python

from langchain_upstage import ChatUpstage
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field
from typing import Literal
from dotenv import load_dotenv
import os

load_dotenv()

UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")
TAVILY_API_KEY= os.getenv("TAVILY_API_KEY")

class RouteQuery(BaseModel):
    """Route a user query to the most relevant datasource."""

    datasource: Literal["vectorstore", "web_search"] = Field(
        ...,
        description="Given a user question choose to route it to `web_search` or a `vectorstore`.",
    )


llm = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-mini',
)

# llm 구조화된 출력 설정
structured_llm_router = llm.with_structured_output(RouteQuery)

system = """You are an expert at routing a user question to a vectorstore or web search.
The vectorstore contains documents related to AI Brief Report(SPRI) including Samsung Gause, Anthropic, etc.
Use the vectorstore for questions on AI related topics. Otherwise, use `web_search`."""


route_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", system),
        ("human", "{question}"),
    ]
)

# 프롬프트 템플릿과 구조화된 LLM 라우터를 결합하여 질문 라우터 생성
question_router = route_prompt | structured_llm_router
```

아래 두 개의 쿼리를 실행해보고 비교해봅시다.  
```Python
# 쿼리 실행
question_router.invoke("삼성전자가 만든 생성형 AI 이름을 찾아줘")
```
```Python
# 쿼리 실행
question_router.invoke("LangCon2025 이벤트의 날짜와 장소는?")
```
다음은 앞에서 구현했었던 `검색` 과 `답변생성`에 대한 메서드들입니다.
```Python

# 검색
def retrieve(state: GraphState) -> GraphState:
    question = state["question"]
    search_results = web_search.run(question)
    state["documents"] = "\n".join(search_results)
    return state
# 답변 생성
def generate(state: GraphState) -> GraphState:
    question = state['question']

    prompt = f"질문: {question}\n검색 결과: {state['documents']}\n해당 질문에 대한 답변을 검색 결과를 참고해서 해줘.\n답변:"

    result = llm.invoke(prompt)

    return {
        "messages": [
            AIMessage(content=result.content)
        ]
    }

```
해당 메서드들을 라우팅 해봅시다.  
사용자의 질문에 따라서 바로 생성을 할지 아니면 검색이후 생성을 할지 라우팅하게 해줍시다.  

```Python

class RouteQuery(BaseModel):
    """Route a user query to the most relevant datasource."""

    datasource: Literal["generate", "retrieve"] = Field(
        ...,
        description="Given a user question choose to route it to `generate` or a `retrieve`.",
    )


llm = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-mini',
)

# llm 구조화된 출력 설정
structured_llm_router = llm.with_structured_output(RouteQuery)

system = """You are an expert at routing a user question to a retrieve or generate.
The retrieve is search the internet to get information.
If you know the answer to that question, do `generate`, and if you need the exact information, do `retrieve`.
"""

route_prompt = ChatPromptTemplaㄴte.from_messages(
    [
        ("system", system),
        ("human", "{question}"),
    ]
)
question_router = route_prompt | structured_llm_router

#라우팅 노드
def route_question(state):
    print("==== [ROUTE QUESTION] ====")
    # 질문 가져오기
    question = state["question"]
    # 질문 라우팅
    source = question_router.invoke({"question": question})
    # 질문 라우팅 결과에 따른 노드 라우팅
    if source.datasource == "retrieve":
        print("\n==== [GO TO RETRIEVE] ====")
        return "retrieve"
    elif source.datasource == "generate":
        print("\n==== [GO TO GENERATE] ====")
        return "generate"
    
```
이제 `조건부 엣지`를 사용해서 그래프를 생성해줍니다.
```Python
workflow = StateGraph(GraphState)

# 노드 추가
workflow.add_node("retrieve",  retrieve)
workflow.add_node("generate", generate)

# 엣지 추가
workflow.add_conditional_edges(
    START,
    route_question,
    {
        "retrieve": "retrieve",
        "generate": "generate",  
    },
)
workflow.add_edge("retrieve", "generate")  # 문서 검색 후 답변 생성
workflow.add_edge("generate", END)  # 답변 생성 후 종료

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
두 가지 질문을 비교해봅시다.  
1. "세계에서 가장 높은 빌딩의 이름과 높이 알려줘"  
    -> 예상 : 검색 -> 생성
2. "1 + 1 은?"  
    -> 예상 : 생성
---
