---
name: MongoDB Marketing Design System
description: Flora-based visual system powering MongoDB's marketing surfaces and exploratory design work.
colors:
  deep-space: "oklch(22.01% 0.044 230.19)"
  forest-deep: "oklch(37.77% 0.073 171.71)"
  forest-floor: "oklch(29.32% 0.05 186.49)"
  charged-signal: "oklch(82.54% 0.237 148.39)"
  pine-active: "oklch(45.84% 0.096 165.11)"
  cool-mist: "oklch(79% 0.012 195)"
  pure-white: "oklch(100% 0 263.28)"
typography:
  display:
    fontFamily: "Euclid Circular A, system-ui, sans-serif"
    fontSize: "clamp(3rem, 7vw, 6rem)"
    fontWeight: 400
    lineHeight: 1.1
    letterSpacing: "-0.01em"
  headline:
    fontFamily: "MongoDB Value Serif, Georgia, serif"
    fontSize: "clamp(2.5rem, 5vw, 3.5rem)"
    fontWeight: 400
    lineHeight: 1.14
    letterSpacing: "0"
  title:
    fontFamily: "Euclid Circular A, system-ui, sans-serif"
    fontSize: "clamp(1.75rem, 4vw, 2.25rem)"
    fontWeight: 500
    lineHeight: 1.33
    letterSpacing: "0"
  body:
    fontFamily: "Euclid Circular A, system-ui, sans-serif"
    fontSize: "1.125rem"
    fontWeight: 400
    lineHeight: 1.78
    letterSpacing: "0"
  label:
    fontFamily: "Euclid Circular A, system-ui, sans-serif"
    fontSize: "1rem"
    fontWeight: 500
    lineHeight: 1.5
    letterSpacing: "0"
  mono:
    fontFamily: "Source Code Pro, Menlo, monospace"
    fontSize: "1rem"
    fontWeight: 400
    lineHeight: 1.5
    letterSpacing: "0"
rounded:
  sm: "4px"
  md: "8px"
  lg: "16px"
  circle: "9999px"
spacing:
  xs: "8px"
  sm: "16px"
  md: "24px"
  lg: "32px"
  xl: "48px"
  2xl: "64px"
  section-y: "clamp(4rem, 9vw, 6rem)"
  gutter-x: "clamp(1rem, 8%, 8.25rem)"
components:
  button-primary:
    backgroundColor: "{colors.charged-signal}"
    textColor: "{colors.deep-space}"
    rounded: "{rounded.sm}"
    padding: "16px 32px"
  button-primary-hover:
    backgroundColor: "oklch(73.5% 0.237 148.39)"
    rounded: "{rounded.circle}"
  button-secondary:
    backgroundColor: "transparent"
    textColor: "{colors.pure-white}"
    rounded: "{rounded.sm}"
    padding: "16px 32px"
  button-secondary-hover:
    backgroundColor: "rgba(255,255,255,0.08)"
    rounded: "{rounded.circle}"
  nav-cta:
    backgroundColor: "{colors.deep-space}"
    textColor: "{colors.pure-white}"
    rounded: "{rounded.sm}"
    padding: "16px 32px"
  nav-cta-hover:
    backgroundColor: "{colors.pine-active}"
    rounded: "{rounded.circle}"
---

# Design System: MongoDB Marketing

## 1. Overview

**Creative North Star: "The Signal in the Dark"**

MongoDB's marketing system operates on a specific material logic: deep navy as the medium, electric green as the transmission. Surfaces are dark not because it's fashionable but because that contrast is load-bearing — MongoDB green reads as a live indicator against it, exactly as it would in a terminal or a monitoring dashboard. Sections don't glow or float; they hold. The system has the weight of infrastructure and the directness of tooling.

This design system is the visual foundation for all MongoDB marketing surfaces: event pages, product marketing, campaign work, technical content, and the exploratory riffs that feed into mongodb.com. It is built on MongoDB's Flora token system — a comprehensive set of OKLCH color values, spacing increments, typographic roles, and motion tokens that has been production-hardened across the real marketing site. Explorations in this repo riff within Flora's constraints; they don't replace them.

What this system rejects: SaaS conference furniture (hero metrics, sponsor logo walls, glassmorphism cards), startup growth-marketing aesthetics (gradient text, superlatives, emoji CTAs), dry enterprise design (gray, flat, joyless), and the AI-generated default aesthetic (cream/beige body backgrounds, Untitled UI component-library feel, eyebrow-on-every-section scaffolding that looks like it came out of a batch generation run).

**Key Characteristics:**
- Deep navy dominates surface area; electric green is an accent, not a wallpaper
- Typography plays two distinct instruments: Value Serif for editorial authority, Euclid Circular A for clarity and precision
- Motion follows Flora tokens — purposeful, calibrated, never decorative
- Layouts earn their asymmetry; photo bleeds and full-height splits are structural, not ornamental
- Every surface must answer: why would a technical visitor stop here?

## 2. Colors: The Signal Palette

A three-layer system: the deep field (navy), the signal (green), and the surface (white for light contexts). All values are OKLCH-native per the Flora token system; hex equivalents are listed for reference and SVG compatibility.

### Primary
- **Charged Signal** (`oklch(82.54% 0.237 148.39)` / #00ed64): MongoDB's brand green. Used for primary CTAs, hover states on links and navigation, active indicators, and benefit checkmarks. Its power comes from rarity — when it appears, it means something is live, active, or ready to act. Never used as a background for body sections.
- **Pine Active** (`oklch(45.84% 0.096 165.11)` / #00684a): The richer, darker green. Used for primary button borders, navigation hover text, and the nav CTA hover background. Grounds the accent without competing with Charged Signal at scale.

### Secondary
- **Forest Deep** (`oklch(37.77% 0.073 171.71)` / #023430): A deep forest-green tonal variant for alternative section backgrounds (currently: speakers section). Signals a surface shift without leaving the brand's green-black axis.
- **Forest Floor** (`oklch(29.32% 0.05 186.49)` / #013227): Deepest green surface. Card photo backgrounds and tertiary section surfaces in the brand theme.

### Neutral
- **Deep Space** (`oklch(22.01% 0.044 230.19)` / #001e2b): The primary dark surface. Near-black, tinted blue-green. Used for body backgrounds, the pricing section, mobile nav overlay, and the dominant surface on dark-theme pages. This is not a generic dark — its specific blue-green tint is brand-load-bearing.
- **Cool Mist** (`oklch(79% 0.012 195)` / #b8c4c2): Muted secondary text on dark backgrounds. Supporting copy, timestamps, subtitles, descriptors. Tinted slightly toward the brand's blue-green axis — never warm gray.
- **Pure White** (`oklch(100% 0 263.28)` / #ffffff): Light-theme body backgrounds, the floating pill nav surface, text on dark sections.

### Named Rules
**The One Green Rule.** Charged Signal covers ≤ 15% of any given screen. The deep navy is the dominant surface; the green is the indicator light. Flooding a hero or section background with brand green turns signal into noise.

**The No-Warmth Rule.** MongoDB's neutrals tilt blue-green, not warm gray. `#b8c4c2` (Cool Mist) is the correct muted text. Substituting a warm or yellow-gray alternative (`#c4bfb8`) breaks the brand temperature and makes the surface feel generic.

## 3. Typography

**Display Font:** Euclid Circular A (system-ui, sans-serif fallback)
**Headline Font:** MongoDB Value Serif (Georgia, serif fallback)
**Mono Font:** Source Code Pro (Menlo, monospace fallback)

**Character:** The pairing creates a productive tension. Value Serif carries institutional weight — the company that has been handling the world's most important data since 2007. Euclid Circular A delivers technical clarity — the engineer who ships at pace. They never compete on the same surface: Value Serif is reserved for editorial display moments, Euclid Circular A owns all UI text, labels, and body copy.

### Hierarchy
- **Display** (Euclid Circular A, weight 400, `clamp(3rem, 7vw, 6rem)`, line-height 1.1, letter-spacing -0.01em): Hero headline only. Uppercase treatment is used here because scale and all-caps together create impact. Do not use uppercase below Display size.
- **Headline** (MongoDB Value Serif, weight 400, `clamp(2.5rem, 5vw, 3.5rem)`, line-height 1.14): Section headings that require editorial weight — pricing titles, featured content headings, prominent labels. The serif creates a material break from surrounding Euclid text; use it when hierarchy needs an instrument change, not just a size change.
- **Title** (Euclid Circular A, weight 500, `clamp(1.75rem, 4vw, 2.25rem)`, line-height 1.33): Section headers, sub-section labels. Medium weight, not bold — holds authority without shouting.
- **Body Large** (Euclid Circular A, weight 400, 18px / 32px line-height): Lead paragraphs, feature descriptions, main section copy. Max line length 65–72ch.
- **Body Base** (Euclid Circular A, weight 400, 16px / 24px): Standard body, card text, navigation items, benefit lists, form fields.
- **Label / Caption** (Euclid Circular A, weight 400–500, 14px / 1.4): Timestamps, disclaimers, micro-copy. Letter-spacing 0.25px at this size.
- **Mono** (Source Code Pro, weight 400, 16px / 24px): Code snippets, version numbers, technical strings.

### Named Rules
**The Two-Instrument Rule.** Value Serif and Euclid Circular A are distinct instruments: use them in contrast, never in competition. A section using Value Serif for the h2 uses Euclid Circular A for everything else. Never use both at similar sizes on the same hierarchy level.

**The Uppercase Ceiling.** Uppercase is reserved for Display-scale headings (≥ 3rem) where scale already provides legibility. Labels, nav items, and body copy: always sentence case. All-caps at body size is unreadable and reads as shouting.

## 4. Elevation

This system uses **tonal surface layering** as its primary depth signal, with one ambient shadow reserved for the floating navigation. Content sections don't lift; they layer.

Section backgrounds step in tone: Pure White (light context) → Deep Space navy → Forest Deep green → Forest Floor. The gradient of tone across sections is the depth cue — not box-shadows. Sections feel grounded and architectural because they have weight, not lift.

The exception is the floating nav pill: it earns a shadow because it is genuinely positioned above the content and moves independently of the scroll.

### Shadow Vocabulary
- **Nav Float** (`0 3px 20px rgba(0,0,0,0.15)`): Ambient shadow under the floating pill nav when scrolled away from the viewport top. Wide blur, low opacity — diffuse ambient, not a hard edge.
- **Nav Stuck** (`0 2px 8px rgba(0,0,0,0.08)`): Reduced shadow when the nav collapses edge-to-edge. Structural minimum, indicating presence over content.
- **Button Glow Ring, Green** (`0 0 0 6px color-mix(in oklch, #00ed64 45%, transparent)`): Radial halo on primary CTA hover. Not a drop shadow — a spatial signal that the button's interaction radius has expanded. Disappears on active press.
- **Button Glow Ring, White** (`0 0 0 6px color-mix(in oklch, #ffffff 18%, transparent)`): Same treatment for secondary/ghost buttons on dark backgrounds.

### Named Rules
**The Flat-By-Default Rule.** Content sections have no box-shadow. If a card or section needs a shadow to feel present, the background color is wrong — fix the surface, don't add lift.

## 5. Components

### Buttons
The signature interaction: at rest, buttons are squared (4px radius); on hover, they morph to pill shape (9999px radius) with a simultaneous 1px upward lift and glow ring expansion. The shape change is the brand behavior — it signals the transition from static UI to ready-to-act state. Motion follows Flora tokens: 250ms `cubic-bezier(0, 0, 0.2, 1)` for the morph and lift, 150ms `ease` for the color shift. On press: `scale(0.985)` at 100ms instant — the feeling of committed action.

Each riff may override this interaction energy (see PRODUCT.md: "Let the riff decide"), but this is the documented baseline.

- **Shape at rest:** 4px radius (`{rounded.sm}`)
- **Shape on hover:** pill (`{rounded.circle}`) + `-1px` translateY
- **Primary:** Charged Signal (#00ed64) background, Deep Space (#001e2b) text, 1px Pine Active (#00684a) border, 16px/32px padding
- **Secondary / Ghost:** Transparent background, white text, 1px white border. Hover: 8% white tint + white glow ring
- **Nav CTA:** Deep Space background, white text, no visible border. Hover: Pine Active background + green glow ring

### Navigation
Floating pill: white background, 16px radius, Nav Float shadow, 12px/32px internal padding. On scroll past the top sentinel, collapses to edge-to-edge with 0px radius and Nav Stuck shadow — transition uses Flora enter duration (250ms) and movement easing. Logo left-aligned; nav links center-right; CTA button rightmost. Nav links: Euclid Circular A 500 16px, Deep Space color; hover shifts to Pine Active (150ms feedback easing). Mobile (≤1024px): hamburger toggle replaces nav links; tap triggers a GSAP-animated clip-path circle-expand overlay in Deep Space with 36px uppercase white links staggered in at Flora enter timing.

### Speaker / Media Cards
Borderless. Photo occupies a 16px radius container (`{rounded.lg}`) with a fixed aspect ratio (456×350 in the current riff). On hover: photo scales to 1.06 over 500ms expressive easing (`cubic-bezier(0.22, 1, 0.36, 1)`) — a slow, decelerated zoom that keeps moving briefly after the card has settled. Name: Euclid Circular A 500, 24px, white. Role/subtitle: 16px, Cool Mist. No card background or border chrome — the photo is the container. The borderless treatment is intentional; card borders here would downgrade the images to thumbnail status.

### Pricing Tiers
Table-style list, not a card grid. Tiers separated by hairline dividers (`rgba(255,255,255,0.16)` top/bottom borders). Tier name in Euclid Circular A 400 20px (white); price in MongoDB Value Serif `clamp(2.5rem, 5vw, 3.5rem)` (white). The typographic contrast between name and price is the visual weight — no card chrome needed. Secondary detail text (date ranges, disclaimers) in Cool Mist.

### Section Headers
Baseline-aligned flex row: heading left, "View all →" link right when present. Title: Euclid Circular A 500, `clamp(1.75rem, 4vw, 2.25rem)`, white. Description: 16px, Cool Mist. Arrow link: white → Charged Signal on hover, with inline SVG arrow translating +2px on hover (Flora enter easing).

### Feature List
Horizontal-rule–divided rows, not cards. Each feature: heading in Euclid Circular A 600 24–32px (white), body in 18px Cool Mist. Rows separated by 1px `rgba(255,255,255,0.16)` bottom border, 32px top/bottom padding. No card chrome. The border is the only visual container.

### Countdown Timer (Signature Component)
High-velocity custom digit-flip component. Documents the expressive ceiling for bespoke motion in this system. Not a reusable primitive — a showcase interaction that demonstrates how far the Flora expressive duration can be pushed within brand.

## 6. Do's and Don'ts

### Do:
- **Do** use Deep Space (`oklch(22.01% 0.044 230.19)` / #001e2b) as the default dark surface. Its specific blue-green tint is the brand. Substituting a generic `#111111` or `#1a1a1a` loses the identity immediately.
- **Do** keep Charged Signal green for interactive affordances and active states only. Its power is in rarity. When it appears, it must mean something.
- **Do** use MongoDB Value Serif for headings that require editorial weight, and Euclid Circular A for everything else. Two instruments, used in contrast, never in competition.
- **Do** follow Flora motion tokens for all transitions: `--flora-motion-duration-feedback` (150ms) for color shifts, `--flora-motion-duration-enter` (250ms) for elements appearing, `--flora-motion-duration-expressive` (500ms) for showcase moments like photo zoom.
- **Do** use tonal surface changes (navy → forest deep → forest floor) as the primary depth signal across sections. Reserve shadows for genuinely lifted elements (nav, modals, tooltips).
- **Do** let photos and media bleed to container or viewport edges when the layout calls for it. Cropped-and-padded images lose the spatial confidence this system is built to project.
- **Do** use horizontal-rule dividers (`rgba(255,255,255,0.16)`) instead of card chrome for list-based content (feature lists, pricing tiers). Card borders downgrade content to thumbnail status.

### Don't:
- **Don't** use gradient text (`background-clip: text` with a gradient). It reads as decorative noise. Emphasis is achieved through weight or size, not gradients.
- **Don't** use the SaaS conference template structure: hero metric with stats below, identical card grids, sponsor logo walls. These are the specific structures to avoid per PRODUCT.md.
- **Don't** use startup growth-marketing aesthetics: superlatives, emoji CTAs, "10,000+ builders" social proof counters, vague "AI-powered" claims. These undermine MongoDB's institutional authority.
- **Don't** soften the palette toward warm gray or beige. The neutral axis tilts blue-green. Cool Mist (`#b8c4c2`) is the correct muted text; warm gray alternatives (`#c4bfb8`) break the brand temperature and make the surface feel unowned.
- **Don't** use the AI-generated default aesthetic: cream/beige body backgrounds, glassmorphism cards (decorative blurs), Untitled UI-style components, small uppercase tracked eyebrow labels above every section.
- **Don't** use `border-left` or `border-right` wider than 1px as a colored accent on cards or callouts. Rewrite as full borders, background tints, or leading icons.
- **Don't** use all-caps body copy. Uppercase is reserved for Display-scale hero headings only.
- **Don't** exceed 15% surface coverage with Charged Signal green on any given screen. It loses signal when it becomes wallpaper.
- **Don't** add box-shadows to content sections or cards at rest. If a surface needs elevation to feel grounded, fix the background color assignment. Don't add lift.
