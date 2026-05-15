# RFC: AI Plugin Concept for AI Gateways

**Status:** Draft
**Date:** 2026-05-14
**Authors:** AI Gateway Architecture Team

**Source Projects:**

- **Praxis** — [github.com/praxis-proxy/praxis](https://github.com/praxis-proxy/praxis/) (Rust proxy framework; see [filters](https://github.com/praxis-proxy/praxis/blob/main/docs/filters.md) and [extensions](https://github.com/praxis-proxy/praxis/blob/main/docs/extensions.md))
- **IPP (llm-d)** — [github.com/llm-d/llm-d-inference-payload-processor](https://github.com/llm-d/llm-d-inference-payload-processor) (Go-based Inference Payload Processor)
- **IPP (AI Gateway)** — [github.com/opendatahub-io/ai-gateway-payload-processing](https://github.com/opendatahub-io/ai-gateway-payload-processing/) (AI Gateway fork with additional plugins)

---

## 1. Motivation

AI gateways face two traffic patterns that both need extensible processing:

1. **Single-pass inference proxying** (`/v1/chat/completions`) -- a request enters, gets mutated (auth, translation, guardrails), is forwarded to a model provider, the response is mutated back, and it exits. One request in, one response out.

2. **Agentic orchestration** (`/v1/responses`) -- a request enters, the gateway runs an iterative loop: it samples from a model, detects tool calls, executes server-side tools, feeds results back, and repeats until the model produces a final answer or a limit is hit. One request in, N internal iterations, one (possibly streamed) response out.

Both patterns need a composable, plugin-based architecture. Two existing designs inform this space:

- **[IPP plugins](https://github.com/llm-d/llm-d-inference-payload-processor)** (Gateway API Inference Extension) -- Go-based `RequestProcessor`/`ResponseProcessor` interfaces with per-request `CycleState`, factory-based instantiation, and Kubernetes-aware handles. Plugins operate on a parsed `InferenceRequest`/`InferenceResponse` (headers + JSON body). They run in a fixed pipeline order. Existing plugins: model-provider-resolver, api-translation, apikey-injection, nemo-guards. The [AI Gateway fork](https://github.com/opendatahub-io/ai-gateway-payload-processing/) extends this with additional plugins (auth, rate-limit, guardrails).

- **[Praxis filters](https://github.com/praxis-proxy/praxis/)** -- Rust-based `HttpFilter`/`TcpFilter` traits with a [pipeline executor](https://github.com/praxis-proxy/praxis/blob/main/docs/filters.md). Filters return `FilterAction` (Continue, Reject, Release, BodyDone) and operate on `HttpFilterContext` (raw HTTP request/response, headers, routing state). Filters can declare conditional execution rules. Pipelines execute forward on request, reverse on response. Praxis also supports [extensions](https://github.com/praxis-proxy/praxis/blob/main/docs/extensions.md) for higher-level composable logic.

This RFC defines the **AI Plugin** concept: a unified abstraction that captures what a plugin is, what it can do, and what it cannot do, applicable to both traffic patterns. The target implementation is **Rust on Praxis** (as filters or extensions). The existing IPP plugins are implemented in Go and will require a rewrite; the transition plan is outside the scope of this document.

---

## 2. Definitions

| Term | Meaning |
|------|---------|
| **AI Plugin** | A composable processing unit that participates in request and/or response processing of AI traffic |
| **Pipeline** | An ordered sequence of plugins executed for a given request lifecycle |
| **CycleState** | Per-request shared state that plugins use to communicate without mutating the request itself |
| **Plugin Action** | The control-flow signal a plugin returns after executing (continue or reject) |
| **Executor** | The runtime that owns the pipeline, invokes plugins, manages the agentic loop, and handles transport-level concerns |

---

## 3. What an AI Plugin IS

An AI plugin is a **named, typed, composable processing unit** with the following properties:

### 3.1 Identity

Every plugin has:

- A **type** -- a fixed identifier for the plugin kind (e.g., `api-translation`, `guardrails`, `inference-caller`). Types are unique within a registry.
- A **name** -- an instance identifier, allowing multiple instances of the same type with different configurations (e.g., two `guardrails` instances with different policies).
- A **boundary classification**:
  - **Local** -- pure in-process computation, no network I/O
  - **External** -- requires calls to external services (model providers, guardrails endpoints, tool backends, Kubernetes APIs)
  - **Sandboxed** -- requires host-level operations (shell execution, filesystem access) that are disabled by default and require explicit admin opt-in

### 3.2 Lifecycle

A plugin has three lifecycle stages:

1. **Creation** -- A factory function receives the plugin name, a configuration blob, and a handle to the runtime environment. It returns an initialized plugin instance or an error. Plugins are created once at startup or configuration reload.
2. **Execution** -- The plugin's hooks are invoked per-request by the pipeline executor. A plugin must be safe for concurrent execution across requests.
3. **Teardown** -- The runtime may signal a plugin to release resources (connections, watchers, caches). Plugins that hold external state (e.g., Kubernetes informers, MCP sessions) must implement graceful shutdown.

### 3.3 Hooks

A plugin declares which hooks it participates in:

| Hook | Direction | When |
|------|-----------|------|
| `on_request` | Forward (pipeline order) | Before the request is forwarded upstream or enters the agentic loop |
| `on_response` | Forward (pipeline order) | After the upstream response is received or the agentic loop produces output |

A plugin may implement one or both hooks. Dual-hook plugins (e.g., `guardrails` with input checking on request and output checking on response, or `api-translation` with format conversion in both directions) use a single plugin instance for both hooks, sharing configuration and connection state.

In the agentic loop, the executor re-enters the pipeline at a configured re-entry point (typically `api-translation`) for each iteration. Plugins before the re-entry point (auth, rate-limit, model-resolver) run once at intake, not per-iteration. The loop re-entry is an executor concern, not a plugin hook.

### 3.4 Plugin Actions

Every hook returns a **Plugin Action** that controls pipeline execution:

| Action | Meaning |
|--------|---------|
| `Continue` | Pass to the next plugin in the pipeline |
| `Reject(status, headers?, body?)` | Short-circuit the pipeline; return an error response to the client immediately |

`Reject` is the primary safety mechanism. Any plugin can reject at any hook. A guardrails plugin that detects a policy violation rejects. A rate-limit plugin that exceeds a threshold rejects. An auth plugin that fails validation rejects.

Control-flow decisions beyond continue/reject -- such as yielding control to the client for function tool handoff, or signaling loop termination -- are expressed through **CycleState values** that the executor reads. Plugins signal intent; the executor acts on it. This keeps the Plugin Action contract simple and avoids conflating plugin-level control flow with executor-level orchestration.

### 3.5 Data Access

Plugins access request and response data through a **context object** passed to each hook. The context provides:

| Access | Scope | Mutability |
|--------|-------|------------|
| Request headers | Per-request | Read/Write (add, set, remove) |
| Request body (parsed) | Per-request | Read/Write (field-level or whole-body replacement) |
| Response headers | Per-request (response hook only) | Read/Write |
| Response body (parsed) | Per-request (response hook only) | Read/Write |
| CycleState | Per-request, shared across all plugins | Read/Write (typed key-value store) |
| Plugin configuration | Per-plugin-instance, immutable after creation | Read-only |
| Runtime handle | Per-plugin-instance | Read-only (access to k8s client, reconcilers, secrets, etc.) |

AI gateway traffic is JSON. Bodies are always parsed before plugins see them. Plugins operate on structured data, not raw byte streams.

### 3.6 Inter-Plugin Communication

Plugins communicate exclusively through **CycleState** -- a per-request, typed key-value store that lives for the duration of a single request (or a single agentic loop invocation, spanning all iterations).

CycleState enables data flow between plugins without coupling them:
- `model-provider-resolver` writes `provider=anthropic` to CycleState
- `api-translation` reads `provider` from CycleState to select the correct translator
- `apikey-injection` reads `credential-ref-name` from CycleState to inject the right API key

CycleState is also the mechanism for plugin-to-executor signaling:
- `loop-controller` writes `loop_action = continue | stop_completed | stop_yield | stop_incomplete | stop_failed`
- `tool-call-handler` writes `tool_routing[]` entries for the executor to dispatch
- Plugins write events to `CycleState.event_queue` for the executor to flush to the client transport

CycleState keys are namespaced by convention. Well-known keys are documented. Plugins may define custom keys for domain-specific communication.

### 3.7 Configuration

Every plugin is configured through a **typed configuration blob** passed to its factory at creation time. The configuration format is plugin-specific. The configuration is validated at creation time -- invalid configuration prevents the plugin from being registered.

Plugins do not read configuration at request time. Runtime behavior changes are achieved through:
- Configuration reload (the runtime destroys and recreates plugin instances)
- CycleState (per-request data set by upstream plugins)
- Conditional execution (the pipeline executor skips plugins based on CycleState predicates)

### 3.8 Conditional Execution

Plugins can be gated by **conditions** evaluated before the plugin's hooks run. Conditions are declared in the pipeline configuration, not inside the plugin. A plugin does not know whether it was conditionally skipped.

Conditions match on **CycleState values** -- e.g., only run `hydrate-prompt` if `has_previous_response_id = true`, only run `tool-call-handler` if `has_tools = true`. This is the primary gating mechanism for separating Chat Completions and Responses API behavior within a single pipeline.

---

## 4. What an AI Plugin CAN Do

### 4.1 Observe

- Read request/response headers and body
- Read CycleState values set by other plugins
- Emit structured logs and metrics
- Write events to `CycleState.event_queue` (the executor flushes these to the client transport)

### 4.2 Transform

- Add, modify, or remove request/response headers
- Modify request/response body fields (e.g., translate between API formats, inject system prompts)
- Replace the entire request/response body (e.g., api-translation between OpenAI and Anthropic formats)
- Rewrite the upstream path or cluster selection (for routing)

### 4.3 Gate

- Reject requests that fail validation, authentication, authorization, rate-limiting, or content policy checks
- Block responses that violate output guardrails

### 4.4 Communicate

- Write to CycleState to share data with downstream plugins in the same request
- Read from CycleState to consume data set by upstream plugins
- Write control-flow signals to CycleState for the executor to act on (e.g., `loop_action`, `tool_routing`)

### 4.5 Call External Services (External-boundary plugins only)

- Call model provider endpoints for inference
- Call guardrails/moderation endpoints for content policy checks
- Call tool backends (MCP servers, vector stores, search APIs, code interpreters)
- Read from and write to persistent stores (response store, conversation store)
- Watch Kubernetes resources (Secrets, CRDs) via the runtime handle

---

## 5. What an AI Plugin CANNOT Do

These constraints are non-negotiable. They define the safety and composability boundaries of the plugin model.

### 5.1 Cannot Control Pipeline Ordering

A plugin cannot specify where it runs in the pipeline. Pipeline ordering is declared in the pipeline configuration by the gateway operator. A plugin does not know which plugins come before or after it. It can only return a Plugin Action (Continue or Reject) to influence control flow.

**Rationale:** Plugins must be composable in arbitrary orders. A plugin that assumes a specific position breaks composability.

### 5.2 Cannot Access Other Plugins Directly

A plugin cannot call, reference, or depend on another specific plugin. Inter-plugin data flow uses CycleState only. There is no plugin-to-plugin RPC, no dependency injection between plugins, and no plugin discovery API.

**Rationale:** Direct plugin coupling creates ordering dependencies, version conflicts, and testing complexity. CycleState provides loose coupling with typed keys.

### 5.3 Cannot Modify CycleState Keys Owned by Other Plugins

CycleState keys follow an **ownership convention**: the plugin that creates a key owns it. Other plugins may read it but should not overwrite it. The runtime does not enforce ownership (CycleState is a shared map), but violating ownership is a contract violation.

For accumulation patterns (e.g., per-iteration token usage), the convention is: one plugin writes per-iteration values to a namespaced key (e.g., `iteration_usage`), and a designated accumulator plugin (e.g., `loop-controller`) reads and sums them into total keys that it owns.

**Rationale:** Prevents subtle bugs where a downstream plugin silently overrides upstream state.

### 5.4 Cannot Maintain Per-Request State Beyond the Request Lifecycle

CycleState is destroyed at the end of each request (or agentic loop invocation). Plugins cannot persist per-request data across requests. Cross-request state (e.g., conversation history, rate-limit counters) must be stored in external systems accessed through the runtime handle.

**Rationale:** The gateway is a stateless pipeline executor. Per-request state that leaks across requests causes memory growth, race conditions, and incorrect behavior under load.

### 5.5 Cannot Fork or Modify the Pipeline at Runtime

A plugin cannot add, remove, or reorder plugins in the pipeline during request processing. The pipeline is immutable for the duration of a request. A plugin cannot spawn sub-pipelines or branch execution.

**Rationale:** Pipeline immutability ensures deterministic behavior, simplifies debugging, and enables static analysis of plugin chains. Background queueing (which forks execution) is an executor responsibility, not a plugin capability.

### 5.6 Cannot Block the Pipeline Indefinitely

Plugins must respect timeout constraints. The runtime enforces per-plugin and per-pipeline timeouts. A plugin that blocks (e.g., waiting for an unresponsive external service) is terminated and the pipeline proceeds with an error.

External-boundary plugins must implement timeouts on all network calls. The default timeout is configurable per plugin instance.

**Rationale:** A hung plugin blocks the entire request and consumes a connection slot. Timeouts ensure bounded latency.

### 5.7 Cannot Execute Arbitrary Code on the Host

Plugins run within the gateway process boundary. They cannot spawn child processes, access the filesystem (except through sanctioned APIs), or execute shell commands. Tools that require host-level access (e.g., `local_shell`, `apply_patch`) must be routed to a sandboxed environment and require explicit opt-in by the operator.

**Rationale:** The gateway runs as shared infrastructure. Arbitrary code execution by plugins is a security boundary violation. Sandboxed-boundary plugins exist for this use case but are disabled by default.

### 5.8 Cannot Bypass the Plugin Action Contract

A plugin must return a Plugin Action from every hook. It cannot silently consume a request, write directly to the client connection, or modify the response outside of the designated context fields. All client-visible behavior flows through the Plugin Action and context mutation APIs.

**Rationale:** The executor owns the client connection and response lifecycle. Plugins that bypass this contract break observability, streaming, and error handling.

### 5.9 Cannot Hold Mutable Singleton State Across Requests

A plugin instance's internal state (set at creation time from configuration) must be immutable or synchronized. Plugins are invoked concurrently across requests. Mutable state that is shared across requests (e.g., a counter, a cache) must use thread-safe primitives and must not affect correctness if lost (e.g., on restart).

**Rationale:** The gateway may run multiple instances with shared-nothing architecture. Plugin state that assumes a single instance causes inconsistency in distributed deployments.

---

## 6. Executor Contract

The executor is the runtime that invokes plugins. It owns concerns that fall outside the plugin contract -- pipeline forking, transport management, and loop orchestration. This section defines what the executor does and why these are not plugin responsibilities.

### 6.1 Pipeline Invocation

The executor creates a pipeline from the configuration, evaluates conditional execution predicates before each plugin, invokes plugin hooks in order, and reads Plugin Actions to determine control flow (continue to the next plugin or short-circuit with a rejection).

### 6.2 Agentic Loop

The executor owns the iteration lifecycle for Responses API requests:

1. After each iteration, reads `loop_action` from CycleState (written by `loop-controller`)
2. If `continue`: re-enters the pipeline at the configured re-entry point (typically `api-translation` request hook)
3. If `stop_*`: exits the loop and routes to completion plugins
4. Reads `tool_routing[]` from CycleState and dispatches to the matching tool executor plugins. If `parallel_dispatch = true`, runs them concurrently

Loop control lives in the executor because it requires re-entering the pipeline (§5.5 prohibits plugins from forking or re-invoking the pipeline).

### 6.3 Streaming Event Delivery

Plugins write events to `CycleState.event_queue`. After each plugin hook returns, the executor flushes the event queue to the client transport (SSE or WebSocket). Plugins never write directly to the client connection (§5.8).

The executor also emits lifecycle events directly (e.g., `response.created`, `response.in_progress`, `response.completed`) since these are executor-level state transitions, not plugin outputs.

### 6.4 Background Queueing

When CycleState `is_background = true`, the executor stores an immediate `queued` response, returns it to the client, and enqueues the pipeline for async processing by a worker. This is a pipeline fork -- it returns to the client AND continues processing -- which plugins cannot do (§5.5).

### 6.5 Function Tool Yield

When the executor reads `loop_action = stop_yield` from CycleState, it exits the loop and delivers a response containing function call items. The client executes the function and resumes with `previous_response_id`. This cooperative handoff is expressed as a CycleState signal, not a Plugin Action, because the executor must coordinate between exiting the loop and assembling the response.

### 6.6 Context Truncation

Truncation is inline logic within specific plugins (`hydrate-prompt` during context reconstruction, `inference-caller` before each model call), not a separate plugin. These plugins read the `truncation` parameter from CycleState and apply it. Making truncation a standalone plugin would require inter-plugin calls (§5.2 violation) and running at two different pipeline positions.

---

## 7. Plugin Classification

Plugins fall into categories based on their role in the AI traffic lifecycle. These categories are conventions, not enforced by the runtime.

### 7.1 Single-Pass Plugins (Chat Completions)

These plugins participate in the request/response flow for standard inference proxying.

| Category | Examples | Hooks Used |
|----------|----------|------------|
| **Auth & Identity** | apikey-injection, token-validation | on_request |
| **Routing & Resolution** | model-provider-resolver, router, load-balancer | on_request |
| **API Translation** | api-translation (OpenAI, Anthropic, Bedrock, Vertex, Azure) | on_request, on_response |
| **Guardrails** | guardrails (NeMo, custom) | on_request, on_response |
| **Observability** | access-log, request-id, metrics | on_request, on_response |
| **Traffic Management** | rate-limit, timeout, circuit-breaker | on_request |

### 7.2 Agentic Loop Plugins (Responses API)

These plugins run within the agentic loop. They use the same two hooks (`on_request`, `on_response`) as single-pass plugins. Agentic behavior is achieved through pipeline positioning and CycleState signals, not through special hooks.

| Category | Examples | Mechanism |
|----------|----------|-----------|
| **Intake** | request-validator, hydrate-prompt, tool-registry, conversation-manager | Runs once at intake via `on_request` |
| **Loop Control** | loop-controller | Writes `loop_action` to CycleState; executor reads it |
| **Inference** | inference-caller | Direct model call within the pipeline |
| **Tool Detection** | tool-call-handler | Reads model output from CycleState, writes `tool_routing[]` |
| **Tool Execution** | server-tool-executor, mcp-executor, host-tool-executor | Executor dispatches based on `tool_routing[]` |
| **Completion** | response-assembler, response-store | Runs after loop exit |

### 7.3 Cross-Cutting Plugins

Some plugins work identically in both single-pass and agentic flows:

- **api-translation** -- translates between the gateway's input format and provider-native formats. In single-pass, it runs once. In the agentic loop, it runs per-iteration.
- **guardrails** -- safety evaluation. In single-pass, checks the incoming request and outgoing response. In the agentic loop, checks each iteration's input and output.
- **apikey-injection** -- injects provider-specific credentials. Works identically in both flows.

---

## 8. Design Comparison: IPP Plugins vs. Praxis Filters

This section maps the two existing designs to the AI Plugin model and identifies what each contributed.

| Aspect | IPP Plugins | Praxis Filters | AI Plugin (this RFC) |
|--------|------------|----------------|---------------------|
| **Language** | Go | Rust | Rust (target: Praxis filters/extensions) |
| **Interface** | `RequestProcessor` / `ResponseProcessor` | `HttpFilter` / `TcpFilter` | `on_request` / `on_response` hooks |
| **Control flow** | Return `error` to abort | Return `FilterAction` (Continue/Reject/Release/BodyDone) | Return `PluginAction` (Continue/Reject) |
| **Body access** | Always parsed (JSON map) | Opt-in (None/ReadOnly/ReadWrite) | Always parsed (JSON -- AI gateway traffic is structured) |
| **Inter-plugin state** | `CycleState` (per-request shared map) | `HttpFilterContext` fields | `CycleState` (explicit, typed, documented keys) |
| **Conditional execution** | Not supported (all plugins run) | `conditions` on filter entries | CycleState-based conditions on pipeline entries |
| **Configuration** | `json.RawMessage` passed to factory | `serde_yaml::Value` passed to `from_config` | Typed configuration blob |
| **External state** | Kubernetes `Handle` (client, reconciler, context) | None (filters are pure functions of config) | Runtime handle (abstraction over k8s, secrets, etc.) |
| **Agentic loop** | Not supported | Not supported | Supported via CycleState signals + executor loop |

### What IPP Contributes

- **CycleState** as the inter-plugin communication primitive. Praxis uses context fields, which couple plugins to the context struct. CycleState is more flexible and extensible.
- **Factory pattern** with configuration and runtime handle. Praxis factories receive only a YAML value; IPP factories also receive a runtime handle for k8s integration.
- **Typed plugin identity** (TypedName with type + instance name).
- **Always-parsed JSON bodies**. AI gateway traffic is structured; the opt-in body access model from Praxis adds complexity without benefit for this domain.

### What Praxis Contributes

- **Explicit control-flow actions** (Continue, Reject) instead of Go errors, which conflate "something went wrong" with "reject this request."
- **Conditional execution** declared in configuration. IPP has no equivalent; every plugin runs on every request. This RFC adopts CycleState-based conditions as the primary gating mechanism.
- **The target runtime.** Praxis is the execution platform. All plugins defined in this RFC will be implemented as Praxis filters or extensions in Rust.

### Current State and Target

The existing IPP plugins (7 plugins: `auth`, `rate-limit`, `model-provider-resolver`, `body-field-to-header`, `base-model-to-header`, `api-translation`, `apikey-injection`) are implemented in Go. The target state is a full Rust implementation on Praxis.

This means all IPP plugins require a rewrite from Go to Rust, and all new plugins (13 for Responses API support) will be written directly in Rust. The Go-to-Rust transition plan -- sequencing, migration strategy, dual-running, and cutover -- is not covered in this document.

---

## 9. Applicability to Responses API Implementation

The Responses API (see `responses-api-agentic-loop.md`) requires plugins that go beyond single-pass processing. The AI Plugin model supports this through:

1. **CycleState as the control plane.** Loop decisions (`loop_action`), tool routing (`tool_routing[]`), and streaming events (`event_queue`) flow through CycleState. The executor reads these signals and orchestrates accordingly.

2. **Executor-owned loop iteration.** The executor re-enters the pipeline at a configured point after tool execution, keeping loop control outside of plugins (§5.5).

3. **Function tool yield via CycleState.** When the model emits a `function` tool call, `loop-controller` writes `loop_action = stop_yield`. The executor exits the loop and delivers a response with function call items. The client resumes with `previous_response_id`.

4. **Plugin categorization** that separates intake, loop control, inference, tool execution, and completion phases (see `agentic-loop-plugins.md`).

5. **Boundary classification** (Local, External, Sandboxed) that informs timeout policies, error handling, and security decisions. External-boundary plugins that call tool backends need different retry and timeout behavior than local plugins that parse JSON.

The existing IPP plugins (`api-translation`, `apikey-injection`, `guardrails`) work in both single-pass and agentic flows without modification -- they implement `on_request`/`on_response` hooks that the executor invokes regardless of traffic pattern.

---

## 10. Open Questions

1. **Should CycleState keys be schema-validated?** Currently keys are string-typed by convention. Enforcing a schema (e.g., `provider` must be one of the known provider constants) would catch bugs earlier but adds rigidity.

2. **Should third-party plugins use WASM for language independence?** The core plugins will be Rust on Praxis. A WASM extension point would allow third-party plugins in other languages at the cost of a performance boundary and a more constrained API.

3. **How does plugin error handling interact with the agentic loop?** In single-pass, a plugin error typically rejects the request. In the agentic loop, a tool execution failure might be recoverable (report the error to the model as a tool result and let it retry). The error-handling contract may need loop-aware semantics.

---

## 11. Summary

An **AI Plugin** is a named, typed, composable processing unit that:

- **IS:** a single-concern function that observes, transforms, or gates AI traffic at `on_request` and/or `on_response` hooks
- **CAN:** read and write request/response data, communicate with other plugins through CycleState, call external services (if classified as External), reject requests, or signal the executor via CycleState
- **CANNOT:** control its position in the pipeline, access other plugins directly, persist state across requests, fork the pipeline, block indefinitely, execute arbitrary code on the host, or bypass the plugin action contract

The **executor** owns orchestration concerns that fall outside the plugin contract: agentic loop iteration, tool dispatch, streaming event delivery, background queueing, and function tool yield. Plugins signal intent through CycleState; the executor acts on it.

This model draws primarily from IPP plugins (CycleState, factory pattern, typed identity, runtime handle, always-parsed JSON bodies) with conditional execution inspired by Praxis filters, and an explicit Plugin Action contract (Continue/Reject) replacing Go's error-based control flow. The target implementation is Rust on Praxis; existing Go IPP plugins require a rewrite (transition plan out of scope).
