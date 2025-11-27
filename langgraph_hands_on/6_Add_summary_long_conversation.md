# LangGraph 핸즈온 - 2. Checkpointer

## 출처
- [`Teddy note : LangGraph 핸즈온`](https://github.com/teddynote-lab/LangGraph-HandsOn)

## 주의 사항
- ⚠️기존 `Teddynote LangGraph 핸즈온`에서는 `openai` 모델을 활용해서 수행을 하지만, 해당 코드에서는 `upstage` 모델을 활용해서 수행을 합니다.

# 6. 대화 기록 요약을 추가하는 방법
대화 기록을 유지하는 것은 지속성의 가장 일반적인 사용 사례 중 하나입니다.  
이는 대화를 지속하기 쉽게 만들어주는 장점이 있습니다.  
하지만 대화가 길어질수록 대화 기록이 누적되어 `context window`를 더 많이 차지하게 됩니다.  
이는 LLM 호출이 더 비싸고 길어지며, 잠재적 오류를 발생시킬 수 있어 바람직 하지 않습니다.  
이를 해결하기 위한 한 가지 방법은 현재까지의 대화 요약본을 생성하고, 이를 최근 N개의 메시지와 함께 사용하는 것입니다.  

이 가이드에서는 이를 구현하는 방법의 예시를 살펴보겠습니다.  

다음 단계가 필요합니다.  
- 대화가 너무 긴지 확인
- 너무 길다면 요약본 생성
- 마지막 N개의 메시지를 제외한 나머지 삭제  

이 과정에서 중요한 부분은 오래된 메시지를 삭제 (`DeleteMessage`)하는 것입니다.  

```Python
from langchain_upstage import ChatUpstage
from langchain_core.messages import SystemMessage, RemoveMessage, HumanMessage
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import MessagesState, StateGraph, START
from langgraph.graph.message import add_messages
from dotenv import load_dotenv
from typing import Literal, Annotated
import os

load_dotenv()

UPSTAGE_API_KEY = os.getenv("UPSTAGE_API_KEY")

llm = ChatUpstage(
    api_key=UPSTAGE_API_KEY,
    model='solar-mini',
)

class State(MessagesState):
    messages: Annotated[list, add_messages]
    summary: str
```
LLM 답변 생성 노드를 구현합니다. 여기서 이전의 대화요약 내용이 있다면 이를 입력에 포함합니다.  
```Python
def generate(state: State):
    # 이전 요약 내용 확인
    summary = state.get("summary", "")

    if summary:
         messages = [
            SystemMessage(content=f"Summary of conversation earlier: {summary}")
        ] + state["messages"]
    else:
        # 이전 메시지만 사용
        messages = state["messages"]

    # 모델 호출
    response = llm.invoke(messages)

    # 응답 반환
    return {"messages": [response]}
```
요약이 필요한 상황인지 판단합니다.  
여기서 메시지 수가 6개 초과라면 요약 노드로 이동합니다.  
```Python
# 대화 종료 또는 요약 결정 로직
def should_continue(state: State) -> Literal["summarize_conversation", END]:
    # 메시지 목록 확인
    messages = state["messages"]

    # 메시지 수가 6개 초과라면 요약 노드로 이동
    if len(messages) > 6:
        return "summarize_conversation"
    return END
```
요약 노드를 구현합니다. 이전 요약정보가 있다면 이를 입력에 포함하고 없다면 새로운 요약 메시지를 생성합니다.

```Python
# 대화 내용 요약 및 메시지 정리 로직
def summarize_conversation(state: State):
    # 이전 요약 정보 확인
    summary = state.get("summary", "")

    # 이전 요약 정보가 있다면 요약 메시지 생성
    if summary:
        summary_message = (
            f"This is summary of the conversation to date: {summary}\n\n"
            "Extend the summary by taking into account the new messages above in Korean."
        )
    else:
        # 요약 메시지 생성
        summary_message = "Create a summary of the conversation above in Korean:"

    # 요약 메시지와 이전 메시지 결합
    messages = state["messages"] + [HumanMessage(content=summary_message)]
    # 모델 호출
    response = model.invoke(messages)
    # 오래된 메시지 삭제
    delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
    # 요약 정보 반환
    return {"summary": response.content, "messages": delete_messages}
```
해당 노드를 반영해서 그래프를 그려봅시다.  
```Python
# 워크플로우 그래프 초기화
workflow = StateGraph(State)

# 대화 및 요약 노드 추가
workflow.add_node("conversation", generate)
workflow.add_node(summarize_conversation)

# 시작점을 대화 노드로 설정
workflow.add_edge(START, "conversation")

# 조건부 엣지 추가
workflow.add_conditional_edges(
    "conversation",
    should_continue,
)

# 요약 노드에서 종료 노드로의 엣지 추가
workflow.add_edge("summarize_conversation", END)

# 워크플로우 컴파일 및 메모리 체크포인터 설정
app = workflow.compile(checkpointer=MemorySaver())
```