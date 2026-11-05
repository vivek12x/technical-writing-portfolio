# API Reference: Semantic Search & Document Ingestion Service

> **Complete REST API specification for programmatic document chunking, vector indexing, and hybrid semantic retrieval.**

---

## 1. Overview & Base URL

The Semantic Search API provides programmatic endpoints to ingest unstructured text documents, generate vector embeddings, and execute low-latency hybrid similarity queries.

| Environment | Base URL |
| :--- | :--- |
| **Production** | `https://api.vectordocs.io/v1` |
| **Sandbox / Staging** | `https://api.sandbox.vectordocs.io/v1` |

All requests must be sent over **HTTPS**. The API accepts and returns standard `application/json` payloads.

---

## 2. Authentication

All API requests require an API key passed in the `Authorization` header as a Bearer token:

```http
Authorization: Bearer sec_live_948fbc210e74b39a
```

| Header | Type | Description |
| :--- | :--- | :--- |
| `Authorization` | string | **Required**. Format: `Bearer <your_api_key>` |
| `Content-Type` | string | **Required** for POST/PUT. Must be `application/json` |
| `X-Request-Id` | string | *Optional*. UUID for idempotent tracing and auditing |

---

## 3. Endpoints

### 3.1 Ingest Document Batch

`POST /v1/documents/ingest`

Submits raw markdown or plain text documents for asynchronous parsing, chunking, and vector indexing.

#### Request Headers
```http
POST /v1/documents/ingest HTTP/1.1
Host: api.vectordocs.io
Authorization: Bearer sec_live_948fbc210e74b39a
Content-Type: application/json
```

#### Request Body Schema

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `collection_id` | string | Yes | The target vector collection or index identifier. |
| `documents` | array[object] | Yes | An array of document objects (max 50 per batch). |
| `documents[].document_id` | string | Yes | Unique client-side document identifier. |
| `documents[].title` | string | Yes | Human-readable title of the document. |
| `documents[].content` | string | Yes | Raw markdown or text content. |
| `documents[].metadata` | object | No | Key-value pairs for filtering (e.g. `tags`, `author`, `access_tier`). |
| `chunking_strategy` | object | No | Custom chunking parameters (`chunk_size`, `chunk_overlap`). Defaults to 500 / 50. |

#### Example Request

```bash
curl -X POST "https://api.vectordocs.io/v1/documents/ingest" \
  -H "Authorization: Bearer sec_live_948fbc210e74b39a" \
  -H "Content-Type: application/json" \
  -d '{
    "collection_id": "col_platform_docs_2026",
    "documents": [
      {
        "document_id": "doc_arch_001",
        "title": "Cloud Snapshot & Immutability Architecture",
        "content": "Enterprise data protection relies on immutable snapshots. When DataLock is enabled...",
        "metadata": {
          "category": "architecture",
          "tier": "enterprise"
        }
      }
    ],
    "chunking_strategy": {
      "chunk_size": 400,
      "chunk_overlap": 40
    }
  }'
```

#### Response (`202 Accepted`)

```json
{
  "status": "queued",
  "job_id": "job_ingest_7841bc90",
  "documents_queued": 1,
  "estimated_completion_ms": 650,
  "created_at": "2026-11-05T08:30:00Z"
}
```

---

### 3.2 Semantic Query

`POST /v1/search/query`

Executes a hybrid semantic and lexical similarity search across an indexed collection, returning ranked chunks and confidence scores.

#### Request Body Schema

| Field | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `collection_id` | string | Yes | - | Identifier of the collection to search. |
| `query` | string | Yes | - | Natural language question or phrase. |
| `top_k` | integer | No | `5` | Maximum number of ranked chunks to return (1-20). |
| `min_similarity` | float | No | `0.70` | Minimum cosine similarity threshold (0.0 to 1.0). Chunks scoring below this are pruned. |
| `hybrid_weight` | float | No | `0.60` | Weight given to dense vector search vs lexical BM25 (`1.0` = pure vector, `0.0` = pure keyword). |
| `filter` | object | No | `null` | Metadata filtering expression (e.g., `{"tier": "enterprise"}`). |

#### Example Request (Python)

```python
import requests

url = "https://api.vectordocs.io/v1/search/query"
headers = {
    "Authorization": "Bearer sec_live_948fbc210e74b39a",
    "Content-Type": "application/json"
}
payload = {
    "collection_id": "col_platform_docs_2026",
    "query": "Can administrators delete snapshots protected by immutability lock?",
    "top_k": 2,
    "min_similarity": 0.75,
    "hybrid_weight": 0.7
}

response = requests.post(url, json=payload, headers=headers)
print(response.json())
```

#### Response (`200 OK`)

```json
{
  "query": "Can administrators delete snapshots protected by immutability lock?",
  "collection_id": "col_platform_docs_2026",
  "results_count": 2,
  "execution_time_ms": 38.4,
  "results": [
    {
      "chunk_id": "chk_9801_001",
      "document_id": "doc_arch_001",
      "title": "Cloud Snapshot & Immutability Architecture",
      "similarity_score": 0.9412,
      "text": "When Compliance Lock is enabled on a snapshot or view, retention periods cannot be shortened or deleted by any user, including super-administrators, until the expiration date.",
      "metadata": {
        "section": "Retention Policies",
        "category": "architecture"
      }
    },
    {
      "chunk_id": "chk_9801_002",
      "document_id": "doc_arch_001",
      "title": "Cloud Snapshot & Immutability Architecture",
      "similarity_score": 0.8105,
      "text": "Legal holds operate independently of scheduled retention locks and prevent automated or manual purging.",
      "metadata": {
        "section": "Legal Hold Overrides",
        "category": "architecture"
      }
    }
  ]
}
```

---

## 4. Error Codes & Handling

The API adheres to standard HTTP status codes:

| HTTP Status | Code Name | Description | Recommended Resolution |
| :--- | :--- | :--- | :--- |
| `200 OK` | `SUCCESS` | Request succeeded. | Process response data. |
| `202 Accepted` | `QUEUED` | Batch ingestion job successfully queued. | Poll `/v1/jobs/{job_id}` for completion status. |
| `400 Bad Request` | `INVALID_PAYLOAD` | Missing required fields or malformed JSON. | Check request body against specification. |
| `401 Unauthorized` | `INVALID_CREDENTIALS` | Missing or invalid API key. | Verify the `Authorization: Bearer <key>` header. |
| `404 Not Found` | `COLLECTION_NOT_FOUND` | Specified `collection_id` does not exist. | Create collection or verify spelling. |
| `413 Payload Too Large`| `BATCH_SIZE_EXCEEDED` | Request exceeds 50 documents or 10 MB total. | Split upload into smaller batches. |
| `429 Too Many Requests`| `RATE_LIMIT_EXCEEDED` | Request volume exceeded allocated quota. | Implement exponential backoff retry. |
| `500 Server Error` | `INTERNAL_ERROR` | An internal processing error occurred. | Retry with backoff; contact support if persistent. |

### Error Response Body Format

All 4xx and 5xx responses return a consistent error object:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Quota exceeded: 120 requests per minute limit reached for current tier.",
    "request_id": "req_8491a02f",
    "retry_after_seconds": 12
  }
}
```

---

*Authored by Vivek as part of the [Technical Writing Portfolio](https://github.com/vivek12x/technical-writing-portfolio).*
