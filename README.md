# Retrieval Server API Usage

This README reflects the current endpoints implemented in `grace_new.api.server`.

## Base URL

- Base URL: `https://songcpu1.cse.ust.hk/status`

If your deployment is behind a reverse proxy prefix (for example `/status`), prepend that prefix to all routes below.

## Local Server

Start the API with the default processed Chroma DB:

```bash
PYTHONPATH=src uv run python -m grace_new.api.server \
  --port 8010 \
  --agentic_model_config model/azure/gpt-5.4-mini.json \
  --text2query_model_config model/azure/gpt-5.4-mini.json
```

When no retrieval backend is specified, Chroma mode defaults to path
`chroma_db_processed_data_20_5`, collection `chunks`, and embedding model
`Qwen/Qwen3-Embedding-0.6B`.

By default, every `/retrieve`, `/text2query`, `/sql`, and `/agentic_query` request
is logged and appended as JSONL to `outputs/server_logs/query_responses.jsonl`.
Use `--query_log_file path/to/file.jsonl` to choose another file, or pass
`--query_log_file ""` to disable saving audit records.

## Available Endpoints

- `GET /` - API metadata and endpoint listing
- `GET /health` - Service health/readiness
- `POST /agentic_query` - Agentic orchestration over RAG + Text2Query
- `POST /retrieve` - Single-query semantic retrieval
- `POST /text2query` - NL question to SQL execution on elderly services DB
- `POST /sql` - Direct read-only SQL execution on elderly services DB
- `GET /docs` - Swagger UI

## 1. Health (`GET /health`)

Returns API readiness for retrieval, text2query, direct SQL, and agentic systems.

Example:

```bash
curl "https://songcpu1.cse.ust.hk/status/health"
```

Response shape:

```json
{
  "status": "healthy",
  "mode": "chroma",
  "ready": true,
  "text2query_ready": true,
  "sql_ready": true,
  "agentic_ready": true
}
```

`ready` describes the semantic retrieval backend. `sql_ready` is `true` when
the configured SQLite database is available for `/sql`.

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
    "question": "列出提供日間護理服務的中心",
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
      "locator": "table=elderly_services; center_id=129",
      "locator_status": "exact",
      "rank": 1,
      "confidence": null,
      "snippet": "center_id=129; center_name=...; address=...; carers_url=https://www.carers.hk/unit/5491",
      "careers_url": "https://www.carers.hk/unit/5491"
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

The `carers_url` values are regenerated from live carers.hk unit indexes with
detail-page matching; see
[`docs/CARERS_URL_GENERATION.md`](docs/CARERS_URL_GENERATION.md).

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

## 5. Direct SQL (`POST /sql`)

Executes a direct read-only SQL query on the configured SQLite DB. Only
`SELECT` and `WITH` queries are accepted; write and DDL commands are rejected.
The endpoint uses `--text2query_db`, which defaults to
`data/structured_data/elderly_services_split.sqlite`.

This endpoint executes SQL supplied by the caller. Expose it only behind a
trusted network boundary or your deployment's authentication layer.

Request fields:

- `sql` (required): read-only SQLite query.
- `max_rows` (optional): maximum rows to return; defaults to 500 and can be set up to 5000.

Main table schema:

Table: `elderly_services`

| Column | Type | Description |
| --- | --- | --- |
| `id` | INTEGER | Surrogate primary key |
| `center_id` | INTEGER | Stable centre/unit identifier shared by paired English and Traditional Chinese rows |
| `organization` | TEXT | Organisation or company name |
| `center_name` | TEXT | Centre or unit name |
| `language` | TEXT | Row language: `en` or `zh-hk` |
| `target_group` | TEXT | Target audience code, for example `8-Elderly` or `8-長者` |
| `center_type` | TEXT | Centre/unit category; may contain multiple semicolon-separated values |
| `district` | TEXT | Districts or service areas; may contain semicolon-separated values and line breaks |
| `ccc_district` | TEXT | Community-care-coupon service district, if applicable |
| `address` | TEXT | Physical address |
| `phone` | TEXT | Phone number |
| `email` | TEXT | Email address |
| `services` | TEXT | Free-text service description |
| `website` | TEXT | Centre/unit website URL |
| `carers_url` | TEXT | carers.hk detail page URL for the centre/unit |

Query-writing notes:

- Most real-world centres have one English row and one `zh-hk` row with the same `center_id`.
- Use `center_id` to pair or deduplicate bilingual rows.
- Prefer `language = 'zh-hk'` and Hong Kong Traditional Chinese terms unless English rows are required.
- Use `LIKE '%keyword%'` for partial matching, especially for `district`, `center_type`, and `services`.
- Common `zh-hk` district values include `東區`, `灣仔區`, `中西區`, `深水埗區`, `觀塘區`, `沙田區`, `元朗區`, `屯門區`, and `全港`.

Example:

```bash
curl -X POST "https://songcpu1.cse.ust.hk/status/sql" \
  -H "Content-Type: application/json" \
  --data-binary @- <<'JSON'
{
    "sql": "SELECT center_id, center_name, district FROM elderly_services WHERE language = 'zh-hk' LIMIT 5;",
    "max_rows": 100
}
JSON
```

Response shape:

```json
{
  "sql": "SELECT ...;",
  "columns": ["center_id", "center_name", "district"],
  "rows": [
    {"center_id": 7, "center_name": "value", "district": "value"}
  ],
  "row_count": 1,
  "truncated": false
}
```

If more rows are available than `max_rows`, `truncated` is `true`.

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


def sql_query(sql: str, max_rows: int = 500):
    response = requests.post(
        f"{BASE_URL}/sql",
        json={"sql": sql, "max_rows": max_rows},
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
