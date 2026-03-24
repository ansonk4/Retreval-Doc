# Retrieval Server API Usage

This README reflects the current endpoints implemented in `src/server.py`.

## Base URL

- Base URL: `https://songcpu1.cse.ust.hk/status`

If your deployment is behind a reverse proxy prefix (for example `/status`), prepend that prefix to all routes below.

## Available Endpoints

- `GET /` - API metadata and endpoint listing
- `GET /health` - Service health/readiness
- `POST /retrieve` - Single-query semantic retrieval
- `POST /retrieve/batch` - Batch semantic retrieval
- `POST /text2query` - NL question to SQL execution on elderly services DB
- `POST /agentic_query` - Agentic orchestration over RAG + Text2Query
- `GET /docs` - Swagger UI

## 1. Health (`GET /health`)

Returns API readiness for retrieval, text2query, and agentic systems.

Example:

```bash
curl "https://songcpu1.cse.ust.hk/status/health"
```

## 2. Single Retrieval (`POST /retrieve`)

Retrieves top-k similar chunks for one query.

Example:

```bash
curl -X POST "https://songcpu1.cse.ust.hk/status/retrieve" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "基督教靈實協會",
    "top_k": 5,
    "return_scores": true,
    "check_sufficiency": false
  }'
```

Response shape:

```json
{
  "query": "string",
  "results": [
    {
      "rank": 1,
      "content": "string",
      "source": "string",
      "similarity_score": 0.88
    }
  ],
  "total_results": 5,
  "sufficiency_result": {
    "sufficient": true,
    "reason": "string"
  }
}
```

Note: `sufficiency_result` is `null` unless `check_sufficiency=true` and sufficiency checker is initialized.

## 3. Batch Retrieval (`POST /retrieve/batch`)

Retrieves documents for multiple queries in one request.

Example:

```bash
curl -X POST "https://songcpu1.cse.ust.hk/status/retrieve/batch" \
  -H "Content-Type: application/json" \
  -d '{
    "queries": ["長者中心地址", "申請服務資格"],
    "top_k": 5,
    "return_scores": true
  }'
```

Response shape:

```json
{
  "queries": ["string"],
  "results": [
    [
      {
        "rank": 1,
        "content": "string",
        "source": "string",
        "similarity_score": 0.9
      }
    ]
  ],
  "total_queries": 1
}
```

## 4. Text2Query (`POST /text2query`)

Converts a natural-language question to SQL and executes it on the configured SQLite DB.

Example:

```bash
curl -X POST "https://songcpu1.cse.ust.hk/status/text2query" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "列出提供日間護理服務的中心"
  }'
```

Response shape:

```json
{
  "question": "string",
  "sql": "SELECT ...",
  "columns": ["col1", "col2"],
  "rows": [
    {"col1": "value", "col2": "value"}
  ],
  "row_count": 1
}
```

## 5. Agentic Query (`POST /agentic_query`)

Runs a planner that can combine RAG retrieval and Text2Query tools.

Example:

```bash
curl -X POST "https://songcpu1.cse.ust.hk/status/agentic_query" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "請比較東區日間護理服務與交通支援",
    "max_steps": 5,
    "rag_top_k": 5,
    "timeout_seconds": 120
  }'
```

Response shape:

```json
{
  "question": "string",
  "answer": "string",
  "used_tools": ["rag_retrieve", "text2query"],
  "steps_executed": 3,
  "max_steps": 5,
  "tool_trace": [
    {
      "step": 1,
      "action": "rag_retrieve",
      "action_input": "string",
      "output_summary": "string"
    }
  ]
}
```

## Python Client Example

```python
import requests

BASE_URL = "https://songcpu1.cse.ust.hk/status"


def retrieve(query: str, top_k: int = 5):
    response = requests.post(
        f"{BASE_URL}/retrieve",
        json={"query": query, "top_k": top_k, "return_scores": True},
        timeout=30,
    )
    response.raise_for_status()
    return response.json()


def text2query(question: str):
    response = requests.post(
        f"{BASE_URL}/text2query",
        json={"question": question},
        timeout=60,
    )
    response.raise_for_status()
    return response.json()

def agentic_query(question: str, max_steps: int = 5, rag_top_k: int = 5, timeout_seconds: int = 120):
    response = requests.post(
        f"{BASE_URL}/agentic_query",
        json={
            "question": question,
            "max_steps": max_steps,
            "rag_top_k": rag_top_k,
            "timeout_seconds": timeout_seconds,
        },
        timeout=timeout_seconds + 10,
    )
    response.raise_for_status()
    return response.json()
```
