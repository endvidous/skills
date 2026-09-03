# Motion

Motion is a material, not a garnish. Every animation should be traceable to
a reason — orienting the user, showing causality, or expressing brand
character at a moment that earns it. "Motion claimed = motion shown": if the
brief or the design says something animates, it has to actually animate,
not just be capable of it.

## The animation decision framework

Before animating anything, answer four questions:

1. **Should it animate at all?** Not everything needs motion. A state
   change that's instant and legible doesn't need an entrance.
2. **What's the purpose?** Orientation (where did this come from/go),
   feedback (the system heard the interaction), continuity (this thing is
   still the same object, just transformed), or delight (an earned,
   scarce moment of personality).
3. **What easing?** See the decision tree below.
4. **How fast?** See the duration table below.

### Easing decision tree
- Entrances and things appearing: ease-out (fast start, gentle settle) —
  the object arrives with momentum and decelerates into place.
- Exits and things disappearing: ease-in (slow start, fast finish) — it
  accelerates away.
- State changes in place (color, size toggles): ease-in-out.
- Anything the user can interrupt mid-flight (drags, gestures, spring
  dismissals): a spring, not a duration-based curve — see below.
- Default authored motion in a design system: exponential ease-out from an
  already-visible default state, not from `scale(0)`/`opacity: 0` at the
  extreme.

### Duration by element type (rough targets)
| Element | Duration |
|---|---|
| Micro-interaction (button press, toggle) | 100–150ms |
| Small UI transition (dropdown, tooltip) | 150–250ms |
| Modal/panel entrance | 250–400ms |
| Page-level or hero entrance | 400–600ms |
| Scroll-driven / choreographed sequence | scrubbed to scroll position, not fixed |

Never bounce or elastic easing outside a deliberately playful/Experience
surface — it reads as toy-like on anything Operate/Read.

## Spring animations

Use springs for anything the user's input can interrupt mid-animation —
drags, gesture dismissals, hover-triggered movement. A duration-based
animation started, then re-triggered mid-flight, jumps; a spring smoothly
redirects because it's driven by velocity and position, not a timeline.

Apple-style spring config as a starting point: `{ type: "spring", duration:
0.5, bounce: 0.2 }`. Traditional physics springs (`stiffness`/`damping`) are
equivalent but harder to reason about by feel — prefer the duration+bounce
parameterization when the tooling supports it.

## Component-level rules

- Buttons: `scale(0.97)` on `:active`, no more. Never animate a button's
  entrance from `scale(0)` — start no smaller than `scale(0.95)`.
- Popovers: origin-aware (grow from the trigger element's position), except
  modals, which stay centered regardless of trigger position.
- Tooltips: skip the hover delay on subsequent tooltips within the same
  interaction session — only the first hover in a sequence should feel
  slow.
- Prefer CSS transitions over keyframe animations for anything interruptible
  UI-driven (hover, focus, toggle) — they're cheaper to interrupt cleanly.
- Use `@starting-style` for enter animations on newly-inserted DOM nodes
  where supported, instead of a JS-driven mount animation.
- Blur can mask an imperfect crossfade between two pieces of content that
  don't perfectly align.

## CSS transform mastery

- `translateY` as a percentage moves relative to the element's own height —
  useful for consistent relative movement regardless of element size.
- `scale()` scales children too; if a child needs to stay unscaled, it
  needs its own inverse transform or to live outside the scaled parent.
- 3D transforms (`rotateX`/`rotateY`, `perspective`) need a `perspective`
  value set on the parent to read as dimensional rather than skewed.
- Set `transform-origin` deliberately — the default (center) is wrong for
  anything that should pivot from an edge or a trigger point.

## clip-path for animation

`clip-path` (inset shape) is the right tool for: tab-style color
transitions, hold-to-delete progress patterns, image reveals, and
before/after comparison sliders. It animates smoothly on GPU and avoids the
layout cost of animating actual dimensions.

## Gesture and drag interactions

- Momentum-based dismissal: use a velocity threshold (~0.11 in normalized
  units) below which a drag snaps back rather than dismisses — a slow drag
  shouldn't accidentally trigger a dismiss.
- Apply damping/resistance at drag boundaries rather than a hard stop —
  friction communicates "you can't go further" better than a wall.
- Use pointer capture so a drag continues tracking even if the pointer
  leaves the element's bounds.
- Guard against multi-touch interfering with a single-pointer gesture.

## Choreography patterns (for Persuade/Experience surfaces)

- **Sticky-stack scroll**: pin a card in place (`start: "top top"`,
  `pin: true`) while the next card scrolls over it, scaling/fading the
  pinned card down as the next one arrives. Canonical for scrollytelling
  sequences.
- **Horizontal pan on vertical scroll**: convert vertical scroll into
  horizontal movement across a wide canvas (`scrub: 1`, end distance tied
  to content width). Reserve for content that's genuinely wide — a gallery,
  a timeline — not as a novelty.
- **Scroll-reveal stagger**: elements fade/rise into view as they enter
  the viewport, staggered 30–80ms apart per item. This is the lightweight
  default (`whileInView`-style) — reach for the heavier scroll-driven
  patterns above only when the content justifies it.
- **Fluid island nav**: a nav bar that morphs shape/size in response to
  scroll position or menu state rather than snapping between fixed states.
- **Magnetic button hover**: a button's content subtly follows the cursor
  within its bounds before the click, then releases — reserve for a
  primary CTA on an Experience surface, not every button.
- **Modal expansion**: a modal that visibly grows from its trigger's
  position with a staggered mask reveal of its contents, rather than
  fading in as a flat overlay.

Marquees (auto-scrolling ticker content): maximum one per page. It stops
being a signature moment the second it's used twice.

## Forbidden animation patterns

- `window.addEventListener("scroll", ...)` driving animation directly —
  always go through a scroll-linked animation library or
  `IntersectionObserver`, never a raw scroll listener re-triggering layout
  reads.
- `requestAnimationFrame` loops that write to React (or equivalent
  framework) state every frame — this thrashes re-renders. Drive continuous
  values (pointer position, scroll offset) through a motion-value primitive
  (e.g. `useMotionValue`/`useTransform`/`useScroll` in Motion) that bypasses
  React's render cycle, never through `useState`.
- Animating `width`/`height`/`top`/`left`/margins casually — these are
  layout-triggering properties. Reach for `transform`/`opacity` first;
  layout properties are acceptable only when the effect genuinely requires
  a layout change and the area affected is small.
- Using a framework's shorthand transform properties (e.g. Framer Motion's
  bare `x`/`y`/`scale` props) without checking whether the library compiles
  them to a single hardware-accelerated `transform` string — some
  shorthand paths are not accelerated, and mixing multiple transform
  properties as separate style writes on Motion's `layout`/`layoutId` can
  fight the library's own optimizations.
- Mixing GSAP or Three.js scroll/animation control with a Motion (Framer
  Motion) tree in the same component subtree — pick one animation system
  per tree and stay in it.

## Performance rules

- Only `transform` and `opacity` are reliably GPU-accelerated across
  browsers; blur, backdrop-filter, filter, clip-path, mask, and colored
  shadows are fine to use for real polish but should be bounded to small,
  isolated areas since they're more expensive to composite.
- CSS variables are inheritable — setting one on a shared parent to animate
  it recomputes for every descendant that reads it; scope custom properties
  to the smallest element that needs them.
- CSS animations/transitions generally beat JS-driven animation under load
  because they run off the main thread more often.
- For programmatic control that still wants hardware acceleration, the Web
  Animations API (WAAPI) is a reasonable middle ground between CSS and a
  full JS animation library.
- `will-change` is a targeted hint for a specific, known-expensive
  animation about to happen — not a default to sprinkle on animated
  elements, and never left on at rest (it forces a new compositor layer,
  which costs memory).

## Accessibility

- Respect `prefers-reduced-motion` — but a global `animation-duration:
  0.01ms !important` kill switch is the wrong implementation because it
  destroys useful feedback along with decoration. Provide an intentional
  reduced alternative that still communicates the state change and
  hierarchy (a fade instead of a slide, an instant state change with a
  clear visual delta) rather than nothing at all.
- Above `MOTION_INTENSITY` ~3 (see SKILL.md's dial framework), a reduced
  alternative isn't optional.
- Gate hover-triggered animation behind an actual hover capability check —
  touch devices report a `hover` pseudo-state on tap that shouldn't trigger
  a desktop-style hover animation.

## Stagger animations

Stagger delays of 30–80ms per item read as a deliberate sequence; below
that it reads as simultaneous, above it reads as sluggish. Longer lists
should compress the per-item delay (or cap total stagger duration) rather
than scaling delay linearly with item count.

## Delight — where earned motion goes

Delight is not a layer of generic whimsy on top of an interface; it's
product character revealed at a moment that earns it. Concentrate it at:
first use, task completion, recovery from an error, or mastery (a power-user
shortcut discovered). Match response intensity to effort and consequence —
a major milestone can expand into something bigger; a routine save should
just feel certain and quick. Never fake a loading delay to stage a
flourish, never let a repeated interaction get old after the hundredth use
(if variation is used, keep it coherent and predictable enough to trust),
and never let delight block, delay, or obscure the primary task.

## Debugging animations

- Test at 0.1–0.25x playback speed (browser devtools animation inspector,
  or a temporary duration multiplier) to catch timing/easing issues
  invisible at full speed.
- Step frame-by-frame through a choreographed sequence to catch pops or
  discontinuities at handoff points between animations.
- Always verify on a real mid-range device, not just a fast dev machine —
  animation that's smooth on a MacBook Pro can drop frames on a mid-range
  Android phone.

## Verify

- One authored moment per section, not scattered effects and not one
  identical entrance repeated on every section.
- Every animation traces to orientation, feedback, continuity, or an
  earned delight moment — nothing animates "because it can."
- Reduced-motion alternative exists and still communicates state.
- No dropped frames on a representative mid-range device.
- Interruptible interactions (drags, hover-triggered motion) redirect
  smoothly rather than jumping or restarting.
