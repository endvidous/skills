# Images, Video and Embeds

## Baseline (apply globally)

```css
img, video, canvas, svg, iframe { max-width: 100%; height: auto; }
```

Tailwind's preflight already does this for `img`/`video` (and makes them `display:block`).

## Prevent layout shift

Always give images their intrinsic dimensions so the browser reserves space before load:

```html
<img src="hero.jpg" width="1600" height="900" alt="…" class="h-auto w-full">
```

For boxes whose shape you control, use `aspect-ratio` + `object-fit`:

```html
<div class="aspect-[4/3] overflow-hidden rounded-lg">
  <img src="…" alt="…" class="size-full object-cover">
</div>
```

## Resolution switching (same image, different sizes)

```html
<img src="card-800.jpg"
     srcset="card-400.jpg 400w, card-800.jpg 800w, card-1600.jpg 1600w"
     sizes="(min-width: 80rem) 25rem, (min-width: 48rem) 33vw, 100vw"
     width="800" height="600" alt="…" loading="lazy" decoding="async">
```

`sizes` must describe the image's *rendered* width at each layout — if a card grid shows 3
columns at ≥48rem, the image is ~33vw there. Wrong `sizes` = downloading 4× too many pixels.

## Art direction (different crop per layout)

```html
<picture>
  <source media="(min-width: 64rem)" srcset="hero-wide.avif" type="image/avif">
  <source media="(min-width: 64rem)" srcset="hero-wide.jpg">
  <img src="hero-tall.jpg" width="800" height="1000" alt="…">
</picture>
```

## Loading

- Above-the-fold hero: no `loading="lazy"`; add `fetchpriority="high"`.
- Everything else: `loading="lazy" decoding="async"`.
- Prefer AVIF/WebP with a JPEG/PNG fallback via `<picture>`.

## Video and iframes

```html
<div class="aspect-video w-full"><iframe class="size-full" src="…" title="…" loading="lazy"></iframe></div>
<video class="aspect-video w-full" playsinline muted controls poster="poster.jpg">…</video>
```

`playsinline` stops iOS forcing full-screen. Autoplaying background video: also `muted loop`, and
respect `prefers-reduced-motion` by pausing it.

## Icons and SVG

- Inline SVG icons sized in `em` (`size-[1.25em]`) scale with adjacent text.
- Give SVGs a `viewBox` and no fixed width/height attributes if they should be fluid.

## Backgrounds

`background-size: cover` crops unpredictably across aspect ratios; pin the important part with
`background-position` (`bg-[position:30%_20%]`) and check both portrait and landscape.
