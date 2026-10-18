# TokenMeter — Product Requirements Document (PRD)

**Version:** 0.3  
**Status:** Refined draft  
**Primary upstream:** OpenRouter  
**Core product loop:** Estimate → Decide → Execute → Compare → Improve  
**MVP focus:** Pre-run estimates for supported text-based requests, followed by actual usage capture

## 1. Product definition

TokenMeter is a pre-run AI cost estimator and usage guard for developers. It estimates the likely cost of an AI request before execution, lets the user decide whether to proceed, forwards supported requests through OpenRouter, captures actual usage/cost after execution, and shows how the result compares with the prediction.

TokenMeter should eventually estimate bounded multi-call tasks, not just single requests. However, it must not claim to predict arbitrary agent workflows when the number of model calls, tool calls, retries, or output tokens is unknown. The MVP therefore starts with reliable single-request estimates and links multiple observed requests into a task for cumulative reporting. Bounded task forecasts are a follow-on feature with explicit user-provided assumptions.

## 2. Product principles

1. **Estimate before execution:** pre-run estimation is a core feature, not a future analytics add-on.
2. **Make uncertainty visible:** provide ranges, assumptions, confidence, and warnings; never imply that a probabilistic estimate is an exact quote.
3. **Separate known zero from unknown:** `$0` is a valid cost when confirmed by pricing/usage semantics; missing price or usage is not zero.
4. **Preserve provenance:** keep requested model, actual model, pricing snapshot, estimate method, usage source, and provider-reported cost.
5. **Do not over-collect prompts:** raw prompts and responses are not persisted by default.
6. **Measure before claiming:** prediction accuracy, latency, and reliability claims require a measured sample and documented method.
7. **Keep the first version focused:** OpenRouter first, text-based chat request flow first, extensible provider adapter architecture second.

## 3. User stories

### US-01 — Estimate before sending
As a developer, I want to submit a request payload for estimation and see a likely cost range before sending it to a model, so that I can decide whether to run it or change the model/output limit.

### US-02 — Understand the estimate
As a developer, I want to see the token assumptions, selected model, current pricing snapshot, confidence, and excluded cost dimensions, so that I know why TokenMeter produced the estimate.

### US-03 — Run the same request
As a developer, I want to execute the same request I estimated, so that I can compare the estimate with the actual result without accidental mismatch between the preview and execution payload.

### US-04 — Compare estimated and actual cost
As a developer, I want to see estimate, actual reported cost, computed cost, and their difference for each completed request.

### US-05 — Track a task across calls
As a developer, I want to group multiple API calls under one task ID and see cumulative actual cost. In a later release, I want to estimate a bounded plan before starting the task.

### US-06 — Treat free models correctly
As a developer, I want a free model with known zero inference pricing to show a `$0` estimate, while missing or stale pricing is clearly identified as unknown.

### US-07 — Avoid overspending
As a developer, I want project-budget warnings and, in a later release, an enforceable task budget that can stop new work before a defined spending ceiling is exceeded.

## 4. Scope by release

### MVP — required

- Workspace/project ownership and authorization.
- OpenRouter credential configuration.
- Versioned OpenRouter model/pricing catalogue snapshots.
- A preflight estimate API and simple browser playground for supported text-based chat requests.
- Low/expected/high estimates when assumptions support a useful range.
- Cost-estimate explanation: input token estimate, output assumption, price dimensions, catalogue timestamp/version, confidence, and warnings.
- Supported OpenRouter-compatible chat-completions gateway, initially focused on non-streaming requests if needed to deliver the vertical slice; streaming is required before calling the proxy generally usable and must be explicitly covered in the release plan.
- Request/response usage capture and provider-reported cost capture when returned.
- Store a catalogue-based computed cost separately when it can be calculated.
- Requested model and resolved/actual model tracking.
- Estimate-to-execution linking and estimate-vs-actual comparison.
- Project usage history, cumulative task IDs, monthly budget configuration, and warnings.
- Automated tests and deployment documentation.

### Follow-on milestones

- Robust streaming estimate/execution handling, including final usage events and interrupted streams.
- Task estimator for bounded workflows defined by explicit steps, models, expected call counts, and per-step output limits/ranges.
- Historical calibration by model and request/task category after a sufficient sample is available.
- Reservation-based task budget enforcement and explicit policy for uncertain upstream results.
- Additional providers and provider billing reconciliation.
- Non-text modalities and tool/search pricing dimensions.

### Explicitly out of scope

- Tracking calls that bypass TokenMeter and have not been instrumented.
- Exact predictions for unknown future tool calls, retries, model routes, or open-ended agent loops.
- Emergent/Lovable/Bolt-specific credits.
- Provider-independent exact tokenization for every model.
- All forms of account balance changes, purchase fees, taxes, and invoice adjustments as if they were request inference cost.
- Automatically retrying an upstream request when doing so may create a second billable execution.

## 5. User workflow

### 5.1 Estimate only

1. User opens the playground or calls the estimate API.
2. User chooses an OpenRouter model or supported router and provides a request payload.
3. User sets an output assumption: explicit output token limit or a labelled response-length preset (for example concise, standard, or long).
4. TokenMeter estimates prompt/input tokens from the complete supported request content, including supported messages and tool definitions.
5. TokenMeter loads pricing from its latest cached catalogue snapshot.
6. TokenMeter returns a cost range, the output assumption, confidence, pricing timestamp/version, and any caveats.
7. User can change the model/output assumption and estimate again without incurring inference cost.

### 5.2 Estimate and execute

1. User reviews an estimate.
2. User selects **Run**.
3. TokenMeter executes the same normalized request payload through OpenRouter, retaining the estimate ID and request ID relationship.
4. TokenMeter returns the upstream response to the caller.
5. TokenMeter stores available token usage, actual/resolved model, provider-reported cost, and status.
6. TokenMeter calculates cost from usage and the relevant pricing snapshot where possible.
7. UI displays predicted range, actual provider-reported cost when present, local calculated cost, and error/difference.

### 5.3 Task grouping

1. A client assigns an optional task ID to one or more requests.
2. Each request is recorded under that task.
3. TokenMeter shows cumulative actual cost and call count for the task.
4. In the follow-on task-estimation release, users can provide a bounded plan (steps, selected models, expected number of calls, and per-step output assumptions) to receive a pre-run task-cost range.

## 6. Estimation behavior

### 6.1 What the estimate response must include

At minimum:

- `estimate_id`
- requested model/router ID
- resolved model candidates where they are known; otherwise a routing uncertainty warning
- estimated input token count and method (`model_tokenizer`, `heuristic`, or other documented method)
- output-token assumption and source (`explicit_limit`, `preset`, or later `historical_calibration`)
- low/expected/high estimated cost in USD when computable
- estimate status (`estimated`, `known_zero`, `unknown`, or `unsupported`)
- confidence indicator (`high`, `medium`, `low`) with an explainable reason
- price catalogue snapshot/version and fetched timestamp
- assumptions and excluded cost dimensions

### 6.2 Request token estimation

- Count all supported request parts: system/developer/user messages, conversation history, and supported tool/function schemas.
- Use a compatible tokenizer when available for the selected model.
- When a compatible tokenizer is unavailable, use a documented heuristic and reduce confidence. Do not describe a heuristic as an exact token count.
- The initial accurate-estimation scope is text-based chat requests. Image/audio/video or unsupported modalities must be rejected for estimation or return `unsupported`/low-confidence with an explicit caveat; they must not be treated as zero-token content.
- Never persist raw prompt/request content by default. Process it in memory and store estimation metadata and a non-reversible payload fingerprint only if needed for linking/comparison.

### 6.3 Output-cost assumptions

Output length is not knowable exactly before generation. The estimator must use one of:

- an explicit maximum output token parameter supplied by the request;
- a user-selected response-length preset mapped to documented token assumptions; or
- historical model/task statistics, only after enough suitable observations exist.

The UI must state which assumption is used. A maximum output limit can support a conditional upper-bound calculation only for the billable dimensions actually bounded by that limit and supported by the provider/model semantics. Do not call it an unconditional guaranteed maximum if tools, retries, external services, or hidden calls can create additional charges.

### 6.4 Cost calculation

Use the exact OpenRouter model ID and applicable price dimensions from the stored catalogue snapshot. Pricing values should be parsed from API strings using `Decimal` or an equivalent exact-decimal representation.

Conceptually:

`estimated_cost = estimated_input_tokens × input_rate + assumed_output_tokens × output_rate + supported_other_dimensions`

Actual field names and price units must be mapped according to the OpenRouter catalogue schema. Do not assume every model has only input/output token pricing. Unsupported dimensions must be disclosed and excluded from the estimate rather than silently treated as free.

### 6.5 Free models and unknown costs

- A known model/variant with a valid zero inference price may return an estimate of exactly `0` and status `known_zero`.
- `openai/gpt-4o` and similar namespaced IDs must be stored as exact IDs, not split or rewritten incorrectly.
- A variant such as `model-id:free` must be retained exactly when that variant is the selected catalogue model.
- `openrouter/free` is a routing choice, not necessarily the concrete model that ultimately serves a request. The estimate must reflect its free-routing semantics and known candidate constraints; the actual returned model must be stored after execution. If a selected routing mode can reach paid models and candidates cannot be bounded, provide a broad/low-confidence estimate or `unknown`, not an invented fixed cost.
- Missing price data, failed catalogue refresh, unknown model IDs, and missing usage must never silently become `$0`.
- Distinguish per-request inference cost from account-level credit purchase fees, taxes, or balance adjustments.

### 6.6 Calibration and accuracy reporting

The MVP can use deterministic pricing plus documented token/output assumptions. Do not use the term “AI prediction” to mask a heuristic.

After enough opt-in completed requests exist, later versions may calibrate output estimates or task cost ranges using anonymized feature summaries. Calibration must be evaluated on held-out observations and by model/request category. Report sample size and metrics. Do not train on raw prompts by default.

## 7. Functional requirements and acceptance criteria

### FR-01 — Workspace, project, and credentials

Users can create projects and configure an OpenRouter API key. The key must be encrypted at rest, hidden after entry, and redacted from logs. TokenMeter gateway keys must be stored as hashes where practical and scoped to a workspace/project.

**Acceptance criteria**
- Unauthorized users cannot read another workspace's projects, estimates, or usage.
- Invalid/revoked gateway credentials are rejected.
- Provider and gateway secrets never appear in normal logs, errors, or telemetry.

### FR-02 — Model catalogue and price snapshots

Fetch available models from OpenRouter's `/api/v1/models` endpoint on a schedule or through an administrator-triggered refresh. Do not call the catalogue endpoint for each estimate. Persist exact model IDs, names, relevant capabilities, pricing fields/dimensions, fetched time, and snapshot/version.

**Acceptance criteria**
- `openai/gpt-4o` can be looked up by exact ID when available in the catalogue.
- Model/variant IDs with suffixes are preserved.
- Price strings are parsed using decimal-safe arithmetic.
- Refresh creates a new snapshot and does not mutate the pricing provenance of previous estimates/usage records.
- A stale or failed catalogue sync is visible and can lower estimate confidence.

### FR-03 — Pre-run estimate API and playground

Provide an endpoint such as `POST /api/v1/estimates` and a basic UI that accepts a supported request payload, model ID, and output assumption. It must return the fields defined in Section 6.1 without sending an inference request to the upstream provider.

**Acceptance criteria**
- Estimation alone does not incur model inference cost.
- Re-estimating after changing the model, prompt, or output limit produces a new estimate.
- Returned estimates include pricing provenance, token method, output assumptions, confidence, and exclusions.
- Unsupported modalities or unbounded routing produce an explicit unsupported/unknown/low-confidence outcome.
- No raw prompt content is persisted by default.

### FR-04 — OpenRouter proxy execution

Provide a supported chat-completions-compatible gateway endpoint (for example `POST /api/v1/chat/completions`). Forward the request to OpenRouter, request usage reporting where supported, preserve supported parameters, and return the upstream response. Attach an internal request ID and link to the estimate/task when supplied.

**Acceptance criteria**
- The same normalized payload used for estimation can be executed without unintended mutation.
- Successful upstream responses are returned to the client.
- Upstream error status and safe error information are propagated predictably.
- Upstream calls are not performed while holding a database transaction open.
- Ambiguous network failures are not blindly retried in a way that may execute and charge the request twice.

### FR-05 — Actual usage capture and cost provenance

Record, when available:

- workspace/project/task/estimate/request IDs;
- provider (`openrouter` initially);
- requested model and actual/resolved model;
- timestamps, status, latency, upstream request/response ID;
- input/output/cached/reasoning token counts where supplied;
- provider-reported request cost if supplied;
- TokenMeter-calculated cost where enough data exists;
- currency, pricing snapshot/version, usage source, and cost status.

Suggested cost statuses: `provider_reported`, `calculated`, `known_zero`, `unknown`, `pending_usage`, and `unsupported`. Statuses should be normalized consistently; never infer a numeric zero from missing data.

**Acceptance criteria**
- A valid provider-reported `$0` is represented as zero, not null.
- Missing cost is null plus an explicit status.
- Provider-reported and catalogue-calculated costs are stored separately when both exist.
- A request can be retained even if usage is missing or incomplete.
- Duplicate usage processing does not double-count totals.

### FR-06 — Estimate-to-actual comparison

Link an estimate to the execution that follows it. Show low/expected/high estimate, actual provider-reported cost when available, calculated cost, absolute difference, and whether actual cost fell inside the estimate range.

**Acceptance criteria**
- An execution with no linked estimate is still tracked but is labelled “not estimated”.
- A known-zero actual result is handled without division-by-zero or misleading percentage error.
- Incomplete/unknown actual cost is not included in accuracy metrics as zero.
- The UI identifies which actual-cost source is displayed.

### FR-07 — Task IDs and cumulative usage

Allow multiple requests to share a developer-supplied or TokenMeter-issued task ID. The MVP records and displays cumulative actual cost and calls per task. Pre-run cost estimation of multi-call plans is a follow-on requirement and must require an explicit bounded plan rather than infer unobservable future calls.

**Acceptance criteria**
- Multiple request records can be retrieved as one task.
- Cumulative totals equal the sum of qualifying records under the same cost-source policy.
- A request cannot be silently reassigned across unauthorized workspaces.

### FR-08 — Project usage and budget warnings

Provide filters and summaries by period, project, model, task, and request status. Allow monthly budgets and configurable warnings (initial defaults 50%, 80%, and 100%). Warnings are monitoring features, not guaranteed hard spending caps.

**Acceptance criteria**
- Dashboard totals reconcile to visible usage records under the selected cost policy.
- Unknown/incomplete costs are shown separately and disclosed in summaries.
- Each threshold crossing produces one deduplicated alert per project/budget period/threshold.
- The budget screen states whether totals are provider-reported, calculated, or mixed.

### FR-09 — Streaming and interrupted responses

Streaming support must capture final usage if OpenRouter returns it for the chosen endpoint/protocol. If a stream terminates before final usage, retain the request and mark usage incomplete/pending. Do not record a failed stream as a known `$0` request merely because usage is absent.

**Acceptance criteria**
- Completed supported streams store one final usage record.
- Duplicate final chunks/reprocessing do not double-count.
- Interrupted streams are labelled incomplete.
- Client cancellation, upstream timeout, and network interruption are covered by tests.

### FR-10 — Reliability and idempotent processing

Assign an internal unique ID to every proxy request. Use unique constraints and idempotent event ingestion for usage and alerts. Keep state transitions explicit for `received`, `forwarding`, `completed`, `failed`, and `unknown_outcome` (or a documented equivalent).

**Acceptance criteria**
- Reprocessing an event does not create a second usage amount.
- Unknown upstream outcomes remain visible for later review; they are not silently retried.
- Local persistence failures are logged safely and recovered where possible.
- Database transaction scopes do not include remote provider waits.

### FR-11 — Privacy and security

- Do not persist raw prompts or completions by default.
- Redact provider keys, gateway keys, and sensitive headers.
- Encrypt provider credentials at rest.
- Enforce project/workspace authorization consistently.
- Apply request size limits, timeouts, rate limiting, and safe exception handling.
- Store only necessary estimation metadata for comparison/calibration.
- Make any later opt-in retention of request content explicit and revocable.

## 8. Proposed data model

The exact schema is a technical design decision, but it must support these concepts:

### `workspaces` and `projects`
Ownership, settings, monthly budget, and optional default output assumptions.

### `provider_credentials`
Provider, encrypted credential material, created/rotated/revoked timestamps, and key status. Never return decrypted credentials in normal API responses.

### `model_catalogue_snapshots`
Source (`openrouter` initially), fetched timestamp, snapshot/version identifier, and sync status.

### `models` / `model_prices`
Exact OpenRouter model ID, display metadata, variants, capabilities, pricing dimensions, decimal price values, units, effective/fetched time, and snapshot reference. Historical snapshots must remain queryable.

### `estimates`
Estimate ID, project ID, requested model/router ID, request fingerprint, estimated input tokens and method, output assumption, low/expected/high costs, currency, confidence, warnings, status, pricing snapshot reference, and timestamp. Do not store raw prompt by default.

### `tasks`
Optional task ID, project ID, label/metadata, planned assumptions where applicable, execution status, and aggregate cost summary.

### `proxy_requests` / `usage_events`
Request ID, linked estimate/task, requested/resolved model IDs, token counts, provider-reported cost, locally calculated cost, status, source, latency, upstream IDs, pricing snapshot, timestamps, and deduplication key. Preserve raw provider response only if necessary and scrubbed; default to normalized fields.

### `budget_alerts`
Project, period, threshold, cost basis, crossing event, status, and sent timestamp.

## 9. API surface (proposed)

- `POST /api/v1/estimates` — estimate supported request cost without executing it.
- `POST /api/v1/chat/completions` — forward a supported OpenRouter chat request through TokenMeter.
- `GET /api/v1/estimates/{estimate_id}` — retrieve an estimate and its assumptions.
- `GET /api/v1/requests/{request_id}` — retrieve request status, usage, costs, and estimate comparison.
- `GET /api/v1/usage` — paginated usage/events filtered by project, date, model, and status.
- `GET /api/v1/projects/{project_id}/summary` — period totals, model breakdowns, and unknown-cost count.
- `GET /api/v1/tasks/{task_id}` — task call list and cumulative actual cost.
- `POST /api/v1/tasks` or request metadata — establish a task grouping (implementation choice).
- `GET/PUT /api/v1/projects/{project_id}/budget` — configure the monthly budget and warning thresholds.

Endpoint paths are proposals to be confirmed in the technical design. Do not implement every endpoint before the first end-to-end request works.

## 10. Non-functional requirements

### Accuracy and transparency
Store the method, assumptions, pricing version, confidence, and exclusions for each estimate. Report accuracy only for records with usable actual-cost data and identify the sample size and metric.

### Performance
Initial target: the preflight estimate endpoint adds less than 100 ms p95 of internal processing under a declared local/load-test condition, excluding UI latency and remote inference. This is a goal to measure, not an achieved claim. Cached catalogue data should be used for estimate requests.

### Correctness
Use decimal-safe calculations. Keep known zero, unknown, pending, and unsupported states distinct. Aggregates must reconcile to underlying events under a documented cost-source rule.

### Reliability
Handle rate limits, upstream errors, timeouts, restarts, interrupted streams, missing usage and local database failures without fabricating costs or double-counting requests.

### Security
Protect credentials and tenant data; do not store raw prompt/response text by default; apply authorization to estimates, tasks, usage, and budget endpoints.

### Observability
Collect structured metrics for estimate count/status/confidence, estimate-vs-actual error, requests and latency, upstream response status, missing usage, price-catalogue freshness, unknown cost, and alert delivery. Telemetry must not include secrets or raw prompts.

### Maintainability
Use a modular monolith with distinct modules for estimation, model catalogue/pricing, OpenRouter adapter/proxy, usage normalization, tasks, budgets, and reporting. Use migrations, type checks, automated tests, and documented ADRs.

## 11. Test strategy and required edge cases

### Estimation tests
- Exact model ID such as `openai/gpt-4o` is preserved.
- `:free` variants are retained and priced using the variant's catalogue data.
- A valid zero-price model produces `$0` with `known_zero` status.
- Unknown model or missing price produces unknown/unsupported, not `$0`.
- Decimal prices and token counts produce expected costs without binary-float drift.
- Input estimation includes all supported messages and tool schemas.
- Unsupported multimodal inputs do not silently become zero cost.
- Low/expected/high results change when the output assumption or model changes.
- Dynamic router estimate is widened or low-confidence when the possible resolved model set is uncertain.
- Estimate calls do not trigger an upstream inference request.

### Execution and accounting tests
- A supported request reaches OpenRouter and its response is relayed.
- Usage reporting is requested where supported.
- Provider-reported zero and locally calculated zero remain valid values.
- Provider-reported and calculated costs can coexist without overwriting each other.
- Missing usage, incomplete stream, and ambiguous failure remain distinct.
- Duplicate event processing does not double-count usage or alerts.
- Pricing refresh does not rewrite prior estimates or cost provenance.
- A completed execution links to the correct estimate and task.

### Security/reliability tests
- Tenant/workspace access checks are enforced on every read/write endpoint.
- Keys and sensitive headers are redacted.
- Upstream 401/429/5xx responses, timeouts, connection resets, and client disconnects are handled.
- Ambiguous upstream execution is not automatically retried without a safe idempotency guarantee.
- Request-size and rate limits work as configured.
- Real PostgreSQL integration tests cover transaction and deduplication behavior.

Use deterministic mock fixtures for normal CI. Keep an optional, explicitly labelled smoke test for a real OpenRouter key; CI must not require paid external API calls.

## 12. Implementation roadmap

### Phase 0 — Freeze product semantics
Write ADRs and tests for estimate ranges, cost statuses, request/task boundary, supported request surface, zero vs. unknown, and dynamic routing behavior.

**Exit gate:** product can explain what the estimate means and what it excludes.

### Phase 1 — Catalogue and deterministic estimate engine
Sync OpenRouter models/prices into snapshots. Implement Decimal-safe pricing, exact model IDs, input-token estimator with method/confidence, explicit output assumptions, and unit tests.

**Exit gate:** given a fixed request fixture and price snapshot, the estimate response is deterministic, explainable, and properly handles paid/zero/unknown pricing.

### Phase 2 — Estimate UI/API vertical slice
Build `POST /estimates` and a small playground that accepts a prompt, model and output assumption. Return range, confidence, assumptions and exclusions without calling the upstream model.

**Exit gate:** a developer can preview costs and compare model/output choices without inference spend.

### Phase 3 — OpenRouter proxy and actual usage
Proxy the same supported request through OpenRouter, capture usage/report cost, persist it and link it to the estimate.

**Exit gate:** at least one successful real or mocked end-to-end request shows estimate, actual usage, and the difference. Use a real-key test only as an optional smoke test.

### Phase 4 — Request history, task grouping and budget alerts
Add usage reports, task IDs, cumulative totals, project budgets, threshold-crossing alerts, and explicit cost-source labels.

**Exit gate:** totals reconcile against test events, duplicate events do not double-count, and threshold alerts are deduplicated.

### Phase 5 — Streaming and failure hardening
Implement supported streaming behavior, final usage capture, interruption handling, retries only where safe, and recovery for incomplete records.

**Exit gate:** failure matrix is covered by automated tests and documented limitations.

### Phase 6 — Bounded task estimation and budget guard
Allow users to describe a bounded plan with known steps/models/call counts and per-step output assumptions. Compute low/expected/high cumulative estimates. Design atomic reservations and safe behavior for uncertain provider outcomes before enforcing hard limits.

**Exit gate:** task-level estimates show assumptions clearly, and enforcement tests show no oversubscription under concurrent requests.

### Phase 7 — Calibration and extension
Use opt-in, privacy-preserving historical features to evaluate whether output/task estimates improve. Add additional provider adapters only after the OpenRouter implementation is reliable.

**Exit gate:** calibration metrics are computed from a held-out sample, with zero-cost and unknown-cost cases reported separately.

## 13. Definition of done for the MVP

A developer can create a project, configure OpenRouter, enter a supported request, see a pre-run estimate and its assumptions without incurring inference cost, choose to execute that request through TokenMeter, receive the upstream response, view captured usage and cost, compare actual against estimated, and receive a budget warning when a configured threshold is crossed.

The repository includes tests for paid, free/zero, unknown, dynamic routing, duplicates, missing usage, security, and provider failures; setup instructions; API docs; a working deployment or repeatable local run; and measured benchmark results. The project must document that request-level estimates do not guarantee the cost of an unbounded multi-step agent task.

## 14. Official references

- OpenRouter model catalogue: https://openrouter.ai/docs/api/api-reference/models/get-models
- OpenRouter model details: https://openrouter.ai/docs/api/api-reference/models/get-model
- OpenRouter usage reporting: https://openrouter.ai/support/
- OpenRouter pricing and billing: https://openrouter.ai/pricing
