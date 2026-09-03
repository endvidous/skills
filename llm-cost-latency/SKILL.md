---
name: llm-cost-latency
description: Use when optimizing LLM API cost or latency, choosing a model tier, setting up prompt caching, batching, streaming, or model routing — or when someone says "the AI feature is too slow" or "the LLM bill is too high".
---

# LLM Cost & Latency

Two rules govern everything here. First: **measure before optimizing** — most teams guess wrong about whether input tokens, output tokens, or model choice dominates their bill. Second: **quality is the product** — work the levers that preserve quality (caching, batching, routing) before the one that trades it away (downgrading the model).

## Decision 0: Know your token economics

- **Output tokens cost ~5× input tokens** on typical current pricing (e.g. Claude tiers run $1/$5, $3/$15, $5/$25, $10/$50 per Mtok input/output). A pipeline that's 90% input-heavy and one that's output-heavy need opposite optimizations — check `usage` data before touching anything.
- **Latency splits the same way:** input is processed in parallel (cheap in time), output is generated token-by-token (expensive in time). Total generation time ≈ output length. The single most effective latency fix is usually *shorter outputs* — a tighter `max_tokens` plus a terseness instruction.
- Count tokens with the provider's counting endpoint against the actual model — never a mismatched tokenizer (e.g. tiktoken for non-OpenAI models undercounts 15–20%+).
- Model tiers are roughly order-of-magnitude steps; verify current prices against provider docs rather than memory — pricing changes faster than any document.

## The lever ladder (work top-down)

1. **Prompt caching** — up to ~90% off repeated context, no quality change (Decision 1)
2. **Batch API** — flat 50% off anything that can wait an hour (Decision 2)
3. **Context hygiene** — stop paying for tokens that don't help (Decision 3)
4. **Output discipline & reasoning budget** — shorter outputs, calibrated thinking effort (Decision 4)
5. **Routing / cascades** — cheaper models where they're sufficient, *proven by evals* (Decision 5)
6. **Response caching** — exact-match first; semantic caching only with eyes open (Decision 6)

Downgrading every request to a cheaper model is the *last* lever, not the first — it's the only one that spends quality.

## Decision 1: Prompt caching

Provider-side prompt caching is a **prefix match**: cache reads cost ~0.1× the base input price, writes ~1.25× (short TTL) to 2× (long TTL); with a 5-minute-TTL write, the second request already breaks even. The discipline that makes it work:

- **Order by stability:** static system prompt and tool definitions first → per-session context → per-turn content last. Any byte change invalidates everything after it.
- **Hunt silent invalidators:** a timestamp or UUID interpolated into the system prompt, unsorted JSON serialization, a per-user tool list — each one silently zeroes your hit rate. Verify with the usage fields (`cache_read_input_tokens` > 0), don't assume.
- **Never swap tools or model mid-conversation** — both rebuild the cache from position zero.
- Multi-turn agents are the killer use case: each turn re-reads the whole history at ~0.1× instead of full price.
- Caching also cuts *latency* — cached prefix tokens skip prefill processing.

## Decision 2: Batch vs real-time

Ask one question per pipeline: **does a human wait on this response?** If not — nightly enrichment, bulk classification, eval runs, report generation, backfills — the batch endpoint gives a flat **50% discount** with typically-under-an-hour turnaround. This is the highest-value/lowest-effort lever most teams skip. Mechanics: results return unordered (key by `custom_id`), and batches compose with prompt caching.

## Decision 3: Context hygiene

- **Unbounded conversation history is a cost leak** — every turn resends everything. Prune stale tool results (context editing), compact old history, cap window growth. See `llm-app-architecture` Decision 7.
- **RAG top-k discipline:** retrieving 40 chunks when reranked-10 scores the same is pure waste — tune k against your eval set, not superstition.
- **Filter tool results in code** before they re-enter the context; the model shouldn't re-read a 50KB API response to use three fields of it.
- Don't over-trim either: cutting context that evals show the model needs is a quality regression sold as an optimization.

## Decision 4: Output discipline and reasoning budget

- Set `max_tokens` deliberately per route (classification ≠ report writing); prompt for the output length you actually want — verbose outputs cost money *and* time.
- **Reasoning depth is a first-class cost lever.** Modern models expose thinking/effort controls (e.g. effort levels low→max, adaptive thinking): high effort for correctness-critical or agentic work, low for routine extraction and chat. Sweep effort levels against your evals — on strong models, low effort often matches the previous generation's maximum. Counter-intuitive but measured: *higher* effort up front can reduce an agent's total cost by cutting turn count — measure end-to-end, not per-request.
- For long agent runs, use task-level token budgets where the platform supports them, so the model paces itself instead of being truncated.

## Decision 5: Model routing and cascades

**Options, simplest first:**
- **One capable model everywhere (default at small scale)** — engineering time is more expensive than tokens until volume says otherwise.
- **Static per-route assignment (default at scale)** — each pipeline/endpoint gets the cheapest tier that passes its eval suite: cheap model for classification and extraction, mid-tier for user chat, top tier for agentic/complex work. Trivial to implement and reason about; this captures most routing value.
- **Learned routing / cascades** — a router picks the model per query, or a cheap model answers and escalates on low confidence. The research is real — cascades matched top-model quality at up to 98% lower cost (FrugalGPT); learned routers cut cost ~2× at 95% quality (RouteLLM) — but you inherit a router to train/maintain, added latency on escalation, and quality that varies per query. Adopt only at volumes where the savings pay for that machinery.

Hard rule: **a downgrade is only a downgrade if evals prove parity.** "Seems fine" on a cheaper model is how quality quietly erodes — route by measurement (see `llm-evals`).

## Decision 6: Response caching

- **Exact/normalized-match caching (safe default):** hash the full prompt (or normalized query + context version); serve identical repeats for free. Works for FAQ-shaped traffic, repeated classifications, idempotent extraction. Invalidate on prompt-version or corpus change.
- **Semantic caching (dangerous — treat as last resort):** serving a cached answer because the *embedding* is similar returns confidently wrong answers on near-miss queries ("cancel my order" vs "cancel my subscription"). If forced: high threshold tuned on a labeled set, cache only self-contained answers (never account- or time-dependent ones), and log cache hits for human sampling.

## Latency playbook (user-facing)

1. **Stream everything user-facing.** Time-to-first-token is the perceived latency; a 20-second response that starts in 400ms feels fine. Also required practically: long non-streaming generations hit HTTP timeouts.
2. **Cut output length** (Decision 4) — total time tracks output tokens.
3. **Prompt-cache the prefix** (Decision 1) — cuts prefill time on long contexts.
4. **Parallelize independent calls** — fan out sub-tasks concurrently instead of sequentially; move sequential tool-call chains into code where possible.
5. **Route latency-critical paths to smaller/faster models** — with eval parity (Decision 5).
6. **Hide latency in UX:** optimistic UI, progress states, speculative prefetch on predictable next actions.

Watch p95/p99, not means — LLM latency is long-tailed, and retry storms on timeout make tails worse (set timeouts + retry budgets per `distributed-reliability`).

## Review checklist

- Usage data examined — is input, output, or model choice the actual cost driver?
- Prompt ordered stable→volatile; cache hit rate verified nonzero; no timestamps/UUIDs in the prefix?
- Anything not human-waited moved to batch (50%)?
- History pruned/compacted; RAG k tuned; tool outputs filtered?
- `max_tokens` and effort/thinking set per route, not global defaults?
- Model downgrades backed by eval parity?
- User-facing paths streamed; tails (p95+) monitored?

## Anti-patterns

- Downgrading the model as the first move, unmeasured
- A timestamp in the system prompt (silent 100% cache miss)
- Rebuilding the tool list per request / per user
- Unbounded chat history resent forever
- Optimizing input cost when output tokens dominate the bill (or vice versa)
- Semantic cache serving stale or wrong-account answers
- Non-streamed 30-second user-facing generations
- Latency-driven max-retries-everywhere (see `distributed-reliability` — retries amplify)
