<script lang="ts">
  import { onMount, onDestroy, tick } from 'svelte';
  import gsap from 'gsap';

  /**
   * Stakeholder review overlay.
   *
   * Pins a pulsing, numbered callout dot into the left margin beside each
   * annotated section. Clicking a dot slides a dark "design rationale" drawer
   * in from the left (GSAP) showing the copy. A bottom-left toggle hides/shows
   * the whole layer so the page can also be presented clean.
   *
   * Add a section by appending to `annotations` — `anchor` is the selector of
   * the element the dot aligns to vertically.
   */
  type Annotation = {
    num: string;
    label: string;
    anchor: string;
    body: string[];
  };

  const annotations: Annotation[] = [
    {
      num: '1',
      label: 'Hero',
      anchor: '.headline',
      body: [
        'The hero opens with one outcome-driven promise — “Ship your AI vision, faster” — so visitors grasp why this event matters within the first second, before they scroll.',
        'Event essentials live in a glassy eyebrow above the headline, while a looping ambient video carries the in-person energy without competing with the copy. Two CTAs split intent: a high-contrast Register Now for ready attendees, and a quieter View Agenda for those still evaluating.',
      ],
    },
    {
      num: '2',
      label: 'Why Attend',
      anchor: '.why-title',
      body: [
        'Directly below the fold we answer the visitor’s first question — “what’s in it for me?” — before asking for anything in return.',
        'Three benefit-led pillars (hands-on AI-native development, career growth, and learning from industry trailblazers) stay scannable at a glance, with supporting detail for anyone who wants to go deeper. A full-bleed photo of a real .local crowd grounds the value props in the experience of being in the room.',
      ],
    },
    {
      num: '3',
      label: 'Featured Speakers',
      anchor: '#speakers .section-header-title',
      body: [
        'People are the strongest proof point for an event, so we surface the lineup early. Leading with recognizable names — MongoDB’s CEO, well-known investors, and hands-on ML engineers — signals the caliber of the room at a glance.',
        'The mix is deliberate: aspirational star power (founders, VCs) alongside practitioners who actually ship AI in production, so the lineup reads as both worth attending and relevant to a working developer. Cards link out to full bios for anyone vetting the agenda before committing.',
      ],
    },
    {
      num: '4',
      label: 'Ticket Pricing',
      anchor: '.pricing-title',
      body: [
        'Pricing is shown openly rather than hidden behind a form — transparency lowers friction and builds trust. The tiered structure (Super Early Bird → General) makes the cost of waiting explicit, creating gentle urgency to register now.',
        'Pairing the tiers with a clear “every ticket includes” list reframes the decision from a cost into a value exchange, and the photo of a full session room reinforces the in-person payoff right before the final Register CTA.',
      ],
    },
    {
      num: '5',
      label: 'Featured Sessions',
      anchor: '#sessions .section-header-title',
      body: [
        'A preview of the agenda answers the practical question behind every registration: “is the content actually worth my day?” Concrete talks, named presenters, and topic pills let visitors quickly self-identify with a track.',
        'We show a representative sample rather than the full schedule to keep the page scannable, with a clear path to the complete agenda for those who want to go deeper before deciding.',
      ],
    },
  ];

  let reviewOn = $state(true);
  // Identity is keyed on the stable `num` string, not the object — assigning an
  // object into $state proxies it, which breaks `=== annotation` comparisons.
  let activeNum = $state<string | null>(null);
  const activeAnn = $derived(
    activeNum ? (annotations.find((x) => x.num === activeNum) ?? null) : null
  );
  let positions = $state(annotations.map(() => ({ top: -9999, ok: false })));

  let layerEl: HTMLDivElement;
  let backdropEl: HTMLDivElement;
  let drawerEl: HTMLElement;
  let closeEl: HTMLButtonElement;

  let openTl: gsap.core.Timeline | null = null;
  let ro: ResizeObserver | null = null;
  let rafId = 0;
  let lateTimers: ReturnType<typeof setTimeout>[] = [];
  let lastFocused: HTMLElement | null = null;

  const STORE_KEY = 'riff-review-notes';

  // Read a Flora motion token (seconds) so GSAP stays in sync with the CSS
  // tokens — including the reduced-motion overrides in global.css.
  function motionToken(varName: string, fallbackSeconds: number): number {
    if (typeof document === 'undefined') return fallbackSeconds;
    const v = getComputedStyle(document.documentElement).getPropertyValue(varName).trim();
    if (!v) return fallbackSeconds;
    return v.endsWith('ms') ? parseFloat(v) / 1000 : parseFloat(v) || fallbackSeconds;
  }

  // ---------- Positioning ----------
  function measure() {
    positions = annotations.map((a) => {
      const el = document.querySelector(a.anchor);
      if (!el) return { top: -9999, ok: false };
      const r = el.getBoundingClientRect();
      return { top: r.top + window.scrollY + r.height / 2, ok: true };
    });
  }

  function scheduleMeasure() {
    cancelAnimationFrame(rafId);
    rafId = requestAnimationFrame(measure);
  }

  // ---------- Dot visibility (toggle) ----------
  function dots(): HTMLElement[] {
    if (!layerEl) return [];
    return Array.from(layerEl.querySelectorAll<HTMLElement>('.review-dot'));
  }

  function animateDotsIn() {
    const enter = motionToken('--flora-motion-duration-enter', 0.25);
    const stagger = motionToken('--flora-motion-delay-stagger', 0.06);
    gsap.fromTo(
      dots(),
      { autoAlpha: 0, scale: 0.4, x: -12 },
      { autoAlpha: 1, scale: 1, x: 0, duration: enter, stagger, ease: 'back.out(2)' }
    );
  }

  function toggleReview() {
    reviewOn = !reviewOn;
    try {
      localStorage.setItem(STORE_KEY, reviewOn ? '1' : '0');
    } catch {}

    if (reviewOn) {
      tick().then(animateDotsIn);
    } else {
      if (activeNum) closeDrawer();
      const exit = motionToken('--flora-motion-duration-exit', 0.2);
      gsap.to(dots(), { autoAlpha: 0, scale: 0.4, duration: exit, ease: 'power2.in' });
    }
  }

  // ---------- Drawer ----------
  function buildOpenTl() {
    openTl?.kill();
    const enter = motionToken('--flora-motion-duration-enter', 0.25);
    const stagger = motionToken('--flora-motion-delay-stagger', 0.06);
    const items = drawerEl.querySelectorAll('[data-stagger]');

    openTl = gsap.timeline({
      paused: true,
      onReverseComplete: () => {
        gsap.set([backdropEl, drawerEl], { visibility: 'hidden' });
        activeNum = null;
        document.body.style.overflow = '';
        lastFocused?.focus?.();
        lastFocused = null;
      },
    });
    openTl.set([backdropEl, drawerEl], { visibility: 'visible' });
    openTl.to(backdropEl, { autoAlpha: 1, duration: enter, ease: 'power2.out' }, 0);
    openTl.fromTo(
      drawerEl,
      { xPercent: -100 },
      { xPercent: 0, duration: enter * 1.2, ease: 'power3.out' },
      0
    );
    openTl.fromTo(
      items,
      { autoAlpha: 0, x: -18 },
      { autoAlpha: 1, x: 0, duration: enter, stagger, ease: 'power2.out' },
      enter * 0.45
    );
  }

  // Re-run just the content stagger when swapping between dots while open.
  function restagger() {
    const enter = motionToken('--flora-motion-duration-enter', 0.25);
    const stagger = motionToken('--flora-motion-delay-stagger', 0.06);
    gsap.fromTo(
      drawerEl.querySelectorAll('[data-stagger]'),
      { autoAlpha: 0, x: -18 },
      { autoAlpha: 1, x: 0, duration: enter, stagger, ease: 'power2.out' }
    );
  }

  async function openDrawer(a: Annotation, trigger: HTMLElement) {
    // Toggle closed if the same dot is clicked again.
    if (activeNum === a.num) {
      closeDrawer();
      return;
    }
    // Swap content if a different dot is clicked while open.
    if (activeNum) {
      activeNum = a.num;
      await tick();
      restagger();
      return;
    }

    lastFocused = trigger;
    activeNum = a.num;
    await tick();
    document.body.style.overflow = 'hidden';
    buildOpenTl();
    openTl?.timeScale(1).play(0);
    closeEl?.focus({ preventScroll: true });
  }

  function closeDrawer() {
    if (!activeNum || !openTl) return;
    const enter = motionToken('--flora-motion-duration-enter', 0.25);
    const exit = motionToken('--flora-motion-duration-exit', 0.2);
    openTl.timeScale(exit > 0 ? (enter * 1.2) / exit : 1).reverse();
  }

  function onKey(e: KeyboardEvent) {
    if (e.key === 'Escape' && activeNum) closeDrawer();
  }

  onMount(() => {
    try {
      reviewOn = localStorage.getItem(STORE_KEY) !== '0';
    } catch {}

    measure();
    if (reviewOn) tick().then(animateDotsIn);

    ro = new ResizeObserver(scheduleMeasure);
    ro.observe(document.body);
    window.addEventListener('resize', scheduleMeasure, { passive: true });
    window.addEventListener('load', measure);
    document.fonts?.ready.then(measure).catch(() => {});
    // Late passes catch lazy images / font swaps that shift layout.
    lateTimers = [setTimeout(measure, 300), setTimeout(measure, 1200)];

    window.addEventListener('keydown', onKey);
  });

  onDestroy(() => {
    ro?.disconnect();
    cancelAnimationFrame(rafId);
    lateTimers.forEach(clearTimeout);
    openTl?.kill();
    window.removeEventListener('resize', scheduleMeasure);
    window.removeEventListener('load', measure);
    window.removeEventListener('keydown', onKey);
    if (typeof document !== 'undefined') document.body.style.overflow = '';
  });
</script>

<!-- Dot layer: spans the document, pinned in the left margin -->
<div bind:this={layerEl} class="review-layer" class:hidden={!reviewOn} aria-hidden={!reviewOn}>
  {#each annotations as a, i (a.num)}
    <!-- Positioning wrapper: Svelte owns `top` here. The button inside is owned
         by GSAP (opacity/scale/transform). Keeping them on separate elements
         stops a measure-triggered style rewrite from wiping GSAP's inline
         styles (which would otherwise hide every dot on resize). -->
    <div class="review-dot-anchor" style="top: {positions[i].top - 15}px">
      <button
        class="review-dot"
        class:is-active={activeNum === a.num}
        type="button"
        tabindex={reviewOn ? 0 : -1}
        aria-label={`Open design rationale: ${a.label}`}
        aria-expanded={activeNum === a.num}
        onclick={(e) => openDrawer(a, e.currentTarget as HTMLElement)}
      >
        <span class="review-dot__num">{a.num}</span>
        <span class="review-dot__hint">{a.label}</span>
      </button>
    </div>
  {/each}
</div>

<!-- Backdrop + drawer -->
<div
  bind:this={backdropEl}
  class="review-backdrop"
  onclick={closeDrawer}
  aria-hidden="true"
></div>

<aside
  bind:this={drawerEl}
  class="review-drawer"
  role="dialog"
  aria-modal="true"
  aria-labelledby="review-drawer-title"
>
  <div class="review-drawer__glow" aria-hidden="true"></div>
  <button bind:this={closeEl} class="review-drawer__close" type="button" aria-label="Close" onclick={closeDrawer}>
    <svg viewBox="0 0 24 24" width="20" height="20" fill="none" aria-hidden="true">
      <path d="M6 6l12 12M18 6L6 18" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
    </svg>
  </button>

  {#if activeAnn}
    <div class="review-drawer__inner">
      <span class="review-drawer__num">{activeAnn.num}</span>
      <p class="review-drawer__eyebrow" data-stagger>Design rationale</p>
      <div class="review-drawer__head" data-stagger>
        <h2 class="review-drawer__title" id="review-drawer-title">{activeAnn.label}</h2>
      </div>
      <div class="review-drawer__body">
        {#each activeAnn.body as p}
          <p data-stagger>{p}</p>
        {/each}
      </div>
    </div>
  {/if}
</aside>

<!-- Toggle — docks half-hidden at the bottom edge; lifts + brightens on hover -->
<button
  class="review-fab"
  class:is-on={reviewOn}
  type="button"
  aria-pressed={reviewOn}
  onclick={toggleReview}
>
  <span class="review-fab__dot" aria-hidden="true"></span>
  <span class="review-fab__label">{reviewOn ? 'Hide notes' : 'Review notes'}</span>
</button>

<style>
  /* ---------- Dot layer ---------- */
  .review-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 0; /* children are absolutely positioned in document coords */
    z-index: 40;
    pointer-events: none;
    --dot-left: clamp(12px, 3.2vw, 40px);
    --review-green: #6fce00;
    --review-green-light: #98ff00;
    --review-green-deep: #3f8a00;
  }

  /* Wrapper holds the document-Y position (set by Svelte via inline `top`).
     The dot's vertical center is baked into `top` (centerY − half height). */
  .review-dot-anchor {
    position: absolute;
    left: var(--dot-left);
    pointer-events: none;
  }

  .review-dot {
    position: relative;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 30px;
    height: 30px;
    padding: 0;
    border: 0;
    background: transparent;
    cursor: pointer;
    color: #ffffff;
    /* GSAP owns the fade/scale/visibility; start hidden to avoid a flash before
       it runs. Interactivity is gated by the layer's .hidden class below. */
    opacity: 0;
    visibility: hidden;
    pointer-events: none;
  }

  .review-layer:not(.hidden) .review-dot {
    pointer-events: auto;
  }

  .review-dot__num {
    position: relative;
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 30px;
    height: 30px;
    border-radius: 999px;
    background: linear-gradient(
      135deg,
      var(--review-green-light) 0%,
      var(--review-green) 48%,
      var(--review-green-deep) 100%
    );
    box-shadow:
      0 2px 10px rgba(79, 156, 0, 0.5),
      inset 0 0 0 1px rgba(0, 30, 43, 0.2);
    font-family: 'Source Code Pro', monospace;
    font-size: 12px;
    font-weight: 600;
    line-height: 1;
    color: #001e2b;
    transition: transform var(--flora-motion-duration-feedback) var(--flora-motion-easing-emphasized);
  }

  /* Pulsing ring */
  .review-dot__num::before {
    content: '';
    position: absolute;
    inset: 0;
    border-radius: 999px;
    border: 2px solid var(--review-green-light);
    opacity: 0;
  }

  @media (prefers-reduced-motion: no-preference) {
    .review-dot__num::before {
      animation: review-pulse 2.4s var(--flora-motion-easing-emphasized) infinite;
    }
  }

  @keyframes review-pulse {
    0% {
      transform: scale(1);
      opacity: 0.7;
    }
    70% {
      transform: scale(2.2);
      opacity: 0;
    }
    100% {
      transform: scale(2.2);
      opacity: 0;
    }
  }

  .review-dot:hover .review-dot__num,
  .review-dot:focus-visible .review-dot__num,
  .review-dot.is-active .review-dot__num {
    transform: scale(1.12);
  }

  .review-dot:focus-visible {
    outline: none;
  }

  .review-dot:focus-visible .review-dot__num {
    box-shadow:
      0 2px 10px rgba(79, 156, 0, 0.45),
      0 0 0 4px rgba(152, 255, 0, 0.5);
  }

  /* Hover label — absolutely positioned so it stays out of the button's
     layout box, keeping the entrance scale anchored on the numbered circle. */
  .review-dot__hint {
    position: absolute;
    left: calc(100% + 12px);
    top: 50%;
    display: inline-flex;
    align-items: center;
    height: 28px;
    padding: 0 12px;
    border-radius: 999px;
    background: rgba(5, 20, 29, 0.92);
    border: 1px solid rgba(152, 255, 0, 0.3);
    font-family: 'Euclid Circular A', sans-serif;
    font-size: 13px;
    font-weight: 500;
    white-space: nowrap;
    color: #ffffff;
    opacity: 0;
    transform: translate(-8px, -50%);
    pointer-events: none;
    transition:
      opacity var(--flora-motion-duration-feedback) var(--flora-motion-easing-emphasized),
      transform var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized);
  }

  .review-dot:hover .review-dot__hint,
  .review-dot:focus-visible .review-dot__hint,
  .review-dot.is-active .review-dot__hint {
    opacity: 1;
    transform: translateX(0);
  }

  /* ---------- Backdrop ---------- */
  .review-backdrop {
    position: fixed;
    inset: 0;
    z-index: 90;
    background: rgba(0, 12, 18, 0.55);
    backdrop-filter: blur(2px);
    -webkit-backdrop-filter: blur(2px);
    opacity: 0;
    visibility: hidden;
  }

  /* ---------- Drawer ---------- */
  .review-drawer {
    position: fixed;
    top: 0;
    left: 0;
    z-index: 100;
    width: min(440px, 92vw);
    height: 100dvh;
    background: #05141d;
    border-right: 1px solid rgba(152, 255, 0, 0.18);
    box-shadow: 24px 0 60px rgba(0, 0, 0, 0.5);
    overflow-y: auto;
    /* Hidden until opened. The off-screen slide offset is driven entirely by
       GSAP (xPercent) — keeping it out of CSS avoids the computed-matrix `x`
       leftover that would otherwise pin the panel off-screen. */
    visibility: hidden;
  }

  .review-drawer__glow {
    position: absolute;
    top: -120px;
    left: -120px;
    width: 360px;
    height: 360px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(152, 255, 0, 0.35) 0%, transparent 70%);
    pointer-events: none;
  }

  .review-drawer__inner {
    position: relative;
    padding: clamp(28px, 5vw, 48px);
    padding-top: clamp(64px, 8vw, 88px);
  }

  .review-drawer__close {
    position: absolute;
    top: 20px;
    right: 20px;
    z-index: 2;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 40px;
    height: 40px;
    border: 1px solid rgba(255, 255, 255, 0.14);
    border-radius: 999px;
    background: rgba(255, 255, 255, 0.04);
    color: #c1c7c6;
    cursor: pointer;
    transition:
      background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      color var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      border-color var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .review-drawer__close:hover {
    background: rgba(152, 255, 0, 0.12);
    border-color: rgba(152, 255, 0, 0.4);
    color: #ffffff;
  }

  .review-drawer__eyebrow {
    margin: 16px 0 24px;
    font-family: 'Source Code Pro', monospace;
    font-size: 12px;
    font-weight: 500;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: #98ff00;
  }

  .review-drawer__head {
    display: flex;
    align-items: center;
    gap: 16px;
    margin-bottom: 24px;
  }

  .review-drawer__num {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 44px;
    height: 44px;
    border-radius: 999px;
    background: linear-gradient(135deg, #98ff00 0%, #6fce00 48%, #3f8a00 100%);
    box-shadow: inset 0 0 0 1px rgba(0, 30, 43, 0.2);
    font-family: 'Source Code Pro', monospace;
    font-size: 15px;
    font-weight: 600;
    color: #001e2b;
  }

  .review-drawer__title {
    margin: 0;
    font-family: 'Euclid Circular A', sans-serif;
    font-weight: 600;
    font-size: 28px;
    line-height: 1.15;
    color: #ffffff;
  }

  .review-drawer__body {
    display: flex;
    flex-direction: column;
    gap: 20px;
  }

  .review-drawer__body p {
    margin: 0;
    font-family: 'Euclid Circular A', sans-serif;
    font-size: 16px;
    line-height: 28px;
    color: #b8c4c2;
  }

  /* ---------- Toggle ---------- */
  .review-fab {
    position: fixed;
    left: clamp(16px, 3vw, 32px);
    bottom: 0;
    z-index: 80;
    display: inline-flex;
    align-items: center;
    gap: 0;
    padding: 13px 16px;
    border-radius: 14px 14px 0 0;
    border: 1px solid rgba(152, 255, 0, 0.18);
    background: rgba(5, 20, 29, 0.72);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    box-shadow: 0 -4px 20px rgba(0, 0, 0, 0.3);
    color: #ffffff;
    font-family: 'Euclid Circular A', sans-serif;
    font-size: 14px;
    font-weight: 500;
    line-height: 1;
    white-space: nowrap;
    cursor: pointer;
    /* Docked: tucked ~halfway below the viewport edge, peeking as a handle.
       The label collapses so only the pulsing dot shows when docked. */
    opacity: 0.82;
    transform: translateY(50%);
    transition:
      transform var(--flora-motion-duration-enter) var(--flora-motion-easing-expressive),
      opacity var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized),
      border-radius var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized),
      background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      border-color var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      box-shadow var(--flora-motion-duration-moderate) var(--flora-motion-easing-emphasized);
  }

  /* Invisible hover bridge: when the button lifts on hover it moves up and away
     from the cursor near the bottom edge — without this the cursor would fall
     into the gap, drop :hover, and the button would flicker up/down. The bridge
     extends the hit area down to (and past) the viewport edge. At rest it sits
     below the fold, so it never intercepts page clicks. */
  .review-fab::after {
    content: '';
    position: absolute;
    left: 0;
    top: 100%;
    width: 100%;
    height: 72px;
    background: transparent;
  }

  /* :focus (not just :focus-visible) so a tap on touch also reveals the label */
  .review-fab:hover,
  .review-fab:focus {
    outline: none;
    opacity: 1;
    transform: translateY(-14px);
    border-radius: 999px;
    border-color: rgba(152, 255, 0, 0.55);
    background: rgba(8, 26, 37, 0.96);
    box-shadow:
      0 14px 34px rgba(0, 0, 0, 0.45),
      0 0 0 4px rgba(152, 255, 0, 0.18);
  }

  .review-fab__label {
    max-width: 0;
    margin-left: 0;
    overflow: hidden;
    opacity: 0;
    transition:
      max-width var(--flora-motion-duration-enter) var(--flora-motion-easing-expressive),
      margin-left var(--flora-motion-duration-enter) var(--flora-motion-easing-expressive),
      opacity var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .review-fab:hover .review-fab__label,
  .review-fab:focus .review-fab__label {
    max-width: 120px;
    margin-left: 10px;
    opacity: 1;
  }

  .review-fab__dot {
    position: relative;
    width: 10px;
    height: 10px;
    border-radius: 999px;
    background: #5d6c74;
    transition: background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .review-fab.is-on .review-fab__dot {
    background: linear-gradient(135deg, #98ff00, #6fce00);
  }

  @media (prefers-reduced-motion: no-preference) {
    .review-fab.is-on .review-fab__dot::after {
      content: '';
      position: absolute;
      inset: 0;
      border-radius: 999px;
      border: 2px solid #98ff00;
      animation: review-pulse 2.4s var(--flora-motion-easing-emphasized) infinite;
    }
  }
</style>
