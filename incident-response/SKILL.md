---
name: incident-response
description: Use when production is broken or degraded (outage, elevated errors, data issue), when designing an on-call/incident process, severity levels, or runbooks, when writing or reviewing a postmortem, or when debugging a live system under time pressure. Complements observability-slo (detection) and migration-playbooks (rollback design).
---

# Incident Response

Two disciplines carry most of the value. **Mitigate first, understand later:** the goal during an incident is to stop user impact, not to find the root cause — rolling back and *then* debugging in calm beats debugging live, and skipping that order adds 90+ minutes to recovery in incidents where rollback was the eventual fix anyway. **Blameless afterward:** people acting in good faith on the information they had are never the root cause; systems that let a mistake become an outage are.

## Decision 1: Severity — decide fast, from user impact

Define levels **before** incidents, by user/business impact with concrete examples — not by which component broke:

| Level | Meaning (typical) | Response |
|---|---|---|
| SEV1 | Product down / data loss / security breach — most users blocked | Page now, all hands, exec + public comms |
| SEV2 | Major feature broken or severe degradation for many | Page now, incident process |
| SEV3 | Partial/degraded, workaround exists | Business hours, ticket |

Rules from PagerDuty's response docs: **unsure between two levels → take the higher**; don't litigate severity mid-incident (adjust in review); anyone may declare an incident — a false alarm costs minutes, a delayed declaration costs the MTTR. Detection should come from SLO burn-rate alerts (`observability-slo`), not from a customer tweet.

## Decision 2: First response — the mitigation ladder

First question, always: **"what changed?"** — deploys, config/flag flips, infra changes, dependency behavior, traffic shape. Deploy-induced regressions alone are ~a fifth of incidents, with a consistent fingerprint: starts within ~30 minutes of a deploy, resolved by rollback. Deploy markers on dashboards (`observability-slo`) make this a ten-second check.

Mitigation ladder — cheapest reversible action first:
1. **Roll back** the suspect change (deploy or flag). You don't need proof it's the cause; you need low cost of being wrong. Rollback safety was designed earlier or not at all (`migration-playbooks` Decision 4 — N−1 compatibility, rehearsed rollback).
2. **Flip the ops flag / kill switch** — disable the failing non-critical feature (the flags `migration-playbooks` says to keep few and documented).
3. **Shed or shift** — rate-limit, serve degraded/cached, fail over.
4. Only then: live-debug forward fixes — highest variance, under the worst conditions.

Discipline: no untracked changes ("fix attempts" that aren't logged become tomorrow's "what changed?"); verify mitigation in the *user-facing* metric, not the theory; the timeline is written as you go (a scribe or a bot-annotated channel), because memory reconstructs it wrong.

## Decision 3: Roles and communication (beyond ~2 responders)

From Google's IMAG / PagerDuty (both derived from emergency-services ICS):

- **Incident Commander** — owns coordination and decisions, **does not touch keyboards to fix things**. The IC debugging is the classic failure: nobody is steering, three people restart the same service.
- **Ops/subject responders** — hands on the system, report findings to IC.
- **Communications lead** (or IC at small scale) — posts updates on a stated cadence (e.g., every 30 min *even if "no change"*) to a status page/stakeholder channel; silence generates exec drive-bys into the war room, which is its own outage.
- Small teams collapse roles into two people — but IC-vs-hands separation survives even at two.
- Handoffs at fatigue boundaries are explicit: state, attempts, next steps — a tired responder freelancing at hour six causes the second incident.

## Decision 4: The postmortem

Every SEV1/SEV2 gets one, written within days, **blameless**, and shared widely — the artifact's purpose is organizational learning, converting one team's outage into everyone's fix:

- Contents: impact (users, duration, SLO budget burned — `observability-slo`), timeline (detection → mitigation → resolution, with the *detection gap* highlighted), contributing causes (plural — "root cause" is usually several conditions aligning), what went well / what didn't, action items.
- Blameless concretely: "the deploy script allowed a config that fails at runtime" not "X pushed a bad config." Naming-and-shaming teaches people to hide information, which destroys the only mechanism that prevents recurrence. Blameless ≠ consequence-free; it means the analysis targets the system.
- **Action items are the product:** each has an owner and a ticket in the normal backlog, or it didn't happen. Prioritize detection/mitigation improvements (make it *shorter* next time) over "prevent this exact bug" (it won't recur; its cousins will). Review the backlog of postmortem actions monthly — unfunded action items are how the same incident happens twice, and the second postmortem says so.
- MTTR is dominated by detection + decision time, not typing time — the postmortem asks "why did it take 40 minutes to notice?" before "why did it break?"

## Decision 5: Prepared, not improvised

- **Runbooks** per service, per alert where possible: the 5 known failure modes, the dashboards, the rollback command, the escalation path, the kill switches. Written the day the feature ships (`observability-slo` instrument-before-ship), tested by someone who didn't write them.
- On-call is staffed humanely (rotation depth, comp) and pages are actionable (`observability-slo` Decision 3) — alert fatigue is the root cause of "we got paged and ignored it."
- Rehearse: game days / tabletop failure drills once in a while beat a pristine process document. The first time someone runs the failover must not be during the failure. An untested runbook is a hypothesis (`migration-playbooks` says the same of rollbacks).

## Review checklist (for an incident process or a specific response)

- Severity levels defined by user impact, with examples, and a "when unsure, go higher" rule?
- Can any engineer declare an incident and page without permission?
- Deploy/change markers visible on the dashboards responders open first?
- Rollback of the last deploy: one command, rehearsed, safe with current data (N−1)?
- IC role separated from hands-on-keyboard? Comms cadence stated?
- Postmortem: blameless language, detection gap analyzed, actions owned + ticketed + reviewed later?
- Runbooks exist for the top failure modes and were tested by a non-author?

## Anti-patterns

- Debugging root cause while users are down and a rollback is available
- "Who did this?" in the incident channel; postmortems naming individuals as causes
- IC fixing things; everyone fixing things and nobody coordinating; untracked fix attempts
- Severity debates during the incident
- Status silence ("we'll update when we know more" — you won't be left alone)
- Postmortem action items with no owner, no ticket, or no follow-up review
- Runbooks written during the incident they were needed for
- Treating a near-miss as a non-event (free lessons, same causes)
- Hero culture: rewarding 3am saves over the boring work that prevents them
