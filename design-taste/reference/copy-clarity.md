# Copy and clarity

Interface copy is a design material, not an afterthought filled in last.
Rewrite unclear text so users understand what happened, what matters, and
what to do next — while preserving factual meaning, product terminology,
and established voice.

## Audit the language

Read the whole interaction path, not isolated strings, and look for:
ambiguous nouns/verbs, internal jargon or assumed knowledge, vague labels
and system-state descriptions, missing consequences/recovery/timing,
inconsistent terminology or capitalization across screens, redundant
headings/intros/helper text/confirmations that all say the same thing, and
text that breaks at realistic widths or under translation expansion.

Ask before changing a factual claim, legal meaning, or domain-specific term
— those aren't clarity calls, they're content decisions.

## Set the message hierarchy per state

For any given state, decide in order: the one fact the user needs right
now; the action available next; supporting context that would actually
change the decision; the tone appropriate to the moment. Say each idea
once — if a heading already explains the state, the intro below it should
add new information or be deleted.

## Rewrite by function

### Actions and navigation
Use a specific verb + object when the outcome isn't already obvious from
context ("Delete project" not "Delete"). Labels describe what will happen,
not the gesture used to trigger it. Keep the same noun/verb pair for the
same concept everywhere in the product — don't call it "delete" in one
place and "remove" in another.

For destructive actions: name the object and the consequence explicitly.
Prefer undo over a confirmation dialog when recovery is genuinely safe.
When confirmation really is necessary, name the action on both the message
and the button — never bare "Yes"/"No"/"OK"/"Submit."

### Forms
Persistent labels, not placeholder-as-label (a placeholder disappears the
moment the user starts typing — it can't do a label's job). Format and
eligibility requirements stated before submission, not only surfaced as an
error after. Required/optional treatment consistent across the whole form.
Validation names what needs attention and how to fix it without blaming
the user; keep the instruction near the field; announce errors accessibly
(not visual-only).

### Errors and permissions
An actionable error answers three things: what failed, why (when known and
useful), and how to recover or what alternative remains. Don't expose an
internal error code as the primary message — it can accompany the human
message but shouldn't replace it. Never promise a cause or resolution the
system doesn't actually know. Treat privacy, payment, deletion, access
loss, and blocked-work errors with real seriousness — warmth is fine, jokes
about lost work or money are not.

### Loading, empty, and success states
Loading text names the real operation happening ("Loading your projects…"
not a generic spinner-only state) when the wait is meaningful; show
determinate progress when it's actually available, never invent progress
that isn't real. An empty state distinguishes first-use from no-results
from filtered-to-nothing from a permissions gap from an actual failure —
each needs different guidance and a clear next action. Success confirms
the completed outcome; mention the next consequence only if it changes
what the user should do next — routine success should stay brief.

### Help and instructional text
Helper text answers the implicit question a user actually has, not a
restatement of the control's visible label. Use progressive disclosure for
uncommon detail rather than showing everything at once. Link text needs to
make sense out of context (never bare "click here"); icon-only controls
need a real accessible name.

## Voice, accessibility, and localization

Keep voice consistent across the product; let tone adapt to the moment
(an error deserves a different register than a success toast). Plain
language without flattening terminology the audience genuinely already
knows — over-simplifying for an expert audience is its own clarity failure.

- Write complete, translatable sentences rather than concatenating string
  fragments — fragment-concatenation breaks in languages with different
  word order.
- Keep variables and numbers structured so a translator can reorder them
  within the sentence.
- Budget space for expansion rather than abbreviating text prematurely to
  fit a fixed container.
- Alt text conveys the image's actual information; decorative images get
  an empty `alt=""`, not a redundant caption repeated as alt text.
- Screen-reader accessible names stay aligned with the visible label and
  outcome — a mismatch between what's announced and what's shown is
  actively disorienting.
- Never rely on punctuation, color, or icon alone to carry meaning — pair
  it with text.

Maintain a short terminology glossary when a product is large enough that
inconsistency has crept in across teams/screens. Don't vary word choice for
literary effect in an interface — that's a novel's job, not a button's.

## Content density (from the visual side of copy)

Run a self-audit on every block of copy before shipping: does every
sentence earn its place, or is it restating something the heading or an
adjacent element already said? Flag fake-precise numbers ("Trusted by
14,382 teams") that have no real source behind them — round them, cite
them, or cut them. Where a spec-sheet or table format would communicate
faster than prose, prefer it.

## Quotes and testimonials

Keep quotes to 3 lines maximum. Never let a quote's punctuation include an
em dash for attribution (`"Quote." — Name, Title`) — restructure as
`"Quote." Name, Title` or use a comma; the em dash ban in
[anti-slop-tells.md](anti-slop-tells.md) applies here too, and attribution
lines are a common place it slips through.

## Verify

- Comprehension holds without hidden product knowledge the reader doesn't
  have.
- Every error, empty state, and decision point is actionable, not just
  descriptive.
- Terminology is consistent product-wide, and factual claims are accurate.
- Copy scans cleanly at target widths and at 200% zoom.
- Long names, localization expansion, pluralization, and dynamic values
  all render without breaking the layout.
- Accessible names and state-change announcements are correct, not just
  the visible text.
- The final copy is as short as it can be without losing meaning or a
  recovery path — and contains zero em dashes anywhere.
