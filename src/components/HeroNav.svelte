<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import gsap from 'gsap';

  let stuck = $state(false);
  let menuOpen = $state(false);
  let sentinelEl: HTMLDivElement;
  let toggleEl: HTMLButtonElement;
  let overlayEl: HTMLDivElement;
  let menuListEl: HTMLUListElement;

  let io: IntersectionObserver;
  let tl: gsap.core.Timeline | null = null;

  // Read a Flora motion token (in seconds) so GSAP stays in sync with the CSS
  // tokens — including the reduced-motion overrides defined in global.css.
  function motionToken(varName: string, fallbackSeconds: number): number {
    if (typeof document === 'undefined') return fallbackSeconds;
    const v = getComputedStyle(document.documentElement).getPropertyValue(varName).trim();
    if (!v) return fallbackSeconds;
    return v.endsWith('ms') ? parseFloat(v) / 1000 : parseFloat(v) || fallbackSeconds;
  }

  function buildTimeline() {
    if (!toggleEl || !overlayEl) return;
    const rect = toggleEl.getBoundingClientRect();
    const cx = rect.left + rect.width / 2;
    const cy = rect.top + rect.height / 2;
    const farthest = Math.hypot(
      Math.max(cx, window.innerWidth - cx),
      Math.max(cy, window.innerHeight - cy)
    );

    tl?.kill();
    // Menu opening → Flora "enter" timing. The circular clip geometry is bespoke,
    // but its duration/easing follow the tokens. Closing reuses this timeline
    // reversed, retimed to "exit" (see playClose).
    const enter = motionToken('--flora-motion-duration-enter', 0.25);
    const stagger = motionToken('--flora-motion-delay-stagger', 0.06);
    tl = gsap.timeline({ paused: true });
    tl.set(overlayEl, { visibility: 'visible' });
    tl.fromTo(
      overlayEl,
      { clipPath: `circle(0px at ${cx}px ${cy}px)` },
      {
        clipPath: `circle(${farthest}px at ${cx}px ${cy}px)`,
        duration: enter,
        ease: 'power2.out', // ≈ easing-enter cubic-bezier(0, 0, 0.2, 1)
      }
    );
    if (menuListEl) {
      // Staggered fade-up of items (Flora section-reveal pattern)
      tl.from(
        menuListEl.children,
        { opacity: 0, y: 24, stagger, duration: enter, ease: 'power2.out' },
        enter * 0.4
      );
    }
  }

  function playClose() {
    // Retime the reversed timeline from "enter" to "exit" duration
    const enter = motionToken('--flora-motion-duration-enter', 0.25);
    const exit = motionToken('--flora-motion-duration-exit', 0.2);
    tl?.timeScale(exit > 0 ? enter / exit : 1);
    tl?.reverse();
  }

  function toggleMenu() {
    if (menuOpen) {
      playClose();
      menuOpen = false;
      document.body.style.overflow = '';
    } else {
      buildTimeline();
      tl?.timeScale(1);
      tl?.play(0);
      menuOpen = true;
      document.body.style.overflow = 'hidden';
    }
  }

  function closeMenu() {
    if (!menuOpen) return;
    playClose();
    menuOpen = false;
    document.body.style.overflow = '';
  }

  function onKey(e: KeyboardEvent) {
    if (e.key === 'Escape') closeMenu();
  }

  onMount(() => {
    io = new IntersectionObserver(
      ([entry]) => {
        stuck = !entry.isIntersecting;
      },
      { threshold: 0 }
    );
    io.observe(sentinelEl);

    window.addEventListener('keydown', onKey);
  });

  onDestroy(() => {
    io?.disconnect();
    tl?.kill();
    window.removeEventListener('keydown', onKey);
    document.body.style.overflow = '';
  });
</script>

<div bind:this={sentinelEl} class="nav-sentinel" aria-hidden="true"></div>

<header class="nav-shell" class:stuck class:menu-open={menuOpen}>
  <nav class="nav">
    <a class="brand" href="#" aria-label="MongoDB .local San Francisco" onclick={closeMenu}>
      <svg
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 298 33"
        class="brand-svg"
        fill="none"
        aria-hidden="true"
      >
        <path fill="#001e2b" d="M189.223 22.8333h1.55c.133 1.3167 1.433 1.9167 2.55 1.9167 1.233 0 2.233-.6667 2.233-1.7667 0-.9333-.583-1.5833-1.683-1.9833l-1.634-.6c-1.833-.65-2.716-1.6666-2.733-3.3333 0-1.9333 1.533-3.1 3.6-3.1 2.4 0 3.583 1.65 3.733 3.2h-1.533c-.3-1.2333-1.267-1.75-2.233-1.75-1.134 0-2.017.6167-2.017 1.6 0 1 .55 1.5 1.7 1.9167l1.517.5499c1.8.65 2.833 1.6834 2.833 3.45 0 2-1.567 3.2667-3.767 3.2667-2.483 0-4.05-1.5667-4.116-3.3667m9.342-1c0-2.3833 1.75-4.3333 4.083-4.3333 1.25 0 2.383.65 2.933 1.45v-1.2833h1.434V26h-1.434v-1.2833c-.55.8-1.683 1.45-2.933 1.45-2.333 0-4.083-1.95-4.083-4.3334m7.133 0c0-1.6833-1.15-3.0166-2.833-3.0166-1.684 0-2.85 1.3333-2.85 3.0166 0 1.6834 1.166 3.0167 2.85 3.0167 1.683 0 2.833-1.3333 2.833-3.0167M209.713 26v-8.3333h1.433V18.95c.567-.9833 1.567-1.45 2.65-1.45 1.867 0 3.15 1.3 3.15 3.3667V26h-1.417v-4.9167c0-1.4-.8-2.2666-1.966-2.2666-1.35 0-2.417 1.1-2.417 3.05V26zm14.051 0V14.1667h6.883v1.4667h-5.333v3.5833h4.216v1.4666h-4.216V26zm8.116 0v-8.3333h1.434v1.6167c.333-1.0667 1.366-1.7 2.333-1.7.217 0 .417.0166.65.0666v1.4834c-.267-.1-.5-.1334-.783-.1334-1.05 0-2.2.9167-2.2 2.8667V26zm5.177-4.1667c0-2.3833 1.75-4.3333 4.084-4.3333 1.25 0 2.383.65 2.933 1.45v-1.2833h1.433V26h-1.433v-1.2833c-.55.8-1.683 1.45-2.933 1.45-2.334 0-4.084-1.95-4.084-4.3334m7.134 0c0-1.6833-1.15-3.0166-2.834-3.0166-1.683 0-2.85 1.3333-2.85 3.0166 0 1.6834 1.167 3.0167 2.85 3.0167 1.684 0 2.834-1.3333 2.834-3.0167M248.205 26v-8.3333h1.434V18.95c.566-.9833 1.566-1.45 2.65-1.45 1.866 0 3.15 1.3 3.15 3.3667V26h-1.417v-4.9167c0-1.4-.8-2.2666-1.967-2.2666-1.35 0-2.416 1.1-2.416 3.05V26zm17.715-2.9333c-.467 1.8-2.1 3.1-4.2 3.1-2.534 0-4.35-1.95-4.35-4.3334 0-2.3833 1.816-4.3333 4.35-4.3333 2.1 0 3.733 1.3 4.2 3.1h-1.534c-.366-1.05-1.366-1.7833-2.666-1.7833-1.734 0-2.9 1.3333-2.9 3.0166 0 1.6834 1.166 3.0167 2.9 3.0167 1.3 0 2.3-.7333 2.666-1.7833zm3.631-8.65c0 .5667-.433 1.0167-1 1.0167-.566 0-1.033-.45-1.033-1.0167 0-.55.467-1.0167 1.033-1.0167.567 0 1 .4667 1 1.0167M267.818 26v-8.3333h1.433V26zm9.9-2.25c0 1.65-1.333 2.4167-3.066 2.4167-1.784 0-3.267-.9-3.384-2.5834h1.467c.2 1.0167 1.067 1.3667 1.933 1.3667.9 0 1.6-.3667 1.6-1.1 0-.6667-.4-1.0667-1.383-1.3333l-1.2-.3167c-1.433-.3667-2.15-1.2667-2.15-2.45 0-1.3833 1.2-2.25 2.883-2.25 1.784 0 2.884 1.05 3.034 2.3833h-1.434c-.216-.7333-.816-1.1833-1.6-1.1833-.8 0-1.433.4-1.433 1.0333 0 .6167.383.9667 1.267 1.2l1.333.35c1.383.35 2.133 1.2167 2.133 2.4667m9.93-.6833c-.466 1.8-2.1 3.1-4.2 3.1-2.533 0-4.35-1.95-4.35-4.3334 0-2.3833 1.817-4.3333 4.35-4.3333 2.1 0 3.734 1.3 4.2 3.1h-1.533c-.367-1.05-1.367-1.7833-2.667-1.7833-1.733 0-2.9 1.3333-2.9 3.0166 0 1.6834 1.167 3.0167 2.9 3.0167 1.3 0 2.3-.7333 2.667-1.7833zm9.948-1.2334c0 2.4-1.85 4.3334-4.35 4.3334-2.516 0-4.366-1.9334-4.366-4.3334s1.85-4.3333 4.366-4.3333c2.5 0 4.35 1.9333 4.35 4.3333m-7.25 0c0 1.6667 1.167 3 2.9 3 1.734 0 2.884-1.3333 2.884-3 0-1.6666-1.15-2.9999-2.884-2.9999-1.733 0-2.9 1.3333-2.9 2.9999"/>
        <path fill="#000" d="M126.311 26.0149c.746 0 1.353-.5834 1.353-1.3067 0-.7466-.607-1.3533-1.353-1.3533-.724 0-1.307.6067-1.307 1.3533 0 .7233.583 1.3067 1.307 1.3067m7.539-.3267.187-.1867v-.7466l-.187-.1867h-.42c-.466 0-.793-.3266-.793-.7933V10.4053l.14-1.3533-.14-.16333h-.56l-2.52.90998-.186.23335v.4433l.233.2333h.35c.467 0 .817.2333.817.7933v12.273c0 .4667-.327.7933-.794.7933h-.42l-.186.1867v.7466l.186.1867zm7.6.3267c3.36 0 6.044-2.4033 6.044-6.0899 0-3.6865-2.684-6.0898-6.044-6.0898s-6.043 2.4033-6.043 6.0898c0 3.6866 2.683 6.0899 6.043 6.0899m0-1.1667c-2.45 0-4.036-1.8899-4.036-4.9232 0-3.0332 1.586-4.9232 4.036-4.9232s4.037 1.89 4.037 4.9232c0 3.0333-1.587 4.9232-4.037 4.9232m13.156 1.1667c3.01 0 4.41-1.8667 5.017-3.7333l-.117-.2333-.256-.1167-.234.0934c-.746 1.4466-1.96 2.5199-3.826 2.5199-2.38 0-4.2-1.8199-4.2-4.8065 0-2.6366 1.47-4.7132 4.13-4.7132 2.076 0 2.893 1.1433 2.893 2.2166v.2333l.187.1867h.63l.186-.1867-.116-2.5899-.164-.2334c-.91-.5133-2.123-.8166-3.476-.8166-3.174 0-6.207 2.2633-6.207 6.2532 0 3.7565 2.73 5.9265 5.553 5.9265m19.002-1.4467c-.466 0-.793-.3266-.793-.7933v-9.4264l-.187-.1866h-.676l-.957 1.7266h-.117c-.7-1.0033-1.983-2.0533-4.106-2.0533-2.87 0-5.577 2.2633-5.577 6.0898 0 4.2699 2.707 6.0899 5.46 6.0899 1.517 0 3.127-.5134 4.293-2.4266v1.9132l.187.1867h2.893l.187-.1867v-.7466l-.187-.1867zm-6.626.35c-2.59 0-3.827-2.4032-3.827-4.9932 0-2.6132 1.237-4.9932 3.827-4.9932 2.52 0 3.826 2.38 3.826 4.9932 0 2.59-1.306 4.9932-3.826 4.9932m13.09.77.187-.1867v-.7466l-.187-.1867h-.42c-.467 0-.793-.3266-.793-.7933V10.4053l.14-1.3533-.14-.16333h-.56l-2.52.90998-.187.23335v.4433l.233.2333h.35c.467 0 .817.2333.817.7933v12.273c0 .4667-.327.7933-.793.7933h-.42l-.187.1867v.7466l.187.1867zM9.5821 3.37024C8.32662 1.88066 7.24551.367798 7.02464.053589c-.02325-.023275-.05813-.023275-.08138 0C6.72239.367798 5.64128 1.88066 4.3858 3.37024-6.39039 17.114 6.08304 26.389 6.08304 26.389l.10461.0698c.093 1.4314.3255 3.4912.3255 3.4912h.92998s.2325-2.0481.3255-3.4912l.10461-.0814c.01163 0 12.48506-9.2634 1.70886-23.00716M6.97814 26.1795s-.55799-.4771-.70911-.7215v-.0233l.67423-14.9657c0-.0465.06976-.0465.06976 0l.67423 14.9657v.0233c-.15112.2444-.70911.7215-.70911.7215m23.34306-3.1448-5.1978-12.6044-.0116-.0348h-4.0441v.8473h.6527c.1981 0 .3845.0812.5244.2205.1398.1393.2098.325.2098.5223l-.1166 12.6856c0 .3946-.3263.7196-.7225.7312l-.6643.0116v.8356h3.9391v-.8356l-.4079-.0116c-.3962-.0116-.7225-.3366-.7225-.7312V12.7168l5.664 13.5328c.0815.1974.268.325.4778.325s.3962-.1276.4778-.325l5.5358-13.2311.0816 11.6527c0 .4062-.3263.7312-.7342.7428h-.4196v.8356h4.6151v-.8356h-.6293c-.3963 0-.7226-.3366-.7342-.7312l-.035-12.6856c0-.4062.3263-.7312.7226-.7428l.6759-.0116v-.8473h-3.9391zm36.2419 2.2543c-.1286-.1287-.1987-.3042-.1987-.5148V18.504c0-1.1932-.3505-2.129-1.0516-2.7958-.6894-.6668-1.6475-1.0061-2.8393-1.0061-1.6709 0-2.9913.6785-3.9144 2.0121-.0117.0234-.0467.0351-.0818.0351-.035 0-.0584-.0234-.0584-.0585l-.4323-1.6728h-.7245l-1.8578 1.0645v.5849h.4791c.222 0 .4089.0585.5374.1755.1286.117.1987.2925.1987.5381v7.3816c0 .2105-.0701.386-.1987.5147-.1285.1286-.3037.1988-.5141.1988h-.4673v.854h4.2765v-.854h-.4674c-.2103 0-.3856-.0702-.5141-.1988-.1285-.1287-.1986-.3042-.1986-.5147v-4.8665c0-.62.1402-1.24.3972-1.8483.2688-.5966.6661-1.0996 1.1919-1.4856.5258-.3861 1.1567-.5733 1.8812-.5733.8179 0 1.4372.2574 1.8228.7721s.5842 1.1815.5842 1.977v6.0362c0 .2106-.0701.3861-.1986.5148-.1286.1286-.3038.1988-.5142.1988h-.4673v.854h4.2765v-.854h-.4673c-.1753 0-.3389-.0585-.4791-.1988m39.0709-13.9376c-1.182-.6295-2.503-.9559-3.929-.9559h-5.5624v.8509h.5447c.2086 0 .394.0816.5795.2681.1738.1749.2665.3731.2665.5829V24.64c0 .2098-.0927.4079-.2665.5828-.1739.1748-.3709.2681-.5795.2681h-.5447v.8509h5.5624c1.426 0 2.747-.3264 3.929-.9558 1.182-.6295 2.144-1.562 2.84-2.751.695-1.189 1.054-2.6227 1.054-4.2547 0-1.6319-.359-3.054-1.054-4.2547-.707-1.2123-1.658-2.1448-2.84-2.7742m1.657 7.0056c0 1.4921-.266 2.751-.788 3.7651-.521 1.0141-1.216 1.7718-2.074 2.2614s-1.808.7344-2.828.7344h-1.124c-.209 0-.394-.0816-.5795-.2681-.1738-.1749-.2665-.373-.2665-.5829V12.4471c0-.2098.0811-.3963.2665-.5829.1735-.1748.3705-.2681.5795-.2681h1.124c1.02 0 1.97.2448 2.828.7344s1.553 1.2473 2.074 2.2614c.522 1.0141.788 2.2731.788 3.7651m15.388.8509c-.514-.5944-1.507-1.0957-2.675-1.3638 1.612-.8043 2.442-1.935 2.442-3.3921 0-.7926-.211-1.5037-.631-2.1098-.421-.6062-1.017-1.0958-1.776-1.4338-.76-.338-1.647-.5129-2.652-.5129h-6.297v.8509h.502c.21 0 .397.0816.584.2681.175.1749.269.3731.269.5829V24.64c0 .2098-.094.4079-.269.5828-.175.1748-.374.2681-.584.2681h-.549v.8509h6.834c1.04 0 2.01-.1748 2.886-.5245s1.577-.8626 2.08-1.5387c.514-.6761.771-1.5037.771-2.4596-.012-1.0257-.316-1.8883-.935-2.6111m-7.839 5.6535c-.175-.1748-.269-.373-.269-.5828v-5.6302h3.248c1.145 0 2.021.2914 2.629.8743.607.5828.911 1.3405.911 2.273 0 .5596-.14 1.1074-.397 1.6087-.269.5128-.666.9208-1.204 1.2356-.525.3147-1.18.4779-1.939.4779h-2.395c-.21.0116-.397-.07-.584-.2565m-.257-7.4136v-5.0007c0-.2098.082-.3963.269-.5829.175-.1748.373-.2681.584-.2681h1.542c1.11 0 1.928.2798 2.442.816.514.5479.771 1.2473.771 2.1099 0 .8859-.246 1.5969-.725 2.1331-.479.5246-1.203.7927-2.161.7927zm-64.2995-2.0022c-.8927-.4879-1.8898-.7435-2.968-.7435s-2.0868.244-2.968.7435c-.8927.4879-1.5999 1.1966-2.1216 2.0911s-.7884 1.94-.7884 3.1017.2667 2.2072.7884 3.1017 1.2289 1.6031 2.1216 2.091c.8928.4879 1.8898.7435 2.968.7435s2.0869-.244 2.968-.7435c.8927-.4879 1.6-1.1965 2.1217-2.091s.7883-1.94.7883-3.1017-.2666-2.2072-.7883-3.1017c-.5217-.9061-1.229-1.6032-2.1217-2.0911m.858 5.1928c0 1.4288-.3478 2.5905-1.0435 3.4269-.684.8365-1.6231 1.2663-2.7825 1.2663-1.1593 0-2.0984-.4298-2.7825-1.2663-.6956-.8364-1.0434-1.9981-1.0434-3.4269 0-1.4289.3478-2.5906 1.0434-3.427.6841-.8364 1.6232-1.2662 2.7825-1.2662 1.1594 0 2.0985.4298 2.7825 1.2662.6957.8364 1.0435 1.9865 1.0435 3.427m40.2299-5.1928c-.8927-.4879-1.8898-.7435-2.968-.7435s-2.0869.244-2.968.7435c-.8927.4879-1.5999 1.1966-2.1216 2.0911s-.7884 1.94-.7884 3.1017.2667 2.2072.7884 3.1017 1.2289 1.6031 2.1216 2.091 1.8898.7435 2.968.7435 2.0869-.244 2.968-.7435c.8927-.4879 1.5999-1.1965 2.1217-2.091.5217-.8945.7883-1.94.7883-3.1017s-.2666-2.2072-.7883-3.1017c-.5218-.9061-1.2406-1.6032-2.1217-2.0911m.8579 5.1928c0 1.4288-.3478 2.5905-1.0434 3.4269-.684.8365-1.6231 1.2663-2.7825 1.2663s-2.0985-.4298-2.7825-1.2663c-.6956-.8364-1.0434-1.9981-1.0434-3.4269 0-1.4405.3478-2.5906 1.0434-3.427.684-.8364 1.6231-1.2662 2.7825-1.2662s2.0985.4298 2.7825 1.2662 1.0434 1.9865 1.0434 3.427m-17.4279-5.9363c-.9341 0-1.7865.1984-2.5571.5952s-1.3778.9337-1.8098 1.6222c-.432.6769-.6539 1.4355-.6539 2.2407 0 .7236.1635 1.3888.5021 1.984.3269.5718.7706 1.0503 1.3311 1.4471l-1.6697 2.2641c-.2102.2801-.2336.6535-.0818.9569.1635.3151.4671.5019.8174.5019h.4787c-.467.3151-.8407.6885-1.0976 1.132-.3036.5018-.4553 1.027-.4553 1.5638 0 1.0037.4437 1.8323 1.3194 2.4508.864.6185 2.0783.9336 3.6079.9336 1.0625 0 2.0783-.175 3.0008-.5135.9341-.3384 1.693-.8402 2.2535-1.4938.5721-.6535.864-1.4471.864-2.3574 0-.957-.3503-1.6338-1.1676-2.2874-.7006-.5485-1.7981-.8403-3.1642-.8403h-4.6705c-.0117 0-.0234-.0116-.0234-.0116s-.0116-.0234 0-.035l1.2144-1.6339c.3269.1517.6305.2451.899.3034.2802.0584.5955.0817.9458.0817.9808 0 1.8682-.1984 2.6388-.5952.7706-.3967 1.3895-.9336 1.8332-1.6221.4437-.6769.6655-1.4355.6655-2.2407 0-.8636-.4203-2.4391-1.5646-3.2444 0-.0117.0117-.0117.0117-.0117l2.5103.2801v-1.1554H76.763c-.6306-.21-1.2844-.3151-1.9616-.3151m1.4011 7.3057c-.4437.2334-.9224.3618-1.4011.3618-.7823 0-1.4713-.2801-2.0551-.8286s-.8757-1.3538-.8757-2.3808.2919-1.8322.8757-2.3807 1.2728-.8286 2.0551-.8286c.4904 0 .9574.1167 1.4011.3618.4437.2334.8056.5951 1.0975 1.0736.2803.4785.4321 1.0737.4321 1.7739 0 .7119-.1402 1.3071-.4321 1.7739-.2802.4785-.6538.8286-1.0975 1.0737m-3.1643 4.2947h3.1643c.8757 0 1.4362.175 1.8098.5485.3736.3734.5605.8753.5605 1.4588 0 .8519-.3386 1.5522-1.0159 2.0773-.6772.5252-1.5879.7936-2.7089.7936-.9807 0-1.7981-.2217-2.3936-.6419-.5954-.4201-.899-1.062-.899-1.8789 0-.5135.1401-.992.4203-1.4121.2803-.4201.6189-.7236 1.0625-.9453"/>
      </svg>
    </a>

    <div class="nav-right">
      <ul class="nav-items">
        <li><a href="#why-attend">Why Attend</a></li>
        <li><a href="#speakers">Speakers</a></li>
        <li><a href="#pricing">Pricing</a></li>
        <li><a href="#sessions">Featured Sessions</a></li>
      </ul>
      <a class="register-btn" href="#register">Register Now</a>
    </div>

    <button
      bind:this={toggleEl}
      class="menu-toggle"
      class:open={menuOpen}
      type="button"
      aria-label={menuOpen ? 'Close menu' : 'Open menu'}
      aria-expanded={menuOpen}
      aria-controls="mobile-menu"
      onclick={toggleMenu}
    >
      <span class="bar bar-top"></span>
      <span class="bar bar-mid"></span>
      <span class="bar bar-bot"></span>
    </button>
  </nav>
</header>

<div bind:this={overlayEl} id="mobile-menu" class="mobile-overlay" aria-hidden={!menuOpen}>
  <ul bind:this={menuListEl} class="mobile-menu-list">
    <li><a href="#speakers" onclick={closeMenu}>Why Attend</a></li>
    <li><a href="#agenda" onclick={closeMenu}>Speakers</a></li>
    <li><a href="#sponsors" onclick={closeMenu}>Pricing</a></li>
    <li><a href="#sponsors" onclick={closeMenu}>Featured Sessions</a></li>
    <li><a class="mobile-register" href="#register" onclick={closeMenu}>Register Now</a></li>
  </ul>
</div>

<style>
  .nav-sentinel {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 1px;
    pointer-events: none;
  }

  .nav-shell {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    z-index: 50;
    padding: 24px clamp(16px, 7.86%, 132px);
    transition: padding var(--flora-motion-duration-enter) var(--flora-motion-easing-movement);
    pointer-events: none;
  }

  .nav-shell.stuck {
    padding: 0;
  }

  .nav {
    pointer-events: auto;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 24px;
    background: #ffffff;
    padding: 12px 32px;
    border-radius: 16px;
    box-shadow: 0 3px 20px rgba(0, 0, 0, 0.15);
    transition:
      border-radius var(--flora-motion-duration-enter) var(--flora-motion-easing-movement),
      box-shadow var(--flora-motion-duration-enter) var(--flora-motion-easing-movement);
  }

  .nav-shell.stuck .nav {
    border-radius: 0;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  }

  .brand {
    display: flex;
    align-items: center;
    text-decoration: none;
    color: #001e2b;
    flex-shrink: 0;
  }

  .brand-svg {
    height: 24px;
    width: auto;
    display: block;
    transition: height 0.25s ease;
  }

  .nav-right {
    display: flex;
    align-items: center;
    gap: 32px;
  }

  .nav-items {
    list-style: none;
    margin: 0;
    padding: 0;
    display: flex;
    align-items: center;
    gap: 32px;
  }

  .nav-items a {
    font-family: 'Euclid Circular A', sans-serif;
    font-weight: 500;
    font-size: 16px;
    line-height: 24px;
    color: #001e2b;
    text-decoration: none;
    transition: color var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .nav-items a:hover {
    color: #00684a;
  }

  .register-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 16px 32px;
    background: #001e2b;
    color: #ffffff;
    text-decoration: none;
    font-family: 'Euclid Circular A', sans-serif;
    font-weight: 500;
    font-size: 16px;
    line-height: 18px;
    border-radius: 4px;
    /* Flora button motion */
    transition:
      background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      box-shadow var(--flora-motion-duration-moderate) var(--flora-motion-easing-emphasized),
      transform var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized),
      border-radius var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized);
  }

  .register-btn:hover {
    background: #00684a;
    transform: translateY(-1px);
    border-radius: var(--radius-border-radius-circle);
    box-shadow: 0 0 0 6px color-mix(in oklch, #00ed64 35%, transparent);
  }

  .register-btn:active {
    transform: scale(0.985);
    border-radius: var(--radius-border-radius-circle);
    transition-duration: var(--flora-motion-duration-instant);
  }

  /* ---------- Hamburger toggle ---------- */
  .menu-toggle {
    display: none;
    position: relative;
    width: 44px;
    height: 44px;
    border: 0;
    background: transparent;
    cursor: pointer;
    padding: 0;
    z-index: 60;
    border-radius: 999px;
    transition: background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .menu-toggle:hover {
    background: rgba(0, 30, 43, 0.06);
  }

  .menu-toggle .bar {
    position: absolute;
    left: 50%;
    width: 22px;
    height: 2px;
    background: #001e2b;
    border-radius: 2px;
    transform-origin: center;
    transition:
      transform var(--flora-motion-duration-enter) var(--flora-motion-easing-movement),
      opacity var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      top var(--flora-motion-duration-enter) var(--flora-motion-easing-movement);
    margin-left: -11px;
  }

  .menu-toggle .bar-top { top: 14px; }
  .menu-toggle .bar-mid { top: 21px; }
  .menu-toggle .bar-bot { top: 28px; }

  .menu-toggle.open .bar { background: #ffffff; }
  .menu-toggle.open .bar-top {
    top: 21px;
    transform: rotate(45deg);
  }
  .menu-toggle.open .bar-mid { opacity: 0; }
  .menu-toggle.open .bar-bot {
    top: 21px;
    transform: rotate(-45deg);
  }

  /* ---------- Mobile overlay ---------- */
  .mobile-overlay {
    position: fixed;
    inset: 0;
    z-index: 55;
    background: #001e2b;
    visibility: hidden;
    clip-path: circle(0 at 100% 0);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
  }

  .mobile-menu-list {
    list-style: none;
    margin: 0;
    padding: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 32px;
    text-align: center;
  }

  .mobile-menu-list a {
    font-family: 'Euclid Circular A', sans-serif;
    font-weight: 500;
    font-size: 36px;
    line-height: 1.1;
    color: #ffffff;
    text-decoration: none;
    text-transform: uppercase;
    letter-spacing: 0.02em;
    transition: color var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .mobile-menu-list a:hover {
    color: #00ed64;
  }

  .mobile-register {
    margin-top: 16px;
    padding: 16px 40px;
    background: #00ed64;
    color: #001e2b !important;
    border-radius: 4px;
    font-size: 18px !important;
    text-transform: none !important;
    transition:
      background var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      color var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback),
      transform var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized),
      border-radius var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized);
  }

  .mobile-register:hover {
    color: #001e2b !important;
    background: #00d959;
    transform: translateY(-1px);
    border-radius: var(--radius-border-radius-circle);
  }

  .mobile-register:active {
    transform: scale(0.985);
    border-radius: var(--radius-border-radius-circle);
    transition-duration: var(--flora-motion-duration-instant);
  }

  /* ---------- Breakpoints ---------- */
  @media (max-width: 1023px) {
    .nav-right {
      display: none;
    }
    .menu-toggle {
      display: inline-block;
    }
    .brand-svg {
      height: 20px;
    }
    .nav {
      padding: 10px 16px 10px 20px;
    }
  }

  @media (max-width: 480px) {
    .brand-svg {
      height: 18px;
    }
    .mobile-menu-list a {
      font-size: 28px;
    }
  }
</style>
