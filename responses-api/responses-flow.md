# Responses API -- Internal Flow

Extracted from the OGX interactive flow simulator (`ogx/docs/src/components/ResponsesFlowSimulator/`). This document captures the actors, phases, flow presets, and toggle-driven behavior that the simulator visualizes as sequence diagrams.

> **Source files:**
> - `ogx/docs/docs/api-openai/responses-flow.mdx` — page definition
> - `ogx/docs/src/components/ResponsesFlowSimulator/flowData.js` — actors, steps, presets, prose
> - `ogx/docs/src/components/ResponsesFlowSimulator/index.jsx` — component and code generation

---

## Actors (Subsystems)

| Actor | Label | Description |
|-------|-------|-------------|
| `client` | Client | Your application making the API call |
| `fastapi` | FastAPI | HTTP endpoint handler — routes, SSE, request validation |
| `impl` | Responses | Core orchestration — request routing, state management, conversation sync |
| `bgworker` | BGWorker | Async worker pool (10 workers) for background response processing |
| `orchestrator` | Orchestrator | Streaming response orchestrator — runs the inference loop and coordinates tool execution |
| `moderation` | Moderation | Input/output guardrail validation via configured moderation policies |
| `inference` | Inference | LLM provider (OpenAI, vLLM, Ollama, Bedrock, etc.) |
| `executor` | Executor | Tool execution dispatcher — routes tool calls to the right backend |
| `vectorio` | VectorIO | Semantic search over vector stores — returns ranked chunks with citations |
| `mcp` | MCP | Model Context Protocol server connection — lazy discovery, session caching |
| `toolruntime` | ToolRuntime | Server-side runtime tool execution (for runtime tools, not function tools) |
| `conversations` | Conversations | Conversation turn management — stores structured items for UI/metadata |
| `store` | Store | SQL persistence for responses and raw chat messages (dual storage) |

**Actor ordering** (left-to-right in sequence diagrams): Client → FastAPI → Responses → BGWorker → Orchestrator → Moderation → Inference → Executor → VectorIO → MCP → ToolRuntime → Conversations → Store

---

## Arrow Styles

| Style | Meaning |
|-------|---------|
| **Solid teal** (`request`) | Outgoing call to a subsystem |
| **Solid gray** (`response`) | Return value from a subsystem |
| **Dashed amber** (`event`) | SSE event (streaming to client) |
| **Dashed purple** (`async`) | Async operation (background queue, polling) |

---

## Toggles (Request Parameters)

The flow changes depending on which parameters are set. Each toggle activates/deactivates specific flow steps.

| Toggle | Description |
|--------|-------------|
| `stream` | Enable SSE streaming |
| `background` | Async processing mode — queue and poll |
| `previous_response_id` | Multi-turn continuation from a prior response |
| `conversation` | Conversation association (dual storage) |
| `file_search` | file_search tool enabled |
| `mcp` | MCP tools enabled |
| `function_tools` | Client-side function tools enabled |
| `guardrails` | Input/output moderation guardrails enabled |

**Validation constraints:**
- `previous_response_id` and `conversation` are **mutually exclusive** (400 error)
- `stream` and `background` **cannot be combined** (400 error — OGX limitation)

---

## Flow Phases

### Phase 1: Request Arrival (always)

```
Client ──POST /v1/responses──► FastAPI
FastAPI ──create_openai_response()──► Responses
```

### Phase 2: Context Loading (conditional)

**When `previous_response_id` is set:**
```
Responses ──get_response_object(prev_id)──► Store
Store ──previous response + messages──► Responses
```

The previous response is loaded from the Store, including its original input and the full chat message history. These are prepended to the current input, giving the model conversational context without requiring the client to resend the entire history.

**When `conversation` is set:**
```
Responses ──list_items(conversation_id)──► Conversations
Conversations ──conversation items──► Responses
Responses ──get_conversation_messages()──► Store
Store ──stored chat messages──► Responses
```

Conversation state is loaded from two sources: the Conversations API provides structured turn items (for UI and metadata), while the Store provides the raw chat messages used for inference. After the response completes, both stores are updated — items are added to the conversation, and the full message array is persisted for the next turn. This dual-storage pattern enables both conversation-level UI and accurate inference continuity.

### Phase 3: Background Queueing (when `background: true`)

```
Responses ──store queued response──► Store
Responses ──enqueue work item──► BGWorker
Responses ──queued response (immediate)──► Client
BGWorker ──start processing──► Orchestrator        [async]
```

In background mode, the request is immediately queued and a response with status `"queued"` is returned to the client. One of 10 async workers picks up the job, runs the full inference loop internally, and updates the stored response. The client polls `GET /v1/responses/{id}` to check progress and retrieve the completed response.

### Phase 3 (alt): Direct Orchestration (when NOT `background`)

```
Responses ──create_response()──► Orchestrator
```

### Phase 4: Streaming Event (when `stream` AND NOT `background`)

```
Orchestrator ──SSE: response.created──► Client     [event]
```

### Phase 5: Input Guardrails (when `guardrails: true`)

```
Orchestrator ──run_guardrails(input)──► Moderation
Moderation ──pass / blocked──► Orchestrator
```

Before the inference loop begins, the combined input text is checked against configured guardrail policies. If a violation is detected, the response is refused immediately.

### Phase 6: MCP Tool Discovery (when `mcp: true`)

```
Orchestrator ──list_mcp_tools(endpoint)──► MCP
MCP ──tool definitions (cached)──► Orchestrator
```

MCP tools are discovered lazily — the orchestrator calls `list_mcp_tools()` on the configured server endpoint when MCP tools first appear. Tool definitions are cached for the duration of the request via `MCPSessionManager`. When the model invokes an MCP tool, the executor reuses the existing session, avoiding redundant connection setup.

### Phase 7: Inference Loop (always — this is the core loop)

The dashed box in the sequence diagram marks the **inference loop** — the model calls tools, receives results, and calls inference again until no more server-side tool calls are needed, a client-side `function_call` is returned, or `max_infer_iters` is reached.

**Inference call (always):**
```
┌─────────────────────────────────────────────────────────┐
│ loop [until no tool_calls or max_iters]                 │
│                                                         │
│ Orchestrator ──openai_chat_completion()──► Inference     │
│ Inference ──completion + tool_calls──► Orchestrator      │
```

**file_search execution (when `file_search: true`):**
```
│ Orchestrator ──execute(file_search)──► Executor          │
│ Executor ──search_vector_store()──► VectorIO             │
│ VectorIO ──ranked chunks + citations──► Executor         │
│ Executor ──file_search result──► Orchestrator            │
```

When the model requests a file search, the Executor queries VectorIO's `search_vector_store()` endpoint with the model's query. VectorIO searches the configured vector stores and returns ranked document chunks with relevance scores. These are formatted with citations and fed back to the model as tool results for the next inference iteration.

**MCP tool execution (when `mcp: true`):**
```
│ Orchestrator ──execute(mcp_tool)──► Executor             │
│ Executor ──invoke_mcp_tool()──► MCP                      │
│ MCP ──tool result──► Executor                            │
│ Executor ──mcp tool result──► Orchestrator               │
```

**Function tool handoff (when `function_tools: true`):**
```
│ Orchestrator ──emit function_call output──► Responses    │
│                   (breaks loop)                          │
└─────────────────────────────────────────────────────────┘
```

Function tools are client-side in the Responses flow. When the model emits a function tool call, OGX returns it as a `function_call` output item and exits the inference loop. The client executes the function and sends the next request with a `function_call_output` item to continue.

### Phase 8: Output Guardrails (when `guardrails: true`)

```
Orchestrator ──run_guardrails(output)──► Moderation
Moderation ──pass / blocked──► Orchestrator
```

After the model generates output, the response text is checked again — an output violation is converted into a refusal response with violation details.

### Phase 9: Persistence (always)

```
Orchestrator ──upsert_response_object()──► Store
```

### Phase 10: Conversation Sync (when `conversation: true`)

```
Responses ──add_items(input + output)──► Conversations
Responses ──store_conversation_messages()──► Store
```

### Phase 11: Response Delivery

**Streaming (when `stream` AND NOT `background`):**
```
Orchestrator ──SSE: response.completed | response.incomplete | response.failed──► Client    [event]
```

With streaming enabled, Server-Sent Events (SSE) are emitted throughout execution. The client receives `response.created` when processing begins, intermediate events for each tool call and output item, and one terminal event: `response.completed`, `response.incomplete`, or `response.failed`.

**Synchronous (when NOT `stream` AND NOT `background`):**
```
Responses ──OpenAIResponseObject──► Client
```

**Background polling (when `background: true`):**
```
Client ──GET /v1/responses/{id} (poll)──► FastAPI      [async]
FastAPI ──get_response_object()──► Store
Store ──completed response──► FastAPI
FastAPI ──OpenAIResponseObject──► Client
```

---

## Flow Presets

### 1. Simple Completion

**Toggles:** none (all defaults)

The minimal flow — request arrives, goes through Responses → Orchestrator → Inference → Store → response returned.

**Active actors:** Client, FastAPI, Responses, Orchestrator, Inference, Store

**Request:**
```http
POST /v1/responses HTTP/1.1
Host: localhost:8321
Authorization: Bearer $API_KEY
Content-Type: application/json

{
  "model": "llama-3.3-70b",
  "input": "Explain how transformers work"
}
```

**Response:**
```json
{
  "id": "resp_abc123",
  "object": "response",
  "status": "completed",
  "model": "llama-3.3-70b",
  "output": [
    {
      "type": "message",
      "role": "assistant",
      "content": [
        {
          "type": "output_text",
          "text": "Transformers are a neural network architecture..."
        }
      ]
    }
  ],
  "usage": {
    "input_tokens": 12,
    "output_tokens": 256,
    "total_tokens": 268
  }
}
```

**Steps:**
1. Client → FastAPI: `POST /v1/responses`
2. FastAPI → Responses: `create_openai_response()`
3. Responses → Orchestrator: `create_response()`
4. Orchestrator → Inference: `openai_chat_completion()` *(loop)*
5. Inference → Orchestrator: `completion + tool_calls` *(loop)*
6. Orchestrator → Store: `upsert_response_object()`
7. Responses → Client: `OpenAIResponseObject`

---

### 2. RAG + Conversation

**Toggles:** `file_search: true`, `conversation: true`

Loads conversation history from dual storage, runs inference with file_search tool for RAG, persists conversation state after completion.

**Active actors:** Client, FastAPI, Responses, Orchestrator, Inference, Executor, VectorIO, Conversations, Store

**Request:**
```http
POST /v1/responses HTTP/1.1
Host: localhost:8321
Authorization: Bearer $API_KEY
Content-Type: application/json

{
  "model": "llama-3.3-70b",
  "input": "What do the uploaded documents say about Q4 results?",
  "tools": [
    {
      "type": "file_search",
      "vector_store_ids": ["vs_abc123"],
      "max_num_results": 5,
      "ranking_options": {
        "ranker": "auto"
      }
    }
  ],
  "conversation": "my-session-001"
}
```

**Response:**
```json
{
  "id": "resp_def456",
  "object": "response",
  "status": "completed",
  "model": "llama-3.3-70b",
  "output": [
    {
      "type": "file_search_call",
      "id": "fs_001",
      "status": "completed",
      "queries": ["Q4 results"],
      "results": [
        {
          "file_id": "file_xyz",
          "filename": "q4-report.pdf",
          "score": 0.92,
          "text": "Q4 revenue increased by 15%..."
        }
      ]
    },
    {
      "type": "message",
      "role": "assistant",
      "content": [
        {
          "type": "output_text",
          "text": "According to the uploaded documents, Q4 results show...",
          "annotations": [
            {
              "type": "file_citation",
              "file_id": "file_xyz",
              "filename": "q4-report.pdf",
              "index": 42
            }
          ]
        }
      ]
    }
  ],
  "usage": {
    "input_tokens": 1840,
    "output_tokens": 312,
    "total_tokens": 2152
  }
}
```

**Steps:**
1. Client → FastAPI: `POST /v1/responses`
2. FastAPI → Responses: `create_openai_response()`
3. Responses → Conversations: `list_items(conversation_id)`
4. Conversations → Responses: `conversation items`
5. Responses → Store: `get_conversation_messages()`
6. Store → Responses: `stored chat messages`
7. Responses → Orchestrator: `create_response()`
8. Orchestrator → Inference: `openai_chat_completion()` *(loop)*
9. Inference → Orchestrator: `completion + tool_calls` *(loop)*
10. Orchestrator → Executor: `execute(file_search)` *(loop)*
11. Executor → VectorIO: `search_vector_store()` *(loop)*
12. VectorIO → Executor: `ranked chunks + citations` *(loop)*
13. Executor → Orchestrator: `file_search result` *(loop)*
14. Orchestrator → Store: `upsert_response_object()`
15. Responses → Conversations: `add_items(input + output)`
16. Responses → Store: `store_conversation_messages()`
17. Responses → Client: `OpenAIResponseObject`

---

### 3. MCP Agent Loop

**Toggles:** `mcp: true`, `stream: true`

Streaming agentic flow with MCP tool discovery and execution. The orchestrator discovers MCP tools lazily, caches tool definitions, and executes MCP tool calls within the inference loop.

**Active actors:** Client, FastAPI, Responses, Orchestrator, Inference, Executor, MCP, Store

**Request:**
```http
POST /v1/responses HTTP/1.1
Host: localhost:8321
Authorization: Bearer $API_KEY
Content-Type: application/json
Accept: text/event-stream

{
  "model": "llama-3.3-70b",
  "input": "List open issues in the repository and summarize the top 3 by priority",
  "tools": [
    {
      "type": "mcp",
      "server_label": "github",
      "server_url": "http://localhost:8080/sse",
      "allowed_tools": ["list_issues", "get_issue"]
    }
  ],
  "stream": true
}
```

**Response (SSE stream):**
```
event: response.created
data: {"type":"response.created","response":{"id":"resp_ghi789","status":"in_progress",...}}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":0,"item":{"type":"mcp_call","id":"mcp_001","server_label":"github","tool_name":"list_issues","arguments":"{\"state\":\"open\"}"}}

event: response.mcp_call.in_progress
data: {"type":"response.mcp_call.in_progress","output_index":0,"item_id":"mcp_001"}

event: response.mcp_call.completed
data: {"type":"response.mcp_call.completed","output_index":0,"item_id":"mcp_001"}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":1,"item":{"type":"message","role":"assistant",...}}

event: response.output_text.delta
data: {"type":"response.output_text.delta","output_index":1,"content_index":0,"delta":"Here are the top 3 open issues by priority:\n\n"}

event: response.output_text.delta
data: {"type":"response.output_text.delta","output_index":1,"content_index":0,"delta":"1. **Critical auth bug** (#142)..."}

event: response.output_text.done
data: {"type":"response.output_text.done","output_index":1,"content_index":0,"text":"Here are the top 3 open issues by priority:\n\n1. **Critical auth bug** (#142)..."}

event: response.completed
data: {"type":"response.completed","response":{"id":"resp_ghi789","status":"completed","usage":{"input_tokens":2100,"output_tokens":480},...}}
```

**Steps:**
1. Client → FastAPI: `POST /v1/responses`
2. FastAPI → Responses: `create_openai_response()`
3. Responses → Orchestrator: `create_response()`
4. Orchestrator → Client: `SSE: response.created` *(event)*
5. Orchestrator → MCP: `list_mcp_tools(endpoint)`
6. MCP → Orchestrator: `tool definitions (cached)`
7. Orchestrator → Inference: `openai_chat_completion()` *(loop)*
8. Inference → Orchestrator: `completion + tool_calls` *(loop)*
9. Orchestrator → Executor: `execute(mcp_tool)` *(loop)*
10. Executor → MCP: `invoke_mcp_tool()` *(loop)*
11. MCP → Executor: `tool result` *(loop)*
12. Executor → Orchestrator: `mcp tool result` *(loop)*
13. Orchestrator → Store: `upsert_response_object()`
14. Orchestrator → Client: `SSE: response.completed | response.incomplete | response.failed` *(event)*

---

### 4. Background Processing

**Toggles:** `background: true`, `file_search: true`, `mcp: true`

Request is queued immediately and processed asynchronously by one of 10 background workers. Client polls for results. Combines file_search and MCP tools.

**Active actors:** Client, FastAPI, Responses, BGWorker, Orchestrator, Inference, Executor, VectorIO, MCP, Store

**Request (submit):**
```http
POST /v1/responses HTTP/1.1
Host: localhost:8321
Authorization: Bearer $API_KEY
Content-Type: application/json

{
  "model": "llama-3.3-70b",
  "input": "What do the uploaded documents say about Q4 results?",
  "tools": [
    {
      "type": "file_search",
      "vector_store_ids": ["vs_abc123"]
    },
    {
      "type": "mcp",
      "server_label": "github",
      "server_url": "http://localhost:8080/sse"
    }
  ],
  "background": true
}
```

**Response (immediate — queued):**
```json
{
  "id": "resp_bg_001",
  "object": "response",
  "status": "queued",
  "model": "llama-3.3-70b",
  "output": [],
  "usage": null
}
```

**Request (poll):**
```http
GET /v1/responses/resp_bg_001 HTTP/1.1
Host: localhost:8321
Authorization: Bearer $API_KEY
```

**Response (completed — after processing):**
```json
{
  "id": "resp_bg_001",
  "object": "response",
  "status": "completed",
  "model": "llama-3.3-70b",
  "output": [
    {
      "type": "file_search_call",
      "id": "fs_002",
      "status": "completed",
      "queries": ["Q4 results"],
      "results": [{"file_id": "file_xyz", "filename": "q4-report.pdf", "score": 0.92, "text": "..."}]
    },
    {
      "type": "mcp_call",
      "id": "mcp_002",
      "server_label": "github",
      "tool_name": "search_code",
      "arguments": "{\"query\":\"Q4 metrics\"}",
      "output": "{\"matches\":[...]}"
    },
    {
      "type": "message",
      "role": "assistant",
      "content": [{"type": "output_text", "text": "Based on the documents and code repository..."}]
    }
  ],
  "usage": {
    "input_tokens": 3200,
    "output_tokens": 580,
    "total_tokens": 3780
  }
}
```

**Steps:**
1. Client → FastAPI: `POST /v1/responses`
2. FastAPI → Responses: `create_openai_response()`
3. Responses → Store: `store queued response`
4. Responses → BGWorker: `enqueue work item`
5. Responses → Client: `queued response (immediate)`
6. BGWorker → Orchestrator: `start processing` *(async)*
7. Orchestrator → MCP: `list_mcp_tools(endpoint)`
8. MCP → Orchestrator: `tool definitions (cached)`
9. Orchestrator → Inference: `openai_chat_completion()` *(loop)*
10. Inference → Orchestrator: `completion + tool_calls` *(loop)*
11. Orchestrator → Executor: `execute(file_search)` *(loop)*
12. Executor → VectorIO: `search_vector_store()` *(loop)*
13. VectorIO → Executor: `ranked chunks + citations` *(loop)*
14. Executor → Orchestrator: `file_search result` *(loop)*
15. Orchestrator → Executor: `execute(mcp_tool)` *(loop)*
16. Executor → MCP: `invoke_mcp_tool()` *(loop)*
17. MCP → Executor: `tool result` *(loop)*
18. Executor → Orchestrator: `mcp tool result` *(loop)*
19. Orchestrator → Store: `upsert_response_object()`
20. Client → FastAPI: `GET /v1/responses/{id} (poll)` *(async)*
21. FastAPI → Store: `get_response_object()`
22. Store → FastAPI: `completed response`
23. FastAPI → Client: `OpenAIResponseObject`

---

### 5. Full Advanced

**Toggles:** `stream: true`, `conversation: true`, `file_search: true`, `mcp: true`, `function_tools: true`, `guardrails: true`

All features enabled — streaming, conversation context, file_search, MCP, function tools, and guardrails. This is the maximum-complexity flow.

**Active actors:** Client, FastAPI, Responses, Orchestrator, Moderation, Inference, Executor, VectorIO, MCP, Conversations, Store

**Request:**
```http
POST /v1/responses HTTP/1.1
Host: localhost:8321
Authorization: Bearer $API_KEY
Content-Type: application/json
Accept: text/event-stream

{
  "model": "llama-3.3-70b",
  "input": "What do the uploaded documents say about Q4 results? Also check the weather in NYC.",
  "tools": [
    {
      "type": "file_search",
      "vector_store_ids": ["vs_abc123"],
      "max_num_results": 5
    },
    {
      "type": "mcp",
      "server_label": "github",
      "server_url": "http://localhost:8080/sse"
    },
    {
      "type": "function",
      "name": "get_weather",
      "description": "Get current weather for a city",
      "parameters": {
        "type": "object",
        "properties": {
          "city": { "type": "string" }
        },
        "required": ["city"]
      }
    }
  ],
  "conversation": "my-session-001",
  "stream": true,
  "guardrails": [{ "id": "content-safety" }]
}
```

**Response (SSE stream — shows key events in the full lifecycle):**
```
event: response.created
data: {"type":"response.created","response":{"id":"resp_full_001","status":"in_progress",...}}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":0,"item":{"type":"file_search_call","id":"fs_003","status":"searching",...}}

event: response.file_search_call.searching
data: {"type":"response.file_search_call.searching","output_index":0,"item_id":"fs_003"}

event: response.file_search_call.completed
data: {"type":"response.file_search_call.completed","output_index":0,"item_id":"fs_003","results":[...]}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":1,"item":{"type":"function_call","id":"fc_001","name":"get_weather","arguments":"{\"city\":\"NYC\"}"}}

event: response.function_call_arguments.done
data: {"type":"response.function_call_arguments.done","output_index":1,"item_id":"fc_001","arguments":"{\"city\":\"NYC\"}"}

event: response.incomplete
data: {"type":"response.incomplete","response":{"id":"resp_full_001","status":"incomplete","incomplete_details":{"reason":"function_tool_yield"},...}}
```

The response status is `incomplete` with reason `function_tool_yield` — the client must execute the `get_weather` function and resume:

**Request (continuation with function result):**
```http
POST /v1/responses HTTP/1.1
Host: localhost:8321
Authorization: Bearer $API_KEY
Content-Type: application/json
Accept: text/event-stream

{
  "model": "llama-3.3-70b",
  "previous_response_id": "resp_full_001",
  "input": [
    {
      "type": "function_call_output",
      "call_id": "fc_001",
      "output": "{\"temperature\":\"72°F\",\"condition\":\"Partly cloudy\"}"
    }
  ],
  "stream": true
}
```

**Response (SSE stream — continuation):**
```
event: response.created
data: {"type":"response.created","response":{"id":"resp_full_002","status":"in_progress","previous_response_id":"resp_full_001",...}}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":0,"item":{"type":"message","role":"assistant",...}}

event: response.output_text.delta
data: {"type":"response.output_text.delta","output_index":0,"content_index":0,"delta":"Based on the Q4 documents, revenue increased by 15%..."}

event: response.output_text.delta
data: {"type":"response.output_text.delta","output_index":0,"content_index":0,"delta":"\n\nAs for NYC, it's currently 72°F and partly cloudy."}

event: response.output_text.done
data: {"type":"response.output_text.done","output_index":0,"content_index":0,"text":"Based on the Q4 documents, revenue increased by 15%...\n\nAs for NYC, it's currently 72°F and partly cloudy."}

event: response.completed
data: {"type":"response.completed","response":{"id":"resp_full_002","status":"completed","usage":{"input_tokens":4200,"output_tokens":340},...}}
```

**Steps:**
1. Client → FastAPI: `POST /v1/responses`
2. FastAPI → Responses: `create_openai_response()`
3. Responses → Conversations: `list_items(conversation_id)`
4. Conversations → Responses: `conversation items`
5. Responses → Store: `get_conversation_messages()`
6. Store → Responses: `stored chat messages`
7. Responses → Orchestrator: `create_response()`
8. Orchestrator → Client: `SSE: response.created` *(event)*
9. Orchestrator → Moderation: `run_guardrails(input)`
10. Moderation → Orchestrator: `pass / blocked`
11. Orchestrator → MCP: `list_mcp_tools(endpoint)`
12. MCP → Orchestrator: `tool definitions (cached)`
13. Orchestrator → Inference: `openai_chat_completion()` *(loop)*
14. Inference → Orchestrator: `completion + tool_calls` *(loop)*
15. Orchestrator → Executor: `execute(file_search)` *(loop)*
16. Executor → VectorIO: `search_vector_store()` *(loop)*
17. VectorIO → Executor: `ranked chunks + citations` *(loop)*
18. Executor → Orchestrator: `file_search result` *(loop)*
19. Orchestrator → Executor: `execute(mcp_tool)` *(loop)*
20. Executor → MCP: `invoke_mcp_tool()` *(loop)*
21. MCP → Executor: `tool result` *(loop)*
22. Executor → Orchestrator: `mcp tool result` *(loop)*
23. Orchestrator → Responses: `emit function_call output (breaks loop)` *(loop)*
24. Orchestrator → Moderation: `run_guardrails(output)`
25. Moderation → Orchestrator: `pass / blocked`
26. Orchestrator → Store: `upsert_response_object()`
27. Responses → Conversations: `add_items(input + output)`
28. Responses → Store: `store_conversation_messages()`
29. Orchestrator → Client: `SSE: response.completed | response.incomplete | response.failed` *(event)*

---

## Code Examples

The simulator generates Python code for each preset. Below are the examples for each flow.

### Simple Completion
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8321/v1", api_key="fake")
response = client.responses.create(
    model="llama-3.3-70b",
    input="Explain how transformers work",
)

print(response.output_text)
```

### RAG + Conversation
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8321/v1", api_key="fake")
response = client.responses.create(
    model="llama-3.3-70b",
    input="What do the uploaded documents say about Q4 results?",
    tools=[
        {
            "type": "file_search",
            "vector_store_ids": ["vs_abc123"],
        },
    ],
    conversation="my-session-001",
)

print(response.output_text)
```

### MCP Agent Loop
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8321/v1", api_key="fake")
response = client.responses.create(
    model="llama-3.3-70b",
    input="List open issues in the repository",
    tools=[
        {
            "type": "mcp",
            "server_label": "github",
            "server_url": "http://localhost:8080/sse",
        },
    ],
    stream=True,
)

for event in response:
    print(event)
```

### Background Processing
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8321/v1", api_key="fake")
response = client.responses.create(
    model="llama-3.3-70b",
    input="What do the uploaded documents say about Q4 results?",
    tools=[
        {
            "type": "file_search",
            "vector_store_ids": ["vs_abc123"],
        },
        {
            "type": "mcp",
            "server_label": "github",
            "server_url": "http://localhost:8080/sse",
        },
    ],
    background=True,
)

# Poll until complete
import time
while response.status in ("queued", "in_progress"):
    time.sleep(1)
    response = client.responses.retrieve(response.id)

print(response.output)
```

### Full Advanced
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8321/v1", api_key="fake")
response = client.responses.create(
    model="llama-3.3-70b",
    input="What do the uploaded documents say about Q4 results?",
    tools=[
        {
            "type": "file_search",
            "vector_store_ids": ["vs_abc123"],
        },
        {
            "type": "mcp",
            "server_label": "github",
            "server_url": "http://localhost:8080/sse",
        },
        {
            "type": "function",
            "name": "get_weather",
            "description": "Get current weather for a city",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"],
            },
        },
    ],
    conversation="my-session-001",
    stream=True,
    guardrail_ids=["content-safety"],
)

for event in response:
    print(event)
```
