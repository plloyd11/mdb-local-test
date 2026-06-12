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
    <a class="brand" href="/" aria-label="MongoDB .local Build Fest" onclick={closeMenu}>
      <svg xmlns="http://www.w3.org/2000/svg" width="244" height="27" fill="none">
        <g clip-path="url(#a)">
          <path fill="#023430" d="M161.661 20.7417h-5.991l-.154-.1546v-.6184l.154-.1546h.619c.386 0 .773-.3865.773-.7731V8.91436c0-.38651-.387-.77302-.773-.77302h-.58l-.193-.19326v-.54112l.193-.19326h5.836c2.571 0 4.271 1.39145 4.271 3.4207 0 1.3721-.831 2.2804-2.184 2.9181 1.72.3672 2.88 1.6427 2.88 3.3627 0 2.5124-2.223 3.8265-4.851 3.8265m-2.86-11.82734v4.32894h2.686c1.759 0 2.513-1.1015 2.513-2.551 0-1.37209-.754-2.55096-2.803-2.55096h-1.623c-.386 0-.773.38651-.773.77302m.773 10.89974h2.049c2.048 0 3.072-1.4688 3.072-2.8989 0-1.546-1.024-2.7442-3.072-2.7442h-2.822v4.87c0 .3866.387.7731.773.7731m18.234 0h.348l.155.1546v.6184l-.155.1546h-2.396l-.155-.1546v-1.4688c-.734 1.1982-1.836 1.894-3.362 1.894-1.74 0-3.305-1.0823-3.305-3.1888v-4.6961c0-.4638-.29-.6571-.676-.6571h-.29l-.194-.1933v-.3671l.155-.1933 2.087-.6378h.464l.155.1547-.155 1.2754v5.1214c0 1.3528.638 2.2997 2.242 2.2997 1.72 0 2.879-1.5267 2.879-2.9761v-3.8265c0-.4638-.29-.6571-.676-.6571h-.29l-.193-.1933v-.3671l.154-.1933 2.088-.6378h.463l.155.1547-.155 1.2754v6.6481c0 .3865.271.6571.657.6571m3.703-10.45524c-.599 0-1.082-.48315-1.082-1.08224 0-.5991.483-1.08224 1.082-1.08224s1.082.48314 1.082 1.08224c0 .59909-.483 1.08224-1.082 1.08224m1.778 11.38284h-3.556l-.155-.1546v-.6184l.155-.1546h.348c.386 0 .657-.2706.657-.6571v-5.7977c0-.4638-.29-.6571-.676-.6571h-.29l-.194-.1933v-.3671l.155-.1933 2.087-.7537h.464l.116.1353-.116 1.1209v6.706c0 .3865.271.6571.657.6571h.348l.155.1546v.6184zm5.264 0h-3.556l-.155-.1546v-.6184l.155-.1546h.348c.386 0 .657-.2706.657-.6571V8.99167c0-.46382-.29-.65708-.677-.65708h-.29l-.193-.19325v-.36719l.155-.19326 2.087-.7537h.464l.116.13528-.116 1.12089V19.157c0 .3865.27.6571.657.6571h.348l.154.1546v.6184zm5.908.2706c-2.281 0-4.522-1.5074-4.522-5.0441 0-3.1694 2.241-5.044 4.618-5.044 1.701 0 2.745.8117 3.344 1.6041V8.99167c0-.46382-.29-.65708-.677-.65708h-.289l-.194-.19325v-.36719l.155-.19326 2.087-.7537h.464l.116.13528-.116 1.12089V19.157c0 .3865.27.6571.657.6571h.348l.154.1546v.6184l-.154.1546h-2.435l-.155-.1546v-1.2755c-.947 1.2755-2.242 1.7007-3.401 1.7007m.27-.9083c2.126 0 3.17-1.9713 3.17-4.1358 0-2.1644-1.044-4.1357-3.17-4.1357-2.125 0-3.169 1.9713-3.169 4.1357 0 2.1645 1.044 4.1358 3.169 4.1358m16.078.6377h-4.522l-.155-.1546v-.6184l.155-.1546h.618c.387 0 .773-.3865.773-.7731V8.91436c0-.38651-.386-.77302-.773-.77302h-.618l-.155-.15461v-.61842l.155-.15461h8.928l.194.19326v.57977l-.348 2.00987-.155.1546h-.541l-.154-.1546c-.155-1.29482-.696-1.85526-1.74-1.85526h-2.28c-.387 0-.773.38651-.773.77302v4.79274h2.879c.387 0 .773-.2125.773-.657v-.2706l.155-.1546h.618l.155.1546v2.7829l-.155.1546h-.618l-.155-.1546v-.2706c0-.5024-.386-.657-.773-.657h-2.879v4.4062c0 .3866.386.7731.773.7731h.618l.155.1546v.6184zm10.366.2706c-3.266 0-4.909-2.4351-4.909-4.9667 0-2.783 1.952-5.1214 4.812-5.1214 3.015 0 4.561 2.3964 4.561 4.9861l-.154.1546h-7.653c0 1.9712 1.526 3.6525 3.594 3.6525 1.817 0 2.918-.9856 3.633-2.2804l.194-.0773.212.0773.097.1933c-.561 1.9325-2.242 3.382-4.387 3.382m-3.343-6.0297.154.1546h5.373c.425 0 .541-.1932.541-.5411 0-1.3335-.966-2.7829-2.841-2.7829-2.049 0-3.131 1.7393-3.227 3.1694m12.614 6.0297c-1.372 0-2.532-.4252-3.247-.9083l-.096-.1933.502-2.0292.193-.1933h.368l.193.1933c-.135 1.6813 1.082 2.3384 2.106 2.3384 1.025 0 1.72-.5991 1.72-1.5847 0-1.3528-1.082-1.8359-2.261-2.2998-1.565-.6184-2.609-1.3721-2.609-2.6476 0-1.6813 1.276-2.7636 3.208-2.7636.986 0 1.759.2899 2.416.6378l.135.1932.097 2.1259-.193.1932h-.445l-.193-.1932c-.058-1.2949-.734-2.1452-1.894-2.1452-.966 0-1.643.6184-1.643 1.4494 0 .8311.522 1.3528 2.397 2.1065 1.295.5218 2.551 1.2562 2.551 2.7636 0 1.778-1.43 2.9569-3.305 2.9569m8.247 0c-1.643 0-2.435-1.0436-2.435-2.4737v-6.4162h-1.411l-.154-.1546v-.6184l.154-.1546h1.024l1.218-2.31909h.56l.155.15461v2.16448h2.57l.155.1546v.6184l-.155.1546h-2.57v6.3195c0 .7924.503 1.4108 1.295 1.4108.599 0 1.121-.1932 1.643-.8503l.154-.058.309.1546.039.1546c-.135.5411-.889 1.7587-2.551 1.7587m-133.694.0807c.621 0 1.125-.4851 1.125-1.0866 0-.6209-.504-1.1254-1.125-1.1254-.602 0-1.087.5045-1.087 1.1254 0 .6015.485 1.0866 1.087 1.0866m6.27-.2717.155-.1552v-.6209l-.155-.1552h-.349c-.389 0-.66-.2717-.66-.6597V8.11228l.116-1.12538-.116-.13583h-.466l-2.095.75673-.156.19403v.36866l.194.19403h.291c.389 0 .68.19403.68.65971V19.2303c0 .388-.272.6597-.66.6597h-.349l-.156.1552v.6209l.156.1552zm6.32.2717c2.794 0 5.025-1.9985 5.025-5.0642s-2.231-5.0643-5.025-5.0643-5.026 1.9986-5.026 5.0643 2.232 5.0642 5.026 5.0642m0-.9702c-2.038 0-3.357-1.5716-3.357-4.094s1.319-4.0941 3.357-4.0941c2.037 0 3.356 1.5717 3.356 4.0941s-1.319 4.094-3.356 4.094m10.94.9702c2.503 0 3.667-1.5523 4.172-3.1045l-.097-.194-.214-.0971-.194.0776c-.621 1.203-1.63 2.0956-3.182 2.0956-1.979 0-3.492-1.5135-3.492-3.9971 0-2.1925 1.222-3.9194 3.434-3.9194 1.727 0 2.406.9508 2.406 1.8433v.194l.155.1553h.524l.155-.1553-.097-2.1537-.136-.194c-.756-.4269-1.765-.6792-2.891-.6792-2.638 0-5.161 1.8821-5.161 5.2001 0 3.1239 2.27 4.9284 4.618 4.9284m15.802-1.203c-.388 0-.66-.2717-.66-.6597v-7.8389l-.155-.1552h-.563l-.795 1.4358h-.097c-.582-.8343-1.65-1.7075-3.415-1.7075-2.387 0-4.638 1.8821-4.638 5.0643 0 3.5507 2.251 5.0642 4.541 5.0642 1.261 0 2.6-.4269 3.57-2.0179v1.591l.155.1552h2.406l.155-.1552v-.6209l-.155-.1552zm-5.511.291c-2.153 0-3.182-1.9985-3.182-4.1522 0-2.1732 1.029-4.1523 3.182-4.1523 2.096 0 3.182 1.9791 3.182 4.1523 0 2.1537-1.086 4.1522-3.182 4.1522m10.886.6403.155-.1552v-.6209l-.155-.1552h-.349c-.388 0-.66-.2717-.66-.6597V8.11228l.116-1.12538-.116-.13583h-.466l-2.095.75673-.155.19403v.36866l.194.19403h.291c.388 0 .679.19403.679.65971V19.2303c0 .388-.272.6597-.66.6597h-.349l-.155.1552v.6209l.155.1552z"/>
          <path fill="#001e2b" d="M7.93639 2.22478C6.89116 1.00263 5.9911-.238622 5.80722-.49642c-.01935-.019096-.0484-.019096-.06775 0-.18388.257798-1.08394 1.49905-2.12917 2.7212-8.97152 11.27632 1.413 18.88612 1.413 18.88612l.0871.0573c.07742 1.1744.27099 2.8644.27099 2.8644h.77424s.19356-1.6805.27099-2.8644l.08709-.0669c.00968.0096 10.39419-7.6002 1.42268-18.87652M5.76851 20.9486s-.46455-.3915-.59036-.592v-.0191l.56132-12.27884c0-.03819.05808-.03819.05808 0l.56131 12.27884v.0191c-.12581.2005-.59035.592-.59035.592m19.43469-2.5898L20.8759 8.01734l-.0097-.02857h-3.3668v.69514h.5433c.165 0 .3202.06666.4366.18093.1165.11427.1747.26663.1747.42852l-.097 10.40814c0 .3238-.2717.5904-.6016.5999l-.5531.0095v.6857h3.2795v-.6857l-.3396-.0095c-.3299-.0095-.6015-.2761-.6015-.5999V9.89328l4.7154 11.10332c.0679.1618.2232.2666.3978.2666.1747 0 .3299-.1048.3978-.2666l4.6088-10.8557.0679 9.5606c0 .3333-.2717.5999-.6113.6094h-.3493v.6857H32.81v-.6857h-.5239c-.3299 0-.6016-.2761-.6113-.5999l-.0291-10.40812c0-.33329.2717-.59992.6016-.60944l.5627-.00953v-.69514h-3.2794zm30.1728 1.7716c-.107-.1046-.1654-.2473-.1654-.4185v-5.0982c0-.9702-.2918-1.7311-.8755-2.2733-.5739-.5421-1.3716-.8179-2.3639-.8179-1.391 0-2.4903.5516-3.2588 1.6359-.0097.0191-.0389.0286-.0681.0286-.0291 0-.0486-.019-.0486-.0476l-.3599-1.3601h-.6031l-1.5468.8655v.4756h.3989c.1848 0 .3404.0476.4475.1427.107.0951.1653.2378.1653.4375v6.0018c0 .1712-.0583.3138-.1653.4185-.1071.1046-.253.1617-.4281.1617h-.3891v.6943h3.5604v-.6943h-.3891c-.1751 0-.321-.0571-.428-.1617-.107-.1047-.1654-.2473-.1654-.4185v-3.9759c0-.5041.1167-1.0082.3307-1.5028.2238-.4851.5545-.8941.9923-1.2079.4377-.3139.963-.4661 1.5661-.4661.681 0 1.1966.2093 1.5176.6278s.4864.9606.4864 1.6074v4.9079c0 .1713-.0584.3139-.1654.4186-.107.1046-.2529.1617-.428.1617h-.3891v.6943h3.5603v-.6943h-.3891c-.1459.019-.2821-.0381-.3988-.1427M87.904 8.77301c-.9841-.51645-2.084-.78424-3.2708-.78424H80.002v.69816h.4534c.1737 0 .3281.06696.4825.21998.1447.14346.2219.30604.2219.47819v10.2908c0 .1721-.0772.3347-.2219.4782-.1448.1434-.3088.2199-.4825.2199h-.4534v.6982h4.6312c1.1868 0 2.2867-.2678 3.2708-.7842.9842-.5165 1.785-1.2816 2.3639-2.2571s.878-2.1519.878-3.4909c0-1.3389-.2991-2.5057-.878-3.4908-.5886-.9946-1.3797-1.75974-2.3639-2.27619m1.3798 5.74789c0 1.2242-.222 2.2571-.6561 3.0892-.4342.832-1.0131 1.4537-1.7271 1.8554-.714.4016-1.5052.6025-2.3542.6025h-.9359c-.1737 0-.3281-.067-.4824-.22-.1448-.1434-.222-.306-.222-.4782V9.66245c0-.17215.0676-.32517.222-.47819.1447-.14346.3087-.21997.4824-.21997h.9359c.849 0 1.6402.20084 2.3542.60252.714.40169 1.2929 1.02339 1.7271 1.85539.4341.8321.6561 1.8745.6561 3.0987m12.8102.6982c-.428-.4878-1.255-.899-2.2276-1.119 1.3426-.6599 2.0326-1.5876 2.0326-2.7831 0-.6503-.175-1.2337-.525-1.73106-.35-.49732-.846-.89901-1.4784-1.17636-.6322-.27736-1.3714-.42081-2.2079-.42081h-5.2425v.69816h.4182c.1751 0 .3307.06695.4863.21997.1459.14346.2237.30605.2237.4782v10.2908c0 .1721-.0778.3347-.2237.4782-.1459.1434-.3112.2199-.4863.2199h-.4572v.6982h5.69c.8656 0 1.6729-.1435 2.4028-.4304.729-.2869 1.313-.7077 1.731-1.2624.428-.5547.642-1.2338.642-2.018-.01-.8416-.263-1.5589-.778-2.1423m-6.5267 4.6385c-.1459-.1435-.2237-.3061-.2237-.4782V14.76h2.704c.9531 0 1.6826.2391 2.1884.7173.506.4782.759 1.0999.759 1.865 0 .459-.117.9085-.331 1.3198-.224.4208-.554.7555-1.0019 1.0138-.4376.2582-.9823.3921-1.6145.3921h-1.994c-.175.0095-.3306-.067-.4863-.2104m-.2139-6.0827V9.67201c0-.17215.068-.32517.2237-.47819.1459-.14346.3112-.21997.4863-.21997h1.2839c.924 0 1.6048.22953 2.0328.66947.428.44948.6419 1.02338.6419 1.73108 0 .7268-.2042 1.3102-.603 1.7502-.3988.4304-1.0018.6503-1.7994.6503zm-53.5318-1.6424c-.7432-.4004-1.5733-.61-2.4709-.61-.8977 0-1.7374.2001-2.471.61-.7432.4003-1.332.9817-1.7663 1.7156-.4344.7339-.6564 1.5917-.6564 2.5448 0 .9532.222 1.811.6564 2.5449.4343.7339 1.0231 1.3153 1.7663 1.7156s1.5733.61 2.471.61c.8976 0 1.7374-.2002 2.4709-.61.7432-.4003 1.332-.9817 1.7664-1.7156.4343-.7339.6563-1.5917.6563-2.5449 0-.9531-.222-1.8109-.6563-2.5448-.4344-.7339-1.0232-1.3058-1.7664-1.7156m.7143 4.2604c0 1.1724-.2896 2.1255-.8687 2.8118-.5695.6862-1.3513 1.0389-2.3165 1.0389s-1.7471-.3527-2.3165-1.0389c-.5792-.6863-.8687-1.6394-.8687-2.8118 0-1.1723.2895-2.1254.8687-2.8117.5694-.6862 1.3513-1.0389 2.3165-1.0389s1.747.3527 2.3165 1.0389c.5791.6958.8687 1.6394.8687 2.8117m33.4928-4.2604c-.7433-.4004-1.5734-.61-2.471-.61-.8977 0-1.7374.2001-2.4709.61-.7433.4003-1.332.9817-1.7664 1.7156-.4343.7339-.6563 1.5917-.6563 2.5448 0 .9532.222 1.811.6563 2.5449.4344.7339 1.0231 1.3153 1.7664 1.7156.7432.4003 1.5732.61 2.4709.61.8976 0 1.7374-.2002 2.471-.61.7432-.4003 1.332-.9817 1.7663-1.7156s.6563-1.5917.6563-2.5449c0-.9531-.222-1.8109-.6563-2.5448s-1.0328-1.3058-1.7663-1.7156m.7142 4.2604c0 1.1724-.2896 2.1255-.8687 2.8118-.5695.6862-1.3513 1.0389-2.3165 1.0389s-1.747-.3527-2.3165-1.0389c-.5791-.6863-.8687-1.6394-.8687-2.8118 0-1.1818.2896-2.1254.8687-2.8117.5695-.6862 1.3513-1.0389 2.3165-1.0389s1.747.3527 2.3165 1.0389c.5695.6958.8687 1.6394.8687 2.8117m-14.509-4.8704c-.7776 0-1.4872.1627-2.1288.4883-.6416.3255-1.147.766-1.5067 1.3309-.3597.5554-.5444 1.1778-.5444 1.8385 0 .5936.1361 1.1394.418 1.6277.2722.4692.6416.8618 1.1082 1.1874l-1.3901 1.8575c-.175.2298-.1944.5362-.068.7852.136.2585.3888.4117.6804.4117h.3986c-.3889.2586-.6999.565-.9138.9288-.2527.4117-.3791.8426-.3791 1.2831 0 .8234.3694 1.5033 1.0985 2.0108.7193.5074 1.7302.766 3.0037.766.8845 0 1.7303-.1437 2.4982-.4213.7777-.2777 1.4095-.6894 1.8761-1.2256.4763-.5363.7194-1.1874.7194-1.9342 0-.7852-.2917-1.3406-.9721-1.8768-.5833-.45-1.497-.6894-2.6344-.6894h-3.8883c-.0097 0-.0194-.0095-.0194-.0095s-.0097-.0192 0-.0288l1.0109-1.3405c.2722.1245.525.2011.7485.249.2333.0478.4958.067.7874.067.8166 0 1.5554-.1628 2.1969-.4883.6416-.3256 1.1568-.7661 1.5262-1.331.3694-.5553.5541-1.1777.5541-1.8384 0-.7086-.35-2.0012-1.3026-2.6619 0-.0096.0097-.0096.0097-.0096l2.09.2298v-.9479h-3.344c-.5249-.1724-1.0693-.2585-1.6331-.2585m1.1665 5.9844c-.3694.1915-.7679.2969-1.1665.2969-.6512 0-1.2248-.2299-1.7108-.6799-.4861-.45-.7291-1.1107-.7291-1.9533s.243-1.5033.7291-1.9533c.486-.4501 1.0596-.6799 1.7108-.6799.4083 0 .7971.0958 1.1665.2969.3694.1915.6708.4883.9138.8809.2333.3926.3597.8809.3597 1.4554 0 .5841-.1167 1.0724-.3597 1.4554-.2333.3926-.5444.6894-.9138.8809m-2.6343 3.5333h2.6343c.7291 0 1.1957.1436 1.5068.45.311.3064.4666.7181.4666 1.1969 0 .699-.2819 1.2735-.8457 1.7044-.5639.4308-1.3221.6511-2.2553.6511-.8165 0-1.497-.182-1.9927-.5267-.4958-.3447-.7485-.8713-.7485-1.5416 0-.4213.1166-.8139.3499-1.1586s.5152-.5936.8846-.7755"/>
        </g>
        <defs>
          <clipPath id="a">
            <path fill="#fff" d="M0 0h243.166v26.4834H0z"/>
          </clipPath>
        </defs>
      </svg>
    </a>

    <div class="nav-right">
      <ul class="nav-items">
        <li><a href="#why-attend-bento">Why Attend</a></li>
        <!-- <li><a href="#speakers">Speakers</a></li> -->
        <li><a href="#pricing">Pricing</a></li>
        <!-- <li><a href="#sessions">Featured Sessions</a></li> -->
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
    /* Inset by the review "push" drawer width (0 by default) so the fixed nav
       stays aligned with the content when the page is pushed aside. */
    left: var(--review-push, 0px);
    right: 0;
    z-index: 50;
    padding: 24px clamp(16px, 7.86%, 132px);
    transition:
      padding var(--flora-motion-duration-enter) var(--flora-motion-easing-movement),
      left var(--flora-motion-duration-enter) var(--flora-motion-easing-emphasized);
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
