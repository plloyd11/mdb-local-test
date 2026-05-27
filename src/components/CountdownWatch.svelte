<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import gsap from 'gsap';

  // Event date: August 13, 2026, 09:00 local
  const EVENT_DATE = new Date('2026-08-13T09:00:00').getTime();

  let days = $state('00');
  let hours = $state('00');
  let minutes = $state('00');
  let seconds = $state('00');

  // Live clock state for the watch face
  let hourDeg = $state(0);
  let minuteDeg = $state(0);
  let secondDeg = $state(0);

  let countdownTimer: ReturnType<typeof setInterval>;
  let clockTimer: ReturnType<typeof setInterval>;

  let pillEl: HTMLElement;
  let digitsEl: HTMLElement;
  let watchEl: HTMLElement;
  let triggerEl: HTMLElement;

  let tl: gsap.core.Timeline;
  let io: IntersectionObserver;

  const pad = (n: number) => String(Math.max(0, Math.floor(n))).padStart(2, '0');

  function updateCountdown() {
    const diff = EVENT_DATE - Date.now();
    if (diff <= 0) {
      days = '00';
      hours = '00';
      minutes = '00';
      seconds = '00';
      return;
    }
    const d = diff / 1000;
    days = pad(d / 86400);
    hours = pad((d % 86400) / 3600);
    minutes = pad((d % 3600) / 60);
    seconds = pad(d % 60);
  }

  function updateClockHands() {
    const now = new Date();
    const s = now.getSeconds() + now.getMilliseconds() / 1000;
    const m = now.getMinutes() + s / 60;
    const h = (now.getHours() % 12) + m / 60;
    secondDeg = s * 6;
    minuteDeg = m * 6;
    hourDeg = h * 30;
  }

  onMount(() => {
    updateCountdown();
    updateClockHands();
    countdownTimer = setInterval(updateCountdown, 1000);
    clockTimer = setInterval(updateClockHands, 1000);

    // Set initial states for GSAP
    gsap.set(watchEl, { opacity: 0, scale: 0.6 });

    tl = gsap.timeline({ paused: true });
    tl.to(digitsEl, { opacity: 0, scale: 0.85, duration: 0.25, ease: 'power2.in' }, 0)
      .to(
        pillEl,
        {
          width: 64,
          height: 64,
          paddingLeft: 0,
          paddingRight: 0,
          paddingTop: 0,
          paddingBottom: 0,
          borderRadius: 999,
          backgroundColor: 'rgba(0, 30, 43, 0.85)',
          borderColor: 'rgba(0, 237, 100, 0.9)',
          duration: 0.55,
          ease: 'power3.inOut',
        },
        0.05
      )
      .to(watchEl, { opacity: 1, scale: 1, duration: 0.3, ease: 'power2.out' }, 0.35);

    io = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          tl.reverse();
        } else {
          tl.play();
        }
      },
      { threshold: 0 }
    );
    io.observe(triggerEl);
  });

  onDestroy(() => {
    clearInterval(countdownTimer);
    clearInterval(clockTimer);
    io?.disconnect();
    tl?.kill();
  });
</script>

<!-- Sentinel placed by parent; we render our own at top of hero -->
<div bind:this={triggerEl} class="hero-sentinel" aria-hidden="true"></div>

<div class="countdown-wrap">
  <div bind:this={pillEl} class="pill" role="timer" aria-label="Time until event">
    <div bind:this={digitsEl} class="digits">
      <div class="item" style="width: 41px;">
        <span class="num">{days}</span>
        <span class="label">Days</span>
      </div>
      <div class="item" style="width: 41px;">
        <span class="num">{hours}</span>
        <span class="label">Hours</span>
      </div>
      <div class="item" style="width: 45px;">
        <span class="num">{minutes}</span>
        <span class="label">Minutes</span>
      </div>
      <div class="item" style="width: 52px;">
        <span class="num">{seconds}</span>
        <span class="label">Seconds</span>
      </div>
    </div>

    <div bind:this={watchEl} class="watch" aria-hidden="true">
      <svg viewBox="0 0 64 64" width="44" height="44">
        <!-- watch case -->
        <circle cx="32" cy="32" r="26" fill="none" stroke="#00ed64" stroke-width="2" />
        <!-- ticks -->
        {#each Array(12) as _, i}
          <line
            x1="32"
            y1="9"
            x2="32"
            y2={i % 3 === 0 ? 13 : 11.5}
            stroke="#00ed64"
            stroke-width={i % 3 === 0 ? 2 : 1}
            stroke-linecap="round"
            transform="rotate({i * 30} 32 32)"
          />
        {/each}
        <!-- hour hand -->
        <line
          x1="32"
          y1="32"
          x2="32"
          y2="20"
          stroke="#ffffff"
          stroke-width="2.5"
          stroke-linecap="round"
          transform="rotate({hourDeg} 32 32)"
        />
        <!-- minute hand -->
        <line
          x1="32"
          y1="32"
          x2="32"
          y2="14"
          stroke="#ffffff"
          stroke-width="2"
          stroke-linecap="round"
          transform="rotate({minuteDeg} 32 32)"
        />
        <!-- second hand -->
        <line
          x1="32"
          y1="34"
          x2="32"
          y2="12"
          stroke="#00ed64"
          stroke-width="1.25"
          stroke-linecap="round"
          transform="rotate({secondDeg} 32 32)"
          style="transition: transform 0.2s cubic-bezier(0.4, 2.5, 0.6, 1);"
        />
        <!-- center cap -->
        <circle cx="32" cy="32" r="2" fill="#00ed64" />
      </svg>
    </div>
  </div>
</div>

<style>
  .hero-sentinel {
    position: absolute;
    top: 90vh;
    left: 0;
    width: 100%;
    height: 1px;
    pointer-events: none;
  }

  .countdown-wrap {
    position: fixed;
    right: 16px;
    bottom: 16px;
    z-index: 40;
    pointer-events: none;
  }

  .pill {
    pointer-events: auto;
    display: flex;
    align-items: center;
    justify-content: center;
    box-sizing: border-box;
    padding: 16px;
    border: 1px solid rgba(0, 237, 100, 0.7);
    border-radius: 8px;
    background: rgba(0, 30, 43, 0.3);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    width: 247px;
    height: 68px;
    position: relative;
    overflow: hidden;
  }

  .digits {
    display: flex;
    align-items: center;
    gap: 12px;
    line-height: 1;
    text-align: center;
  }

  .item {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
    flex-shrink: 0;
  }

  .num {
    font-family: 'Source Code Pro', monospace;
    font-weight: 500;
    font-size: 20px;
    color: #ffffff;
    line-height: 1;
    font-variant-numeric: tabular-nums;
  }

  .label {
    font-family: 'Euclid Circular A', sans-serif;
    font-weight: 400;
    font-size: 12px;
    color: #b8c4c2;
    line-height: 1;
    white-space: nowrap;
  }

  .watch {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    pointer-events: none;
  }
</style>
