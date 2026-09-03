---
name: llm-app-architecture
description: Use when designing or reviewing any LLM-powered feature or system — chatbots, RAG pipelines, agents, tool use, multi-agent systems, summarization/extraction/classification pipelines, or AI workflow automation. Forces the single-call vs workflow vs agent decision, RAG vs long-context vs fine-tuning choice, retrieval design, tool-surface design, context management for long-running agents, and prompt-injection trust boundaries. Provider-agnostic; fires whenever "add AI/LLM" appears in a task.
---

# LLM Application Architecture

The governing rule, from Anthropic's [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents): **find the simplest solution possible, and only increase complexity when demonstrably needed**. Most "agent" requests are a single prompt; most "multi-agent" requests are a workflow. Every tier you add costs latency, money, and debuggability.

## Decision 1: Single call vs workflow vs agent

- **Single LLM call (default)** — classification, extraction, summarization, Q&A, rewriting. One request, one response, maybe a schema-constrained output. If you can write the steps down, you probably don't need more than this plus code.
- **Workflow** — LLM calls orchestrated through *predefined code paths*. Predictable, testable, debuggable. Choose when the task decomposes into known steps.
- **Agent** — the model dynamically directs its own process and tool use. Choose only when the path genuinely can't be predicted in advance.

Before building an agent, all four must hold (Anthropic's agent criteria):
1. **Complexity** — multi-step and hard to fully specify in advance ("turn this design doc into a PR", not "extract the title").
2. **Value** — the outcome justifies the extra cost and latency.
3. **Viability** — the model is actually capable at this task type.
4. **Cost of error** — errors can be caught and recovered (tests, review, rollback).

Any "no" → drop a tier.

## Decision 2: Which workflow pattern

From [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — pick by task shape, not preference:

| Pattern | Shape of the task | Cost |
|---|---|---|
| **Prompt chaining** | Decomposes into fixed sequential steps; each step's output feeds the next; you can add programmatic checks between steps | Latency × steps |
| **Routing** | Distinct input categories better handled by specialized prompts/models | One extra classification call |
| **Parallelization** | Independent subtasks, or multiple attempts you'll vote/merge over | Tokens × branches |
| **Orchestrator–workers** | Subtasks can't be predicted upfront but a central call can enumerate them | Highest; hardest to debug |
| **Evaluator–optimizer** | Clear evaluation criteria exist and iteration measurably helps (e.g. writing, code that must pass checks) | 2–N× per output |

Default: chaining or routing. Orchestrator–workers is the last workflow stop before a true agent.

## Decision 3: Grounding the model in your knowledge

**Options:** stuff it in the context vs RAG vs fine-tuning.

- **Long context / prompt stuffing (default when the corpus fits)** — simplest, no infra, and with prompt caching the repeated-context cost drops ~90% (see `llm-cost-latency`). Caveats: cost scales per request with corpus size, and retrieval quality inside long contexts is position-dependent — models retrieve worst from the *middle* of long contexts ([Lost in the Middle, Liu et al., TACL 2024](https://aclanthology.org/2024.tacl-1.9.pdf)); put critical material at the start or end.
- **RAG** — when the corpus is too large, changes frequently, or needs per-user permissioning. You take on a retrieval-quality problem: the model can't cite what retrieval didn't fetch. See Decision 4.
- **Fine-tuning** — for *style, format, and task behavior*, not knowledge injection. Facts in weights go stale and can't be permissioned or audited; fine-tuning also loses to RAG on freshness. Choose it when few-shot prompting can't hit the required output distribution and you have hundreds+ of quality examples.

These compose: RAG for facts + prompting/fine-tune for behavior is normal.

## Decision 4: Retrieval design (when RAG won)

Naive vector-only RAG is the baseline everyone outgrows. The measured upgrade path ([Anthropic: Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)):

1. **Hybrid search** — embeddings *and* BM25, rank-fused. Embeddings miss exact identifiers (error codes, SKUs, function names); BM25 misses paraphrase. Together they cover each other.
2. **Contextual chunks** — prepend a short generated summary situating each chunk in its document before embedding/indexing; contextual embeddings alone cut retrieval failures by 35%.
3. **Rerank** — retrieve wide (top-150), rerank to the top-20 you actually pass to the model. The full stack (contextual embeddings + contextual BM25 + reranking) cut top-20 retrieval failure by 67% (5.7% → 1.9%).

Rules: evaluate retrieval separately from generation (recall@k on a labeled set — see `llm-evals`); chunking follows document structure, not fixed character counts; store metadata for filtering (date, tenant, source) — most production "RAG bugs" are permission and freshness bugs.

## Decision 5: Tool surface design

- **Few, focused tools beat many.** Every tool competes for the model's attention; a giant flat list degrades selection. If tools exceed ~20, use dynamic tool search/loading rather than stuffing all schemas in context.
- **Descriptions are prescriptive about *when*, not just what** — "Call this when the user asks about current prices or recent events" measurably beats a what-it-does description. The description is a prompt, not documentation.
- **Broad tool vs dedicated tool:** a bash/query-anything tool gives breadth but hands your harness an opaque string. Promote an action to a dedicated typed tool when you need to **gate** it (confirmation before irreversible actions), **render** it (custom UI), **audit** it, or **parallelize** it safely (Anthropic's agent-design heuristic). Start broad; promote on demand.
- Tool results the model must see verbatim stay small; big intermediate results get filtered in code before re-entering context.

## Decision 6: Single agent vs multi-agent

**Default: one agent.** Multi-agent is a specific tool, not a maturity level. Anthropic's research system found a multi-agent setup outperformed single-agent by 90.2% on *breadth-first, parallelizable research* — while consuming ~15× the tokens of a chat interaction ([How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)).

Choose multi-agent when: subtasks are genuinely parallel and independent (fan-out reads, per-item processing), or context isolation is the point (each worker gets a clean window for its slice). Avoid when subtasks share state or need tight coordination — agents can't cheaply share context mid-flight, and coordination bugs dominate. One level of delegation; deeper hierarchies are undebuggable.

## Decision 7: Context management for long-running agents

Three mechanisms, by lifetime:
- **Context editing/pruning** — clear stale tool results within a session; keeps the transcript lean without losing structure.
- **Compaction/summarization** — when the conversation approaches the window limit, summarize earlier context. Lossy; keep the durable facts elsewhere.
- **Memory (files/store)** — state that must survive across sessions. Give the agent an explicit place to write learnings and tell it when to consult it.

Long-running agents typically need all three. Also: the context window is a budget — track it like one, and prefer sub-agents or code-side filtering over letting one context absorb everything.

## Decision 8: Structured output

When downstream code parses the response, constrain it — schema-enforced structured output (or strict tool schemas) over parse-the-markdown-and-pray. Validate at the boundary anyway (the schema guarantees shape, not sense). Prompt-level "respond in JSON" is the option of last resort.

## Trust boundaries (prompt injection)

Prompt injection is the #1 LLM application risk ([OWASP LLM01](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)), and the dangerous variant is **indirect**: instructions hidden in content the model reads — web pages, retrieved documents, emails, tickets, tool outputs.

- Everything retrieved or tool-returned is **data, not instructions** — delimit it, and never grant it operator authority.
- Design for the model being compromised: **least-privilege tools** (the injection can only do what the tools allow), no secrets in prompts, human confirmation on irreversible/outward-facing actions.
- Treat model *output* as untrusted too — sanitize before rendering (XSS via generated HTML) or executing.
- Detection/filtering is defense-in-depth, not the boundary. The boundary is what the tools are physically able to do.

## Review checklist

- Is the simplest tier being used? What demonstrated failure justifies each escalation?
- Corpus fits in context? → why is there a RAG pipeline?
- Retrieval evaluated separately from generation? Hybrid + rerank considered?
- Tool descriptions say *when*; irreversible actions gated; tool count sane?
- Long-running: context budget managed (edit/compact/memory)?
- Injection: what's the worst an instruction embedded in retrieved content could do with these tools?
- Parsed outputs schema-constrained?
- Are there evals? (No evals → see `llm-evals` before shipping.)

## Anti-patterns

- An agent for a task with fully specifiable steps (that's a workflow)
- Multi-agent as the starting architecture; agent frameworks adopted before the plain-API version was tried
- RAG over a corpus that fits in a cached context
- Fine-tuning to teach facts
- One mega-prompt accreting every feature's instructions (route instead)
- Giant flat tool lists; tool descriptions without trigger conditions
- Retrieved content concatenated into the prompt with operator-level authority
- "It works on my three test prompts" as the ship criterion

## Sources

- [Building Effective Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)
- [Contextual Retrieval — Anthropic Engineering](https://www.anthropic.com/engineering/contextual-retrieval)
- [How we built our multi-agent research system — Anthropic Engineering](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Lost in the Middle: How Language Models Use Long Contexts — Liu et al., TACL 2024](https://aclanthology.org/2024.tacl-1.9.pdf)
- [OWASP GenAI: LLM01 Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
- [Patterns for Building LLM-based Systems & Products — Eugene Yan](https://eugeneyan.com/writing/llm-patterns/)
