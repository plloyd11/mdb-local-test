<script lang="ts">
  const items = [
    {
      q: 'Who should attend mongodb.local?',
      a: 'mongodb.local is for developers, data scientists, architects, and technical leaders building with MongoDB and AI — from first-time users to seasoned experts.',
    },
    {
      q: 'Is parking available at the venue?',
      a: 'Limited paid parking is available nearby, but we recommend public transit or rideshare, as the venue fills up quickly.',
    },
    {
      q: 'Will sessions be recorded?',
      a: 'Yes. Most keynote and breakout sessions are recorded and shared with registered attendees after the event.',
    },
    {
      q: 'Can I get a group discount?',
      a: 'Groups of five or more can request discounted tickets — reach out to the events team for a group code before you register.',
    },
    {
      q: "What's your refund policy?",
      a: "Tickets are refundable up to 14 days before the event. After that, you're welcome to transfer your ticket to a colleague.",
    },
  ];

  let openStates = $state(items.map(() => false));

  function toggle(i: number) {
    openStates[i] = !openStates[i];
  }
</script>

<div class="faq-list">
  {#each items as item, i}
    <div class="faq-item" class:open={openStates[i]}>
      <button
        class="faq-q"
        id={`faq-q-${i}`}
        aria-expanded={openStates[i]}
        aria-controls={`faq-panel-${i}`}
        onclick={() => toggle(i)}
      >
        <span class="faq-q-text">{item.q}</span>
        <span class="faq-icon" aria-hidden="true"></span>
      </button>
      <div class="faq-panel" id={`faq-panel-${i}`} role="region" aria-labelledby={`faq-q-${i}`}>
        <div class="faq-panel-inner">
          <p>{item.a}</p>
        </div>
      </div>
    </div>
  {/each}
</div>

<style>
  .faq-list {
    display: flex;
    flex-direction: column;
  }

  .faq-item {
    border-top: 1px solid rgba(255, 255, 255, 0.16);
  }

  .faq-item:last-child {
    border-bottom: 1px solid rgba(255, 255, 255, 0.16);
  }

  .faq-q {
    width: 100%;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 48px;
    padding: 32px 0;
    margin: 0;
    background: none;
    border: 0;
    cursor: pointer;
    text-align: left;
    color: #ffffff;
  }

  .faq-q-text {
    font-family: 'Euclid Circular A', sans-serif;
    font-weight: 500;
    font-size: 24px;
    line-height: 32px;
    color: #ffffff;
    transition: color var(--flora-motion-duration-feedback) var(--flora-motion-easing-feedback);
  }

  .faq-q:hover .faq-q-text {
    color: #00ed64;
  }

  .faq-q:focus-visible {
    outline: 2px solid #00ed64;
    outline-offset: 4px;
    border-radius: 4px;
  }

  /* Plus → minus morph */
  .faq-icon {
    position: relative;
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    margin-top: 0;
  }

  .faq-icon::before,
  .faq-icon::after {
    content: '';
    position: absolute;
    left: 50%;
    top: 50%;
    background: #00ed64;
    border-radius: 2px;
  }

  .faq-icon::before {
    width: 20px;
    height: 2px;
    transform: translate(-50%, -50%);
  }

  .faq-icon::after {
    width: 2px;
    height: 20px;
    transform: translate(-50%, -50%);
    transition: transform var(--flora-motion-duration-feedback) var(--flora-motion-easing-movement);
  }

  .faq-item.open .faq-icon::after {
    /* rotate the vertical bar onto the horizontal one → minus */
    transform: translate(-50%, -50%) rotate(90deg);
  }

  /* Animated panel (grid-rows 0fr → 1fr) */
  .faq-panel {
    display: grid;
    grid-template-rows: 0fr;
    transition: grid-template-rows var(--flora-motion-duration-enter) var(--flora-motion-easing-movement);
  }

  .faq-item.open .faq-panel {
    grid-template-rows: 1fr;
  }

  .faq-panel-inner {
    overflow: hidden;
  }

  .faq-panel-inner p {
    max-width: 900px;
    margin: 0;
    padding: 0 48px 32px 0;
    font-family: 'Euclid Circular A', sans-serif;
    font-weight: 400;
    font-size: 18px;
    line-height: 32px;
    color: #b8c4c2;
  }

  @media (max-width: 600px) {
    .faq-q {
      gap: 24px;
    }

    .faq-q-text {
      font-size: 20px;
    }
  }
</style>
