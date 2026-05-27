# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm dev       # Start dev server at localhost:4321
pnpm build     # Build production site to ./dist/
pnpm preview   # Preview production build locally
pnpm astro     # Run Astro CLI commands (e.g. pnpm astro add)
```

Package manager is **pnpm**. Node >= 22.12.0 required.

No test or lint scripts are configured yet.

## Stack

- **Astro 6** — static site generator with file-based routing (`src/pages/` → URL routes)
- **Svelte 5** — interactive component layer; use `.svelte` files for stateful/reactive UI
- **TypeScript** — strict mode via `astro/tsconfigs/strict`
- **Tailwind CSS 4** — loaded as a Vite plugin (not PostCSS), configured in `astro.config.mjs`

## Architecture

### Component model

Astro pages wrap content in `src/layouts/Layout.astro` via `<slot />`. Pages live in `src/pages/`, interactive islands use Svelte with Astro's `client:*` directives.

### Design system

`src/styles/global.css` defines a comprehensive CSS variable system with MongoDB design tokens:

- **Colors** — `--color-{name}-{shade}` (10 shades in oklch for black, green, blue, red, purple, yellow)
- **Typography** — `--font-{heading|body|mono}`, `--text-{h1–h6}`, `--text-{xl|lg|md|sm|xs}`
- **Spacing** — `--space-{0–12}` scale
- **Shadows** — `--shadow-{sm|md|lg|xl}`
- **Themes** — `.dark` and `.brand` class variants

**Fonts loaded from `/public/fonts/`:** Euclid Circular A (body), MongoDB Value Serif (headings), Source Code Pro (mono). Font faces declared in `/public/fonts.css`.

Use these CSS variables rather than raw Tailwind color/spacing values to stay consistent with the design system.
