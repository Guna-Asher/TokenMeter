# TokenMeter — Business Requirements Document (BRD)

**Version:** 0.2  
**Status:** Draft  
**Product:** TokenMeter  
**Category:** AI API usage tracking and cost management  
**Primary upstream integration:** OpenRouter

## 1. Business problem

Developers use many AI models through API-based applications, and understanding usage across projects can be difficult. They need to know which project or model consumes the most, what individual requests cost, and whether their spending is likely to exceed a budget.

TokenMeter will provide a project-level view of AI API usage and estimated or provider-reported inference cost. OpenRouter is the first upstream integration because it exposes a unified API across many models, including paid models, free variants, and routing options.

## 2. Business objective

Build a developer-focused service that:

- tracks supported AI requests routed through TokenMeter;
- captures token usage returned by OpenRouter;
- calculates cost using OpenRouter's model catalogue when possible;
- preserves OpenRouter-reported request cost when provided;
- groups usage by workspace, project, model, and time period;
- alerts developers when recorded spending crosses configurable thresholds; and
- lays out a clean path for adding other upstream providers later.

## 3. Target users

- Independent developers building AI applications.
- Small engineering teams developing AI-powered products.
- AI-first startups managing usage across projects.
- Developers experimenting with paid and free models.

## 4. Value proposition

TokenMeter should help developers answer:

1. How much AI inference cost has each project incurred?
2. Which requested and actual models were used?
3. How many input and output tokens did requests consume?
4. Which requests or models account for the largest costs?
5. Have I crossed a budget threshold, and what might I spend by month end?
6. Was a request free, paid, or not yet costed because usage/pricing data was unavailable?

## 5. Business goals and success measures

The initial release aims to:

- route supported OpenRouter chat-completion requests and return the upstream result;
- record usage for successful responses when usage data is available;
- avoid double-counting when a usage event is processed more than once;
- represent zero-cost usage as a valid numeric value, not as missing data;
- show model/project spending, token counts, status and usage provenance;
- trigger budget alerts once per relevant threshold crossing; and
- document actual reliability and latency through test results.

Success must be demonstrated with automated tests and a working deployment. Performance or accuracy claims must not be made before measurement.

## 6. Product scope

### Initial release

- Workspaces and projects.
- OpenRouter as the first upstream provider.
- OpenRouter-compatible chat-completion proxy endpoint.
- Model catalogue synchronization from OpenRouter's models API.
- Capture of input/output token usage and other supported usage details.
- Cost calculation for supported text-token pricing dimensions.
- Preservation of OpenRouter-reported request cost when returned.
- Paid models, free model variants, and zero-cost routing/models.
- Usage history and project/model breakdowns.
- Monthly project budgets, warning thresholds, and a basic usage-based forecast.
- Secure key handling, structured logs, and automated tests.

### Out of scope for the first release

- Integrations with Emergent, Lovable, Bolt, or their platform-specific credits.
- Direct support for every provider outside OpenRouter.
- Full cost accounting for image, audio, video, tool execution, or other non-token-priced dimensions; data structures should remain extensible for them.
- Importing and reconciling all OpenRouter account billing/activity data.
- Subscription billing for TokenMeter itself.
- Guaranteed pre-request hard spending caps. The first release offers tracking and warnings; reliable enforced caps require a separate reservation/admission-control design.
- Advanced automated model selection or prompt optimization.

## 7. Business assumptions and constraints

- Automatic request-level tracking covers only traffic routed through TokenMeter. Requests that bypass it will not be observed by the proxy.
- The first version uses a user's OpenRouter API key to call OpenRouter.
- OpenRouter model identifiers are structured strings, for example `openai/gpt-4o`, and may include variants such as `:free`; the exact identifier must be retained.
- Dynamic routers such as `openrouter/free` may resolve to a concrete model. TokenMeter must retain both the model requested by the client and the actual model reported by OpenRouter when available.
- OpenRouter can return usage information, including cost, when usage inclusion is requested. TokenMeter should treat this as provider-reported data and separately retain its own catalogue-based calculation when it can compute one.
- A zero cost can be a valid result for a free model. Zero must never be confused with unknown or missing cost.
- Model pricing and availability can change. The pricing catalogue must be versioned or time-stamped so historical calculations are explainable.
- TokenMeter tracks inference cost, not necessarily every account-level charge such as credit-purchase fees, account adjustments, or all billing effects. These categories must not be silently mixed.

## 8. Business risks

- Incorrect or stale catalogue pricing can lead to incorrect estimates.
- Usage data may be absent if a response is interrupted or the upstream does not return final usage.
- Dynamic routing can make the resolved model differ from the requested model.
- Streaming responses can terminate before the final usage event is received.
- Free models may have rate limits or availability constraints that can change over time.
- A proxy adds latency and becomes a service that must be operated reliably.
- Provider credentials and request metadata must be protected.
- A warning feature does not itself stop further upstream spending.

## 9. References

- OpenRouter model catalogue API: https://openrouter.ai/docs/api/api-reference/models/get-models
- OpenRouter support — usage reporting and `usage: { "include": true }`: https://openrouter.ai/support/
- OpenRouter usage accounting announcement: https://openrouter.ai/blog/announcements/smarter-charts-inline-svgs-and-live-usage-accounting/
- OpenRouter free models: https://openrouter.ai/collections/free-models
