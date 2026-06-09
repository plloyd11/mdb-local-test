# Product

## Register

brand

## Users

Developers, architects, technical decision-makers, and engineering leaders who interact with MongoDB's marketing website. Depending on the surface: a developer evaluating MongoDB Atlas for a new project; a VP of Engineering comparing data platforms; a community member reading about a local event. They're technical enough to spot lazy design and skeptical enough to distrust hype. They've seen every SaaS marketing playbook.

This repo is also used by the MongoDB marketing design team for exploratory work: prototyping new page concepts, testing visual directions, riffing on layout or motion ideas before they go into the design system or production site. Some surfaces will be event pages; others will be product marketing, documentation-adjacent, or brand campaign work.

## Product Purpose

This is a design exploration workspace for MongoDB's marketing website. It exists to prototype, test, and refine visual directions using MongoDB's Flora design system — components, tokens, typography, and motion — in a live Astro + Svelte environment. The current riff is the MongoDB .local San Francisco 2026 event landing page (August 13, 2026 at The Midway), but the repo will host multiple explorations over time.

Success looks like: design directions that feel authentically MongoDB — not templated, not derivative — that could credibly ship on mongodb.com.

## Brand Personality

Confident, technical, high-energy. MongoDB's institutional authority (a company that handles the world's most important data) meets the pace and directness of the engineering teams that build with it. Not neutral, not enterprise-gray — a brand with a real point of view on where software is going.

## Anti-references

- Generic SaaS marketing templates: hero metrics, gradient text, sponsor logo walls, identical card grids
- Startup hype and growth-marketing aesthetics: emoji CTAs, superlatives, "10,000+ builders" social proof, vague "AI-powered" claims
- Dry enterprise design: gray layouts, dense info tables, no personality, no energy — looks like a procurement portal
- AI-generated default aesthetic: cream/beige backgrounds, glassmorphism cards, Untitled UI component-library feel, eyebrow-on-every-section scaffolding

## Design Principles

1. **MongoDB's brand, not a template wearing MongoDB colors.** Deep navy, charged green, Value Serif headings — used with intention, not as a skin over a generic layout.
2. **Specific over impressive.** Real feature names, real session titles, real people. Specific nouns and active verbs beat vague benefit claims every time.
3. **Motion as punctuation.** The Flora motion system is in place; use it to create rhythm and signal transitions, not to decorate. Every animated element should communicate something.
4. **Earn each section's presence.** No section exists to fill space. If a block can't answer "why would a visitor stop and read this?", cut or collapse it.
5. **High contrast, committed palette.** MongoDB green on deep navy is a power combination. Don't dilute it with gray softening. The color system carries the brand energy — use it at full intensity.

## Accessibility & Inclusion

WCAG 2.1 AA minimum. All body text ≥ 4.5:1 contrast. Large text and interactive elements ≥ 3:1. Reduced-motion honored via Flora token overrides (already in global.css). Video hero has a poster fallback. No information conveyed by color alone.
