# Scaled Previews (A4/Letter pages, slides, canvases, device frames)

Some things must look identical at every screen size: a resume that will be printed, a 16:9
slide, a social-media image, a phone mockup. If you let their HTML reflow to the available
width, line breaks, page breaks and font sizes change and the preview lies. So:

**Render at the real size, then scale the whole thing.**

| Format | Size at 96 CSS px/in |
|---|---|
| A4 portrait | 794 × 1123 px (210 × 297 mm) |
| US Letter portrait | 816 × 1056 px (8.5 × 11 in) |
| 16:9 slide | 1920 × 1080 (or 1280 × 720) |

You can also author pages in physical units (`width: 210mm`) — browsers map them at 96px/in —
which keeps print CSS and screen preview identical.

## The trap: `transform: scale()` doesn't change layout size

A 794px page scaled to 0.5 *looks* 397px wide but still occupies 794 × 1123 in layout. You
get a phantom horizontal scrollbar and a huge empty gap below the page. Fix it by giving the
page a wrapper whose size is the *scaled* size:

```html
<section class="@container min-w-0 overflow-auto bg-gray-200 p-4 sm:p-8" data-fit-preview>
  <div class="mx-auto" data-scale-box>                                <!-- sized by JS -->
    <article class="origin-top-left bg-white shadow-xl" data-page
             style="width:794px;height:1123px">…resume…</article>
  </div>
</section>
```

Size the wrapper with a ResizeObserver:

```js
function fitPreview(pane, { mode = 'width', maxScale = 1 } = {}) {
  const box = pane.querySelector('[data-scale-box]');
  const page = pane.querySelector('[data-page]');
  const layout = () => {
    const cs = getComputedStyle(pane);
    const w = pane.clientWidth - parseFloat(cs.paddingLeft) - parseFloat(cs.paddingRight);
    const h = pane.clientHeight - parseFloat(cs.paddingTop) - parseFloat(cs.paddingBottom);
    const pw = page.offsetWidth, ph = page.offsetHeight;   // unscaled layout size
    let scale = w / pw;
    if (mode === 'page') scale = Math.min(scale, h / ph);
    scale = Math.min(maxScale, scale);
    page.style.transformOrigin = 'top left';
    page.style.transform = `scale(${scale})`;
    box.style.width = `${pw * scale}px`;
    box.style.height = `${ph * scale}px`;
    box.style.marginInline = 'auto';
  };
  const ro = new ResizeObserver(layout);
  ro.observe(pane); ro.observe(page);   // page height can grow with content
  layout();
  return () => ro.disconnect();
}
```

Give the pane `scrollbar-gutter: stable`, otherwise a vertical scrollbar appearing changes the
available width and the scale can oscillate. Keep both panes mounted when switching tabs on narrow
screens; the observer re-fits automatically when a hidden pane becomes visible.

Why JS rather than pure CSS: CSS can't yet reliably divide one length by another to get a
unitless scale factor across browsers, so `calc(100cqi / 794px)` is not dependable. ResizeObserver
is cheap, runs only on resize, and works everywhere.

## Alternatives

- **CSS `zoom`** (standardised; in all major engines since Firefox 126, 2024) *does* change layout
  size, so no wrapper sizing is needed: `style="zoom: 0.6"`. Caveats: it affects
  `getBoundingClientRect`/pointer coordinates differently from transform, and some libraries
  (drag handles, canvas, rich-text editors) mis-measure inside it. Fine for read-only previews;
  prefer transform for interactive canvases.
- **Container-query units for content that should scale with its box without a fixed source
  size** — e.g. a card where text is `font-size: 4cqi`. Good for thumbnails; not for print-exact
  documents, because rounding at small sizes changes line breaks.
- **Iframe preview**: render the document in an iframe at real width and scale the iframe.
  Isolates CSS from the app (great for user-authored templates and email previews).

## Multi-page documents

Render each page as its own fixed-size box with a gap between, and scale the whole stack as
one unit. Decide page breaks from the real-size layout (or with print CSS:
`break-inside: avoid` on sections, `@page { size: A4; margin: 0 }`), never from the scaled one.

## Zoom controls

Offer Fit width / Fit page / 50 / 75 / 100 / 150%. Keyboard: Ctrl/Cmd + `+`/`-` are the browser's,
so don't hijack them; use buttons or `Ctrl+Alt` combinations. At zoom > fit, the preview pane's
`overflow-auto` provides panning; keep `min-w-0` on the pane so it can scroll instead of growing.

## Legibility floor

At very small scales a preview is decoration, not feedback. If `scale < ~0.4`, that's the signal
the layout should switch modes (tabs/sheet; see layout-patterns §2) rather than keep shrinking.
