# Inference Proxy Conformance — Gap Analysis

Analysis of the IPP (Inference Payload Processor) against the [Inference Proxy Conformance Guidelines](https://docs.google.com/document/d/1yDzs9ehHFxqYbufOmY-sEXiEtn9nhXUKHx4uKjasC5Q) draft by Ben Browning.

## Context

The conformance doc assumes a **transparent same-format proxy** (same API in, same API out). IPP is an **API translator** — it intentionally transforms between OpenAI and provider-native formats (Anthropic Messages API, Vertex GenerateContent, etc.). This distinction is important: sections on "preserve all fields" and "don't modify responses" apply differently depending on whether IPP is in passthrough mode (OpenAI provider) or translation mode (Anthropic/Vertex native).

## Praxis Direction — How These Gaps Close

The AI Gateway direction ([Architecture & Direction](https://docs.google.com/document/d/1-c6qyeLpS2y1aCUUj6CCOtwtXu7AkAkQZuNU878qCpQ/edit)) replaces the ext_proc-attached IPP with the [Praxis](https://github.com/praxis-proxy/praxis) converged data plane, where every capability is a filter in one in-process chain ([praxis-ai](https://github.com/praxis-proxy/ai), [interactive visualizer](https://htmlpreview.github.io/?https://github.com/n1hility/ai-gateway-docs/blob/praxis-responses/responses-api/index.html)). Several gaps below are structural consequences of the callout protocol and close by construction in one process; others become ordinary filter work with a natural landing spot.

| Gap # | Status on Praxis |
|---|---|
| 2 (streaming translation) | Closes by construction — SSE is parsed in-process (`SseFrameParser`, `openai_stream_events`, `anthropic_stream_events`); no per-chunk callout to buffer or corrupt chunk boundaries. |
| 4 (`/v1/responses`) | Implemented as the Responses filter set in praxis-ai (format/validate/store/rehydrate/`responses_proxy`); agentic tool loop specified in [proposal 00354](https://github.com/praxis-proxy/ai/blob/main/docs/proposals/00354_responses-api-filters.md). |
| 8 (`/v1/messages` as input) | Implemented — `anthropic_messages_format` / `anthropic_to_openai` / `anthropic_stream_events` provide native Anthropic Messages ingress. |
| 1, 3 (field preservation, `stream_options`) | Translation is a filter over the parsed body in one process; forward-by-default field handling is a filter-local fix rather than a plugin-chain rebuild. |
| 5 (client disconnect) | Praxis owns both connections in-process (Pingora); no ext_proc boundary to lose disconnect signals across. |
| 10, 12 (`X-Request-Id`, body limits) | Core built-ins: `request_id` filter and `body_limits` config ceilings. |
| 6, 7, 9, 11 | Remain ordinary roadmap items (benchmarks, legacy endpoints, embeddings, typed proxy errors), now with a single substrate to land in. |

## Part 1: IPP Gaps Against the Conformance Doc

### HIGH Priority

| # | Guideline | IPP Status | Notes |
|---|-----------|-----------|-------|
| 1 | Preserve ALL unknown request fields | Anthropic translator drops anything it doesn't explicitly copy | Root cause of PRs #137, #188, #189. Fix should be to forward all unrecognized fields by default instead of playing whack-a-mole. |
| 2 | Streaming response translation (Anthropic) | Untested — translator rebuilds full response body, unclear behavior with SSE chunks | May buffer entire response or corrupt chunk boundaries. Needs validation. |

### MEDIUM Priority

| # | Guideline | IPP Status | Notes |
|---|-----------|-----------|-------|
| 3 | `stream_options` forwarding | Not forwarded in translation path | `include_usage`, `continuous_usage_stats` silently dropped. |
| 4 | `/v1/responses` endpoint | Not supported | Required for agentic workflows with tool use and reasoning chains. |
| 5 | Client disconnect → close upstream | Not implemented in ext_proc | Wastes provider resources and tokens on abandoned requests. |
| 6 | Chunk latency target (10-50ms per chunk) | Not measured | No benchmarks for streaming latency through ext_proc. |

### LOW Priority

| # | Guideline | IPP Status | Notes |
|---|-----------|-----------|-------|
| 7 | `/v1/completions` endpoint | Not supported | Legacy, but SHOULD per doc. |
| 8 | `/v1/messages` as input | We translate TO Anthropic format, but don't accept Anthropic-format requests as input | Doc says SHOULD support for Anthropic-compatible clients. |
| 9 | `/v1/embeddings` endpoint | Not supported | Only relevant if proxying embedding models. |
| 10 | `X-Request-Id` forwarding | Not implemented | Useful for distributed tracing across client → proxy → provider. |
| 11 | Proxy-specific error types | Errors use `errcommon.Error` but don't use distinct types like `proxy_auth_error` vs `proxy_rate_limit` | Could confuse clients about error source (proxy vs provider). |
| 12 | Request body size limits | Not enforced | Doc recommends for DoS prevention. |

### N/A — By Design

| # | Guideline | IPP Status | Notes |
|---|-----------|-----------|-------|
| 13 | MUST NOT modify response structure | Anthropic/Azure response translation modifies responses by design | This is IPP's core value — normalizing all provider responses to OpenAI format. |
| 14 | MUST NOT validate beyond proxy needs | OpenAI translator validates `messages` field (PR #146) | Intentional — returns 400 instead of letting provider return 500. Contradicts guideline but improves UX. |
| 15 | CORS header propagation | Handled by Istio/gateway, not ext_proc | Not IPP's responsibility. |

## Part 2: Gaps in the Conformance Doc

Things the doc doesn't cover that real-world inference proxies need.

### API Translation Proxies

The doc assumes same-format in/out. Many production proxies translate between formats (OpenAI → Anthropic, OpenAI → Vertex). The doc treats all response modification as prohibited, but translation is the core value of multi-provider gateways.

**Needed:** A section on "Translation Proxies" with separate conformance rules — preserve semantics, not bytes. Define what "conformance" means when the proxy intentionally changes the wire format.

### Credential Injection Patterns

The doc covers auth passthrough, termination, and injection, but doesn't address provider-specific credential patterns. Real proxies read credentials from K8s Secrets or Vault and inject different headers per provider:
- OpenAI: `Authorization: Bearer <key>`
- Anthropic: `x-api-key: <key>`
- Azure: `api-key: <key>`
- Bedrock: `Authorization: Bearer <encoded-key>`

**Needed:** Guidance on provider-specific credential injection and header naming.

### Multi-Provider Routing

No mention of routing to different backends based on request content. A proxy fronting multiple providers must inspect the `model` field to route to the correct backend. The doc says "don't validate parameters" but the proxy MUST read `model` to route.

**Needed:** Acknowledge routing-by-parameter (e.g., `model` field) as a valid proxy responsibility, distinct from parameter validation.

### Path Rewriting

The doc doesn't address different path conventions per provider:
- OpenAI: `/v1/chat/completions`
- Anthropic: `/v1/messages`
- Vertex: `/v1beta1/projects/{project}/locations/{location}/endpoints/{endpoint}/chat/completions`
- Bedrock: `/model/{model-id}/converse`

A multi-provider proxy must rewrite paths per provider.

**Needed:** A section on path handling for multi-backend proxies, including custom path prefixes for non-standard endpoints.

### Response Format Normalization

The doc says preserve responses exactly, but multi-provider proxies need to normalize responses to a common format. If the proxy accepts OpenAI-format requests and forwards to Anthropic, the response must be translated back:
- `stop_reason: "end_turn"` → `finish_reason: "stop"`
- `content[].text` → `choices[].message.content`
- `input_tokens/output_tokens` → `prompt_tokens/completion_tokens`

**Needed:** Conformance criteria for response translation — what must be preserved semantically even when the wire format changes.

### Streaming with API Translation

The doc covers SSE passthrough but not format conversion during streaming. When translating Anthropic's streaming format (different event types: `content_block_start`, `content_block_delta`, `message_stop`) to OpenAI SSE format, the chunk structure changes fundamentally.

**Needed:** Guidelines for streaming translation conformance — latency, semantic fidelity, chunk boundary handling.

### Service Mesh / ext_proc Integration

No mention of Envoy ext_proc, Istio sidecars, or service mesh patterns. Many production proxies run as ext_proc filters inside Envoy, not as standalone HTTP proxies. This affects:
- Body buffering modes (buffered vs streamed)
- Header mutation semantics (set vs append vs remove)
- Streaming behavior (ext_proc processes body chunks, not raw SSE)

**Needed:** A section on conformance for ext_proc-based proxies and their constraints.

### CRD / Config-Driven Behavior

The doc assumes static proxy configuration. Kubernetes-native proxies use CRDs (ExternalModel, ExternalProvider) to dynamically configure provider endpoints, credentials, and routing at runtime.

**Needed:** Acknowledge dynamic configuration as a valid pattern and its implications for conformance testing.

### Rate Limiting Interaction

The doc mentions rate limiting briefly but doesn't address token-based rate limiting. Token-based rate limiting (e.g., Kuadrant TokenRateLimitPolicy) requires the proxy to extract `usage` from responses and report it. This conflicts with "don't inspect responses."

**Needed:** Guidance on how rate limiting and usage metering interact with the "don't modify responses" principle.

### Health Check / Readiness Endpoints

Not mentioned at all. Proxies in Kubernetes need `/healthz` and `/readyz` endpoints. Inference servers also have health endpoints that the proxy should expose or forward.

**Needed:** Requirements for health/readiness endpoints in proxy deployments.
