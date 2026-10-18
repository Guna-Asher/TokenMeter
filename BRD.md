# TokenMeter — Business Requirements Document (BRD)

**Version:** 0.3  
**Status:** Refined draft  
**Product category:** Pre-run AI cost estimation and task-level spending control  
**First upstream integration:** OpenRouter  
**Primary audience:** Developers and small teams building with AI APIs

## 1. Executive summary

TokenMeter helps developers understand the likely cost of an AI API request or a bounded AI task **before they run it**. After execution, TokenMeter records available actual usage and cost, compares the result with the original estimate, and helps developers make better estimates over time.

The product is not primarily a retrospective analytics dashboard. Its central workflow is **estimate → decide → execute → compare → improve**.

OpenRouter is the first integration because it provides one API and a model catalogue covering many models, including paid models, free variants, and routing options. The initial release will focus on supported text-based requests routed through TokenMeter. Additional providers and coding-platform integrations are outside the initial scope.

## 2. Business problem

Developers can incur AI API costs while prototyping, debugging, asking for code changes, or running multi-step AI workflows. Most usage details become clear only after a request completes. Before running, developers may not know how much context is being sent, how much output may be generated, whether a model is free or paid, or how repeated model calls can multiply cost.

A simple token-price calculator is not enough for many workflows. It may omit system prompts, tool definitions, conversation history, output-length uncertainty, retries, repeated agent calls, and dynamic model routing. Conversely, no estimator can precisely predict an arbitrary model response before it has been generated.

TokenMeter addresses this gap by producing a transparent cost range before execution, identifying the assumptions behind it, recording what actually happened, and helping users limit avoidable overspend.

## 3. Product vision

**Before you run an AI request or task, know its likely cost and what could make it more expensive.**

TokenMeter should become a trusted preflight layer for developers using AI APIs: estimate cost before execution, observe actual cost afterward, and manage the cumulative cost of bounded tasks and workflows.

## 4. Target users and primary jobs

### Primary users

- Independent developers building AI-powered applications.
- Developers experimenting with multiple models and prompt designs.
- Small engineering teams working across several AI projects.
- AI-first startups that need lightweight cost visibility and spending controls.

### User jobs

1. Estimate the likely cost of a request before sending it to a model.
2. Compare model choices for the same request.
3. Understand the assumptions and uncertainty behind an estimate.
4. See whether the last request cost more or less than predicted.
5. Track several API calls as one task and estimate a bounded workflow before starting it.
6. Receive warnings before project spending exceeds a chosen budget.
7. Avoid accidentally spending more than intended during experiments.

## 5. Value proposition

TokenMeter provides:

- **Pre-run estimate:** low, expected, and high cost estimates where inputs permit a meaningful range.
- **Transparent assumptions:** estimated input tokens, assumed output length, selected model, catalogue version, and any exclusions.
- **Actual-versus-estimated comparison:** record usage after execution and report prediction error.
- **Task grouping:** attribute multiple observed API calls to one developer-defined task.
- **Budget awareness:** alert users when recorded spending or a planned estimate approaches a budget.
- **Provider portability over time:** keep a provider-neutral usage and pricing domain, starting with OpenRouter.

## 6. Business goals and success measures

The project should demonstrate useful, honest pre-run estimates and reliable post-run accounting. Metrics must be measured from real tests and opt-in usage data; they must not be presented as achieved before evidence exists.

### Product metrics

- **Estimate coverage:** percentage of supported requests for which TokenMeter can return an estimate instead of `unknown`.
- **Estimate range coverage:** percentage of completed requests whose actual cost falls inside the displayed estimate range.
- **Expected-estimate error:** mean absolute error between expected and actual costs, reported alongside sample size and model/request category.
- **Pre-run adoption:** percentage of tracked executions that have a stored estimate linked to them.
- **Task attribution:** percentage of observed requests assigned to a project and, when used, a task.
- **Budget alert correctness:** percentage of tested threshold crossings that create exactly one correct alert.
- **Gateway reliability and latency:** measured using documented test conditions.

For zero-cost requests, percentage error is undefined or misleading; use absolute error and separate free-model results from paid-model accuracy metrics. For unknown or incomplete costs, do not silently treat the value as zero.

### Initial engineering success criteria

- A developer can estimate and then execute a supported OpenRouter request.
- An estimate is stored with the model, price snapshot, output assumptions, method, and confidence.
- Actual usage is linked to the estimate when the same execution proceeds through TokenMeter.
- Paid cost, known zero cost, and unknown cost remain distinct in storage and UI.
- Duplicate event processing does not double-count actual usage or budget totals.
- Test and benchmark evidence is included in the repository.

## 7. Scope and release boundaries

### First usable release (MVP)

- Workspaces and projects.
- OpenRouter as the only upstream provider.
- A model catalogue sync with versioned pricing snapshots.
- Pre-run estimates for supported text-based chat requests.
- An estimate response that includes low/expected/high cost where supportable, token estimates, cost assumptions, confidence, and warnings.
- A simple web playground where a user selects a model, enters a prompt/request, sees the estimate, and can choose to run it.
- An OpenRouter-compatible gateway for the supported request surface.
- Post-run usage capture, OpenRouter-reported cost capture when available, and catalogue-based calculation when possible.
- Actual-versus-estimated comparison.
- Project-level usage history and monthly budget warnings.
- Task identifiers that can group multiple API calls, initially with basic cumulative-cost reporting.
- Automated tests for pricing, zero-cost models, missing information, duplicate events, and common provider failures.

### Follow-on release

- Bounded multi-call task estimates where users supply expected call counts, models, and token/output budgets per step.
- Task-level budget enforcement using reservations and explicit handling for uncertain outcomes.
- Better model-specific tokenization and estimates calibrated from historical usage.
- Forecasting across recent usage and task categories.
- Streaming and more advanced tool-call workflows if not completed in the MVP.
- Reconciliation with additional OpenRouter account-level reports where supported.

### Out of scope for the initial release

- Integrations with Emergent, Lovable, Bolt, or other AI app-builder credit systems.
- Claims that TokenMeter can estimate hidden calls made outside its gateway/SDK.
- Exact cost prediction for open-ended coding agents or workflows whose number of model calls, tool calls, retries, or generated tokens is unknown.
- Direct integration with all providers besides OpenRouter.
- Full accounting for all account-level charges, credit purchase fees, taxes, or adjustments; inference cost must be labelled separately.
- Image, audio, video, embedding, web-search, or other non-text cost dimensions unless explicitly supported and tested. The data model should allow future dimensions.
- Advanced autonomous model selection or automatic prompt rewriting.
- A claim of production readiness before testing, security review, and operating evidence.

## 8. High-level user workflow

1. The user creates a workspace and project.
2. The user configures an OpenRouter credential securely.
3. The user enters or instruments a supported request and selects a model or supported router.
4. TokenMeter estimates the input size, uses known catalogue prices, applies the stated output assumption, and returns a cost range with caveats.
5. The user decides whether to run the request, change the model, reduce the output limit, or cancel.
6. If executed through TokenMeter, the gateway forwards the request and returns the provider response.
7. TokenMeter records usage and cost information returned by OpenRouter, plus its own calculated cost if available.
8. TokenMeter compares actual cost with the estimate and updates project/task totals.
9. Budget notifications fire when configured thresholds are crossed.

## 9. Key assumptions and product truths

- TokenMeter can automatically observe only requests routed through the gateway or explicitly instrumented through a supported SDK/API.
- A pre-run estimate is a prediction, not a guaranteed price, unless it is a mathematically justified upper bound under explicit constraints.
- Input token counts may be exact only when a compatible tokenizer and complete request payload are available; otherwise they are estimates and must be labelled accordingly.
- Output token count is not known before generation. It must be supplied as a limit/assumption or estimated as a range using a documented method.
- A free model can have a valid `$0` inference price. `$0` is not the same as unknown price, missing usage, or incomplete execution.
- Dynamic routing may resolve to a different model than the requested router. The pre-run estimate must represent the possible routing uncertainty, and the post-run record must retain the actual model when returned.
- Provider-reported cost and TokenMeter-calculated cost should be stored separately when both are available.
- Pricing and model availability change, so each estimate and calculation must retain price-catalogue provenance.
- TokenMeter's inference-cost view may differ from OpenRouter account-level balance changes due to charges or account adjustments outside per-request inference cost.

## 10. Risks and mitigations

| Risk | Business impact | Mitigation |
|---|---|---|
| Estimates appear more precise than reality | Users lose trust | Show ranges, assumptions, confidence, and exclusions; evaluate against actual usage. |
| Dynamic routing changes the serving model | Estimate may be misleading | Display route vs. resolved model; widen range or return low-confidence/unknown when candidates cannot be bounded. |
| Prompt/token estimate is inaccurate | Cost range misses actual cost | Use compatible tokenizers where available; otherwise identify heuristic estimation and lower confidence. |
| Output, retries, or tool calls exceed assumptions | Actual cost exceeds estimate | Show assumptions; use explicit output/call limits; later implement task budget guard. |
| Stale or missing prices | Incorrect estimate | Version catalogue snapshots, refresh regularly, and never replace unknown pricing with zero. |
| Stream interruption or network failure | Actual usage uncertain | Mark incomplete/unknown and reconcile when possible; do not fabricate cost. |
| Gateway stores sensitive request content | Privacy/security exposure | Do not persist raw prompts or responses by default; redact credentials and sensitive headers. |
| Scope grows into a general observability platform | Delayed core value | Prioritize estimate → execution → comparison before expanding provider coverage or dashboards. |

## 11. Product positioning

TokenMeter is not merely a historical spend dashboard or a generic per-token calculator. Its intended focus is the combination of:

1. a pre-run cost estimate with transparent assumptions;
2. an execution path that links the estimate to actual usage;
3. learning from estimate-versus-actual differences; and
4. task-level budgeting as task grouping and measurement mature.

The MVP must prove that the pre-run estimate is useful and that it is reliably connected to what actually happened.

## 12. Initial source references

- OpenRouter model catalogue: https://openrouter.ai/docs/api/api-reference/models/get-models
- OpenRouter model details and pricing fields: https://openrouter.ai/docs/api/api-reference/models/get-model
- OpenRouter usage reporting: https://openrouter.ai/support/
- OpenRouter pricing and billing distinctions: https://openrouter.ai/pricing
