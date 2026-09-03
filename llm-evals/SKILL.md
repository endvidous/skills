---
name: llm-evals
description: Use when building, shipping, or changing any LLM-powered feature — before changing a prompt, swapping a model, or adding an AI capability — and whenever someone asks "is the AI output good?" or reports quality regressions. Covers eval-first development, error analysis, code assertions vs LLM-as-judge vs human review, judge bias mitigation, golden datasets, regression harnesses in CI, and agent/trajectory evaluation.
---

# LLM Evals

Unsuccessful AI products almost always share a root cause: no robust evaluation system ([Hamel Husain: Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/)). Without evals, every prompt tweak and model swap is a blind bet — you cannot know whether you improved the product or broke it for a case you're not looking at. The eval suite is to an LLM feature what the test suite is to code: the thing that makes change safe.

## Decision 1: When to invest in evals

**Before the feature ships, and before any prompt/model change after that.** The eval-first loop: collect real examples → label what "good" means → automate the check → then iterate on prompts/models against it. Iterating on prompts before having evals is optimizing against your own memory of three examples.

**Start with error analysis, not metrics.** Read your actual traces — 30–50 of them — and categorize the failures before choosing what to measure. The failure taxonomy tells you which evals are worth building; a metric chosen before looking at data measures the wrong thing precisely (the core method Husain/Shankar teach: [look at your data first](https://hamel.dev/blog/posts/evals-faq/)).

## Decision 2: What kind of check, per failure mode

Three levels, cheapest first — use the cheapest level that can detect the failure ([Husain's three levels](https://hamel.dev/blog/posts/evals/)):

1. **Code assertions (default wherever possible)** — deterministic checks, run like unit tests: output parses against the schema, required fields present, no PII/user-ID leakage, banned phrases absent, citations point at retrieved chunks, length bounds, refusal-when-required. Cheap, fast, zero judge noise. A surprising fraction of "quality" failures reduce to assertions.
2. **LLM-as-judge** — for genuinely subjective criteria (faithfulness, tone, helpfulness). Powerful but a measurement instrument that itself needs calibration — see Decision 4.
3. **Human review** — the ground truth that validates level 2, and the periodic sample of production traces that catches what the suite doesn't test yet.

Retrieval systems get a separate, fully deterministic eval: recall@k / precision@k on labeled query→chunk pairs. Never debug generation quality while retrieval quality is unmeasured (see `llm-app-architecture` Decision 4).

## Decision 3: The dataset

- **Source:** real production failures first, then hand-written edge cases, then synthetic generation to fill gaps — in that priority order. A synthetic set nobody has read is noise with a progress bar.
- **Size:** 20–100 carefully labeled examples beats 1,000 unread ones. Grow the set from every production incident — each escaped failure becomes a permanent eval case (same discipline as regression tests).
- **Structure:** each case = input + context + a *per-criterion* expected judgment, not a single holistic grade.
- **Contamination:** the moment you tune prompts against a set, it stops measuring generalization. Keep a held-out slice you never look at while iterating, or rotate fresh cases in before big decisions.

## Decision 4: Judge design (LLM-as-judge done properly)

A strong LLM judge can match human agreement (~80%+, the same level as human–human agreement), but carries known, measured biases ([Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena"](https://arxiv.org/abs/2306.05685)):

| Bias | Mitigation |
|---|---|
| **Position bias** (favors first/last answer in pairwise comparison) | Randomize or swap order and require consistency; treat swap-inconsistent verdicts as ties |
| **Verbosity bias** (prefers longer answers regardless of quality) | Judge per-criterion against a rubric, not holistic "which is better"; include length-matched counterexamples in judge validation |
| **Self-enhancement bias** (favors its own model family's outputs) | Judge with a different model than the generator, or validate the judge against human labels before trusting it |

Hard rules ([Husain's judge guide](https://hamel.dev/blog/posts/llm-judge/index.html)):
- **Binary pass/fail per criterion, never 1–10 scores.** A "helpfulness of 4.2 vs 4.7" is meaningless and unactionable; "faithful-to-source: fail" is a bug report.
- **One criterion per judge prompt** — an omnibus judge blurs everything into vibes. Faithfulness, completeness, and tone are separate judges.
- **Critique before verdict** — require the judge to quote the evidence, then decide.
- **Validate the judge itself:** label a sample with a human expert, measure judge–human agreement, have the human critique the judge's mistakes, fold critiques back into the judge prompt, re-measure. The judge is only trusted after this loop converges. An unvalidated judge is a random-number generator with confidence.

## Decision 5: Regression harness

- Evals run **on every prompt, model, retrieval, or tool-description change** — in CI or a pre-merge script, exactly like tests. Prompts are versioned artifacts in the repo, not strings edited in a dashboard.
- Track scores over time per criterion; a change ships when it improves the target criterion **without regressing the others** — LLM changes routinely fix one behavior and silently break another, which is the whole reason the suite exists.
- Model upgrades get the full suite plus a human-review sample before rollout; provider model updates count as changes.
- Budget note: judge calls cost money — run assertions on every commit, judges on merges/nightly, full human-anchored calibration monthly-ish.

## Decision 6: Evaluating agents (multi-step)

Single-response evals don't transfer directly to agents:
- **End-state over trajectory (default):** define what "done" looks like (files changed, ticket updated, correct answer with citations) and check the end state; agents legitimately reach goals by different paths.
- **Trajectory checks where the path matters:** required tool called, no forbidden tool called, no irreversible action without confirmation, loop terminated within budget.
- **Rubric-graded outcomes** for open-ended deliverables: explicit, independently gradeable criteria ("CSV has a numeric `price` column"), not vibes ("data looks good") — vague criteria produce noisy grading loops.
- Instrument token spend and turn count as eval outputs too — a fix that doubles cost is a finding, not a footnote (see `llm-cost-latency`).

## Online evaluation (production)

The offline suite is necessary, not sufficient. In production: capture thumbs/edit-rate/abandonment as weak labels; sample traces for scheduled human review; feed every confirmed failure back into the dataset (the flywheel — [Eugene Yan's patterns](https://eugeneyan.com/writing/llm-patterns/)). A/B test major changes on real traffic when the offline delta is small — offline evals rank variants, production decides.

## Review checklist

- Error analysis done on real traces before metrics were chosen?
- Every failure mode covered at the cheapest sufficient level (assertion > judge > human)?
- Judges: binary, per-criterion, critique-first, different model or human-validated, position/verbosity bias handled?
- Judge–human agreement measured and acceptable?
- Held-out set intact (not tuned against)?
- Evals wired into the change process (CI / pre-merge), prompts versioned?
- Retrieval evaluated separately (recall@k)?
- Production traces sampled for human review; failures flow back into the dataset?

## Anti-patterns

- Shipping on vibes ("looks good on my examples")
- 1–10 scores; one omnibus "quality" judge
- Judge same-model-as-generator, never validated against humans
- Tuning prompts against the full eval set, then reporting its score
- Large synthetic datasets nobody has read
- Evals run once at launch, never on subsequent prompt edits
- Debugging generation while retrieval is unmeasured
- Treating the judge's verdict as ground truth instead of a calibrated instrument

## Sources

- [Your AI Product Needs Evals — Hamel Husain](https://hamel.dev/blog/posts/evals/) · [LLM judge guide](https://hamel.dev/blog/posts/llm-judge/index.html) · [Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena — Zheng et al., NeurIPS 2023](https://arxiv.org/abs/2306.05685)
- [Patterns for Building LLM-based Systems & Products — Eugene Yan](https://eugeneyan.com/writing/llm-patterns/)
- [LLM-as-a-judge complete guide — Evidently AI](https://www.evidentlyai.com/llm-guide/llm-as-a-judge)
