---
name: billing-payments
description: Use when building or reviewing anything that touches money or plans — payment provider integration (Stripe et al.), subscriptions, checkout, plan changes, proration, entitlements/feature gating, usage-based/metered billing, invoices, failed payments/dunning, refunds, or billing webhooks. Covers the provider-as-source-of-truth rule, webhook reconciliation, the entitlement layer, subscription state machines, and dunning. Complements api-contracts (webhook mechanics, idempotency) and distributed-reliability.
---

# Billing & Payments

Two rules generate most of this file. **One:** never store, log, or transit raw card data — use the provider's hosted fields/checkout so card numbers never touch your servers, which keeps you in the lightest PCI compliance tier (SAQ-A) instead of a real audit ([Stripe PCI guide](https://stripe.com/guides/pci-compliance)). **Two:** for everything the provider manages (subscriptions, payment state), **the provider is the source of truth and your database is a projection of it** — billing bugs are almost always a projection that drifted, and the fix is reconciliation, not cleverer event handling.

## Decision 1: Integration surface

- **Hosted checkout (Stripe Checkout-class) — default.** Provider hosts the payment page; you get SAQ-A PCI scope, built-in SCA/3DS, wallets, tax hooks. Cost: less UI control. Start here; the conversion argument for custom checkout is rarely backed by your data.
- **Embedded fields (Elements-class)** — custom UI, card data still goes direct to the provider. Note the compliance step-up: fully custom forms with embedded fields can land in SAQ A-EP, which has real web-security requirements ([SAQ scope differences](https://cside.com/blog/can-you-use-stripe-for-pci-dss)).
- **Your own card handling** — no. That's full PCI DSS; it's a business decision with an auditor, not an engineering choice.
- Multi-provider abstraction layers are speculative until you have a signed reason (a market your provider doesn't serve). One provider, used deeply, beats an adapter over two used shallowly.

## Decision 2: Webhook-driven projection — the core loop

Your handler pattern for every billing event ([Stripe webhook docs](https://docs.stripe.com/webhooks), [subscription webhooks](https://docs.stripe.com/billing/subscriptions/webhooks)):

1. **Verify the signature** on the raw body (`api-contracts` webhook rules), ack `200` fast, process async — the provider's timeout is ~10s and slow handlers cause retry double-processing.
2. **Dedupe by event ID** — delivery is at-least-once (`distributed-reliability` idempotent-consumer pattern).
3. **Never trust event order.** Events race; a stale `subscription.updated` can arrive after a newer one. The embedded object snapshot is for routing/logging only — **refetch the live object from the provider API and project *that*** into your DB. This one rule eliminates the "paying customer lost access" class of bug.
4. Projection writes are idempotent upserts keyed on the provider's object ID; multiple writes in one event → `$transaction`-equivalent.
- Webhooks *will* be missed (outage, bug, endpoint misconfig). Run a **periodic reconciliation job** that lists active subscriptions from the provider API and diffs against your projection — the safety net that turns silent drift into a metric.
- Checkout success: the redirect/return URL is UX only; **entitlement is granted by the webhook/reconciled state**, never by the user landing on `/success` (users close tabs; attackers visit URLs).

## Decision 3: The entitlement layer

Separate **billing state** (provider's domain: subscriptions, invoices, payment status) from **entitlements** (your domain: what this account can do — features, limits, seats). Product code asks the entitlement layer (`can(account, feature)`, `limit(account, metric)`); *only* the billing module maps provider state → entitlements. Reasons: plan changes stop being code-wide find-and-replace; comps/trials/enterprise overrides become data instead of `if (customer === 'bigco')`; and billing-provider migration stops touching product code. Entitlement checks are authorization checks — enforce server-side, deny by default (`security-boundaries`). Cache them briefly if hot, respecting the revocation window (`caching-strategy`).

## Decision 4: Subscription lifecycle

- Model your projection on the **provider's state machine** (Stripe: `incomplete → active → past_due → canceled/unpaid`, plus `trialing`) rather than inventing your own booleans — `is_premium` can't represent "payment failing, in grace period," and that state is where the money is.
- **Cancellation defaults to end-of-period** (immediate cancel + no refund = support tickets; immediate + refund = proration math). Offer pause where churn data justifies it.
- **Upgrades apply immediately with proration; downgrades apply at period end** — the standard because it matches money already paid. Let the provider compute proration; verify with its preview API rather than reimplementing.
- **Trials:** without-card trials convert worse but sign up more; with-card trials must send a pre-charge reminder (required by law in several jurisdictions, and good faith everywhere).
- Test lifecycle logic against provider **test clocks** (simulated time) — renewal/dunning bugs are otherwise only discoverable monthly, in production (`testing-strategy`).

## Decision 5: Failed payments — dunning

Involuntary churn (expired/declined cards) is silent revenue loss; the recovery machinery ([dunning guides](https://baremetrics.com/blog/dunning-management)):

- **Use the provider's smart retries** (ML-timed, decline-code-aware) over any hand-rolled schedule.
- **Grace period, not cliff:** `past_due` keeps access for ~7–14 days while retries and emails run; degrade before you cut off.
- **Dunning emails** (a sequence over the retry window, each with a one-click update-card link — provider-hosted billing portal makes this nearly free) plus **pre-expiry card warnings**; pause-don't-cancel for high-value accounts, with human outreach.
- Instrument recovery rate and involuntary-churn rate (`observability-slo` business metrics) — this funnel is invisible unless measured.

## Decision 6: Money-handling rules

- **Integer minor units** (cents), never floats; currency stored beside every amount (`api-contracts` field conventions). Your projection stores what the provider charged, not what you recomputed.
- Refunds/credits/disputes are **new ledger entries, never mutations** of past records — accountants and auditors need the history; model money movements append-only (`data-modeling` state rules).
- Any charge you initiate (usage billing, one-offs) carries an **idempotency key** — a retried network call must not charge twice (`api-contracts`, `distributed-reliability`).
- Usage-based/metered billing: meter in your domain (events → aggregation you can re-derive), report aggregates to the provider; keep raw events long enough to re-bill a disputed period.
- Taxes: use a tax engine (Stripe Tax-class) from the first international customer — VAT/GST registration thresholds are a legal problem, not a rounding one.

## Review checklist

- Card data path: hosted fields/checkout only? Nothing card-shaped in logs, DB, or LLM prompts?
- Webhooks: signature verified, fast-ack + async, deduped by event ID, live-object refetch instead of trusting order?
- Reconciliation job diffing provider ↔ projection, with drift alerting?
- Entitlements: single layer, server-side, mapped only from billing module, overrides as data?
- Success page grants nothing; webhook grants everything?
- `past_due`/grace modeled? Proration delegated to provider? Test clocks in tests?
- Amounts in integer minor units + currency? Refunds append-only? Idempotency keys on charges?

## Anti-patterns

- `is_premium: boolean` as the whole billing model
- Trusting webhook event payloads/order over a live API refetch; no reconciliation job
- Granting access on the checkout redirect
- Entitlement `if plan == 'pro'` checks scattered through product code
- Hand-rolled retry schedules; hard cutoff at first failed payment
- Recomputing proration/tax yourself; floats for money
- Editing historical invoices/charges in place
- Custom card forms "for the design" without noticing the PCI tier change
- Multi-provider abstraction before a second provider exists

## Sources

- [Stripe: Using webhooks with subscriptions](https://docs.stripe.com/billing/subscriptions/webhooks) · [Stripe webhooks](https://docs.stripe.com/webhooks) · [How subscriptions work](https://docs.stripe.com/billing/subscriptions/overview)
- [Stripe: PCI compliance guide](https://stripe.com/guides/pci-compliance) · [SAQ-A vs A-EP scope with Stripe](https://cside.com/blog/can-you-use-stripe-for-pci-dss)
- [Stripe webhooks end-to-end: signatures, idempotency, replay](https://appycodes.dev/blog/stripe-webhooks-end-to-end-2026/) · [Subscription webhooks that survive the edge cases](https://amplifiedcreations.com/journal/stripe-subscription-webhooks)
- [Dunning management guide — Baremetrics](https://baremetrics.com/blog/dunning-management) · [Involuntary churn & failed payments](https://dodopayments.com/blogs/involuntary-churn-failed-payments)
