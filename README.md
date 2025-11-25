# langgraph_practice_note
📗: LangGraph Practice Note Repo

## 기본 환경설정
- 프로젝터 root 위치에 `.env` 파일 생성

### LLM API Key 설정
- 필요한 `key` 지정
    - `OPENAI_API_KEY`=""
    - ...

`예시`
```
OPENAI_API_KEY="your_api_key"
```

### LangSmit 환경 설정
- `LangSmith`의 추적을 원하는 경우 `LangSmith API Key` 발급 이후 다음과 같이 설정
    - [LangSmith Key 발급](https://smith.langchain.com)
    - 회원가입 이후 -설정 -API Keys 발급
```
LANGSMITH_API_KEY="your_langsmith_api_key"
LANGSMITH_TRACING="true"
LANGSMITH_ENDPOINT="https://api.smith.langchain.com"
LANGSMITH_PROJECT="your_smith_project_name"
```