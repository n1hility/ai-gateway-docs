# Responses API -- Agentic Loop Phases

This document describes the agentic loop as defined by the **Open Responses specification** and implemented by **OGX** (the server). Where OGX diverges from or extends the spec, the text is marked with **\[OGX\]**.

> **Spec reference:** [Open Responses specification](https://github.com/openresponses/openresponses) — `specification.mdx` and OpenAPI schemas in `schema/`.

## Overview

The Responses API (`POST /v1/responses`) is an OpenAI-compatible endpoint that supports **server-side agentic orchestration**. Unlike the Chat Completions API (single inference call), the Responses API runs an **iterative loop**: the server samples from the model, inspects the output for tool calls, executes internally-hosted tools, feeds results back, and repeats until the model produces a final answer or a limit is reached.

```
Client ──POST /v1/responses──> Server ──agentic loop──> Model Provider
                                  │
                                  ├── tool execution (file_search, web_search, MCP, ...)
                                  │
                                  └── streamed events back to client (SSE / WebSocket)
```

## Key Request Parameters

The spec's `CreateResponseBody` defines 25+ parameters. The ones most relevant to agentic loop behavior:

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | Model identifier |
| `input` | array | Input items (messages, tool outputs, references, etc.) |
| `instructions` | string | System-level instructions for the model |
| `tools` | array | Tool definitions (see Tool Taxonomy below) |
| `tool_choice` | string/object | Controls tool invocation: `auto`, `required`, `none`, or a specific tool |
| `max_tool_calls` | integer ≥ 1, nullable | Limits the number of tool calls in the agentic loop |
| `previous_response_id` | string | Links to a prior response for multi-turn continuation |
| `stream` | boolean | Enable SSE streaming |
| `background` | boolean | Async processing mode (response is queued, not streamed inline) |
| `truncation` | string | Context window overflow handling: `auto` or `disabled` |
| `reasoning` | object | Reasoning behavior configuration |
| `store` | boolean | Whether to persist the response |
| `conversation` | object | Conversation association |
| `service_tier` | string | Processing priority |
| `temperature`, `top_p` | number | Sampling parameters |
| `metadata`, `user` | object/string | Metadata fields |

> **\[OGX\]** OGX adds `max_infer_iters` (default: 10) as its own loop-iteration limiter and `guardrails: bool` for content moderation. These are not part of the Open Responses spec. The spec-defined equivalent for loop limiting is `max_tool_calls`.

## Tool Taxonomy

The spec classifies tools into two hosting categories:

### Externally-hosted tools
Control is yielded outside the server's loop:

| Tool Type | Description |
|-----------|-------------|
| `function` | Client-defined tools — control is yielded back to the developer |
| `mcp` | MCP server-hosted tools — the server orchestrates calls to external MCP servers, but control is **not** yielded back to the developer |

> **Note on MCP:** Although MCP tools are externally-hosted (running on separate MCP servers), the spec explicitly states that control is not first yielded back to the developer. The server handles MCP tool execution within the agentic loop, similar to internally-hosted tools.

### Internally-hosted tools
The server executes these directly within the agentic loop:

| Tool Type | Description |
|-----------|-------------|
| `web_search` | Web search |
| `file_search` | File/document search |
| `code_interpreter` | Code execution sandbox |
| `image_gen` | Image generation |
| `computer` | Computer use (screen, mouse, keyboard) |
| `local_shell` | Local shell command execution |
| `function_shell` | Function-style shell execution |
| `apply_patch` | Code patching tool |
| `custom` | Custom server-side tool implementations |

## Agentic Loop Phases

### Phase 1: Request Intake & Validation

- Client sends `POST /v1/responses` with the parameters described above
- Server validates input
- **\[OGX\]** OGX checks guardrails (content moderation via a configured `moderation_endpoint`). This is not part of the spec.
- If `background: true`, the server queues the response for async processing
  - Emits: `response.queued`
  - The client can poll or reconnect later to retrieve results
- If `previous_response_id` is set, loads prior context: `previous.input + previous.output + new.input`
  - The spec states: *"The server is responsible for translating this logical concatenation into whatever internal representation it uses. Providers MAY apply truncation or compaction as needed."*
- Emits: `response.created`, `response.in_progress`
- Prepares tools list and converts input items to internal representation

### Phase 2: Model Sampling

- The server samples from the model (the spec says *"API samples from model"* — the internal mechanism is implementation-specific)
- **\[OGX\]** OGX implements this as a `POST /v1/chat/completions` call. This is an OGX implementation detail, not a spec requirement.
- Response is streamed back chunk by chunk
- Emits: `response.output_item.added`, `response.output_text.delta`, `response.content_part.done`
- If reasoning is enabled: `response.reasoning.delta`, `response.reasoning.done`

### Phase 3: Tool Call Detection

- Server parses the model's output looking for tool calls
- Three routing outcomes:
  - **No tool calls** → proceed to Phase 6 (completion)
  - **Only `function` tool calls** → proceed to Phase 5 (control yielded to developer)
  - **Server-orchestrated tools** (internally-hosted + MCP) → proceed to Phase 4
  - **Mixed** (both server-orchestrated and `function` calls in same turn) → the server executes server-orchestrated tools and yields the `function` calls to the client simultaneously. **\[OGX\]** OGX handles this by executing server-side tools and returning function calls to the client in the same response.

### Phase 4: Server-Side Tool Execution

- Server executes internally-hosted tools and MCP tools
- Tools may run in parallel if `parallel_tool_calls=true`
- Tool results are appended to the conversation context
- Emits tool-specific events (see Streaming Events below)
- **Loop back to Phase 2** for the next model sampling iteration
- Loop continues until:
  - No more tool calls are emitted
  - `max_tool_calls` limit is reached → response status becomes `incomplete`
  - `max_output_tokens` exhausted → response status becomes `incomplete`
  - **\[OGX\]** `max_infer_iters` reached (OGX-specific, default: 10)

### Phase 5: Developer Control Handoff (Function Calls)

- When the model emits a `function_call` item, **control is yielded back to the developer** (spec language)
- The function call is streamed to the client: `response.function_call_arguments.delta` → `response.function_call_arguments.done`
- Response completes with the function call in the output
- Client executes the tool externally and sends a **new request** with:
  - `previous_response_id` pointing to this response
  - `function_call_output` item containing the tool result
- This starts a **new agentic loop** from Phase 1

### Phase 6: Completion & Response

- Model produces a final text response (no tool calls)
- Server emits: `response.output_item.done`, `response.completed`
- Response is persisted to the responses store (if `store: true`)
- Final `ResponseResource` object sent to client with full input/output items and usage stats
  - The response object type is `"response"` (`"object": "response"`)

### Terminal States

| Status | Cause | Details |
|--------|-------|---------|
| `queued` | `background: true` — response is awaiting processing | Transient state; transitions to `in_progress` |
| `completed` | Model finished normally | — |
| `incomplete` | `max_tool_calls` reached, `max_output_tokens` exhausted, or **\[OGX\]** `max_infer_iters` exceeded | `incomplete_details.reason` carries a structured reason string |
| `failed` | Exception during inference or tool execution | — |

> **Note:** The spec defines `status` as `type: string` (not a closed enum), intentionally allowing extensibility beyond these well-known values.

## Transport

### SSE (Server-Sent Events)
The primary streaming transport. Set `stream: true` in the request to receive events via SSE.

### WebSocket
The spec also defines WebSocket as an alternative transport with specific semantics:
- Single in-flight response per connection
- Client sends `response.create` messages to initiate
- 60-minute connection limit
- Reconnection via `previous_response_id` for continuation
- Supports the same event types as SSE

## Streaming Events Summary

The spec defines **51+ streaming event types**. The table below covers key events per phase. See the [full specification](https://github.com/openresponses/openresponses) for the authoritative list.

| Phase | Key Events |
|-------|-----------|
| Queued | `response.queued` |
| Intake | `response.created`, `response.in_progress` |
| Model Sampling | `response.output_item.added`, `response.output_text.delta`, `response.content_part.done`, `response.output_text.done` |
| Reasoning | `response.reasoning.delta`, `response.reasoning.done`, `response.reasoning_summary.delta/done`, `response.reasoning_summary_part.added/done` |
| Refusal | `response.refusal.delta`, `response.refusal.done` |
| Text Annotations | `response.output_text.annotation_added` |
| Tool Detection | `response.output_item.done` (with tool call type) |
| File Search | `response.file_search_call.searching`, `response.file_search_call.completed` |
| Web Search | `response.web_search_call.searching`, `response.web_search_call.completed` |
| Code Interpreter | `response.code_interpreter_call.interpreting`, `response.code_interpreter_call.code.delta/done` |
| MCP | `response.mcp_call.in_progress`, `response.mcp_call.completed`, `response.mcp_call.failed`, `response.mcp_call.arguments.delta/done` |
| MCP List Tools | `response.mcp_list_tools.in_progress/completed/failed` |
| Image Generation | `response.image_gen_call.generating`, `response.image_gen_call.partial_image`, `response.image_gen_call.completed` |
| Shell | `response.shell_call.command.delta`, `response.shell_call.command.done`, `response.shell_call.in_progress` |
| Apply Patch | `response.apply_patch_call.operation_diff.delta/done` |
| Custom Tools | `response.custom_tool_call.input.delta/done` |
| Function Call | `response.function_call_arguments.delta`, `response.function_call_arguments.done` |
| Completion | `response.completed`, `response.incomplete`, `response.failed` |
| Error | `error` |

## Input Item Types

The spec's `ItemParam` union defines 25 input item types:

`message`, `reasoning`, `compaction_summary`, `system_message`, `developer_message`, `assistant_message`, `function_call`, `function_call_output`, `custom_tool_call`, `custom_tool_call_output`, `computer_call`, `computer_call_output`, `web_search_call`, `image_gen_call`, `code_interpreter_call`, `file_search_call`, `local_shell_call`, `local_shell_call_output`, `mcp_approval_request`, `mcp_approval_response`, `apply_patch_tool_call`, `apply_patch_tool_call_output`, `function_shell_call`, `function_shell_call_output`, `item_reference`

## Additional Spec Features

### `allowed_tools`
A cache-preserving mechanism to restrict which tools the model can invoke without changing the `tools` list. Useful for dynamically narrowing tool access per request.

### `background` Processing
When `background: true`, the server returns immediately with a `queued` response. The client can poll for the completed response or use `previous_response_id` to continue once processing finishes.

## Multi-Turn Continuation

The `previous_response_id` mechanism enables stateful conversations without resending history:

```
Turn 1: POST /v1/responses {input: "What's the weather?", tools: [get_weather]}
  → Response: function_call(get_weather, {location: "SF"})    [resp_1]

Turn 2: POST /v1/responses {previous_response_id: "resp_1", input: [function_call_output(...)]}
  → Response: "It's sunny and 72F in San Francisco"           [resp_2]

Turn 3: POST /v1/responses {previous_response_id: "resp_2", input: "Plan my day"}
  → Response: "Here's your plan for a sunny day..."           [resp_3]
```

The server reconstructs context as: `resp_N.input + resp_N.output + new_input`. Providers MAY apply truncation or compaction as needed during this reconstruction.

---

## \[OGX\] Implementation Details

> **This section is OGX-specific.** None of this is part of the Open Responses specification.

Each iteration of the agentic loop generates a **separate Chat Completions request** to the model provider. An agentic loop with 3 iterations produces 3 independent `/v1/chat/completions` calls:

```
Iteration 1:  OGX ──/v1/chat/completions──> Model Provider
                                             ↓
              OGX <── response (with tool calls) ┘

              OGX executes tools locally

Iteration 2:  OGX ──/v1/chat/completions──> Model Provider
              (messages now include tool results)  ↓
              OGX <── response (with more tool calls) ┘

              OGX executes tools locally

Iteration 3:  OGX ──/v1/chat/completions──> Model Provider
              (messages include all tool results)  ↓
              OGX <── response (final answer) ─────┘
```

### \[OGX\] Key Differences from Spec

| Aspect | Open Responses Spec | OGX Implementation |
|--------|--------------------|--------------------|
| Loop limiter | `max_tool_calls` | `max_infer_iters` (default: 10) |
| Content moderation | Not defined | `guardrails: bool` + `moderation_endpoint` |
| Model sampling | "API samples from model" (abstract) | `POST /v1/chat/completions` |
| Incomplete reason | `incomplete_details.reason` (free string) | Reports `max_infer_iters exceeded` |

---

## Single-Pass vs Multi-Pass: IPP Plugin Placement

The IPP (Inference Payload Processor) plugin pipeline was designed for **single-pass** Chat Completions traffic: one request in, one response out. In the **multi-pass** Responses API flow, each agentic loop iteration generates a nested `/v1/chat/completions` call through IPP. Not all IPP plugins should re-execute on every iteration — some are invariant, some lack the context they need, and some are handled better in the agentic loop.

### The Problem: IPP Sees a Fragment

When the client sends `POST /v1/responses`, the request body contains:

```json
{
  "model": "claude-haiku-4-5",
  "input": "What do the docs say about Q4?",
  "previous_response_id": "resp_123",
  "tools": [...]
}
```

The **conversation history does not exist in the request body**. It is reconstructed later by agentic loop plugins (`conversation-manager`, `hydrate-prompt`) that load prior responses and conversation items from external stores. IPP plugins that inspect request content at intake or per-iteration see only the bare `input` field — not the full conversation context.

This has direct implications for which plugins are effective at which stage.

### Plugin Disposition: Single-Pass vs Multi-Pass

| IPP Plugin | Single-Pass (Chat Completions) | Multi-Pass Intake | Multi-Pass Per-Iteration | Rationale |
|---|---|---|---|---|
| **Auth (Authorino)** | Runs | Runs | **Skipped** | API key validated once at intake. Re-validation per-iteration is redundant — the key cannot change mid-loop. |
| **Rate Limit (check)** | Runs | Runs | **Skipped** | Quota checked at intake. Per-iteration checking is redundant — the intake gate already confirmed quota exists. |
| **Rate Limit (report)** | Runs | — | **Runs** | Token accounting must happen per-iteration. Each model call consumes tokens; Limitador needs the actual usage to decrement the quota correctly. |
| **guardrails (request)** | Runs | **Skipped in IPP** | **Skipped in IPP** | In single-pass, the full message history is in the request body — guardrails can evaluate the complete conversation. In multi-pass, IPP sees only the bare `input` field without conversation history. Input guardrails run **post-hydration in the agentic loop** instead, where the full reconstructed context is available. |
| **guardrails (response)** | Runs | — | **Skipped in IPP** | In the agentic loop, response guardrails run in the agentic loop itself, which has richer context: tool results, conversation history, and the original Responses API items. The IPP response guardrails would check a Chat Completions-formatted translation of the same content — less context, same cost. |
| **intelligent-model-selection** | Runs | Skipped | **Skipped** | Model selection is invariant across loop iterations. The model was chosen at intake and does not change mid-loop. |
| **model-provider-resolver** | Runs | Skipped | **Skipped** | The provider, target model ID, and endpoint URL are resolved from the ExternalModel CRD once. These are static for the lifetime of the request. |
| **body-field-to-header** | Runs | Skipped | **Skipped** | Extracts the `model` field to `X-Gateway-Model-Name` header. The model name is invariant — no need to re-extract per-iteration. |
| **api-translation (request)** | Runs | — | **Runs** | Each iteration sends a different messages array (with appended tool results). The translation from OpenAI Chat Completions format to provider-native format must happen per-iteration. |
| **apikey-injection** | Runs | Skipped | **Skipped** | The provider API key is read from a Kubernetes Secret once. The credential does not change between iterations. |
| **api-translation (response)** | Runs | — | **Runs** | Each iteration's provider response must be translated back to OpenAI Chat Completions format so the agentic loop can parse tool calls and build Responses API output items. |

### Where Guardrails Actually Run

Because IPP guardrails are ineffective for Responses API traffic (they see a content fragment without history), the agentic loop owns guardrails for multi-pass flows:

| Check | Where | When | Context Available |
|---|---|---|---|
| **Input safety** | Agentic loop (`guardrails` plugin, `on_request` hook) | After hydration, before first iteration | Full conversation history + new input + tool definitions |
| **Output safety** | Agentic loop (`guardrails` plugin, `on_response` hook) | After each iteration's model response | Model output + tool results + full conversation context |
| **Input safety** | IPP (`guardrails` request plugin) | **Skipped for Responses API** | Would only see bare `input` — no history |
| **Output safety** | IPP (`guardrails` response plugin) | **Skipped for Responses API** | Agentic loop already checked with richer context |

> **Note:** Input guardrails in the agentic loop should run in **all** Responses API flows (Simple, RAG, MCP, Background, Full Advanced), not only in flows that explicitly enable guardrails. Any flow that supports `previous_response_id` or `conversation` can carry multi-turn context that requires safety evaluation.

### CycleState Isolation

The nested IPP call operates with its **own CycleState**, isolated from the agentic loop's CycleState (see `rfc-ai-plugin-concept.md` §3.6). This means values resolved at intake (provider, target model, credential reference) cannot be implicitly shared with per-iteration IPP runs.

To skip redundant plugins per-iteration, the implementation must either:

1. **Forward resolved values via headers** — the agentic loop's `inference-caller` sets headers (e.g., `X-Agentic-Iteration: true`, pre-resolved provider headers) that IPP plugins read to short-circuit
2. **Use conditional execution** — IPP pipeline configuration gates plugins on a CycleState predicate (e.g., `skip_if: is_agentic_iteration`) set by a thin intake plugin that reads the forwarding headers
3. **Thin IPP mode** — a separate, minimal IPP pipeline configuration for agentic loop calls that only includes api-translation and rate-limit-report

The choice between these mechanisms is an implementation decision outside the scope of this document, but the principle holds: **plugins that resolve invariant data or lack sufficient context should not re-execute per-iteration**.
