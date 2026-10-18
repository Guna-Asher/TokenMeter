# TokenMeter — Product Requirements Document (PRD)

**Version:** 0.2  
**Status:** Draft  
**Primary upstream:** OpenRouter  
**Initial API surface:** OpenRouter-compatible Chat Completions  
**Initial cost scope:** Text input/output token usage, with an extensible schema for other billing dimensions

## 1. Product description

TokenMeter is a usage tracking and cost management service for developers using AI APIs. A developer routes supported requests through TokenMeter; TokenMeter authenticates the project, forwards the request to OpenRouter, returns the upstream response, records available usage information, calculates cost where possible, and displays usage in a dashboard.

OpenRouter is the first integration. The internal product must keep provider-specific response parsing separate from generic usage, pricing, project, budget, and reporting logic so that additional providers can be added later without rewriting the core domain.

## 2. Primary user workflow

1. A developer creates a TokenMeter account/workspace and a project.
2. The developer configures an OpenRouter API key securely.
3. TokenMeter issues a project-specific gateway credential.
4. The developer changes the application to call TokenMeter's gateway rather than calling OpenRouter directly.
5. The application sends a supported request with an OpenRouter model ID, such as `openai/gpt-4o`, or a router such as `openrouter/free`.
6. TokenMeter forwards the request and requests usage reporting with `usage: { "include": true }` where supported by the endpoint.
7. TokenMeter streams or returns the upstream response to the client.
8. TokenMeter captures the requested model, resolved/returned model when available, usage, reported cost and request status.
9. TokenMeter calculates an additional catalogue-based cost when sufficient pricing and usage data exist.
10. The dashboard shows usage, cost basis, project totals, alerts, and incomplete/uncertain records.

Traffic that bypasses TokenMeter is outside the gateway's automatic request-level tracking scope.

## 3. Core cost semantics

Every usage record must make the source of its cost explicit. At minimum, support these fields/statuses:

- `provider_reported_cost`: cost reported by OpenRouter in the response, if present.
- `calculated_cost`: TokenMeter's cost calculation from usage and a stored pricing version, if possible.
- `cost_currency`: initially USD for OpenRouter-reported and catalogue costs.
- `cost_status`: `provider_reported`, `calculated`, `known_zero`, `unknown`, or `pending_usage`.
- `pricing_version_id`: version of the catalogue record used for calculation, if any.
- `usage_source`: e.g. `openrouter_response`, `openrouter_stream_final_chunk`, or `reconciliation_import` in a future release.

A provider-reported zero is a valid value. A catalogue price of zero is also a valid value. Unknown pricing must be represented as `null` plus an explicit unknown status, never as `0`.

When both provider-reported cost and a catalogue calculation are available, store both. The dashboard may use provider-reported request cost as the primary displayed figure, while retaining the calculated figure for validation and comparison. Do not silently overwrite one with the other when they differ.

## 4. Model identifier rules

- Persist the exact requested `model` string sent by the client.
- Persist the exact `model` string returned by OpenRouter when present.
- Preserve namespaces and variant suffixes, including identifiers like `openai/gpt-4o`, `some-author/some-model:free`, and routers like `openrouter/free`.
- Do not use a display name as the unique model identity.
- Do not strip `:free` or other suffixes as a generic normalization step. A variant can affect behavior and pricing.
- Synchronize model IDs, canonical slugs, labels, context/capability metadata, and pricing from OpenRouter's `/api/v1/models` API.
- For dynamic routing, price usage against the concrete model returned by OpenRouter if available and supported by the catalogue. If the resolved model is unavailable but the response includes a reported cost, preserve that provider-reported cost. If neither reliable cost basis is available, mark the cost unknown instead of guessing.

## 5. Functional requirements

### FR-01: Workspaces and projects

Users can create, view, update, and archive projects. Each gateway credential is scoped to a workspace/project. Every usage record belongs to one authorized project.

**Acceptance criteria**
- Projects can be created and listed by their owner.
- Usage queries always enforce workspace/project authorization.
- Invalid, revoked, or out-of-scope gateway credentials are rejected.

### FR-02: OpenRouter credentials

Users can configure an OpenRouter API key. It must be encrypted at rest, masked in the UI, excluded from logs and errors, and revocable.

**Acceptance criteria**
- Valid keys can authorize upstream calls.
- Invalid upstream keys produce a safe, actionable error.
- Secret values never appear in logs, usage records, or ordinary API responses.
- Revoked credentials cannot authorize future requests.

### FR-03: OpenRouter gateway

Implement a TokenMeter gateway endpoint that accepts supported OpenRouter-compatible chat-completion requests and forwards them to OpenRouter's chat-completions endpoint. Include usage reporting in the upstream request where supported. Return the provider response to the client while recording relevant metadata.

The initial release supports non-streaming and streaming text chat requests. Other modalities and endpoint families are future work unless specifically added to scope.

**Acceptance criteria**
- A valid non-streaming request reaches OpenRouter and the client receives the response.
- Streaming data is forwarded as Server-Sent Events without waiting for the full response.
- Upstream errors and timeouts are returned in a predictable, safe form.
- The gateway does not claim a request succeeded if the outcome is uncertain.
- Provider-specific request/response handling is isolated behind an adapter.

### FR-04: Usage capture

Capture, when available:

- Workspace/project and internal request IDs.
- Requested model ID and returned/resolved model ID.
- OpenRouter response/request ID where available.
- Start/end timestamps and latency.
- Input/prompt tokens and output/completion tokens.
- Cached input tokens and reasoning tokens when present in usage details.
- Total tokens and raw usage payload version/schema metadata as appropriate.
- HTTP/outcome status and retry/stream outcome metadata.
- Provider-reported cost, currency, usage source, and calculated cost status.

Do not store raw prompts or completions by default. Avoid persisting authorization headers and secrets.

**Acceptance criteria**
- Successful requests with usage data create a usage record.
- Reprocessing a record or event cannot double-count usage.
- Missing usage is marked pending/unknown rather than filled with invented counts.
- Failed requests remain visible with a status, even when they have zero billable usage or no usage data.

### FR-05: Dynamic model catalogue

Use OpenRouter's `/api/v1/models` endpoint as the first model catalogue source. Store the exact API model ID, canonical slug if supplied, human-readable name, and supported pricing fields relevant to the MVP. Refresh the catalogue on a scheduled or administrator-triggered basis; do not fetch it on every user request.

Pricing fields returned by OpenRouter may be serialized as strings. Parse them using decimal-safe arithmetic. Store each fetched catalogue snapshot/version and its retrieval time; don't rewrite historical usage costs when new catalogue data arrives.

**Acceptance criteria**
- `openai/gpt-4o` can be stored and looked up as an exact model identifier if present in the current catalogue.
- Model identifiers with variant suffixes are preserved.
- Removed or unavailable models remain referable from historical usage records.
- Pricing refreshes create a new version/snapshot rather than mutating the provenance of past calculations.

### FR-06: Cost calculation

For the initial text-token scope, calculate cost from the appropriate model pricing and usage dimensions, using decimal-safe arithmetic. Use the exact resolved model and relevant price version when possible.

Conceptually:

`calculated_cost = input_tokens × input_price_per_token + output_tokens × output_price_per_token + supported_additional_charges`

The actual formula must use the price units and fields exposed by the chosen OpenRouter catalogue record. Do not assume every future model's price is only input/output tokens; keep a pricing-dimension model extensible for image, audio, request-based or other billing units.

**Acceptance criteria**
- Test fixtures produce expected costs for known paid model prices.
- A known zero-price model/variant may produce a calculated cost of exactly `0` and a `known_zero` or equivalent known-price status.
- Unknown/missing price is represented as unknown, not zero.
- A missing usage field is not interpreted as zero tokens unless the response semantics explicitly justify that interpretation.
- Cost rounding/precision rules are documented and tested.
- Historical results retain pricing provenance.

### FR-07: Streaming usage

Request usage data for streaming calls where supported and process the final usage event/chunk when it arrives. Persist partial request metadata separately from final usage. If the connection ends before final usage arrives, mark the request's usage as incomplete/pending rather than assigning a guessed cost.

**Acceptance criteria**
- A completed stream with a final usage chunk creates one final usage result.
- Duplicate final chunks or retry processing do not double-count the request.
- Interrupted streams are visible and do not appear as known-zero-cost requests by default.

### FR-08: Dashboard and reports

Display:
- total provider-reported and/or calculated inference cost for a selected period;
- usage grouped by project, requested model, and resolved model;
- input/output token counts and request count;
- cost-status breakdown (reported, calculated, known zero, unknown/pending);
- request history, including errors and incomplete usage;
- daily spend trends.

The UI must not add unknown costs to numeric totals as if they were zero without clearly showing that some records are uncosted.

**Acceptance criteria**
- Summary totals use consistent time ranges and filters.
- Requested and actual model grouping can be distinguished for dynamic routers.
- Paginated history is available.
- Unknown/pending costs are separately visible.

### FR-09: Budgets and alerts

Users can set a monthly project budget and warning thresholds, initially 50%, 80%, and 100%. The first release provides monitoring/alerts only. Threshold basis should be explicit: use provider-reported cost when available and otherwise a documented calculated-cost fallback, with the dashboard showing any uncosted usage caveat.

Deduplicate alerts by project, budget period, threshold, and crossing event. Avoid repeated alerts for every subsequent request beyond the same threshold.

**Acceptance criteria**
- A threshold crossing creates a single alert for that period/threshold crossing.
- Duplicate usage ingestion does not duplicate alerts.
- New budget periods reset threshold state.
- The alert and UI identify which cost basis was used.

### FR-10: Forecast

Estimate end-of-period cost from a documented, simple baseline using recent observed usage. Label the result as an estimate; show insufficient-data status when history is too sparse. Explain that forecast reliability depends on usage stability and complete cost data.

### FR-11: Idempotency and error handling

Every proxied call receives a unique internal ID. Deduplicate local processing with stable request/event identifiers where available. Do not blindly retry an ambiguous upstream request, since the model call may have completed and incurred cost before a network failure.

**Acceptance criteria**
- Reprocessing captured usage cannot double-count it.
- Ambiguous upstream outcomes are identifiable for review/reconciliation.
- Database transactions do not remain open while the proxy awaits OpenRouter.
- Transient local persistence failures use bounded retry/recovery where safe.

### FR-12: Security and privacy

- Enforce workspace/project authorization on all data routes.
- Encrypt upstream API keys at rest and redact them in logs.
- Do not store raw prompts/responses by default.
- Apply request-size limits, timeouts, and suitable rate limits.
- Record administrative changes to keys and projects.
- Do not log full headers or sensitive payloads in exception traces.

## 6. Proposed data model additions

In addition to workspaces, projects, gateway credentials, budgets, alerts, and usage events, include:

### `model_catalogue_versions`

- `id`
- `source` (initially `openrouter`)
- `fetched_at`
- `catalogue_version` or snapshot identifier

### `models`

- `id`
- `provider_source` (`openrouter` initially)
- `model_id` (exact OpenRouter ID)
- `canonical_slug` where supplied
- `display_name`
- modality/capability metadata needed for the MVP
- active/current catalogue snapshot reference

### `model_prices`

- model/catalogue version reference
- price dimension, such as input token or output token
- decimal price value and price unit
- effective/fetched timestamps

### `usage_events`

- internal event/request ID
- project/workspace ID
- upstream source
- requested model ID
- resolved model ID, nullable
- input/output/cached/reasoning token counts, nullable when not supplied
- `provider_reported_cost`, nullable
- `calculated_cost`, nullable
- currency
- cost status and usage source
- catalogue/pricing version reference, nullable
- request status, latency, timestamps
- upstream response/request ID, nullable
- deduplication key or unique event identifier

Numeric columns must permit a value of exactly zero. Unknown values should be nullable and accompanied by explicit status fields.

## 7. Non-functional requirements

### Correctness
Usage and aggregate totals must remain consistent under retries, duplicate ingestion, and concurrent updates. Zero, unknown, and pending values must remain distinct.

### Performance
Initial target: less than 100 ms of additional p95 gateway processing latency at a declared test load, excluding upstream model generation time. This is a proposed goal, not a verified claim. Measure it with repeatable benchmarks.

### Reliability
Handle OpenRouter errors, timeouts, rate limits, process restarts, stream interruption, missing usage, and failed local persistence without fabricating costs.

### Scalability
Keep gateway, provider adapter, usage normalization, pricing, budget, and reporting code as separate modules within a modular monolith. Move alerts and catalogue refresh into background jobs when helpful.

### Observability
Track request latency, upstream status, provider failures, usage records, cost status distribution, catalogue sync failures, alert delivery failures, and incomplete requests. Never include provider secrets in telemetry.

## 8. Test requirements

Must include tests for:

- paid model price calculations;
- model ID with namespace, e.g. `openai/gpt-4o`;
- a `:free` model variant with $0 pricing;
- router request ID versus concrete returned model ID, including `openrouter/free` behavior;
- valid provider-reported `usage.cost` of `0`;
- missing usage versus zero usage;
- price catalogue string values converted to exact decimal values;
- price snapshot refresh without mutation of historical calculations;
- repeated usage ingestion and duplicate final streaming usage chunks;
- interrupted streams and missing final usage;
- upstream 401/429/5xx, timeout, and ambiguous network failure;
- tenant isolation, API key redaction, and authorization checks;
- budget thresholds with reported, calculated, and unknown costs.

Use mocked HTTP responses for deterministic unit tests and integration tests against a real PostgreSQL instance. Keep a separately marked optional smoke test for a real OpenRouter API key; automated CI must not depend on paid external API calls.

## 9. Release plan

### Phase 1 — Domain and tests
Define usage and cost states, model-ID handling, event deduplication, pricing units, and acceptance tests.

### Phase 2 — OpenRouter catalogue and cost engine
Fetch/cache models, persist catalogue versions, implement decimal cost calculation, and test paid/zero/unknown prices using fixtures.

### Phase 3 — First end-to-end proxy
Proxy a non-streaming chat request, capture OpenRouter usage and reported cost, store one usage event, and return the response.

### Phase 4 — Streaming and failure recovery
Forward Server-Sent Events, capture final usage, and handle interrupted streams and ambiguous upstream outcomes.

### Phase 5 — Reports and budgets
Add request history, project/model totals, budget thresholds, alerts, and a basic forecast.

### Phase 6 — Security, reliability, and deployment
Test concurrency, deduplication, database recovery, credential security, request limits, deployment and observed latency.

### Phase 7 — Additional capabilities
Add further OpenRouter endpoints/modalities, billing/activity imports and reconciliation, other provider adapters, and hard pre-request budget enforcement only after defining safe reservation semantics.

## 10. Definition of done

A developer can configure an OpenRouter credential, make supported text requests through TokenMeter, receive non-streaming and streaming responses, inspect request usage and cost provenance, see correct paid and zero-cost model handling, filter project/model reports, and receive threshold alerts.

The repository includes automated tests, migrations, API documentation, setup instructions, measured performance results, a deployed demo, and documented limitations. Claims of accuracy, reliability, or production readiness must be supported by executed tests and measurements.
