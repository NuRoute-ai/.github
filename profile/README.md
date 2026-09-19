# NuRoute

**A provider-agnostic LLM gateway. Route every request to the cheapest model that can still answer it well.**

Most teams building on LLMs are over-provisioned: they picked a frontier model once, pointed every request at it, and never went back. That's a rational choice, since you can't tell in advance which requests need the expensive model. So you pay frontier prices for all of them, including the majority a far cheaper model would answer just as well.

NuRoute's auto routing removes that trade-off. For every prompt it predicts the cheapest class of model that can still answer it well, and sends it there. Hard prompts still reach a frontier model. Easy ones stop costing like they do.

## How a routing decision gets made

Every request goes through two separate stages, kept apart on purpose:

```
   your prompt
        |
        v
  1. Tier prediction              "How capable does the
     prompt -> small | medium | large    model need to be?"
        |
        v
  2. Model selection              "Given that floor, which
     governance filter -> strategy -> winner   model wins under your policy?"
        |
        v
   provider call
        |
        v
   response + X-Router-* headers
   + auditable decision record
```

Tier prediction is a property of the prompt. Model selection is a property of your organization: enabled providers, regions, budget ceilings, strategy. Splitting them means your compliance rules never get baked into a model, and the model never needs retraining just because you added a provider.

### Stage 1: predicting the tier

The router scores a prompt across all three tiers and routes to whichever scores highest. There's no confidence threshold layered on top, because the cost and quality trade-off was already priced in at training time. A threshold on top would just override a trained judgment with a second, separately tuned opinion about the same question.

It's a linear classifier over hand-crafted features read straight off the request text, measuring size, structure, composition, and phrasing. No embeddings, no LLM call, no second model inference anywhere in the decision. That keeps routing overhead sub-millisecond: 0.4ms median, 0.9ms at p95, in-process, on a call that otherwise takes seconds. Nothing is billed for it and no tokens are consumed.

On the held-out evaluation set, the tier mix comes out to 21% small, 17% medium, 62% large. That still lands at 41% of always-frontier spend while retaining 95% of its quality, because the requests routed elsewhere are the cheapest ones to serve and the frontier tier is priced far above the others. Your own mix will differ. It's a property of your traffic, not a setting.

Two behaviors worth knowing:
- Degenerate input (empty, whitespace-only, non-text prompts) never fails a request. It short-circuits to the cheapest tier and returns a normal decision.
- A router outage degrades to your configured strategy, not to an error. The tier call carries a 2-second deadline; if it times out, the request proceeds without a predicted tier and your strategy selects from the full catalog. Routing never fails a request just because the classifier is unavailable.

### Stage 2: choosing the model

Once a tier is fixed, NuRoute expands it into every model/provider pair you could legitimately be served by, then narrows and ranks:

1. **A hard governance filter runs first**, and applies identically whether the tier came from the router or you named a model yourself. Candidates get excluded for a disabled or unhealthy provider, a provider outside your permitted regions, a missing capability, or a breach of your latency or cost ceilings. Every exclusion is recorded with its reason.
2. **Your strategy ranks what survives.** `cheapest`, `fastest`, and `highest_quality` rank on a single axis. `balanced` (the default) normalizes cost, latency, and quality across the survivors and combines them under weights you control. Your preferred-provider order applies as a bounded nudge on top, so a preference can break a close call but never override a large difference.

Models too close to distinguish are treated as tied and separated on cost instead, which is why `highest_quality` often returns the same model as `cheapest` within a tier.

Latency estimates start from benchmarks, then get replaced by your own observed per-token latency once a model has enough real traffic on your account, so routing reflects your actual regions, prompt shapes, and provider load, not a generic number.

## When things go wrong

| Situation | What NuRoute does |
|---|---|
| No model in the predicted tier is eligible | Escalates upward to a stronger tier, never down. Response carries `X-Router-Tier-Escalated`. |
| Nothing is eligible at any tier | Returns a clear `no_available_model` error rather than silently serving something you excluded. |
| Your policy excludes every provider | Applies your configured `fallbackBehavior`, with the relaxation recorded on the decision. |
| A provider rate-limits, times out, or goes down | Excludes that provider and re-routes to the next candidate in the same tier, up to three attempts (auto mode). |
| The tier router itself is unavailable | Falls back to strategy routing over the full catalog. The request is served, not failed. |

Routing never silently downgrades quality to save money, and a router outage never fails your request outright.

## Everything is auditable

Every response carries its routing decision back to you in headers: `X-Router-Tier`, `X-Router-Confidence`, `X-Router-Model`, `X-Router-Estimated-Saving`, and `X-Router-Tier-Escalated` when relevant. Every decision is also persisted with its full candidate set, exclusion reasons, the strategy applied, and the resulting cost, retrievable through routing history or the dashboard's request explorer.

## Quick start

```python
from nuroute import NuRouteClient

client = NuRouteClient(
    api_key="aicp-...",
    base_url="https://your-nuroute-gateway",
)

response = client.chat.complete(
    model="auto",  # predicts the tier and routes it
    messages=[{"role": "user", "content": "Hello!"}],
)

print(response["choices"][0]["message"]["content"])
```

Prefer a specific model? Pin it directly:

```python
response = client.chat.complete(
    model="claude-sonnet-4-6",
    provider="anthropic",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

## SDKs and integrations

| | |
|---|---|
| [`sdk-python`](https://github.com/NuRoute-ai/sdk-python) | Official Python SDK |
| [`sdk-js`](https://github.com/NuRoute-ai/sdk-js) | Official TypeScript / JavaScript SDK |
| [`n8n-nodes-nuroute`](https://github.com/NuRoute-ai/n8n-nodes-nuroute) | n8n community node |
| [`dify-nuroute`](https://github.com/NuRoute-ai/dify-nuroute) | Dify model provider plugin |

NuRoute also exposes an OpenAI-compatible `/v1/chat/completions` endpoint, so most existing OpenAI SDKs and tools work by pointing them at your gateway URL.

## Streaming, tools, and multi-turn

- **Streaming** is supported. Routing resolves before the stream opens, so a routing failure returns a normal JSON error instead of a half-written stream.
- **Multi-turn** conversations are read for recent context, not just the latest message, and the tier is decided fresh per request, so a conversation can move between tiers as it gets harder or easier.
- **Tool calling** is supported; requests carrying tools route only to providers that declare function calling. That check is at the provider level, not per-model reliability, so for quality-sensitive tool traffic, naming a model explicitly is the safer choice.
- **Structured output** (`response_format`) isn't honored yet; it's ignored. For reliable JSON today, name a model you know supports it and instruct it in the prompt.

## Why NuRoute

- **A trained classifier, not a rule engine.** Labels come from priced, graded outcomes across tiers, weighted by what a wrong decision actually costs, not from difficulty heuristics.
- **Full auditability.** Every routing decision is traceable, with exclusion reasons and the full candidate set retrievable after the fact.
- **Never fails silently.** Escalation only ever goes up in quality, and a router outage degrades gracefully instead of failing requests.
- **Drop-in adoption.** OpenAI-compatible, so you can start with `model="auto"` on non-critical traffic and pin specific models wherever you need certainty.

## Docs and links

- Documentation: [nuroute.ai/docs](https://nuroute.ai/docs)
- Routing performance: [nuroute.ai/docs/concepts/routing-performance](https://nuroute.ai/docs/concepts/routing-performance)
- Website: [nuroute.ai](https://nuroute.ai)
- Support: [support@nuroute.ai](mailto:support@nuroute.ai)

## Get involved

Issues and pull requests are welcome on any of the repos above. If you're evaluating NuRoute for production traffic and have questions about routing behavior, governance, or a specific integration, reach out at [support@nuroute.ai](mailto:support@nuroute.ai).
