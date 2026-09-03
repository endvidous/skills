# Typography, color, and layout

The three load-bearing structural decisions of any design. Each section
below has: what to decide before touching CSS, how to apply it, and how to
verify the result.

## Typography

### Set the system before editing
State, in order: the roles the interface actually needs (heading, body,
label, metadata, data); the intended contrast between those roles; the
reading measure and density; which faces/weights are already authoritative
if extending an existing surface; and any performance/localization/a11y
constraints. Use the fewest roles and families that make the hierarchy
unmistakable — combine size, weight, space, and tone rather than asking size
alone to carry every distinction.

### Face selection
Avoid the banned-default list in [anti-slop-tells.md](anti-slop-tells.md).
For a page that needs a real point of view (Persuade/Experience surfaces),
choose a face like an object from the subject's own world rather than a
category association — "it's a bookshop so it wants hand-lettering" is the
reflex the calibration below exists to catch. For Operate/Read surfaces,
workhorse system stacks and established UI faces are the right call, not a
compromise.

If a display serif is genuinely justified, rotate outside the default eight
rather than reaching for Fraunces or Instrument Serif reflexively. Verify
italic descenders have enough line-height clearance to not clip.

### Apply
- Body copy floor: 16px on the web (iOS Safari force-zooms focused inputs
  under 16px, which breaks form layouts). 14px only for genuinely secondary
  text.
- Prose measure: 45–75 characters (craft-floor: 65–75ch). Tune line-height
  inversely with measure — wider lines need more leading.
- Light text on dark surfaces needs compensation on three axes at once:
  slightly more line-height, a touch more tracking, one step more weight
  if the face supports it.
- Tracking floor around -0.04em; -0.02 to -0.03em usually reads better in
  practice than pushing to the floor.
- Display type max ~6rem; balance headings so line breaks don't strand a
  single word.
- Numeric/tabular/code features (`font-variant-numeric: tabular-nums`) for
  data that needs alignment across rows.
- Load only the font weights actually used; provide metric-compatible
  fallbacks; never block text rendering while a font loads.
- Paragraph rhythm: use spacing OR first-line indentation, never both —
  combining them double-marks the same boundary.
- Preserve browser zoom, user font-size settings, and platform text
  scaling. Never override them.

### Verify
- Heading/body/label/metadata roles are distinguishable at a glance, with
  detail blurred (the squint test).
- Long headings, localization expansion (German runs ~30% longer than
  English), 200% zoom, and narrow containers don't break the hierarchy.
- No unexplained reflow or invisible text during font load.
- Repeated roles stay pixel-identical across screens and states.

## Color

### Audit before choosing
Identify what's already a confirmed brand commitment vs. what's still open.
If a brand identity already exists, preserve it — a color pass refines
roles inside it, it doesn't replace the identity. If this is genuinely new
work, name the intended emotional temperature, dominant relationship,
contrast range, and color dosage before picking any hex values.

### The AI-purple / lila rule
Default gradient purples (`#8B5CF6`-adjacent violets used as a default
"tech" accent with no brand reason) are the single most recognizable
AI-generated-design tell. If purple is the actual brand color, use it — but
arriving at it by default, with no brief signal, is the tell.

### Banned default "premium consumer" palette
The cream-and-warm-neutral family (`#f5f1ea` and its immediate relatives)
paired with a terracotta/rust accent has become as much of a default as AI
purple — recognizable on sight as "the safe premium SaaS palette." If the
brief doesn't specifically call for warm minimalism, reach for one of these
instead:

- **Cold luxury** — near-black, cool grays, a single icy accent.
- **Forest** — deep greens, warm neutrals, no accent color needed.
- **Black and tan** — true black, warm tan/leather, minimal palette.
- **Cobalt + cream** — a saturated blue against a genuinely warm cream
  (not the banned default shade), used with intent.
- **Terracotta + slate** — if terracotta is wanted, pair it against cool
  slate instead of warm cream to avoid the default combination.
- **Olive + brick + paper** — three-tone, editorial register.
- **Pure monochrome + one pop color** — grayscale with a single saturated
  accent reserved for the primary action only.

### Color strategy (choose one before picking hex values)
- **Restrained** — neutrals plus one accent. The default for Operate/Read
  surfaces.
- **Committed** — one saturated color carries 30–60% of the surface.
- **Full palette** — 3–4 named roles, each with a real job.
- **Drenched** — the surface *is* the color. Reserved for Persuade/
  Experience work with real permission to commit.

### Build roles, not a bag of swatches
Canvas/elevated surfaces, primary/secondary text, action/focus/selection,
borders/separators, semantic (success/warning/error/info), and data
categories if needed. Prefer OKLCH for new web palettes — lightness and
chroma adjust predictably. Choose hue from product meaning, not category
default. Let the strongest color own a deliberate region rather than
scattering small accents; don't spend the primary action's color on
decoration. On colored surfaces, derive secondary text from that hue or the
foreground — never plain washed-out gray on a color.

### Contrast (WCAG AA minimums)
| Content | Minimum |
|---|---|
| Body text | 4.5:1 |
| Large text | 3:1 |
| Controls, icons, focus indicators | 3:1 |

Verify computed pairs, not eyeballed ones — check hover/focus/disabled
states, overlays, text-on-image, and both light and dark themes. Never let
color alone carry a meaning; pair it with text, shape, icon, or position.

### Verify
- Every color has a stable role, not a decorative one-off.
- Attention lands on the intended action or state.
- Palette holds up across quiet, dense, error, and empty states.
- Dark mode is composed on purpose, not a mechanical inversion of light
  mode; no pure `#000`/`#fff`.
- The result reads as this specific product, not a generic "colorful"
  treatment.

## Layout

### Diagnose before moving boxes
Apply the squint test: with detail blurred, can you still identify the
primary element, secondary element, and major groups in reading order? If
not, the problem is structural, not decorative — don't reach for spacing
tweaks to fix a hierarchy problem.

### Set the spatial thesis
Name: the primary reading/task path; what belongs together vs. what must
separate; which element leads and which supports; the intended density and
spacing rhythm; how structure changes across viewport, input mode, and
content extremes (empty, overflowing, localized).

### Apply
- Group by proximity before reaching for containers or borders.
- Build deliberate rhythm from contrast between tight and generous
  intervals — one repeated spacing value flattens everything to equal
  weight.
- Use a documented spacing scale. A 4-unit base scale generally covers the
  useful middle steps an 8-only scale misses.
- Responsive behavior should be structural (reorder/collapse/reveal based
  on what's actually still important), not just proportional shrinking.
- `min-h-[100dvh]`, never `h-screen`, for viewport-height sections — mobile
  browser chrome makes `100vh` unreliable.
- Prefer CSS Grid over flex-math for two-dimensional layout; don't fight
  flexbox into doing grid's job.
- Avoid center-bias as a default — asymmetry is a legitimate, often
  stronger default for Persuade/Experience work.
- Keep touch targets usably large even when their visible mark is small.

### Anti-repetition caps (see also anti-slop-tells.md)
- Hero section: everything must fit in the viewport at load — cap top
  padding around `pt-24`, cap the number of stacked hero elements at 4.
- Nav: single line, 80px height cap.
- Bento grids: cell count should map to real content groups, not an
  arbitrary decorative grid.
- Eyebrow labels: never (see the absolute ban) — but if this file is being
  read in isolation, restraint elsewhere follows the same logic: no more
  than once per three sections for any other repeated micro-label pattern.
- Zigzag alternating layouts: cap at two cycles before breaking pattern.

## Materiality, shadows, and cards

- Shadows need both offset and blur — a zero-offset colored halo is
  decoration, not depth.
- Declare elevation once (border OR shadow, not both stacked on the same
  element).
- Card radii: 12–16px. Pills are for small controls only, not cards.
- Shape consistency lock: once a corner radius and shadow language are
  chosen for a component family, every instance of that family uses them —
  don't let one card round more than its siblings.
- Real illustration or none (see anti-slop-tells.md) — crisp vector
  geometry, diagrams, and shader effects remain first-class, sketch-style
  SVG scenes don't.

## Verify (combined)

- Contrast, spacing, and type checks above all pass on computed values,
  not visual approximation.
- The squint test still reveals primary/secondary/groups at every
  supported viewport.
- Long content, localization expansion, zoom, and empty states don't break
  structure.
- Keyboard, touch, and screen-reader order agree with the visual order.
