# Unified Pipeline -- AI Plugin Catalog

This document catalogs the unified plugin pipeline for the AI Inference Gateway. The pipeline handles both `/v1/chat/completions` (single-pass inference proxying) and `/v1/responses` (multi-iteration agentic orchestration) using a single set of plugins. Plugins comply with the [AI Plugin RFC](rfc-ai-plugin-concept.md) and use **conditional execution** (RFC §3.8) to gate flow-specific behavior.

The pipeline merges the existing IPP (Inference Payload Processor) plugins with the agentic loop plugins derived from the [Responses API agentic loop specification](responses-api-agentic-loop.md). See [IPP Plugin Catalog](#ipp-plugin-catalog) for the existing implementations being reused and [Plugin Overlap Analysis](#plugin-overlap-analysis) for the issues that motivated the unification.

> **Naming convention:** plugin names use kebab-case, following the existing IPP convention (`model-provider-resolver`, `body-field-to-header`, etc.).

---

## Architecture: Plugins vs. Executor

Not everything in the agentic loop is a plugin. The RFC's plugin contract (Plugin Actions, CycleState, no direct client writes, no pipeline forking) constrains what a plugin can do. Three concerns fall outside this contract and are **executor responsibilities**:

| Concern | Why not a plugin | Owner |
|---------|-----------------|-------|
| **Background queueing** | Returns to the client AND continues processing — a pipeline fork (RFC §5.5) | Executor / API routing layer |
| **SSE/WebSocket streaming** | Writes directly to the client connection (RFC §5.8); other plugins depend on it (RFC §5.2) | Executor / transport layer |
| **Context truncation** | Used by multiple plugins as a shared utility (RFC §5.2 violation); needs to run at two different pipeline positions | Inline logic within `hydrate-prompt` and `inference-caller` |

The **loop executor** owns the iteration lifecycle: it invokes plugins in order, reads their Plugin Actions (Continue, Reject, Yield), re-enters the loop when tool results are appended, and flushes streaming events from CycleState to the client transport. Plugins signal intent (e.g., `loop-controller` writes `loop_action = "continue" | "stop"` to CycleState); the executor acts on it.

See [Executor Responsibilities](#executor-responsibilities) for details.

---

## Plugin Overview

The unified pipeline processes both `/v1/chat/completions` and `/v1/responses` requests. A request flows through: **gateway intake → inference (loops for Responses API) → completion**. Within the inference loop: **api-translation → model call → api-translation → guardrails → tool detection → tool execution → loop back**.

Each plugin is classified by:

- **Boundary:** **Local** (in-process, no external calls), **External** (network calls to external services), or **Sandboxed** (host-level operations, disabled by default)
- **IPP Status:** **Exists** (already implemented in IPP), **Extends** (existing IPP plugin, extended for Responses API), or **New** (no IPP equivalent)
- **Complexity:** **S** (simple logic, <500 LoC), **M** (moderate, 500–2000 LoC), **L** (complex, 2000+ LoC or many integrations), **Undefined** (scope not yet defined)
- **Flows:** **Both** (Chat Completions + Responses API) or **Responses only**

| # | Phase | Plugin | Boundary | IPP | Size | Flows | Description |
|---|-------|--------|----------|-----|------|-------|-------------|
| 1 | Intake | `auth` | External | Exists | S | Both | Validates API key via MaaS API, injects identity headers |
| 2 | Intake | `rate-limit` | External | Exists | S | Both | Token-based quota check (request) + usage reporting (response) |
| 3 | Intake | `request-validator` | Local | Extends | M | Both | Format-aware validation: Chat Completions messages/params OR Responses API input/tools/conversation |
| 4 | Intake | `hydrate-prompt` | External | New | L | Responses | Loads `previous_response_id` chain, resolves `item_reference`, handles `compaction_summary`, inline truncation |
| 5 | Intake | `conversation-manager` | External | New | M | Responses | Dual-hook: loads conversation items + messages (`on_request`), saves state (`on_response`) |
| 6 | Intake | `tool-registry` | External | New | M | Responses | Resolves tools, discovers MCP servers, applies `allowed_tools` filtering |
| 7 | Intake + Loop | `guardrails` | External | Extends | M | Both | Dual-hook: input guardrails (`on_request`) + output guardrails per iteration (`on_response`). NeMo Guardrails endpoint. |
| 8 | Intake | `model-provider-resolver` | External | Exists | S | Both | Watches `ExternalModel` CRDs, resolves model → provider + endpoint + target model ID |
| 9 | Loop | `api-translation` | Local | Extends | L | Both | Format-aware: Responses API items → provider-native OR Chat Completions messages → provider-native. 5 providers. Streaming events, refusal detection, annotations. |
| 10 | Loop | `body-field-to-header` | Local | Exists | S | Both | Extracts `model` field → `X-Gateway-Model-Name` header for routing |
| 11 | Loop | `base-model-to-header` | External | Exists | S | Both | Resolves LoRA adapter → base model via K8s ConfigMaps → `X-Gateway-Base-Model-Name` header |
| 12 | Loop | `apikey-injection` | External | Exists | S | Both | Reads provider API key from K8s Secret, injects auth header |
| 13 | Loop | `inference-caller` | External | New | M | Both | Direct model call with inline truncation, context window check |
| 14 | Loop | `tool-call-handler` | Local | New | S | Responses | Detects, classifies, and routes tool calls from model output |
| 15 | Loop | `loop-controller` | Local | New | S | Both | Signals `continue`/`stop_completed`/`stop_yield`/`stop_incomplete` to executor; tracks iteration + token limits |
| 16 | Loop | `server-tool-executor` | External | New | L | Responses | Dispatch table to 6+ tool backends (file_search, web_search, code_interpreter, image_gen, computer, custom) |
| 17 | Loop | `host-tool-executor` | Sandboxed | New | M | Responses | Executes local_shell/apply_patch in sandbox. Disabled by default, admin opt-in. |
| 18 | Loop | `mcp-executor` | External | New | L | Responses | MCP protocol: session management, tool invocation, approval flow |
| 19 | Completion | `response-assembler` | Local | New | M | Responses | Builds `ResponseResource` with output items, usage, context edits, echo-back fields |
| 20 | Completion | `response-store` | External | New | S | Responses | Persists response when `store: true` |

> **Plugin count: 20.** Of these, **7 already exist in IPP** (reused as-is or extended) and **13 are new** for Responses API support. Dual-hook plugins (`api-translation`, `guardrails`, `conversation-manager`, `rate-limit`) use `on_request` / `on_response` hooks within a single plugin instance.
>
> **Complexity breakdown:** 9× S, 6× M, 4× L. The L-sized plugins (`hydrate-prompt`, `api-translation`, `server-tool-executor`, `mcp-executor`) are the critical path items.

---

## Open Points

Items that are specified in the plugin descriptions but not yet covered by flow scenarios or require further design.

| # | Area | Description | Reference |
|---|------|-------------|-----------|
| 1 | MCP approval flow | The `mcp-executor` describes a 5-step approval round-trip (server yields `mcp_approval_request`, client resumes with `mcp_approval_response`), but no flow scenario demonstrates this interaction pattern. Needs a dedicated flow (e.g., Flow 8) or a sub-scenario under Flow 3. | [mcp-executor](#mcp-executor) |
| 2 | Error/failure scenarios | No flow scenario demonstrates failure paths — inference timeout, tool execution error, or guardrail mid-loop rejection with retry. The [Error Handling](#error-handling) section documents the mechanics but a concrete flow (e.g., Flow 9: guardrail rejects model output mid-loop, model retries, succeeds on second attempt) would validate the design. | [Error Handling](#error-handling) |
| 3 | `rate-limit` per-iteration accounting | The unified pipeline changed `rate-limit` from running N+1 times (nested IPP) to running once at intake + reading per-iteration usage from CycleState. This new behavior is only described in the [Architectural Notes](#architectural-notes) — the plugin itself has no detailed section documenting the CycleState-based accounting, the `iteration_usage` key, or how per-iteration usage feeds into `loop-controller` totals. | [Architectural Notes](#unified-pipeline-architecture) |
| 4 | `intelligent-model-selection` | Planned IPP plugin that selects optimal model/provider based on cost, latency, and availability. Scope and complexity are undefined. Once designed, it should be added to the plugin table (Intake phase, Local boundary, Both flows) and the participation matrix. | — |

---

## Phase 1: Request Intake

### `request-validator`

> **IPP Status: Extends** — IPP has implicit Chat Completions validation. This plugin merges it with the Responses API validator into a single format-aware plugin. **Complexity: M**

Validates the incoming request body based on the API path. For `/v1/chat/completions`, validates the Chat Completions schema (model, messages, sampling parameters). For `/v1/responses`, validates against the spec's `CreateResponseBody` schema. Writes `request_format` (`chat_completions` or `responses_api`) to CycleState for downstream plugins to branch on.

This plugin performs **validation only** — it does not route or classify. Conditional plugin activation (e.g., whether `hydrate-prompt` or `conversation-manager` runs) is declared in the **pipeline configuration**, which matches on CycleState values written by this plugin.

- **Boundary:** Local
- **Input:** raw request body
- **Output:** validated and normalized parameters written to CycleState
- **CycleState writes (always):** `request_format`, `validated_params`
- **Validates (Responses API):**
  - `model` — required string
  - `input` — array of `ItemParam` (26 input item types including `item_reference`, `compaction_summary`, `mcp_approval_request/response`)
  - `tools` — array of tool definitions with correct types
  - `tool_choice` — `auto` | `required` | `none` | specific tool reference
  - `max_tool_calls` — integer >= 1, nullable
  - `max_output_tokens` — integer >= 1, nullable
  - `truncation` — `auto` | `disabled`
  - `reasoning` — reasoning behavior configuration
  - `text` — `TextParam` with `format` (plain text, JSON schema w/ optional `strict` mode) and `verbosity` (`VerbosityEnum`)
  - `temperature`, `top_p`, `presence_penalty`, `frequency_penalty`, `top_logprobs` — sampling parameter bounds
  - `include` — array of `IncludeEnum` values controlling which additional data to include in the response
  - `stream`, `background`, `store` — boolean flags
  - `previous_response_id` — valid response ID format
  - `conversation`, `metadata`, `user`, `service_tier`
  - `prompt_cache_key` — string, max 64 chars
  - `prompt_cache_retention` — `PromptCacheRetentionEnum`
  - `safety_identifier` — string, max 64 chars (stable identifier for safety monitoring)
  - `parallel_tool_calls` — boolean
- **Mutual exclusivity:** rejects if both `previous_response_id` and `conversation` are set (400 error)
- **Mutual exclusivity:** rejects if both `stream` and `background` are set (400 error)
- **CycleState writes:** `validated_params`, `has_previous_response_id`, `has_conversation`, `has_tools`, `is_background`, `is_streaming`
- **Plugin Action:** Continue (on success) or Reject (on validation failure)
- **Error:** rejects with structured error if validation fails

### `hydrate-prompt`

> **IPP Status: New** — no IPP equivalent. **Complexity: L**

Hydrates the full prompt by loading and merging conversation history when `previous_response_id` is set.

- **Boundary:** External (reads from the response store)
- **Conditional execution:** only runs when CycleState `has_previous_response_id = true`
- **Input:** `previous_response_id` and `input` from CycleState
- **Output:** fully hydrated prompt = `previous.input + previous.output + new.input`, written to CycleState
- **Behavior:**
  - Loads the referenced response from the response store
  - Recursively resolves chained `previous_response_id` for multi-turn conversations
  - **`item_reference` resolution:** resolves `ItemReferenceParam` items (`{type: "item_reference", id: "msg_123"}`) by looking up the referenced item in the response store and replacing the reference with actual item content. Returns Reject if the referenced item doesn't exist
  - **`compaction_summary` processing:** if the input contains `CompactionSummaryItemParam` items (produced by the compaction API), includes them as-is in the hydrated prompt — they replace the conversation history they summarize
  - Merges prior input/output with the new request's input items into a single hydrated prompt
  - **Inline truncation:** if `truncation = "auto"`, trims older context to fit the model's context window; if `truncation = "disabled"`, returns Reject when context exceeds the window
  - **Context edits:** when truncation is applied, writes a `ContextEdit` record to CycleState (`{type: "truncation", summary: "...", details: {items_removed, tokens_removed}}`). The `response-assembler` includes these in the `ResponseResource.context_edits` field
- **Plugin Action:** Continue (on success) or Reject (on 404, unresolvable `item_reference`, or context overflow with truncation disabled)
- **Error:** 404 if `previous_response_id` or referenced item is not found

### `tool-registry`

> **IPP Status: New** — no IPP equivalent. **Complexity: M**

Resolves and registers the set of tools available for the current request.

- **Boundary:** External (MCP tool discovery requires connecting to MCP servers)
- **Input:** `tools` array and `allowed_tools` from the request
- **Output:** registered tool set with resolved executors
- **Behavior:**
  - Classifies each tool into externally-hosted (`function`, `mcp`) or internally-hosted (`web_search`, `file_search`, `code_interpreter`, `image_gen`, `computer`, `local_shell`, `function_shell`, `apply_patch`, `custom`)
  - Applies `allowed_tools` filtering (cache-preserving restriction of which tools the model can invoke)
  - Resolves `mcp` tool definitions by connecting to MCP servers and listing available tools (emits `response.mcp_list_tools.in_progress/completed/failed`)
  - Applies `tool_choice` constraints
  - Stores the resolved tool map in CycleState for downstream plugins
  - **Size bound:** MCP discovery can return many tools with complex JSON schemas. The plugin should enforce a configurable maximum tool count (e.g., `max_tools_per_request`) and reject or truncate oversized tool sets to prevent unbounded CycleState growth

### `conversation-manager`

> **IPP Status: New** — no IPP equivalent. **Complexity: M**

Manages conversation-based context loading and persistence. A dual-hook plugin (`on_request` for load, `on_response` for save) using the `conversation` parameter (mutually exclusive with `previous_response_id` / `hydrate-prompt`).

- **Boundary:** External (reads/writes to conversation store)
- **Conditional execution:** only runs when CycleState `has_conversation = true`
- **Mutual exclusivity:** if `previous_response_id` is also set, the `request-validator` rejects the request (400 error)

**`on_request` hook (intake phase):**
- **Input:** `conversation` object from the request
- **Output:** hydrated context from conversation history (items + raw messages) written to CycleState
- **Behavior:**
  - Loads structured conversation items from the conversation store (for UI/metadata)
  - Loads raw chat messages from the message store (for inference context)
  - Merges conversation history with the new request's input items

**`on_response` hook (completion phase):**
- **Input:** finalized `ResponseResource` with input + output items
- **Output:** updated conversation state
- **Behavior:**
  - Writes input + output items to the conversation store
  - Persists the full message array for the next turn's inference context

- **Dual storage:** mirrors OGX's pattern where conversation items and raw messages are stored separately, enabling both conversation-level UI and accurate inference continuity

> **Why not merge `hydrate-prompt` and `conversation-manager`?** They are peers (both load context), not superset/subset. They differ on every axis:
>
> | | `hydrate-prompt` | `conversation-manager` |
> |--|---|---|
> | **Data model** | Immutable response chain — frozen snapshots linked by ID | Mutable conversation — a living thread that accumulates turns |
> | **Store** | Response store (single, read-only) | Dual store: conversation items (UI) + message store (inference) |
> | **Read** | Recursive chain walk (`resp_3 → resp_2 → resp_1`) | Direct load of current conversation state |
> | **Write** | None — `response-store` persists the new link independently | `on_response` writes back items + messages after completion |
> | **Hooks** | `on_request` only | `on_request` + `on_response` |
> | **Unique features** | `item_reference` resolution, `compaction_summary`, inline truncation with `ContextEdit` | Dual-storage sync, conversation-level UI metadata |
> | **History owner** | Client (chooses which response to chain from) | Server (maintains canonical state) |
>
> A merged plugin would have two code paths sharing nothing but an interface — different stores, different read patterns, different write lifecycles, different spec features. The mutual exclusivity is already enforced by the validator; encoding it inside a single plugin adds no benefit.

> **`background` mode** is handled by the **executor**, not a plugin. When `background: true`, the executor returns an immediate `queued` response to the client and enqueues the pipeline for async processing. This is a pipeline fork — returning to the client AND continuing processing — which violates the Plugin Action contract (RFC §5.5). See [Executor Responsibilities](#executor-responsibilities).

---

## Guardrails

### `guardrails`

> **IPP Status: Extends** — merges IPP `request-guard (nemo)` + `response-guard (nemo)` into a single dual-hook plugin. **Complexity: M**

Evaluates safety and content policies on both the incoming request and the model's response. A single plugin with `on_request` and `on_response` hooks, sharing configuration and connection management for the moderation endpoint (e.g., NVIDIA NeMo Guardrails, a configured `moderation_endpoint`).

- **Boundary:** External (calls guardrails/moderation endpoint)
- **Configurable:** enabled/disabled per request or globally (e.g., `guardrails: bool`)
- **Replaces:** the separate `request-guard (nemo)` and `response-guard (nemo)` plugins in the IPP pipeline — same NeMo endpoint, same evaluation logic, one plugin instance instead of two

**`on_request` hook (intake phase):**
- **Input:** hydrated prompt from CycleState
- **Output:** pass / block decision
- **Behavior:** sends the request content to the moderation endpoint. On violation: returns **Reject** — the executor short-circuits and returns an error. On pass: returns **Continue**

**`on_response` hook (each loop iteration):**
- **Input:** translated model response from CycleState (post `api-translation` response hook)
- **Output:** pass / block decision
- **Behavior:** sends the response content to the moderation endpoint. On pass: returns **Continue**. On violation: returns **Reject** — but mid-loop rejection semantics differ from intake rejection (see below)
- **Plugin Action:** Continue or Reject

**Mid-loop rejection semantics:** When the `on_response` hook rejects during an agentic loop iteration (not the final response), the executor does NOT terminate the entire request. Instead:
1. The violating model output is discarded
2. A guardrail violation is written to CycleState as a synthetic tool result (telling the model its output was blocked and why)
3. The executor continues the loop — the model receives the violation feedback and produces a new response
4. If the model's output is blocked for `max_guardrail_retries` consecutive iterations (configurable, default: 2), the executor terminates with status `failed` and reason `guardrail_violation`

This avoids wasting completed tool results from prior iterations while still enforcing content policies.

---

## API Translation

### `api-translation`

> **IPP Status: Extends** — merges IPP `api-translation` (Chat Completions ↔ provider) with Responses API format support. The provider-specific translation core (Anthropic, Bedrock, Vertex, Azure) is shared. **Complexity: L**

Format-aware translation between the gateway's input format and provider-native formats. Reads `request_format` from CycleState to determine the input parser: for `chat_completions`, parses messages/params; for `responses_api`, parses input items/tools. Provider-specific output logic is shared regardless of input format, eliminating the double-hop translation (Responses API → Chat Completions → provider).

A single plugin with `on_request` and `on_response` hooks, sharing provider configuration (endpoint URLs, headers, version strings).

- **Boundary:** Local (pure data transformation, no network calls)
- **Provider support:** OpenAI (native passthrough), Anthropic, Azure OpenAI, AWS Bedrock, Google Vertex AI
- **Plugin Action:** Continue

**`on_request` hook (before inference):**
- **Input:** Responses API input items OR Chat Completions messages, tools, and parameters from CycleState
- **Output:** provider-native request body written to CycleState
- **Behavior:**
  - Converts input (Responses API items or Chat Completions messages) to provider-native messages format
  - Converts tool definitions to provider-native tool format
  - Applies `instructions` as system message in the provider-specific way
  - Translates `text.format` to provider-native `response_format` (e.g., `{type: "json_schema", ...}` for OpenAI, equivalent for other providers)
  - Translates `text.verbosity` to provider-native parameters (if supported)
  - Passes `prompt_cache_key` / `prompt_cache_retention` to provider-native caching parameters (if supported)
  - Passes all sampling parameters: `temperature`, `top_p`, `presence_penalty`, `frequency_penalty`, `top_logprobs`, `max_output_tokens`
  - Rewrites paths and headers as needed (e.g., `/v1/messages` for Anthropic, `/v1/chat/completions` for OpenAI)
  - Adds provider-specific headers (e.g., `anthropic-version: 2023-06-01`)

**`on_response` hook (after inference):**
- **Input:** provider-native response from CycleState (streamed or buffered)
- **Output:** Responses API output items + streaming events written to CycleState
- **Behavior:**
  - Converts provider-native response structure to output format (Responses API output items or Chat Completions choices)
  - Normalizes content fields (e.g., Anthropic `content[]` → Responses API output items)
  - Maps stop reasons (e.g., `end_turn` → `stop`, `tool_use` → tool call items)
  - **Refusal detection:** detects provider-native refusal indicators (e.g., OpenAI `finish_reason: "content_filter"`, Anthropic refusal content) and produces `RefusalContent` in output items + `response.refusal.delta/done` streaming events
  - **Annotation creation:** when tool results contain citations, creates annotation objects (`FileCitationBody`, `UrlCitationBody`) attached to output text content parts and emits `response.output_text.annotation_added` events
  - **Reasoning summaries:** maps provider reasoning summary outputs to `response.reasoning_summary.delta/done` and `response.reasoning_summary_part.added/done` events
  - **`include` filtering:** reads the `include` array from CycleState to determine which optional data to include (e.g., `file_search_call.results`, `message.output_text.logprobs`, `reasoning.encrypted_content`)
  - Normalizes token usage (e.g., `input_tokens`/`output_tokens` → unified `usage` object) and writes per-iteration usage to CycleState for accumulation
  - **Streaming event production:** maps provider-specific chunks to spec events (e.g., Anthropic `content_block_delta` → `response.output_text.delta`, OpenAI `choices[0].delta` → `response.output_item.added` + `response.output_text.delta`, reasoning tokens → `response.reasoning.delta/done`)
  - Writes events to `CycleState.event_queue` — the executor flushes them to the client transport

---

## Loop Control

### `loop-controller`

> **IPP Status: New** — no IPP equivalent. For Chat Completions, always signals `stop_completed` (single-pass). **Complexity: S**

Evaluates loop state and **signals** whether the executor should continue iterating or terminate. The plugin does not control pipeline execution directly — it writes a `loop_action` decision to CycleState, and the **executor** acts on it (RFC §5.1, §5.5).

- **Boundary:** Local
- **Input:** current iteration state from CycleState (`current_tool_call_count`, `current_iteration`, `total_output_tokens`, `tool_routing`)
- **Output:** writes `loop_action` to CycleState: `continue` | `stop_completed` | `stop_incomplete` | `stop_yield` | `stop_failed`
- **Termination signals:**
  - No tool calls in model output → `stop_completed` (but see `tool_choice: required` below)
  - Only `function` tool calls → `stop_yield` (executor exits loop; `response-assembler` includes function call items in output)
  - `max_tool_calls` limit reached → `stop_incomplete` (with `incomplete_details.reason`)
  - `max_output_tokens` exhausted → `stop_incomplete`
  - Exception during inference or tool execution → `stop_failed`
- **`tool_choice: required` enforcement:** if `tool_choice = "required"` AND the model produces no tool calls, the plugin does NOT signal `stop_completed`. Instead it writes `loop_action = continue` with a retry flag, and the executor re-invokes inference (up to `max_tool_choice_retries`, default: 1). If the model still doesn't call a tool after retries, the plugin signals `stop_failed` with `error.code = "tool_choice_violation"`
- **Usage accumulation:** reads per-iteration token usage from CycleState (written by `api-translation` response hook) and accumulates `total_input_tokens`, `total_output_tokens`, `total_cached_tokens`, `total_reasoning_tokens` across all iterations. The `response-assembler` reads these totals to build the `Usage` object
- **CycleState reads/writes:** `current_tool_call_count`, `current_iteration`, `total_output_tokens`, `total_input_tokens`, `total_cached_tokens`, `total_reasoning_tokens`, `tool_choice`
- **Plugin Action:** Continue (always — the decision is in CycleState, not the Plugin Action)

**Loop ownership:** The executor owns the loop. After each iteration, the executor reads `loop_action` from CycleState. If `continue`, it re-invokes the pipeline from `api-translation` (request hook) through `tool-call-handler`, then fans out to the appropriate executor plugins based on `tool_routing[]`. This keeps loop control in the executor where it belongs, while `loop-controller` provides the decision logic as a composable plugin.

---

## Phase 2: Model Sampling

### `inference-caller`

> **IPP Status: New** — in the pre-unification design, IPP forwarded the request to the model provider via Gateway routing; there was no explicit inference-caller plugin. In the unified pipeline, this plugin makes direct model calls after `apikey-injection` has set up credentials. **Complexity: M**

Calls the model provider for inference. Reads the translated provider-native request from CycleState and forwards it directly to the model endpoint.

- **Boundary:** External (calls the model provider endpoint)
- **Input:** provider-native request from CycleState (written by `api-translation` request hook), sampling parameters, provider credentials (injected by `apikey-injection`)
- **Output:** provider-native model response written to CycleState (consumed by `api-translation` response hook)
- **Behavior:**
  - **Inline truncation:** before each call, checks context size against the model's context window. If `truncation = "auto"`, trims older context to fit and writes a `ContextEdit` record to CycleState; if `truncation = "disabled"` and context exceeds the window, returns Reject
  - Applies all sampling parameters from CycleState (delegated from `api-translation` request hook)
  - Applies `reasoning` configuration
  - Makes a direct HTTP call to the model provider endpoint using the credentials and path from CycleState
  - Writes response chunks to CycleState (the `api-translation` response hook produces streaming events from these)
- **Plugin Action:** Continue (on success) or Reject (on truncation overflow with `disabled` mode)

---

## Phase 3: Tool Call Detection

### `tool-call-handler`

> **IPP Status: New** — no IPP equivalent; Chat Completions has no server-side tool execution. **Complexity: S**

Detects, classifies, and routes tool calls from model output in a single pass.

- **Boundary:** Local
- **Input:** model output (from `api-translation` response hook)
- **Output:** routing decisions + streaming events written to CycleState
- **Behavior:**
  - Extracts tool call items from the model response
  - Resolves each tool call against the registered tool set (from `tool-registry`)
  - Tags each call as externally-hosted (`function`, `mcp`) or internally-hosted
  - Detects `mcp_approval_request` output items — writes them to CycleState with `has_mcp_approval = true` so the executor can trigger a yield (same mechanism as function tool handoff)
  - Writes routing decisions to CycleState — does not reference or invoke specific executor plugins by name (RFC §5.2). The executor reads CycleState and uses **pipeline configuration conditional execution rules** to gate which executor plugins run
- **CycleState writes:**
  - `tool_routing[]` — list of `{tool_call_id, executor_type, arguments}` entries (e.g., `executor_type: "file_search"`, `executor_type: "mcp"`, `executor_type: "function"`)
  - `has_server_side_tools` — boolean, true if any internally-hosted or MCP tool calls exist
  - `has_function_tools` — boolean, true if any `function` tool calls exist
  - `parallel_dispatch` — boolean, from `parallel_tool_calls` request parameter
- **Event queue writes:**
  - `response.output_item.done` (with tool call type) for each detected tool call
  - `response.function_call_arguments.delta`, `response.function_call_arguments.done` (for function tool calls)
- **Routing outcomes (as CycleState signals):**
  - **No tool calls** → `tool_routing` is empty; `loop-controller` reads this and signals `stop_completed`
  - **Only server-orchestrated tools** → executor fan-outs to matching executor plugins based on `executor_type`, then continues the loop
  - **Only `function` tools** → `loop-controller` reads `has_function_tools` and signals `stop_yield`; executor exits loop, `response-assembler` includes function call items in output
  - **Mixed** (both server-orchestrated and `function` calls) → server-side tools execute, their results are appended to CycleState **and** included as output items in the response alongside the `function_call` items, then `loop-controller` signals `stop_yield`. The client receives a single response containing both server-side tool results and function call items — matching the spec's "simultaneously" semantics. On resume, the client sends `function_call_output`; the model sees both the server-side results (already processed) and the function output
- **Plugin Action:** Continue

---

## Phase 4: Tool Execution

All tool executor plugins follow the same interface contract:

- **Input:** tool call arguments (from CycleState, keyed by `tool_routing[]` entries)
- **Output:** tool result (appended to CycleState for next iteration)
- **Error:** tool execution failure is reported back to the model as a tool error result
- **Dispatch:** the executor reads `tool_routing[]` from CycleState and invokes the matching executor plugins. If `parallel_dispatch = true`, server-side tools run concurrently

### `server-tool-executor`

> **IPP Status: New** — no IPP equivalent. **Complexity: L**

Executes all internally-hosted tools via a configuration-driven backend dispatch table. Instead of one plugin per spec tool type, a single plugin routes to the appropriate backend based on `executor_type` from `tool_routing[]`.

- **Boundary:** External (calls tool backend services)
- **Backend dispatch table:**

| Tool Type | Backend | Events |
|-----------|---------|--------|
| `file_search` | Vector store service | `file_search_call.searching`, `file_search_call.completed` |
| `web_search` | Web search API (when available) | `web_search_call.searching`, `web_search_call.completed` |
| `code_interpreter` | Sandbox service (when available) | `code_interpreter_call.interpreting`, `code_interpreter_call.code.delta/done` |
| `image_gen` | Image gen service (when available) | `image_gen_call.generating`, `image_gen_call.partial_image`, `image_gen_call.completed` |
| `computer` | Remote desktop service (when available) | (tool-specific) |
| `custom` | Pluggable backend registry | `custom_tool_call.input.delta`, `custom_tool_call.input.done` |

- **Behavior:**
  - Reads `tool_routing[]` entries with matching `executor_type` values
  - Dispatches each tool call to the configured backend URL for that tool type
  - Formats results (e.g., citation formatting for `file_search`)
  - **`include` filtering:** reads the `include` array from CycleState to decide result detail level (e.g., only include `file_search_call.results` detail if `include` contains it; same for `web_search_call.results`, `code_interpreter_call.outputs`, `computer_call_output.output.image_url`)
  - Writes tool-specific events to `CycleState.event_queue`
  - When a backend is not deployed, returns a structured "unsupported tool" error as the tool result (the model sees the error and can adapt)
- **Configuration:** each tool type maps to a backend URL, timeout, and event type templates. Adding a new tool type is a configuration change, not a new plugin
- **Plugin Action:** Continue

### `host-tool-executor`

> **IPP Status: New** — no IPP equivalent. **Complexity: M**

Executes all Sandboxed tools (`local_shell`, `function_shell`, `apply_patch`) with a shared security gate. These tools perform host-level operations that are disabled by default and require explicit admin opt-in.

- **Boundary:** Sandboxed (RFC §5.7 prohibits raw host/filesystem access; must route to a sandboxed environment)
- **Default:** **disabled** — requires explicit admin opt-in via gateway configuration
- **Tool types:**

| Tool Type | Behavior | Events |
|-----------|----------|--------|
| `local_shell` | Shell command execution | `shell_call.command.delta/done`, `shell_call.in_progress` |
| `function_shell` | Function-style shell execution | (tool-specific) |
| `apply_patch` | File patch application | `apply_patch_call.operation_diff.delta/done` |

- **Behavior:**
  - Checks per-tool-type enable flag before execution (shared security gate)
  - Routes to the sandboxed execution environment
  - Writes tool-specific events to `CycleState.event_queue`
- **Plugin Action:** Continue (or Reject if the tool type is not enabled)

### `mcp-executor`

> **IPP Status: New** — no IPP equivalent. **Complexity: L**

Orchestrates calls to external MCP servers. Kept as a separate plugin because MCP has protocol-specific concerns that differ from the "call backend → return result" pattern of other tools.

- **Boundary:** External (calls remote MCP servers)
- **Why separate:** MCP requires session lifecycle management (connect → authenticate → invoke → reuse across calls within one request), an approval flow (`mcp_approval_request/response` input item types), and coupling with `tool-registry` for session reuse between discovery and execution
- **Behavior:**
  - Reuses the MCP session established during `tool-registry` discovery
  - Sends the tool call with arguments
  - Receives the tool result
  - Appends result to CycleState
- **MCP approval flow:** when an MCP server requires approval before tool execution:
  1. The server returns an `mcp_approval_request` output item (`{server_label, tool_name, arguments}`)
  2. The `mcp-executor` writes this to CycleState and the executor treats it as a yield — similar to function tool handoff. The response is delivered to the client with the approval request in its output
  3. The client reviews and resumes with a new `POST /v1/responses` containing `previous_response_id` and an `mcp_approval_response` input item (`{approved: bool, reason?}`)
  4. On resume, `hydrate-prompt` loads the previous response (which contains the pending MCP call). The `mcp-executor` checks the approval decision: if approved, executes the tool; if denied, writes a denial result to CycleState and the loop continues
  5. This uses `loop_action = stop_yield` — the same mechanism as function tool yield
- **Events:** `response.mcp_call.in_progress`, `response.mcp_call.completed`, `response.mcp_call.failed`, `response.mcp_call.arguments.delta`, `response.mcp_call.arguments.done`
- **Plugin Action:** Continue

---

## Phase 5: Completion

### `response-assembler`

> **IPP Status: New** — no IPP equivalent; Chat Completions passes provider response through directly. **Complexity: M**

Builds the final `ResponseResource` object with all spec-required fields.

- **Boundary:** Local
- **Input:** accumulated output items, usage statistics, validated request params, context edits — all from CycleState
- **Output:** complete `ResponseResource` with `"object": "response"`
- **Fields assembled:**
  - `id` — unique response identifier (generated)
  - `object` — `"response"` (constant)
  - `created_at` — timestamp from pipeline start (from CycleState)
  - `completed_at` — timestamp at assembly time (nullable if not completed)
  - `status` — `completed` | `incomplete` | `failed` (from `loop_action` in CycleState)
  - `incomplete_details` — reason structure if status is `incomplete` (from CycleState)
  - `error` — error object if `status = failed` (from CycleState)
  - `output` — array of output items (messages, tool calls, function calls, etc.)
  - `usage` — accumulated token usage across all iterations (`input_tokens`, `output_tokens`, `total_tokens`, `input_tokens_details.cached_tokens`, `output_tokens_details.reasoning_tokens`)
  - `context_edits` — array of `ContextEdit` records from truncation/compaction (from CycleState)
  - **Echo-back fields** (copied from validated request in CycleState): `model`, `instructions`, `metadata`, `input` (post-hydration), `tools`, `tool_choice`, `truncation`, `parallel_tool_calls`, `text`, `temperature`, `top_p`, `presence_penalty`, `frequency_penalty`, `top_logprobs`, `max_output_tokens`, `max_tool_calls`, `reasoning`, `user`, `store`, `background`, `service_tier`, `prompt_cache_key`, `prompt_cache_retention`, `safety_identifier`, `conversation`
  - `previous_response_id` — from request (if set)
  - `billing` — billing information (if applicable)
- **Emits:** `response.output_item.done`, `response.completed` | `response.incomplete` | `response.failed`

### `response-store`

> **IPP Status: New** — no IPP equivalent. **Complexity: S**

Persists the response to the response store when `store: true`.

- **Boundary:** External (writes to the configured store backend)
- **Input:** finalized `ResponseResource`
- **Output:** stored response (retrievable by ID)
- **Behavior:**
  - Writes the complete response to the configured store backend
  - The stored response is retrievable via `GET /v1/responses/{id}` and can be referenced by future requests via `previous_response_id`
- **Skipped:** when `store: false`

---

## Executor Responsibilities

The following concerns are **not plugins** — they are owned by the loop executor (the runtime that invokes plugins).

### Background Queueing

When CycleState `is_background = true`, the executor:
1. Stores an immediate `queued` response via `response-store`
2. Emits `response.queued` (spec-defined event; delivered in the response body since streaming is not active — `stream` + `background` are mutually exclusive)
3. Returns the `queued` response to the client
4. Enqueues the pipeline for async processing by a worker pool
5. The worker updates the stored response status to `in_progress` (emits `response.in_progress`), runs the full plugin pipeline, and updates the stored response on completion

This is a pipeline fork (RFC §5.5) — it cannot be expressed as a Plugin Action. The executor owns it.

Client polling (`GET /v1/responses/{id}`) is handled by the API routing layer, not the loop executor.

### Streaming Event Delivery

The executor owns the client transport (SSE or WebSocket). Plugins never write directly to the client connection (RFC §5.8).

**How it works:**
1. Plugins write events to `CycleState.event_queue` (a structured list of spec-defined event objects)
2. After each plugin hook returns, the executor flushes `event_queue` to the client transport
3. This happens synchronously between plugin invocations — no batching delay for streaming

**Transports:**

- **SSE** — primary transport, enabled by `stream: true`. Standard HTTP response with `Content-Type: text/event-stream`. Each event is a JSON object prefixed by `data: `. The connection stays open until the terminal event (`response.completed`, `response.incomplete`, `response.failed`, or `error`) is sent.

- **WebSocket** — alternative transport defined by the spec with additional constraints:
  - **Single in-flight response per connection** — only one `POST /v1/responses` can be processing at a time. The client must wait for completion before initiating another request on the same connection.
  - **Client-initiated via `response.create`** — the client sends a `response.create` message (JSON) over the WebSocket to initiate a request, rather than using HTTP POST. The message body matches the `CreateResponseBody` schema.
  - **60-minute connection limit** — the server closes WebSocket connections after 60 minutes. Long-running agentic loops that exceed this limit must be resumed via `previous_response_id` on a new connection.
  - **Reconnection via `previous_response_id`** — if a WebSocket connection drops mid-stream, the client reconnects and sends a new `response.create` with `previous_response_id` pointing to the interrupted response. The server resumes from the last persisted state.
  - **Same event types as SSE** — the executor flushes `CycleState.event_queue` identically regardless of transport. Transport selection is transparent to plugins.

The executor also emits lifecycle events directly (e.g., `response.created`, `response.in_progress`, `response.completed`) since these are executor-level state transitions, not plugin outputs.

Supports 51+ event types defined by the spec (see [streaming events summary](responses-api-agentic-loop.md#streaming-events-summary)).

### Loop Iteration

The executor owns the agentic loop. After each iteration:
1. Reads `loop_action` from CycleState (written by `loop-controller`)
2. If `continue`: re-invokes the pipeline from `api-translation` (request hook) through `tool-call-handler`, fans out to executor plugins based on `tool_routing` entries in CycleState
3. If `stop_*`: exits the loop and routes to completion plugins

The executor also handles tool execution fan-out: it reads `tool_routing[]` from CycleState and invokes the matching executor plugins (based on `executor_type`). If `parallel_dispatch = true`, it runs them concurrently. This keeps routing in the executor (where conditional execution belongs per RFC §3.8) rather than in plugins.

### Context Truncation

Truncation is **inline logic** within `hydrate-prompt` and `inference-caller`, not a separate plugin. These plugins read the `truncation` parameter from CycleState and apply it:
- `auto` — trim older context to fit the model's context window
- `disabled` — return Reject if context exceeds the window

Making truncation a separate plugin would require inter-plugin calls (RFC §5.2 violation) and running at two different pipeline positions (impossible in a linear pipeline).

### CycleState Lifetime

CycleState is created once per request and **persists for the entire agentic loop** (all iterations). It is destroyed when the request completes.

- **Cross-iteration state** (tool call counts, accumulated output tokens, tool results) accumulates in CycleState across iterations — this is required for the loop to function
- **Size growth:** the executor should monitor CycleState size and enforce limits. Tool results from N iterations can accumulate significant data. The `loop-controller`'s `max_tool_calls` limit implicitly bounds growth
- **Isolation:** each request gets its own CycleState. Nested pipeline calls (e.g., `inference-caller` triggering the IPP pipeline) get their own isolated CycleState

### Error Handling

Errors can occur at every layer — plugin hooks, model provider calls, tool execution, and the transport itself. Error handling is a cross-cutting concern shared between plugins and the executor.

**Plugin-level errors (Reject action):**
- Any plugin can return **Reject** at any hook point. The executor short-circuits the pipeline and returns an error response to the client.
- During intake (before the loop starts), a Reject terminates the request immediately.
- During an agentic loop iteration, Reject semantics depend on the plugin:
  - `guardrails` mid-loop rejection: the executor discards the violating output, writes a synthetic tool result to CycleState, and continues the loop (up to `max_guardrail_retries`, then `stop_failed`). See [Guardrails](#guardrails).
  - `inference-caller` rejection (truncation overflow with `disabled` mode): terminates the request.
  - Other plugin rejections mid-loop: terminate the request.

**Model provider errors (`inference-caller`):**
- HTTP 429 (rate limited), 500 (server error), timeouts, or malformed responses from the model provider are caught by `inference-caller`.
- On error, `inference-caller` writes a structured error to CycleState (`inference_error` with HTTP status, provider error body, and whether the error is retryable).
- `loop-controller` reads `inference_error` and signals `stop_failed`. The `response-assembler` includes the error in the `ResponseResource.error` field.
- Retry policy: `inference-caller` may retry transient errors (429, 503) with backoff up to a configurable `max_inference_retries` (default: 0 — no retries). Non-transient errors (400, 401, 403) are never retried.

**Tool execution errors:**
- Tool executor plugins (`server-tool-executor`, `host-tool-executor`, `mcp-executor`) do **not** Reject on tool failure. Instead, they write a structured error as the tool result in CycleState.
- The model receives the error as a tool result on the next iteration and can adapt (retry a different tool, report the error to the user, etc.).
- This follows the spec: tool execution failure is recoverable within the agentic loop.

**Streaming `error` event:**
- The spec defines an `error` event type (distinct from `response.failed`) for unrecoverable errors that occur mid-stream — e.g., a transport failure, an internal exception after streaming has started, or a background worker crash.
- The executor emits `error` events directly to the client transport when an unrecoverable error occurs after `response.in_progress` has been sent.
- If the error occurs before streaming starts, the executor returns a standard HTTP error response (no SSE event).
- After emitting an `error` event, the executor closes the transport. The `response-assembler` sets `status = "failed"` and the `response-store` persists the failed state.

**Background mode errors:**
- If the async worker encounters an error, it updates the stored response with `status = "failed"` and the error details. The client discovers the failure on the next poll (`GET /v1/responses/{id}`).
- No `error` event is emitted to the client (no active transport in background mode).

---

## Plugin Execution Flow

```
POST /v1/chat/completions  OR  POST /v1/responses
       │
       ▼
┌──────────────────────────────────────────────┐
│  GATEWAY INTAKE (both flows)                  │
│  auth (Authorino)                   [E] [IPP] │
│  rate-limit (Limitador)             [E] [IPP] │
│  request-validator                  [L] [EXT] │
└──────────────┬───────────────────────────────┘
               │
       ┌───────▼───────────────────────────────┐
       │  Responses API only:                   │
       │  hydrate-prompt                   [E]  │ ◄── if previous_response_id
       │    OR conversation-manager (load) [E]  │ ◄── if conversation
       │  tool-registry                    [E]  │ ◄── if tools
       └───────┬───────────────────────────────┘
               │
       ┌───────▼───────────────────────────────┐
       │  guardrails (on_request)  [E] [EXT]   │ ◄── if enabled (both flows)
       │  model-provider-resolver  [E] [IPP]   │
       └───────┬───────────────────────────────┘
               │
  ┌────────────▼──────────────────────────────────────────────────┐
  │ EXECUTOR: if background → queue & return queued response       │
  │           if streaming  → open SSE/WebSocket transport         │
  │           loop begins, emits lifecycle events                  │
  └────┬──────────────────────────────────────────────────────────┘
       │
       ▼  ◄═══════════════════════════════════════════════╗
┌──────────────────────────────────────┐                  ║
│  INFERENCE LOOP (both flows;         │                  ║
│  runs once for Chat Completions)     │                  ║
│                                      │                  ║
│  api-translation (request)  [L][EXT] │                  ║
│  body-field-to-header       [L][IPP] │                  ║
│  base-model-to-header       [E][IPP] │                  ║
│  apikey-injection           [E][IPP] │                  ║
│          ─── model call ───          │                  ║
│  api-translation (response) [L][EXT] │                  ║
│  guardrails (on_response)   [E][EXT] │ ◄── if enabled   ║
│  rate-limit (usage)         [E][IPP] │                  ║
│  tool-call-handler          [L][NEW] │ ◄── Responses    ║
│  loop-controller            [L][NEW] │                  ║
└───┬───┬──────────────────────────────┘                  ║
    │   │                                                 ║
 no │   │ server-side tools (Responses only)              ║
tools   ▼                                                 ║
    │  ┌─────────────────────────┐                        ║
    │  │  EXECUTOR: fan-out      │                        ║
    │  │  based on tool_routing  │                        ║
    │  │                         │                        ║
    │  │  server-tool-executor[E]│                        ║
    │  │  host-tool-executor  [S]│                        ║
    │  │  mcp-executor        [E]│                        ║
    │  └────────────┬────────────┘                        ║
    │               │ results → CycleState                ║
    │               ╚═════════════════════════════════════╝
    │
    │  (function tools: executor reads stop_yield,
    │   response-assembler includes function_call in output,
    │   client resumes with previous_response_id)
    ▼
┌──────────────────────────────────────┐
│  COMPLETION (Responses only)         │
│  response-assembler             [L]  │
│  response-store                 [E]  │
│  conversation-manager (save)    [E]  │ ◄── if conversation
└──────────────────────────────────────┘
       │
  ┌────▼──────────────────────────────────────────────────────────┐
  │ EXECUTOR: flushes terminal event, closes transport             │
  └───────────────────────────────────────────────────────────────┘

[L] = Local    [E] = External    [S] = Sandboxed
[IPP] = Exists in IPP    [EXT] = Extends IPP    [NEW] = New
```

---

## Flow Scenarios

This section maps the OGX Responses API flow presets (see [responses-flow.md](responses-flow.md)) to the unified plugin chain. Each scenario is adapted for **spec-compliant** behavior — where OGX's implementation diverges from the Open Responses spec, the unified pipeline follows the spec (see [responses-api-agentic-loop.md](responses-api-agentic-loop.md)).

> **Note:** All flows begin with the common **gateway intake** plugins (`auth` → `rate-limit` → `request-validator` → `model-provider-resolver`), then `api-translation` → `body-field-to-header` → `base-model-to-header` → `apikey-injection` run before each model call. These are omitted from the plugin chains and step-by-step tables below for brevity — only Responses-specific plugins are shown.

### Plugin Participation Matrix

Which plugins are active in each flow scenario. The "CC" column shows Chat Completions (single-pass). Plugins not listed are inactive (skipped via conditional execution) for that flow. IPP status and complexity are shown for quick reference.

| Plugin | IPP | Size | CC | 1. Simple | 2. RAG | 3. MCP | 4. BG | 5. Full | 6. Multi | 7. Func |
|--------|-----|------|----|:---------:|:------:|:------:|:-----:|:-------:|:--------:|:-------:|
| `auth` | IPP | S | x | x | x | x | x | x | x | x |
| `rate-limit` | IPP | S | x | x | x | x | x | x | x | x |
| `request-validator` | EXT | M | x | x | x | x | x | x | x | x |
| `hydrate-prompt` | NEW | L | | | | | | | x | x* |
| `conversation-manager` | NEW | M | | | x | | | x | | |
| `tool-registry` | NEW | M | | | x | x | x | x | | x |
| `guardrails` | EXT | M | x† | | | | | x | | |
| `model-provider-resolver` | IPP | S | x | x | x | x | x | x | x | x |
| `api-translation` | EXT | L | x | x | x | x | x | x | x | x |
| `body-field-to-header` | IPP | S | x | x | x | x | x | x | x | x |
| `base-model-to-header` | IPP | S | x | x | x | x | x | x | x | x |
| `apikey-injection` | IPP | S | x | x | x | x | x | x | x | x |
| `inference-caller` | NEW | M | x | x | x | x | x | x | x | x |
| `tool-call-handler` | NEW | S | | | x | x | x | x | | x |
| `loop-controller` | NEW | S | x‡ | x | x | x | x | x | x | x |
| `server-tool-executor` | NEW | L | | | x | | x | x | | |
| `host-tool-executor` | NEW | M | | | | | | | | |
| `mcp-executor` | NEW | L | | | | x | x | x | | |
| `response-assembler` | NEW | M | | x | x | x | x | x | x | x |
| `response-store` | NEW | S | | x | x | x | x | x | x | x |

\* `hydrate-prompt` is active in the **resume** leg of Flow 7 (when `previous_response_id` + `function_call_output` is sent).
† `guardrails` for Chat Completions runs when enabled (replaces IPP `request-guard` + `response-guard`).
‡ `loop-controller` for Chat Completions is trivial: always signals `stop_completed`.

**Notes:**
- `guardrails` has dual hooks: `on_request` (intake) and `on_response` (each loop iteration) — both active in Flow 5
- `api-translation` has dual hooks: `on_request` (before inference) and `on_response` (after inference) — both active in every flow
- `host-tool-executor` is disabled by default; only active when admin-enabled and a Sandboxed tool type is requested
- **7 plugins exist in IPP** (reused as-is or extended); **13 are new** for Responses API

**Executor-owned concerns** (not plugins — see [Executor Responsibilities](#executor-responsibilities)):
- **Background queueing** — active in Flow 4 (executor queues and returns immediately)
- **Streaming event delivery** — active in Flow 3, 5 (executor flushes `CycleState.event_queue` to SSE/WebSocket)
- **Context truncation** — inline within `hydrate-prompt` and `inference-caller` (all flows where context may exceed model window)
- **Loop iteration** — all flows (executor reads `loop_action` from CycleState)
- **Function tool yield** — active in Flow 5, 7 (executor reads `loop_action = stop_yield`, exits loop; `response-assembler` includes function call items in output)

---

### Flow 1: Simple Completion

The minimal path — a single inference call with no tools, no history, no streaming.

**Spec reference:** Phase 1 → Phase 2 → Phase 3 (no tool calls) → Phase 6.

**Toggles:** none

**Plugin chain:**
```
request-validator → api-translation (request) → inference-caller →
api-translation (response) → loop-controller (no tool calls → stop) →
response-assembler → response-store
```

**Step-by-step:**

| # | Actor | Action |
|---|-------|--------|
| 1 | `request-validator` | Validates `POST /v1/responses` body |
| 2 | *executor* | Emits `response.created`, `response.in_progress` (lifecycle events) |
| 3 | `api-translation` | `on_request`: translates Responses API input items to provider-native format |
| 4 | `inference-caller` | Calls model provider directly (credentials and path set by `apikey-injection` and `api-translation`) |
| 5 | `api-translation` | `on_response`: translates provider response back to Responses API output items |
| 6 | `loop-controller` | No tool calls → writes `loop_action = stop_completed` to CycleState |
| 7 | *executor* | Reads `loop_action`, exits loop |
| 8 | `response-assembler` | Builds `ResponseResource` |
| 9 | `response-store` | Persists response (if `store: true`) |
| 10 | *executor* | Emits `response.completed`, delivers response to client |

**Note:** `tool-call-handler` is skipped — no tools are registered (conditional execution based on CycleState `has_tools = false`).

**OGX divergence:** OGX routes through FastAPI → Responses impl → Orchestrator. The agentic loop collapses these into the plugin chain directly. The agentic loop also adds `api-translation` (request/response hooks) which OGX handles implicitly via its provider layer.

---

### Flow 2: RAG + Conversation

File search tool with conversation context. Loads conversation history, runs inference in a loop with file_search, persists conversation state.

**Spec reference:** Phase 1 (with context load) → Phase 2 → Phase 3 → Phase 4 (file_search) → loop → Phase 6.

**Toggles:** `file_search: true`, `conversation: true`

**Plugin chain:**
```
request-validator → conversation-manager (load) → tool-registry →
api-translation (request) → inference-caller → api-translation (response) →
tool-call-handler → loop-controller → server-tool-executor →
[loop back] → ... →
response-assembler → response-store → conversation-manager (save)
```

**Step-by-step:**

| # | Actor | Action |
|---|-------|--------|
| 1 | `request-validator` | Validates request, detects `conversation` param |
| 2 | `conversation-manager` | `on_request`: loads conversation items + stored messages into context |
| 3 | `tool-registry` | Registers `file_search` tool, resolves vector store IDs |
| 4 | *executor* | Emits `response.created`, `response.in_progress` |
| 5 | `api-translation` | `on_request`: translates hydrated context + tools to provider format |
| 6 | `inference-caller` | Calls model — model emits `file_search` tool call |
| 7 | `api-translation` | `on_response`: translates response to Responses API items |
| 8 | `tool-call-handler` | Detects `file_search` tool call, writes routing to CycleState (`executor_type: "file_search"`) |
| 9 | `loop-controller` | Server-side tools → writes `loop_action = continue` |
| 10 | *executor* | Reads `tool_routing`, invokes `server-tool-executor` |
| 11 | `server-tool-executor` | Dispatches `file_search` → vector store, returns ranked chunks + citations |
| 12 | *executor* | Appends tool results to CycleState, re-enters loop |
| 13 | `api-translation` | `on_request`: translates updated context for next iteration |
| 14 | `inference-caller` | Calls model with file_search results — model produces final answer |
| 15 | `api-translation` | `on_response`: translates final response |
| 16 | `loop-controller` | No tool calls → writes `loop_action = stop_completed` |
| 17 | *executor* | Exits loop |
| 18 | `response-assembler` | Builds `ResponseResource` with output + citations |
| 19 | `response-store` | Persists response |
| 20 | `conversation-manager` | `on_response`: saves input + output items to conversation, persists messages |
| 21 | *executor* | Delivers response to client |

**OGX divergence:** OGX uses dual storage (Conversations API for structured items + Store for raw chat messages). The agentic loop introduces `conversation-manager` to handle both reads and writes. OGX's VectorIO actor maps to the `file_search` dispatch in `server-tool-executor`.

**Note:** `conversation-manager` is defined in [Phase 1: Request Intake](#conversation-manager).

---

### Flow 3: MCP Agent Loop

Streaming agentic flow with MCP tool discovery and execution. The model calls MCP tools iteratively until it produces a final answer.

**Spec reference:** Phase 1 → MCP discovery → Phase 2 → Phase 3 → Phase 4 (MCP) → loop → Phase 6.

**Toggles:** `mcp: true`, `stream: true`

**Plugin chain:**
```
request-validator → tool-registry (MCP discovery) →
api-translation (request) → inference-caller → api-translation (response) →
tool-call-handler → loop-controller → mcp-executor →
[loop back] → ... →
response-assembler → response-store
```

**Step-by-step:**

| # | Actor | Action |
|---|-------|--------|
| 1 | `request-validator` | Validates request with `stream: true` |
| 2 | `tool-registry` | Connects to MCP server, discovers tools (`list_mcp_tools`), caches definitions; writes `response.mcp_list_tools.*` events to `event_queue` |
| 3 | *executor* | Flushes `event_queue` → SSE (`response.mcp_list_tools.in_progress/completed`) |
| 4 | *executor* | Emits `response.created`, `response.in_progress` via SSE |
| 5 | `api-translation` | `on_request`: translates to provider format including MCP tool definitions |
| 6 | `inference-caller` | Calls model — model emits MCP tool call |
| 7 | `api-translation` | `on_response`: translates response, writes `response.output_text.delta` events to `event_queue` |
| 8 | *executor* | Flushes `event_queue` → SSE (streaming text deltas) |
| 9 | `tool-call-handler` | Detects MCP tool call, writes routing to CycleState (`executor_type: "mcp"`) |
| 10 | `loop-controller` | Server-side tools present → writes `loop_action = continue` |
| 11 | *executor* | Reads `tool_routing`, invokes `mcp-executor` |
| 12 | `mcp-executor` | Invokes MCP tool on remote server, writes result + `response.mcp_call.*` events to `event_queue` |
| 13 | *executor* | Flushes `event_queue` → SSE, appends tool results to CycleState, re-enters loop |
| 14 | ... | (repeat steps 5–13 until no tool calls) |
| 15 | `loop-controller` | No tool calls → writes `loop_action = stop_completed` |
| 16 | *executor* | Exits loop |
| 17 | `response-assembler` | Builds final `ResponseResource` |
| 18 | `response-store` | Persists response |
| 19 | *executor* | Emits `response.completed` via SSE, closes transport |

**OGX divergence:** OGX caches MCP sessions via `MCPSessionManager` per request. The agentic loop's `tool-registry` handles discovery and `mcp-executor` reuses sessions within the loop — same semantics, different actor decomposition.

---

### Flow 4: Background Processing

Request is queued and processed asynchronously. Client receives immediate `queued` response, polls for completion. Combines file_search and MCP tools.

**Spec reference:** Phase 1 (`background: true`, emits `response.queued`) → async processing → Phase 2–6 in background → client polls.

**Toggles:** `background: true`, `file_search: true`, `mcp: true`

**Plugin chain:**
```
request-validator → tool-registry →
  ┌── EXECUTOR: stores queued response, returns immediately to client
  └── async worker: api-translation (request) → inference-caller →
      api-translation (response) → tool-call-handler → loop-controller →
      server-tool-executor / mcp-executor →
      [loop] → response-assembler → response-store
```

**Step-by-step:**

| # | Actor | Action |
|---|-------|--------|
| 1 | `request-validator` | Validates request with `background: true` |
| 2 | `tool-registry` | Resolves file_search + MCP tool definitions |
| 3 | *executor* | Detects `is_background = true` in CycleState; stores `queued` response via `response-store`, returns it to client immediately, enqueues pipeline for async processing |
| — | | *(async worker picks up)* |
| 4 | *executor* | Emits `response.in_progress` (status transition, no client transport — stored for polling) |
| 5 | `api-translation` | `on_request`: translates context to provider format |
| 6 | `inference-caller` | Calls model |
| 7 | `api-translation` | `on_response`: translates response |
| 8 | `tool-call-handler` | Detects tool calls, writes routing decisions to CycleState |
| 9 | `loop-controller` | Server-side tools → writes `loop_action = continue` |
| 10 | *executor* | Reads `tool_routing`, invokes `server-tool-executor` / `mcp-executor` |
| 11 | `server-tool-executor` | Dispatches `file_search` → vector store |
| 12 | `mcp-executor` | Invokes MCP tool |
| 13 | *executor* | Appends tool results to CycleState, re-enters loop |
| 14 | ... | (repeat steps 5–13 until no tool calls) |
| 15 | `loop-controller` | No tool calls → writes `loop_action = stop_completed` |
| 16 | `response-assembler` | Builds final `ResponseResource` |
| 17 | `response-store` | Updates stored response with completed result |

**Client polling** (`GET /v1/responses/{id}`) is handled by the API routing layer, not the loop executor.

**OGX divergence:** OGX uses a 10-worker `BGWorker` pool. The agentic loop executor's async backend is implementation-agnostic — the worker pool is pluggable.

---

### Flow 5: Full Advanced

All features enabled — streaming, conversation, file_search, MCP, function tools, and guardrails.

**Spec reference:** All phases active. Guardrails are not spec-defined (extension, mirrors OGX).

**Toggles:** `stream: true`, `conversation: true`, `file_search: true`, `mcp: true`, `function_tools: true`, `guardrails: true`

**Plugin chain:**
```
request-validator → conversation-manager (load) →
tool-registry (MCP discovery) → guardrails (on_request) →
api-translation (request) → inference-caller → api-translation (response) →
guardrails (on_response) → tool-call-handler → loop-controller →
  ├── server-tool-executor → [loop]
  ├── mcp-executor → [loop]
  └── (function tools: executor yields via stop_yield)
→ response-assembler → response-store → conversation-manager (save)
```

**Step-by-step:**

| # | Actor | Action |
|---|-------|--------|
| 1 | `request-validator` | Validates full request |
| 2 | `conversation-manager` | `on_request`: loads conversation items + stored messages |
| 3 | `tool-registry` | Registers file_search, MCP (with discovery), function tools |
| 4 | `guardrails` | `on_request`: evaluates input against guardrail policies |
| 5 | *executor* | Emits `response.created`, `response.in_progress` via SSE |
| 6 | `api-translation` | `on_request`: translates to provider format |
| 7 | `inference-caller` | Calls model |
| 8 | `api-translation` | `on_response`: translates response, writes streaming events to `event_queue` |
| 9 | *executor* | Flushes `event_queue` → SSE (text deltas, reasoning, etc.) |
| 10 | `guardrails` | `on_response`: evaluates model output against guardrail policies |
| 11 | `tool-call-handler` | Detects tool calls (file_search + MCP + function), writes routing: `has_server_side_tools = true`, `has_function_tools = true` |
| 12 | `loop-controller` | Mixed tools with function calls → writes `loop_action = continue` (execute server-side tools before yielding) |
| 13 | *executor* | Reads `tool_routing`, invokes server-side executors |
| 14 | `server-tool-executor` | Dispatches `file_search` → vector store |
| 15 | `mcp-executor` | Invokes MCP tool |
| 16 | *executor* | Appends server-side results to CycleState as output items |
| 17 | `loop-controller` | Server-side tools done, function tools remain → writes `loop_action = stop_yield` |
| 18 | *executor* | Reads `stop_yield`, exits loop |
| 19 | `response-assembler` | Builds `ResponseResource` with server-side tool results **and** function_call items in output (client sees both simultaneously) |
| 20 | `response-store` | Persists response |
| 21 | `conversation-manager` | `on_response`: saves input + output to conversation |
| 22 | *executor* | Emits `response.completed` via SSE, closes transport |

**OGX divergence:** OGX checks guardrails via its `Moderation` actor with `guardrail_ids`. The agentic loop uses the `guardrails` plugin (dual-hook, implementation-neutral). OGX cannot combine `stream` + `background` — the agentic loop follows this same constraint.

---

### Flow 6: Multi-Turn Continuation

Client resumes a prior response using `previous_response_id`. This is the spec's primary multi-turn mechanism — no tools, just conversational continuity.

**Spec reference:** Phase 1 (context reconstruction from `previous_response_id`) → Phase 2 → Phase 6.

**Toggles:** `previous_response_id: set`

**Plugin chain:**
```
request-validator → hydrate-prompt (includes inline truncation) →
api-translation (request) → inference-caller → api-translation (response) →
loop-controller (no tools → stop) → response-assembler → response-store
```

**Step-by-step:**

| # | Actor | Action |
|---|-------|--------|
| 1 | `request-validator` | Validates request, detects `previous_response_id` |
| 2 | `hydrate-prompt` | Loads previous response from store, reconstructs context; applies inline truncation if `truncation = "auto"` |
| 3 | *executor* | Emits `response.created`, `response.in_progress` |
| 4 | `api-translation` | `on_request`: translates hydrated context to provider format |
| 5 | `inference-caller` | Calls model with full conversation history (applies inline truncation before call) |
| 6 | `api-translation` | `on_response`: translates response |
| 7 | `loop-controller` | No tool calls → writes `loop_action = stop_completed` |
| 8 | *executor* | Exits loop |
| 9 | `response-assembler` | Builds `ResponseResource` |
| 10 | `response-store` | Persists response (new `previous_response_id` chain link) |
| 11 | *executor* | Emits `response.completed`, delivers response to client |

**Note:** Context truncation is handled **inline** by `hydrate-prompt` (during context reconstruction) and `inference-caller` (before each model call) — there is no separate `truncation-handler` plugin. See [Context Truncation](#context-truncation).

**OGX divergence:** OGX loads from its Store (SQL). The agentic loop's `hydrate-prompt` is store-agnostic. Note that `previous_response_id` and `conversation` are **mutually exclusive** per the spec — if `conversation` is set, `conversation-manager` is used instead of `hydrate-prompt`.

---

### Flow 7: Function Tool Round-Trip

Client-side function tools — the model emits a `function_call`, control is yielded to the client, who executes the tool and resumes with `function_call_output`.

**Spec reference:** Phase 1 → Phase 2 → Phase 3 → Phase 5 (yield) → client executes → new Phase 1 with `previous_response_id` + `function_call_output`.

**Toggles:** `function_tools: true`

**Plugin chain (first request):**
```
request-validator → tool-registry →
api-translation (request) → inference-caller → api-translation (response) →
tool-call-handler → loop-controller (stop_yield) →
response-assembler → response-store
```

**Plugin chain (resume request):**
```
request-validator → hydrate-prompt →
api-translation (request) → inference-caller → api-translation (response) →
loop-controller (no tools → stop) →
response-assembler → response-store
```

**Step-by-step (first request):**

| # | Actor | Action |
|---|-------|--------|
| 1 | `request-validator` | Validates request with function tool definitions |
| 2 | `tool-registry` | Registers function tools (externally-hosted) |
| 3 | *executor* | Emits `response.created`, `response.in_progress` |
| 4 | `api-translation` | `on_request`: translates to provider format with function tool schema |
| 5 | `inference-caller` | Calls model — model emits `function_call` |
| 6 | `api-translation` | `on_response`: translates response |
| 7 | `tool-call-handler` | Detects `function_call` item, writes `has_function_tools = true`, `has_server_side_tools = false`; emits `response.function_call_arguments.delta/done` to `event_queue` |
| 8 | `loop-controller` | Only function tools → writes `loop_action = stop_yield` |
| 9 | *executor* | Reads `stop_yield`, exits loop |
| 10 | `response-assembler` | Builds response with `function_call` items in output |
| 11 | `response-store` | Persists response |
| 12 | *executor* | Emits `response.completed`, delivers response to client |

**Step-by-step (resume):**

| # | Actor | Action |
|---|-------|--------|
| 1 | `request-validator` | Validates resume request with `function_call_output` |
| 2 | `hydrate-prompt` | Loads previous response, appends `function_call_output` to context; applies inline truncation if needed |
| 3 | *executor* | Emits `response.created`, `response.in_progress` |
| 4 | `api-translation` | `on_request`: translates full context including tool result |
| 5 | `inference-caller` | Calls model with function result — model produces final answer |
| 6 | `api-translation` | `on_response`: translates response |
| 7 | `loop-controller` | No tool calls → writes `loop_action = stop_completed` |
| 8 | *executor* | Exits loop |
| 9 | `response-assembler` | Builds final `ResponseResource` |
| 10 | `response-store` | Persists response (chain: resp_1 → resp_2) |
| 11 | *executor* | Emits `response.completed`, delivers response to client |

---

## Open Gaps

Features that require external services, API endpoints, or design decisions beyond the plugin layer.

### Tool Backend Services

**Severity:** Medium (web search) / Low (others)

The `server-tool-executor` dispatch table includes entries for spec-defined tools that have no backend services deployed:

| Tool Type | Backend Required | Status |
|-----------|-----------------|--------|
| `web_search` | Web search API (e.g., Tavily, Brave, SerpAPI) | No backend |
| `code_interpreter` | Sandboxed code execution environment | No backend |
| `image_gen` | Image generation service | No backend |
| `computer` | Remote desktop/screen environment | No backend |

When a backend is not deployed, `server-tool-executor` returns a structured "unsupported tool" error. Adding backend support is a configuration change (backend URL + timeout), not a plugin change.

---

### Response Retrieval API

**Severity:** Medium — required for `background` mode and `previous_response_id` to work

The API routing layer (outside the agentic loop) must implement:
- `GET /v1/responses/{id}` — retrieve a response (background polling, client retrieval)
- `GET /v1/responses/{id}/input_items` — retrieve input items
- `DELETE /v1/responses/{id}` — delete a stored response

These are REST endpoints that read from `response-store`, not plugins.

---

### Compaction API (`POST /v1/responses/compact`)

**Severity:** High — blocks lossless context management for long conversations

The spec defines a standalone endpoint that compresses conversation context into encrypted `compaction_summary` items. The plugin layer accepts `compaction_summary` items as input (handled by `hydrate-prompt`), but nothing produces them.

**What's missing:**
- The `/v1/responses/compact` API endpoint (API routing layer)
- A compaction service that produces encrypted compaction summaries

---

### `allowed_tools` Cache-Preserving Filtering

**Severity:** Low — optimization feature

The spec defines `allowed_tools` to restrict tool invocation without changing the `tools` list (for prompt cache preservation). `tool-registry` mentions this but the behavior is not fully specified.

**Recommendation:**
1. `tool-registry` registers all tools from `tools` (cache key stability), applies `allowed_tools` as a filter
2. `tool-call-handler` validates tool calls against `allowed_tools` and rejects unauthorized calls

---

### `service_tier` Consumption

**Severity:** Medium — validated and echoed back but not consumed

The `request-validator` validates `service_tier` and `response-assembler` echoes it. But no plugin or executor uses it to affect behavior (queue priority, provider selection). Defer until the gateway's service tier model is defined.

---

### Streaming + Background Mutual Exclusion

**Severity:** Low — design constraint

OGX enforces that `stream` and `background` cannot be combined (400 error). The Open Responses spec does not explicitly forbid this. The `request-validator` enforces this constraint; it can be relaxed if the spec evolves.

---

### Summary

| Gap | Severity | Resolution |
|-----|----------|------------|
| Tool backend services | Medium / Low | Config-driven; returns "unsupported" until backend deployed |
| Response retrieval API | Medium | API routing layer, not a plugin |
| Compaction API | High | API routing + external compaction service |
| `allowed_tools` filtering | Low | Extend `tool-registry` and `tool-call-handler` |
| `service_tier` consumption | Medium | Defer until gateway service tier model defined |
| Stream + background exclusion | Low | Enforced in validator; relaxable later |

---

## IPP Plugin Catalog (Reference)

The Inference Payload Processor (IPP) is the existing `ext_proc` filter implementation. The plugins listed below are the baseline — they are reused as-is or extended in the unified pipeline. This section is a reference for the existing implementations; see [Plugin Overview](#plugin-overview) for the unified 21-plugin inventory.

The IPP processes each request in a single pass: request plugins run forward, the request is forwarded to the model provider, response plugins run in reverse.

### Infrastructure Plugins

These plugins handle cross-cutting concerns (auth, rate-limiting) and are not part of the core IPP plugin chain but run within the same `ext_proc` process.

| Plugin | Boundary | Hooks | Description |
|--------|----------|-------|-------------|
| `auth (Authorino)` | External | `on_request` | Validates the MaaS API key by calling `MaaS API /internal/v1/api-keys/validate`. Injects `X-MaaS-Username` and `X-MaaS-Subscription` headers. Rejects with 401 on failure. |
| `rate-limit (Limitador)` | External | `on_request`, `on_response` | Token-based rate limiting via `TokenRateLimitPolicy`. On request: checks if token quota allows. On response: reads `usage.total_tokens` and reports consumption. Rejects with 429. |

### Request Plugins

Run forward (in pipeline order) before the request is forwarded to the model provider.

| Plugin | Boundary | Description |
|--------|----------|-------------|
| `request-guard (nemo)` | External | NVIDIA NeMo Guardrails — evaluates safety policies on the incoming request. Can block requests that violate content policies. **Coming soon.** |
| `intelligent-model-selection` | Local | Automatically selects the optimal model or provider based on request characteristics, cost, latency, and availability. **Coming soon.** |
| `model-provider-resolver` | External | Uses a Kubernetes reconciler to watch `ExternalModel` CRDs. Resolves the model name to a provider, target model ID, and endpoint URL. Writes `provider`, `targetModel`, `endpoint` to CycleState. |
| `body-field-to-header` | Local | Extracts the `model` field from the JSON request body and sets it as the `X-Gateway-Model-Name` HTTP header for model-based routing. |
| `base-model-to-header` | External | Resolves LoRA adapter names to their base model using Kubernetes ConfigMaps. Sets `X-Gateway-Base-Model-Name` header so EPP can select pods running the correct base model. |
| `api-translation (request)` | Local | Translates OpenAI Chat Completions format to provider-native API. Handles path rewriting (`/v1/chat/completions` → `/v1/messages`), header mutations (`anthropic-version`), and body transformation. |
| `apikey-injection` | External | Reads the provider API key from a Kubernetes Secret (labeled `inference.networking.k8s.io/bbr-managed`) and injects the appropriate auth header (`Authorization: Bearer` for OpenAI, `x-api-key` for Anthropic, `api-key` for Azure). |

### Response Plugins

Run in reverse (reverse pipeline order) after the model provider responds.

| Plugin | Boundary | Description |
|--------|----------|-------------|
| `api-translation (response)` | Local | Translates provider-native response back to OpenAI Chat Completions format. Normalizes `content` → `choices`, `stop_reason` → `finish_reason`, and usage tokens. |
| `response-guard (nemo)` | External | NVIDIA NeMo Guardrails — evaluates content policies on the provider response after `api-translation` converts it to OpenAI format. **Coming soon.** |

> **IPP plugin count: 11** (2 infrastructure + 7 request + 2 response). Of these, `api-translation` and `rate-limit` are dual-hook plugins (request + response) implemented as single plugin instances.

---

## Plugin Overlap Analysis

The unified pipeline merges the existing IPP plugins (11) with the agentic loop plugins (14). This section documents the overlaps that drove the unification and the problems it solves.

### Overlap Matrix

| Concern | IPP Plugin(s) | Unified Plugin | Resolution |
|---------|--------------|----------------|------------|
| **Guardrails** | `request-guard (nemo)` + `response-guard (nemo)` (2 plugins) | `guardrails` (1 dual-hook plugin) | **3 → 1.** Same NeMo endpoint, same evaluation logic. IPP split this into two plugins; the unified model uses a single dual-hook plugin for both flows. |
| **API translation** | `api-translation` (request + response) | `api-translation` (format-aware, reads `request_format`) | **2 → 1.** Provider-specific logic (Anthropic, Bedrock, Vertex, Azure) is shared. The unified plugin branches on input format, not on pipeline identity. |
| **Request validation** | implicit Chat Completions validation | `request-validator` (format-aware) | **2 → 1.** Single plugin validates both Chat Completions and Responses API bodies, writes `request_format` to CycleState. |
| **Credential injection** | `apikey-injection` | `apikey-injection` (reused as-is) | **1 → 1.** Runs once in the unified pipeline before `inference-caller`. No longer duplicated via nested IPP calls. |
| **Model resolution** | `model-provider-resolver` | `model-provider-resolver` (reused as-is) | **1 → 1.** Same. |
| **Rate limiting** | `rate-limit (Limitador)` | `rate-limit` (reused, CycleState accounting) | **1 → 1.** Runs once at intake + per-iteration usage via CycleState. No longer re-entered via nested calls. |
| **Auth** | `auth (Authorino)` | `auth` (reused as-is) | **1 → 1.** Runs once at intake. No longer re-validated per inference iteration. |

> **Net reduction: 25 → 20 plugins** (4 eliminated by merging: `request-guard`, `response-guard`, duplicate `api-translation`, duplicate `request-validator`; 1 deferred: `intelligent-model-selection`, see [Open Points](#open-points)).

### The Nested Pipeline Problem (Solved)

Before unification, a Responses API request triggered **two nested plugin pipelines** per inference iteration:

```
Client → Gateway → [Agentic Loop Pipeline] → inference-caller
                                                    │
                                                    ▼
                                              Gateway → [IPP Pipeline]
                                                    │
                                                    ▼
                                              Model Provider
```

This caused four concrete problems, all resolved by the unified pipeline:

1. **Auth ran N+1 times** — once on the initial request, then once per inference iteration. The unified pipeline runs auth once at intake.
2. **Rate-limit ran N+1 times** — redundant quota checks that could race. The unified pipeline checks once at intake and reports per-iteration usage via CycleState.
3. **API translation ran twice per iteration** — `Responses API items → Chat Completions messages → Anthropic messages`. The unified `api-translation` plugin goes directly from input format to provider-native, eliminating the intermediate hop.
4. **Guardrails ran in both layers** — same NeMo endpoint called from both IPP and the agentic loop. The unified `guardrails` plugin handles both in a single dual-hook instance.

---

## Unified Pipeline Architecture

One request, one pipeline. The pipeline uses **conditional execution** ([AI Plugin RFC](rfc-ai-plugin-concept.md) §3.8) to gate Responses-only plugins based on `request_format` in CycleState. Chat Completions requests skip all agentic plugins with zero runtime cost.

### Pipeline Flow

```
POST /v1/chat/completions  OR  POST /v1/responses
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│  GATEWAY INTAKE (runs once per request)                          │
│                                                                  │
│  auth (Authorino)                              [E] [IPP]    S    │
│  rate-limit (Limitador)                        [E] [IPP]    S    │
│  request-validator                             [L] [EXT]    M    │
│  hydrate-prompt                                [E] [NEW]    L    │ ◄─ Responses, if previous_response_id
│  conversation-manager (load)                   [E] [NEW]    M    │ ◄─ Responses, if conversation
│  tool-registry                                 [E] [NEW]    M    │ ◄─ Responses, if tools
│  guardrails (input)                            [E] [EXT]    M    │ ◄─ if enabled
│  model-provider-resolver                       [E] [IPP]    S    │
└──────────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│  INFERENCE (once for Chat Completions, loops for Responses)      │
│                                                                  │
│  api-translation (request)                     [L] [EXT]    L    │
│  body-field-to-header                          [L] [IPP]    S    │
│  base-model-to-header                          [E] [IPP]    S    │
│  apikey-injection                              [E] [IPP]    S    │
│                    ─── model call ───                             │
│  api-translation (response)                    [L] [EXT]    L    │
│  guardrails (output)                           [E] [EXT]    M    │ ◄─ if enabled
│  rate-limit (usage)                            [E] [IPP]    S    │
│                                                                  │
│  ┌─ Responses API only ───────────────────────────────────────┐  │
│  │  tool-call-handler                          [L] [NEW]  S  │  │
│  │  loop-controller                            [L] [NEW]  S  │  │
│  │  server-tool-executor                       [E] [NEW]  L  │  │
│  │  host-tool-executor                         [S] [NEW]  M  │  │
│  │  mcp-executor                               [E] [NEW]  L  │  │
│  │  ════ loop back to api-translation (request) ═════════    │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│  COMPLETION (Responses API only)                                 │
│                                                                  │
│  response-assembler                            [L] [NEW]    M    │
│  response-store                                [E] [NEW]    S    │
│  conversation-manager (save)                   [E] [NEW]    M    │ ◄─ if conversation
└──────────────────────────────────────────────────────────────────┘

[L] = Local   [E] = External   [S] = Sandboxed
[IPP] = Exists in IPP   [EXT] = Extends IPP   [NEW] = New plugin
S/M/L = Complexity       Undef = Scope undefined
```

### Architectural Notes

**CycleState is unified.** One CycleState per request, shared across all plugins and loop iterations. Per-iteration state (tool results, token counts) accumulates in the same CycleState. Plugins namespace their keys by convention (RFC §5.3).

**Loop re-entry point.** The agentic loop re-enters at `api-translation (request)` after tool execution. Plugins before `api-translation` (auth, rate-limit, model-resolver, apikey-injection) run once at intake, not per iteration. Credentials and routing don't change between iterations.

**`inference-caller` makes direct model calls.** Credential injection and path rewriting happen in the same pipeline, before `inference-caller`. No re-entry through the gateway. Error propagation: if the model provider returns an error, `inference-caller` writes it to CycleState, and `loop-controller` evaluates it as `stop_failed`.

**Rate-limit per-iteration accounting.** `rate-limit` runs once at intake (quota check). Per-iteration, `inference-caller` writes `iteration_usage` to CycleState and `rate-limit` reads it during the response phase. `loop-controller` accumulates totals. This is more accurate than N independent quota checks that could race.
