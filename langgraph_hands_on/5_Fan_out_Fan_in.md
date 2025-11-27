# LangGraph 핸즈온 - 2. Checkpointer

## 출처
- [`Teddy note : LangGraph 핸즈온`](https://github.com/teddynote-lab/LangGraph-HandsOn)

## 주의 사항
- ⚠️기존 `Teddynote LangGraph 핸즈온`에서는 `openai` 모델을 활용해서 수행을 하지만, 해당 코드에서는 `upstage` 모델을 활용해서 수행을 합니다.

# 5. Fan-out/Fan-in
LangGraph에서 Fan-out / Fan-in은 복잡한 LLM 워크플로우를 관리하기 위해 중요한 패턴입니다.  

Fan-out은 단일 입력을 여러 병렬 작업으로 분배하는 패턴으로 하나의 프롬프트나 쿼리를 여러 LLM, 도구, 또는 처리 단계로 동시에 전송하여 다양한 관점이나 접근 방식을 얻을 수 있게 합니다. 이는 복잡한 문제를 더 작고 전문화된 하위 작업으로 분할하거나 동일한 작업에 대해 여러 모델의 결과를 비교할 때 유용합니다.  

Fan-in은 Fan-out의 역과정으로 여러 병렬 작업의 결과를 단일 출력이나 다음 단계로 통합합니다. 이는 다양한 모델이나 도구에서 생성된 결과를 종합하여 더 완전하고 정확한 최종 응답을 만들거나 여러 에이전트의 작업을 조정할 때 사용합니다.  

```Python
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

# 상태 정의 (add_messages 리듀서 내용)
class State(TypedDict):
    aggregate: Annotated[list, add_messages]

# 노드 값 반환 클래스
class ReturnNodeValue:
    def __init__(self, node_secret: str):
        self._value = node_secret

    def __call__(self, state: State) -> Any:
        print(f"Adding {self._value} to {state['aggregate']}")
        return {"aggregate": [self._value]}
    
builder = StateGraph(State)

builder.add_node("a", ReturnNodeValue("I'm A"))
builder.add_edge(START, "a")
builder.add_node("b", ReturnNodeValue("I'm B"))
builder.add_node("c", ReturnNodeValue("I'm C"))
builder.add_node("d", ReturnNodeValue("I'm D"))

builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)

# 그래프 컴파일
graph = builder.compile()
```

## 일부만 Fan-out 하는 방법
(Fan-out 의 순서 조정)  
조건부 엣지를 두어 일부만 Fan-out 할 수 있습니다. 

```Python
from typing import Annotated, Sequence
from typing_extensions import TypedDict
from langgraph.graph import END, START, StateGraph


# 상태 정의(add_messages 리듀서 사용)
class State(TypedDict):
    aggregate: Annotated[list, add_messages]
    which: str


# 노드별 고유 값을 반환하는 클래스
class ReturnNodeValue:
    def __init__(self, node_secret: str):
        self._value = node_secret

    def __call__(self, state: State) -> Any:
        print(f"Adding {self._value} to {state['aggregate']}")
        return {"aggregate": [self._value]}


# 상태 그래프 초기화
builder = StateGraph(State)
builder.add_node("a", ReturnNodeValue("I'm A"))
builder.add_edge(START, "a")
builder.add_node("b", ReturnNodeValue("I'm B"))
builder.add_node("c", ReturnNodeValue("I'm C"))
builder.add_node("d", ReturnNodeValue("I'm D"))
builder.add_node("e", ReturnNodeValue("I'm E"))


# 상태의 'which' 값에 따른 조건부 라우팅 경로 결정 함수
def route_bc_or_cd(state: State) -> Sequence[str]:
    if state["which"] == "cd":
        return ["c", "d"]
    elif state["which"] == "bc":
        return ["b", "c"]
    else:
        return ["b", "c", "d"]


# 전체 병렬 처리할 노드 목록
intermediates = ["b", "c", "d"]

builder.add_conditional_edges(
    "a",
    route_bc_or_cd,
    intermediates,
)
for node in intermediates:
    builder.add_edge(node, "e")


# 최종 노드 연결 및 그래프 컴파일
builder.add_edge("e", END)
graph = builder.compile()
```
이제 조건을 다르게 해서 실행해 봅시다.  
`조건 1`
```Python
# 그래프 실행(which: bc 로 지정)
result = graph.invoke({"aggregate": [], "which": "bc"})
print("===" * 30)
print(result["aggregate"])
```
`조건 2`
```Python
# 그래프 실행(which: cd 로 지정)
result = graph.invoke({"aggregate": [], "which": "cd"})
print("===" * 30)
print(result["aggregate"])
```