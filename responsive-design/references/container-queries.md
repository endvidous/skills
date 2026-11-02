# Container Queries

Media queries ask "how wide is the window?". Container queries ask "how wide is the box I'm
in?". Reusable components should almost always ask the second question: the same product card
appears in a 280px sidebar, a 3-column grid and a full-width hero on one screen.

Supported in all evergreen browsers since 2023 (size queries). Use freely.

## Plain CSS

```css
.card-slot { container-type: inline-size; container-name: card; }

.card { display: grid; gap: 1rem; }
@container card (min-width: 28rem) {
  .card { grid-template-columns: 10rem minmax(0, 1fr); }
}
```

- `inline-size` (width only) is what you want 99% of the time. `size` also tracks height but
  requires the container to have an explicit height, or it collapses to 0.
- A container can't style itself based on its own size — only its descendants. Put
  `container-type` on a wrapper.
- Name containers when nesting, so a query hits the intended ancestor.

## Tailwind v4 (built in)

```html
<div class="@container">
  <article class="grid gap-4 @md:grid-cols-[10rem_minmax(0,1fr)] @3xl:gap-8">…</article>
</div>

<!-- named -->
<div class="@container/sidebar"> … <p class="@lg/sidebar:text-lg">…</p></div>

<!-- arbitrary & max -->
<div class="@min-[30rem]:flex @max-md:hidden">…</div>
```

Default container sizes: `@3xs` 16rem · `@2xs` 18rem · `@xs` 20rem · `@sm` 24rem · `@md` 28rem ·
`@lg` 32rem · `@xl` 36rem · `@2xl` 42rem · `@3xl` 48rem · `@4xl` 56rem · `@5xl` 64rem ·
`@6xl` 72rem · `@7xl` 80rem.

Tailwind v3 needs the `@tailwindcss/container-queries` plugin; syntax is the same.

## Container query units

`cqi` (1% of container inline size), `cqb`, `cqw`, `cqh`, `cqmin`, `cqmax`.

```css
.tile h3 { font-size: clamp(1rem, 0.6rem + 3cqi, 1.75rem); }
```

Great for type and padding that should track the component's width. Keep a `rem` term in the
`clamp()` so user font-size preferences still apply.

## When to use which

| Question | Use |
|---|---|
| Should the sidebar be a drawer? Should the page be one column? | Media query (viewport) |
| Should this card stack its image above or beside its text? | Container query |
| Should the toolbar show labels or icons only? | Container query on the toolbar |
| Is the device touch / does it hover? | Media query (`pointer`, `hover`) |
| Is the screen short? | Media query (`max-height`) |
| Is the panel the user resized narrow? | Container query (media queries can't see it) |

The last row is why split-pane apps especially benefit: when users drag a divider, the viewport
doesn't change but every component inside the pane needs to adapt.
