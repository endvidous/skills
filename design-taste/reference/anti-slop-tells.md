# Anti-slop tells

The enumerated list of patterns that make a design read as generic,
AI-default, or assembled-not-built. Most entries are defaults a brief can
earn back by explicitly asking for the effect (e.g. a genuinely neobrutalist
brief earns hard offset shadows). Two entries are hard bans with no
exception, flagged below.

## Absolute bans — no brief earns these back

- **Kicker / eyebrow label above a heading** ("INTRODUCING", small tracked
  caps above a title). The heading carries its own weight; a label above it
  is a tell that the heading alone didn't feel confident enough.
- **Em dash (`—`) and en-dash-as-separator (`–`) anywhere in shipped
  copy** — headlines, eyebrows, pills, body text, quotes, attributions,
  captions, buttons, alt text. Zero em dashes, binary, non-negotiable. Use a
  regular hyphen `-` or restructure the sentence. This is the single most
  reliable AI-writing tell and it's checked last, on every string, before
  anything ships.

## Banned defaults (the brief can override these)

### Fonts
Inter, Roboto, Arial, Open Sans, Helvetica, and — once a face is specifically
chosen for a page's identity — the "AI display-serif" rotation itself
(Fraunces, Instrument Serif, Playfair Display, Cormorant, Lora, Crimson,
Newsreader) and the "AI display-sans" rotation (Syne, Space Grotesk, Space
Mono, IBM Plex, Inter-as-display, DM Sans, DM Serif, Outfit, Plus Jakarta
Sans, Instrument Sans) read as defaults precisely because every model reaches
for them first. Naming one anyway needs a reason no other face satisfies —
"it's a book so it wants a serif" is not that reason.

### Color
See [typography-color-layout.md](typography-color-layout.md) for the full
palette-ban list (the "AI purple" lila rule, the cream/terracotta premium
palette). The tell isn't any one color — it's landing in the same 2-3
palette families every unscoped brief converges on.

### Icons and glyphs
- Thick-stroke, generic icon-font glyphs standing in for a considered icon
  system.
- Unicode/emoji doing an icon's job. Icons are drawn — a real library or
  authored SVG, one consistent stroke and weight.
- Middle-dot (`·`) rationed to at most one per composition; it becomes a
  tell the moment it's the default separator everywhere.

### Layout scaffolds
- Same-size icon + heading + text cards as the default page structure.
  Cards are the lazy container; nested cards (a card inside a card) are
  always wrong.
- The hero-metric template: big number, small label, a row of supporting
  stats, an accent color. Recognizable on sight because every SaaS landing
  page defaults to it.
- Section numbers (01 / 02 / 03) unless the sequence itself carries
  information the reader needs.
- Split-header layouts (headline left, supporting image right, symmetric
  50/50) as the default hero — earn it, don't default to it.
- A modal for a task that needs neither interruption nor protected focus.
- Symmetric three-column grids and edge-to-edge navbars as an unexamined
  default rather than a decision.
- Zigzag alternation (image-left/image-right/image-left…) run past two
  cycles without a break.
- Bento grids with a cell count that doesn't map to real content groups, or
  with every cell drawing from the same background treatment.

### Surface habits
- Gradient text — emphasis comes from weight or size, not a color ramp
  across letters.
- Glass/blur used as decoration rather than a specific, motivated effect.
- A colored `border-left`/`border-right` thicker than 1px on cards, list
  items, callouts, or alerts.
- Hard offset shadows (`box-shadow: 4px 4px 0`) outside a genuinely
  neobrutalist world — a zero-blur block shadow is a costume, not a depth
  system.
- Sparklines, progress rings, and soft-shadowed rounded rectangles standing
  in for actual content.
- Monospace as a costume for "technical" rather than for code, data, or
  measurement it's actually displaying.
- A system display face (Impact, Arial Black, the platform sans) doing
  display duty on a page that's supposed to have its own identity — the
  nearest installed font is a failure, not a fallback.
- Geometric masks (a circle, polygon, or radial-gradient cutout)
  approximating a photo's edge instead of a real alpha matte or cutout
  asset — reads worse than not doing the effect.
- Light/dark theme picked by category convention instead of the actual use
  scene (who, where, what ambient light).
- Sketch-style SVG illustration, `loose-sketch`/`doodle` class names, or
  `feTurbulence` grain — reads as amateur. This bans SVG imitating
  pictures, never SVG doing geometry: crisp vector shapes, diagrams, and
  shader effects remain fine.
- `repeating-linear-gradient` stripes or two-axis grid-overlay backgrounds
  with nothing underneath them — textures need to come from the subject's
  real world (a map, a blueprint, a measuring tool), not a CSS pattern
  standing in for one.

### Production-test tells (specific to shipped, screenshot-tested surfaces)
- Version labels dropped into a hero ("v2.0", build numbers) as decoration.
- Decorative status dots with no real state behind them.
- `<br>`-forced line breaks inside italicized headlines.
- Vertical rotated text as a decorative device.
- Fake product-preview screenshots built out of divs rather than real
  captured UI.
- "Quietly in use at" / "Field notes"-style poetic micro-labels with no
  real backing data.
- Generic step labels ("Stage 1", "Stage 2") where the content has a real
  name available.
- Pills overlaid on photography as a default decoration.
- Photo-credit-style captions used purely as a layout filler, not attached
  to a real credit.
- Version footers, decorative text strips at the bottom of a hero, floating
  top-right sub-text with no function.
- Filled-track progress bars used decoratively rather than showing real
  progress.
- Locale/time/weather strips with no real data source behind them.
- Manufactured scroll cues (bouncing chevron) on pages that don't need one.
- `border-top` + `border-bottom` applied to every row in a list — pick one
  or use spacing instead.

### Content tells
- The "Jane Doe" effect: placeholder names, fabricated logos, and generic
  testimonials that read as obviously synthetic. If demonstration content
  is unavoidable in greenfield work, author it at full fidelity and label
  it synthetic rather than let it read as a stand-in.
- Fake-precise numbers ("Trusted by 14,382 teams") with no real source.
  Round, cite, or cut.
- Quotes/testimonials longer than 3 lines, or containing an em dash (see
  the absolute ban above).

## Impeccable's page-scaffold refusals (complementary, philosophy-level)

These come from a stricter "refuse by default" posture — treat them as the
same list at a higher bar, useful when a brief explicitly asks for
restraint or a premium/editorial register:

- Card radii outside 12–16px; pills reserved for small controls only.
- Elevation declared twice (a 1px border *and* a wide soft shadow on the
  same element — "the ghost card"). Pick one.
- Tracking below -0.04em as a floor; -0.02 to -0.03em usually reads better
  in practice.
- Real illustration or none — no sketch-style stand-ins (see above).
- Claims and configuration values come from supplied truth. Naming a
  concept and then ironizing it in the copy is not the same as making an
  unlabeled false claim, but fabricated specifics (real-sounding customer
  counts, fake logos) are never acceptable outside explicitly labeled
  synthetic content.

## Why this list exists

None of these are bad in isolation — hard shadows are correct in a
neobrutalist world, kickers exist because someone once used one well. The
tell is reaching for them by default, without the brief asking, because
they're the fastest path to "looks designed." Recognizing that is the
difference between deciding and defaulting.
