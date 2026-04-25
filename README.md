# Retrieval Server API Usage

This README reflects the current endpoints implemented in `grace_new.api.server`.

## Base URL

- Base URL: `https://songcpu1.cse.ust.hk/status`

If your deployment is behind a reverse proxy prefix (for example `/status`), prepend that prefix to all routes below.

## Available Endpoints

- `GET /` - API metadata and endpoint listing
- `GET /health` - Service health/readiness
- `POST /agentic_query` - Agentic orchestration over RAG + Text2Query
- `POST /retrieve` - Single-query semantic retrieval
- `POST /text2query` - NL question to SQL execution on elderly services DB
- `GET /docs` - Swagger UI

## 1. Health (`GET /health`)

Returns API readiness for retrieval, text2query, and agentic systems.

Example:

```bash
curl "https://songcpu1.cse.ust.hk/status/health"
```

## 2. Agentic Query (`POST /agentic_query`)

Runs a planner that can combine RAG retrieval and Text2Query tools.

`/agentic_query` keeps a single endpoint and now supports two final response modes:

- `citation` (default): concise claim-focused answer with inline reference ids plus a structured `references` list.
- `synthesis`: legacy concise synthesized answer behavior.

Example:

```bash
curl -X POST "https://songcpu1.cse.ust.hk/status/agentic_query" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "請比較東區日間護理服務與交通支援",
    "mode": "citation",
    "max_steps": 5,
    "rag_top_k": 5,
    "timeout_seconds": 120
  }'
```

Citation response shape:

```json
{
  "question": "string",
  "mode": "citation",
  "answer": "Short cited answer [R1][R2].",
  "references": [
    {
      "id": "R1",
      "tool": "rag_retrieve",
      "source_type": "document",
      "path": "https://www.brainandspine.com.hk/zh-hant/tag/imaging/",
      "locator": "page=1; chunk=0; chars=0-314; lang=zh-TW",
      "locator_status": "exact",
      "rank": 1,
      "confidence": 0.8123,
      "snippet": "Very short evidence snippet..."
    },
    {
      "id": "R2",
      "tool": "text2query_sql",
      "source_type": "structured_row",
      "path": "/abs/path/data/structured_data/Elderly service information 20260302 split.xlsx",
      "locator": "table=elderly_services; center_id=7",
      "locator_status": "exact",
      "rank": 1,
      "confidence": null,
      "snippet": "center_id=7; center_name=...; address=...",
      "careers_url": "https://www.carers.hk/unit/7"
    }
  ],
  "used_tools": ["rag_retrieve", "text2query_sql"],
  "steps_executed": 3,
  "max_steps": 5,
  "total_elapsed_seconds": 12.34,
  "final_response_seconds": 1.21,
  "fallback_synthesis_seconds": null,
  "final_step": {
    "step": 4,
    "action": "final",
    "generation_mode": "citation_planner_final",
    "elapsed_seconds": 1.21
  },
  "tool_trace": [
    {
      "step": 1,
      "action": "rag_retrieve",
      "action_input": "string",
      "output_summary": "string",
      "planner_seconds": 0.85,
      "tool_seconds": 0.14,
      "step_total_seconds": 0.99
    }
  ]
}
```

Notes:

- `references` is empty in `synthesis` mode.
- In `citation` mode, `references` contains the references cited by the final `answer`. Reference ids are assigned from the full evidence catalog, so ids can skip numbers when the answer cites only part of the catalog.
- `locator_status="partial"` means the system could not recover a precise path/location token from the tool output and did not invent one.

## 3. Single Retrieval (`POST /retrieve`)

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

## 4. Text2Query (`POST /text2query`)

Converts a natural-language question to SQL and executes it on the configured SQLite DB.

The current default structured source is:

- Excel: `data/structured_data/Elderly service information 20260302 split.xlsx`
- SQLite: `data/structured_data/elderly_services_split.sqlite`

For centre/unit row queries, `center_id` is always included in the returned rows. If the generated SQL omits it, the server adds it by matching the returned row back to the structured table.

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
  "columns": ["center_id", "col1", "col2"],
  "rows": [
    {"center_id": 7, "col1": "value", "col2": "value"}
  ],
  "row_count": 1
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

def agentic_query(
    question: str,
    mode: str = "citation",
    max_steps: int = 5,
    rag_top_k: int = 5,
    timeout_seconds: int = 120,
):
    response = requests.post(
        f"{BASE_URL}/agentic_query",
        json={
            "question": question,
            "mode": mode,
            "max_steps": max_steps,
            "rag_top_k": rag_top_k,
            "timeout_seconds": timeout_seconds,
        },
        timeout=timeout_seconds + 10,
    )
    response.raise_for_status()
    return response.json()
```
