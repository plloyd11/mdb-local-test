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
        'The hero section establishes a high-energy tone within the first three seconds. The headline frames the flagship series around an immediate, high-value outcome, while the countdown ticker and location anchor create implicit urgency to drive action.',
        'We deploy a dual-CTA strategy to segment traffic immediately: a high-contrast "Register Now" button paths high-intent users directly to checkout, while a secondary "View Agenda" button captures exploratory users. The underlying visual framework is purpose-built to house a high-impact sizzle reel, instantly signaling a premium, can\'t-miss event experience the moment the page loads.',
      ],
    },
    {
      num: '2',
      label: 'Why Attend',
      anchor: '.why-title',
      body: [
        'Rather than overwhelming users with a dense logistical schedule early on, we use a scannable, three-pillar benefit framework mapped to distinct attendee motivations.',
        'The adjacent high-energy photography acts as an intentional narrative layer. By showing real developers actively collaborating under the flagship branding, we humanize the tech ecosystem and allow prospective attendees to instantly visualize themselves in the room, turning abstract benefits into concrete emotional investment.',
      ],
    },
    {
      num: '3',
      label: 'Featured Speakers',
      anchor: '#speakers .section-header-title',
      body: [
        'People are the strongest proof point for an event, so we surface the lineup early. This section serves as a powerful credibility loop designed to eliminate buyer friction and build trust.',
        'The clean, uniform card layout is optimized for rapid scannability, proving at a single glance that the sheer caliber of expertise justifies the attendee\'s time and budget.',
      ],
    },
    {
      num: '4',
      label: 'Ticket Pricing',
      anchor: '.pricing-title',
      body: [
        'This component uses transparent, tiered pricing to leverage the psychological principle of loss aversion, incentivizing immediate registration to lock in the lowest rate.',
        'To overcome budget objections and streamline corporate expense approvals, we position a distinct "Every ticket includes" value checklist directly alongside the costs. This structure shifts the user\'s cognitive focus from an expense to a high-ROI investment, backed by immersive presentation imagery that reinforces the premium physical scale of a flagship event.',
      ],
    },
    {
      num: '5',
      label: 'Featured Sessions',
      anchor: '#sessions .section-header-title',
      body: [
        'This section provides the tangible proof points that back up our high-level marketing narrative. By highlighting concrete, tactical sessions across diverse, tagged tracks (Data Modeling, Analytics, Microservices), we allow different technical personas to quickly validate that the agenda is directly relevant to their daily workflows.',
        'It eliminates abstract uncertainty for middle-of-the-funnel visitors, using a "View all sessions" secondary action as a progressive disclosure path to keep the landing page light, fast, and hyper-scannable.',
      ],
    },
  ];

  // Stage-setting overview shown in a centered modal (auto-opens once per visitor).
  const intro = {
    eyebrow: 'Stakeholder review',
    title: 'Context',
    context: [
      'This interactive prototype introduces the new design framework for the MongoDB .local flagship event series, using the San Francisco event as our structural baseline. All copy, prices, and session tracks are currently placeholders.',
      'Before diving into visual aesthetics, this walkthrough is designed to align us on product strategy, user psychology, and conversion intent. Use the numbered markers down the left edge of the page to explore the design rationale behind each modular section.',
    ],
    goals: [
      'Cultivate High-Energy Event Narrative: We leverage photography and a high-impact sizzle reel to tell a cohesive, immersive story. This captures the energy of a live flagship experience to turn passive interest into active registrations.',
      'Maximize Immediate Conversion: Capture and communicate the core value proposition within the first three seconds of landing to drive registrations.',
      'Establish Instant Credibility: Leverage a structured matrix of industry-leading speakers, enterprise validation, and transparent pricing to eliminate friction and build buyer trust.',
      'Optimize for Scannability: Present a high-level, benefit-first narrative layout that allows casual browsers to scan quickly, while giving high-intent users low-friction pathways to dive deeper on demand.',
    ],
    cta: 'Start the walkthrough',
  };

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

  let overviewOpen = $state(false);
  let overlayEl: HTMLDivElement;
  let overviewEl: HTMLElement;
  let overviewCloseEl: HTMLButtonElement;
  let overviewCtaEl: HTMLButtonElement;

  let openTl: gsap.core.Timeline | null = null;
  let overviewTl: gsap.core.Timeline | null = null;
  let ro: ResizeObserver | null = null;
  let rafId = 0;
  let lateTimers: ReturnType<typeof setTimeout>[] = [];
  let lastFocused: HTMLElement | null = null;
  let overviewLastFocused: HTMLElement | null = null;

  const STORE_KEY = 'riff-review-notes';
  const INTRO_KEY = 'riff-review-intro-seen';

  // Below PUSH_MIN there's no room to push the page aside → fall back to overlay.
  const PUSH_MIN = 1024;
  let pushActive = false;
  let pinRaf = 0;

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

  // ---------- Push (page reflows aside instead of being covered) ----------
  function prefersPush(): boolean {
    return typeof window !== 'undefined' && window.innerWidth >= PUSH_MIN;
  }

  // Set the `--review-push` inset (px) that reflows the page. Because reflow
  // changes page height, the section being reviewed would drift — so we pin its
  // viewport position across the transition with a short rAF loop (which also
  // resolves in ~1 frame under reduced-motion, where the transition is instant).
  function setPush(px: number) {
    const durMs = motionToken('--flora-motion-duration-enter', 0.25) * 1000;
    const anchorEl = activeAnn ? document.querySelector(activeAnn.anchor) : null;
    const target = anchorEl ? anchorEl.getBoundingClientRect().top : null;

    document.documentElement.style.setProperty('--review-push', `${px}px`);

    cancelAnimationFrame(pinRaf);
    if (anchorEl && target != null) {
      const start = performance.now();
      const step = () => {
        const delta = anchorEl.getBoundingClientRect().top - target;
        if (Math.abs(delta) > 0.5) window.scrollBy(0, delta);
        if (performance.now() - start < durMs + 80) pinRaf = requestAnimationFrame(step);
      };
      pinRaf = requestAnimationFrame(step);
    }
  }

  function clearPush() {
    pushActive = false;
    document.documentElement.style.setProperty('--review-push', '0px');
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
        if (pushActive) clearPush();
        lastFocused?.focus?.();
        lastFocused = null;
      },
    });
    openTl.set(drawerEl, { visibility: 'visible' });
    // Push mode reflows the page aside, so skip the dim backdrop entirely —
    // the whole design stays visible. Overlay mode keeps the dimming.
    if (!pushActive) {
      openTl.set(backdropEl, { visibility: 'visible' });
      openTl.to(backdropEl, { autoAlpha: 1, duration: enter, ease: 'power2.out' }, 0);
    }
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

    pushActive = prefersPush();
    if (pushActive) {
      // Reflow the page aside; keep it scrollable/interactive (no scroll lock).
      setPush(drawerEl.getBoundingClientRect().width);
    } else {
      // Overlay fallback: dim + lock scroll (the original behavior).
      document.body.style.overflow = 'hidden';
    }

    buildOpenTl();
    openTl?.timeScale(1).play(0);
    closeEl?.focus({ preventScroll: true });
  }

  function closeDrawer() {
    if (!activeNum || !openTl) return;
    // Animate the page back into place in sync with the drawer sliding out.
    if (pushActive) setPush(0);
    const enter = motionToken('--flora-motion-duration-enter', 0.25);
    const exit = motionToken('--flora-motion-duration-exit', 0.2);
    openTl.timeScale(exit > 0 ? (enter * 1.2) / exit : 1).reverse();
  }

  // ---------- Overview modal ----------
  function buildOverviewTl() {
    overviewTl?.kill();
    const enter = motionToken('--flora-motion-duration-enter', 0.25);
    const stagger = motionToken('--flora-motion-delay-stagger', 0.06);
    const items = overviewEl.querySelectorAll('[data-ov-stagger]');

    overviewTl = gsap.timeline({
      paused: true,
      onReverseComplete: () => {
        gsap.set([overlayEl, overviewEl], { visibility: 'hidden' });
        overviewOpen = false;
        document.body.style.overflow = '';
        overviewLastFocused?.focus?.({ preventScroll: true });
        overviewLastFocused = null;
      },
    });
    overviewTl.set([overlayEl, overviewEl], { visibility: 'visible' });
    overviewTl.to(overlayEl, { autoAlpha: 1, duration: enter, ease: 'power2.out' }, 0);
    // Card transform is animated here — centering lives on the flex wrapper, not
    // a transform on the card, so GSAP never fights a CSS translate.
    overviewTl.fromTo(
      overviewEl,
      { autoAlpha: 0, y: 12, scale: 0.98 },
      { autoAlpha: 1, y: 0, scale: 1, duration: enter * 1.2, ease: 'power3.out' },
      0
    );
    overviewTl.fromTo(
      items,
      { autoAlpha: 0, y: 12 },
      { autoAlpha: 1, y: 0, duration: enter, stagger, ease: 'power2.out' },
      enter * 0.4
    );
  }

  async function openOverview(trigger?: HTMLElement) {
    if (overviewOpen) return;
    overviewLastFocused = trigger ?? (document.activeElement as HTMLElement | null);
    overviewOpen = true;
    await tick();
    document.body.style.overflow = 'hidden';
    buildOverviewTl();
    overviewTl?.timeScale(1).play(0);
    overviewCtaEl?.focus({ preventScroll: true });
  }

  function closeOverview() {
    if (!overviewOpen || !overviewTl) return;
    try {
      localStorage.setItem(INTRO_KEY, '1');
    } catch {}
    const enter = motionToken('--flora-motion-duration-enter', 0.25);
    const exit = motionToken('--flora-motion-duration-exit', 0.2);
    overviewTl.timeScale(exit > 0 ? (enter * 1.2) / exit : 1).reverse();
  }

  function onKey(e: KeyboardEvent) {
    if (e.key !== 'Escape') return;
    if (overviewOpen) closeOverview();
    else if (activeNum) closeDrawer();
  }

  function onResize() {
    scheduleMeasure();
    // If the drawer is open and the viewport crosses the push threshold, close
    // it rather than leave a half-applied push/overlay state.
    if (activeNum && prefersPush() !== pushActive) closeDrawer();
  }

  // Click anywhere outside the drawer (the page, or the overlay-mode backdrop)
  // closes it. Clicks on a callout dot are ignored — those toggle/swap via
  // their own handler. `activeNum` is already set on the opening click, and the
  // dot lives inside `layerEl`, so opening never self-closes.
  function onDocClick(e: MouseEvent) {
    if (!activeNum) return;
    const t = e.target as Node | null;
    if (!t || drawerEl?.contains(t) || layerEl?.contains(t)) return;
    closeDrawer();
  }

  onMount(() => {
    try {
      reviewOn = localStorage.getItem(STORE_KEY) !== '0';
    } catch {}

    measure();
    if (reviewOn) tick().then(animateDotsIn);

    ro = new ResizeObserver(scheduleMeasure);
    ro.observe(document.body);
    window.addEventListener('resize', onResize, { passive: true });
    window.addEventListener('load', measure);
    document.fonts?.ready.then(measure).catch(() => {});
    // Late passes catch lazy images / font swaps that shift layout.
    lateTimers = [setTimeout(measure, 300), setTimeout(measure, 1200)];

    // Stage-setting overview: auto-open once per visitor, after the page paints.
    let introSeen = false;
    try {
      introSeen = localStorage.getItem(INTRO_KEY) === '1';
    } catch {}
    if (!introSeen) lateTimers.push(setTimeout(() => openOverview(), 350));

    window.addEventListener('keydown', onKey);
    document.addEventListener('click', onDocClick);
  });

  onDestroy(() => {
    ro?.disconnect();
    cancelAnimationFrame(rafId);
    cancelAnimationFrame(pinRaf);
    lateTimers.forEach(clearTimeout);
    openTl?.kill();
    overviewTl?.kill();
    window.removeEventListener('resize', onResize);
    window.removeEventListener('load', measure);
    window.removeEventListener('keydown', onKey);
    document.removeEventListener('click', onDocClick);
    if (typeof document !== 'undefined') {
      document.body.style.overflow = '';
      document.documentElement.style.setProperty('--review-push', '0px');
    }
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

<!-- Backdrop (overlay mode only) — closing is handled by the document click
     listener so it works for both overlay and push (no backdrop) modes. -->
<div
  bind:this={backdropEl}
  class="review-backdrop"
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

<!-- Overview modal — stage-setting "Context & goals" -->
<div
  bind:this={overlayEl}
  class="review-modal-backdrop"
  onclick={closeOverview}
  aria-hidden="true"
></div>

<div class="review-modal-wrap">
  <section
    bind:this={overviewEl}
    class="review-modal"
    role="dialog"
    aria-modal="true"
    aria-labelledby="review-modal-title"
  >
    <div class="review-modal__glow" aria-hidden="true"></div>
    <button
      bind:this={overviewCloseEl}
      class="review-modal__close"
      type="button"
      aria-label="Close"
      onclick={closeOverview}
    >
      <svg viewBox="0 0 24 24" width="20" height="20" fill="none" aria-hidden="true">
        <path d="M6 6l12 12M18 6L6 18" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
      </svg>
    </button>

    <div class="review-modal__inner">
      <p class="review-modal__eyebrow" data-ov-stagger>{intro.eyebrow}</p>
      <h2 class="review-modal__title" id="review-modal-title" data-ov-stagger>{intro.title}</h2>
      {#each intro.context as p}
        <p class="review-modal__context" data-ov-stagger>{p}</p>
      {/each}
      <p class="review-modal__goals-label" data-ov-stagger>Goals</p>
      <ul class="review-modal__goals">
        {#each intro.goals as g}
          <li data-ov-stagger><span class="review-modal__bullet" aria-hidden="true"></span>{g}</li>
        {/each}
      </ul>
      <button
        bind:this={overviewCtaEl}
        class="review-modal__cta"
        type="button"
        data-ov-stagger
        onclick={closeOverview}
      >
        {intro.cta}
      </button>
    </div>
  </section>
</div>

<!-- Review toolbar — docks half-hidden at the bottom edge; lifts + brightens on hover -->
<div class="review-dock">
  <button
    class="review-dock__btn"
    type="button"
    aria-haspopup="dialog"
    onclick={(e) => openOverview(e.currentTarget as HTMLElement)}
  >
    <span class="review-dock__icon" aria-hidden="true">
      <svg viewBox="0 0 24 24" width="18" height="18" fill="none">
        <circle cx="12" cy="12" r="9" stroke="currentColor" stroke-width="2" />
        <path d="M12 11.5v4.5" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
        <circle cx="12" cy="7.8" r="1.2" fill="currentColor" />
      </svg>
    </span>
    <span class="review-dock__label">Context &amp; goals</span>
  </button>

  <span class="review-dock__sep" aria-hidden="true"></span>

  <button
    class="review-dock__btn review-dock__btn--notes"
    class:is-on={reviewOn}
    type="button"
    aria-pressed={reviewOn}
    onclick={toggleReview}
  >
    <span class="review-fab__dot" aria-hidden="true"></span>
    <span class="review-dock__label">{reviewOn ? 'Hide notes' : 'Show notes'}</span>
  </button>
</div>

<style>
  /* ---------- Dot layer ---------- */
  .review-layer {
    position: absolute;
    top: 0;
    /* Ride along with the push so the dots sit just right of the open drawer
       (clickable to swap sections). `width: calc(...)` avoids horizontal
       overflow. Vertical positions re-measure via the ResizeObserver. */
    left: var(--review-push, 0px);
    width: calc(100% - var(--review-push, 0px));
    height: 0; /* children are absolutely positioned in document coords */
    z-index: 40;
    pointer-events: none;
    transition:
      left var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized),
      width var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized);
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

  /* Hint shows only on hover/keyboard-focus — not while the dot's drawer is
     open (.is-active), so it doesn't linger over the design. */
  .review-dot:hover .review-dot__hint,
  .review-dot:focus-visible .review-dot__hint {
    opacity: 1;
    transform: translate(0, -50%);
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

  /* ---------- Review toolbar (bottom-left dock) ---------- */
  .review-dock {
    position: fixed;
    left: clamp(16px, 3vw, 32px);
    bottom: 0;
    z-index: 80;
    display: inline-flex;
    align-items: center;
    gap: 2px;
    padding: 10px 14px;
    border-radius: 16px 16px 0 0;
    border: 1px solid rgba(152, 255, 0, 0.18);
    background: rgba(5, 20, 29, 0.72);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    box-shadow: 0 -4px 20px rgba(0, 0, 0, 0.3);
    /* Docked: tucked ~halfway below the viewport edge; only the icons peek.
       Labels collapse so the docked handle reads as a small toolbar tab. */
    opacity: 0.82;
    transform: translateY(50%);
    transition:
      transform var(--flora-motion-duration-enter) var(--flora-motion-easing-expressive),
      opacity var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized),
      border-radius var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized),
      border-color var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      box-shadow var(--flora-motion-duration-moderate) var(--flora-motion-easing-emphasized);
  }

  /* Invisible hover bridge: when the dock lifts it moves up and away from the
     cursor near the bottom edge — without this the cursor falls into the gap,
     drops :hover, and the dock flickers up/down. The bridge extends the hit
     area down past the viewport edge. At rest it sits below the fold, so it
     never intercepts page clicks. */
  .review-dock::after {
    content: '';
    position: absolute;
    left: 0;
    top: 100%;
    width: 100%;
    height: 72px;
    background: transparent;
  }

  /* :focus-within (not just :hover) so keyboard/touch also reveals the labels */
  .review-dock:hover,
  .review-dock:focus-within {
    opacity: 1;
    transform: translateY(-14px);
    border-radius: 999px;
    border-color: rgba(152, 255, 0, 0.5);
    background: rgba(8, 26, 37, 0.96);
    box-shadow:
      0 14px 34px rgba(0, 0, 0, 0.45),
      0 0 0 4px rgba(152, 255, 0, 0.16);
  }

  .review-dock__btn {
    position: relative;
    display: inline-flex;
    align-items: center;
    padding: 8px 10px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: #ffffff;
    font-family: 'Euclid Circular A', sans-serif;
    font-size: 14px;
    font-weight: 500;
    line-height: 1;
    white-space: nowrap;
    cursor: pointer;
    transition: background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .review-dock__btn:hover,
  .review-dock__btn:focus-visible {
    outline: none;
    background: rgba(152, 255, 0, 0.1);
  }

  .review-dock__icon {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 18px;
    height: 18px;
    color: #98ff00;
  }

  .review-dock__sep {
    width: 1px;
    align-self: stretch;
    margin: 6px 2px;
    background: rgba(152, 255, 0, 0.18);
  }

  .review-dock__label {
    max-width: 0;
    margin-left: 0;
    overflow: hidden;
    opacity: 0;
    transition:
      max-width var(--flora-motion-duration-enter) var(--flora-motion-easing-expressive),
      margin-left var(--flora-motion-duration-enter) var(--flora-motion-easing-expressive),
      opacity var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .review-dock:hover .review-dock__label,
  .review-dock:focus-within .review-dock__label {
    max-width: 150px;
    margin-left: 9px;
    opacity: 1;
  }

  /* Notes-toggle pulsing dot */
  .review-fab__dot {
    position: relative;
    width: 10px;
    height: 10px;
    border-radius: 999px;
    background: #5d6c74;
    transition: background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .review-dock__btn--notes.is-on .review-fab__dot {
    background: linear-gradient(135deg, #98ff00, #6fce00);
  }

  @media (prefers-reduced-motion: no-preference) {
    .review-dock__btn--notes.is-on .review-fab__dot::after {
      content: '';
      position: absolute;
      inset: 0;
      border-radius: 999px;
      border: 2px solid #98ff00;
      animation: review-pulse 2.4s var(--flora-motion-easing-emphasized) infinite;
    }
  }

  /* ---------- Overview modal ---------- */
  .review-modal-backdrop {
    position: fixed;
    inset: 0;
    z-index: 95;
    background: rgba(0, 12, 18, 0.62);
    backdrop-filter: blur(3px);
    -webkit-backdrop-filter: blur(3px);
    opacity: 0;
    visibility: hidden;
  }

  /* Full-screen flex wrapper centers the card. Centering lives HERE, never as a
     transform on the card itself — GSAP owns the card's transform. */
  .review-modal-wrap {
    position: fixed;
    inset: 0;
    z-index: 96;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: clamp(16px, 4vw, 40px);
    pointer-events: none;
  }

  .review-modal {
    position: relative;
    width: min(560px, 100%);
    max-height: calc(100dvh - clamp(32px, 8vw, 80px));
    overflow-y: auto;
    background: #05141d;
    border: 1px solid rgba(152, 255, 0, 0.18);
    border-radius: 16px;
    box-shadow: 0 30px 80px rgba(0, 0, 0, 0.6);
    pointer-events: auto;
    visibility: hidden;
  }

  .review-modal__glow {
    position: absolute;
    top: -140px;
    left: -120px;
    width: 380px;
    height: 380px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(152, 255, 0, 0.18) 0%, transparent 70%);
    pointer-events: none;
  }

  .review-modal__close {
    position: absolute;
    top: 18px;
    right: 18px;
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

  .review-modal__close:hover {
    background: rgba(152, 255, 0, 0.12);
    border-color: rgba(152, 255, 0, 0.4);
    color: #ffffff;
  }

  .review-modal__inner {
    position: relative;
    padding: clamp(32px, 5vw, 48px);
  }

  .review-modal__eyebrow {
    margin: 0 0 18px;
    font-family: 'Source Code Pro', monospace;
    font-size: 12px;
    font-weight: 600;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: #98ff00;
  }

  .review-modal__title {
    margin: 0 0 24px;
    font-family: 'Euclid Circular A', sans-serif;
    font-weight: 600;
    font-size: clamp(28px, 4vw, 32px);
    line-height: 1.15;
    color: #ffffff;
  }

  .review-modal__context {
    margin: 0 0 16px;
    font-family: 'Euclid Circular A', sans-serif;
    font-size: 16px;
    line-height: 28px;
    color: #b8c4c2;
  }

  .review-modal__goals-label {
    margin: 28px 0 14px;
    font-size: 32px;
    font-weight: 500;
    color: #fff;
  }

  .review-modal__goals {
    list-style: none;
    margin: 0 0 32px;
    padding: 0;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .review-modal__goals li {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    font-family: 'Euclid Circular A', sans-serif;
    font-size: 16px;
    line-height: 26px;
    color: #ededed;
  }

  .review-modal__bullet {
    flex-shrink: 0;
    width: 8px;
    height: 8px;
    margin-top: 9px;
    border-radius: 999px;
    background: #009FFD
  }

  .review-modal__cta {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 14px 28px;
    border: 0;
    border-radius: 6px;
    background: var(--color-background-brand);
    color: #001e2b;
    font-family: 'Euclid Circular A', sans-serif;
    font-weight: 600;
    font-size: 15px;
    line-height: 1;
    cursor: pointer;
    transition:
      background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      transform var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized),
      box-shadow var(--flora-motion-duration-moderate) var(--flora-motion-easing-emphasized);
  }

  .review-modal__cta:hover {
    transform: translateY(-1px);
    box-shadow: 0 0 0 5px rgba(152, 255, 0, 0.18);
    border-radius: var(--radius-border-radius-circle);
  }

  .review-modal__cta:active {
    transform: scale(0.99);
    transition-duration: var(--flora-motion-duration-instant);
  }
</style>
