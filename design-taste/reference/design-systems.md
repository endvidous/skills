# Design systems

Which official design system (or deliberate aesthetic direction) fits a
given brief, so a build doesn't reinvent a component library that already
exists for the platform it's targeting.

## Brief type → official system

| Brief signals | System | Why |
|---|---|---|
| Enterprise/internal Microsoft-ecosystem tool | Fluent UI (React or Web Components) | Matches Windows/Office visual language and behavior |
| General cross-platform product UI, no strong brand pull | Material Web | Broadest community support, strong a11y defaults |
| Enterprise/data-dense B2B, especially IBM-adjacent | Carbon | Built for dense data and enterprise workflows |
| E-commerce admin/merchant tooling | Shopify Polaris | Purpose-built for commerce operations UI |
| Jira/Confluence-adjacent or Atlassian-ecosystem tool | Atlaskit | Matches Atlassian product family conventions |
| Developer tool, GitHub-adjacent | Primer (CSS or Brand) | Matches GitHub's visual language |
| UK public sector | GOV.UK Frontend | Legally/conventionally expected for UK gov services |
| US public sector / federal | USWDS | Legally/conventionally expected for US federal sites |
| Fast-prototype, no strong brand needs | Bootstrap | Fastest to a working, familiar UI |
| Headless, full custom visual control needed | Radix Themes or shadcn/ui | Unstyled/lightly-styled primitives, accessible behavior included |
| Fast custom build, utility-first | Tailwind (default for most greenfield web work) | No fighting a component library's opinions |

## Aesthetic-not-system directions

These aren't official component libraries — they're genre/aesthetic
choices to build from scratch with the system above as the behavioral
substrate:

- **Glassmorphism** — translucent, blurred-background panels layered over
  a colorful ground. Use `backdrop-filter` deliberately, bounded to small
  areas (expensive to composite at scale).
- **Bento** — asymmetric grid of variously-sized content cells, each
  self-contained. Cell count and sizing must map to real content groups
  (see anti-slop-tells.md).
- **Brutalism/neobrutalism** — raw, unstyled-looking, hard offset shadows,
  visible borders, monospace accents. The one place hard `box-shadow: 4px
  4px 0` shadows are earned by default.
- **Editorial** — magazine-style typographic hierarchy, generous
  whitespace, asymmetric layout, often serif display type.
- **Dark tech** — near-black grounds, one neon/saturated accent, glowing
  edges. Recognizable enough to be a default itself if reached for
  unprompted — see anti-slop-tells.md's calibration note.
- **Aurora/mesh gradients** — soft, blurred multi-color gradient
  backgrounds. Effective sparingly; becomes a tell fast when it's the
  default background for everything.
- **Kinetic type** — typography as the primary animated/interactive
  element, not just static display text.
- **Apple-style "liquid glass"** — there is no official `liquid-glass.css`
  or shipped library for this look. Approximate it honestly: layered
  `backdrop-filter` blur, layered borders/shadows for the glass edge, a
  dark-mode variant, and a `prefers-reduced-transparency` fallback that
  drops to a solid surface. Never claim or imply it's using Apple's actual
  implementation.

## Install references (verify current package names before using — these
drift)

- Material Web: `@material/web`
- Fluent UI React: `@fluentui/react-components`; Fluent Web Components:
  `@fluentui/web-components`
- Carbon: `@carbon/react` (or `carbon-components` for the vanilla CSS
  build)
- Shopify Polaris: `@shopify/polaris`
- Atlaskit: individual `@atlaskit/*` packages per component, no single
  meta-package
- Primer: `@primer/react` (React) or `@primer/css` (CSS-only)
- GOV.UK Frontend: `govuk-frontend`
- USWDS: `@uswds/uswds`
- Bootstrap: `bootstrap`
- Radix Themes: `@radix-ui/themes`
- shadcn/ui: not an npm package — components are copied into the project
  via its CLI (`npx shadcn@latest add <component>`)

Always verify a package still exists and check its current major version
before assuming an install command works — these ecosystems move.

## Canonical documentation sources

Check the official docs site for each system rather than trusting a
remembered API shape — component APIs and props drift across major
versions. Prefer the system's own docs domain over third-party tutorials
when verifying an exact prop or class name.

## Apple Liquid Glass — honest web approximation

Since no official web library exists, a reasonable from-scratch
approximation:

```css
.liquid-glass-web-approx {
  background: rgba(255, 255, 255, 0.12);
  backdrop-filter: blur(20px) saturate(180%);
  -webkit-backdrop-filter: blur(20px) saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.25);
  border-radius: 20px;
  box-shadow:
    0 8px 32px rgba(0, 0, 0, 0.15),
    inset 0 1px 0 rgba(255, 255, 255, 0.3);
}

@media (prefers-color-scheme: dark) {
  .liquid-glass-web-approx {
    background: rgba(20, 20, 22, 0.35);
    border-color: rgba(255, 255, 255, 0.12);
  }
}

@media (prefers-reduced-transparency: reduce) {
  .liquid-glass-web-approx {
    background: var(--surface-solid, #f2f2f2);
    backdrop-filter: none;
    -webkit-backdrop-filter: none;
  }
}
```

This is a labeled approximation, not a reproduction of Apple's actual
rendering — never present it to a user or in copy as "Apple's Liquid
Glass," only as inspired by it.

## Verify

- The chosen system matches the platform/ecosystem the product actually
  lives in, not just visual taste.
- Package names and install commands were checked against current docs,
  not assumed from memory.
- An aesthetic direction (bento, glass, brutalism) was chosen because the
  brief calls for it, not reached for as a default — cross-check against
  [anti-slop-tells.md](anti-slop-tells.md).
