<script lang="ts">
  import { onDestroy, onMount } from 'svelte';

  export let hideUserCursor = false;

  const invisTimeStage1 = 2000;
  const invisTimeStage2 = 4000;

  let cX = 0;
  let cY = 0;
  let down = false;
  let invisStage = 0;
  let invisTimeout: ReturnType<typeof setTimeout> | undefined = undefined;
  const mm = (ev: MouseEvent) => {
    cX = ev.clientX;
    cY = ev.clientY;
    if (invisTimeout) clearTimeout(invisTimeout);
    invisTimeout = setTimeout(() => {
      invisStage = 1;
      invisTimeout = setTimeout(() => {
        invisStage = 2;
      }, invisTimeStage2);
    }, invisTimeStage1);
    invisStage = 0;
  };
  let md = () => (down = true);
  let mu = () => (down = false);
  $: {
    if (typeof document !== 'undefined')
      document.documentElement.classList[hideUserCursor ? 'add' : 'remove'](
        'hidecur',
      );
  }
  onMount(() => {
    document.body.addEventListener('mousemove', mm);
    document.body.addEventListener('mousedown', md);
    document.body.addEventListener('mouseup', mu);
  });
  onDestroy(() => {
    document.body.removeEventListener('mousemove', mm);
    document.body.removeEventListener('mousedown', md);
    document.body.removeEventListener('mouseup', mu);
  });
</script>

<div
  style="--curs-y:{cY}px;--curs-x:{cX}px;"
  class="c"
  class:down
  data-invis-stage={invisStage}
/>

<style lang="scss">
  :global(html.hidecur) {
    &,
    :global(*) {
      cursor: none !important;
    }
  }
  div.c {
    pointer-events: none;
    position: fixed;
    transform: translate(-50%, -50%);
    width: 22px;
    height: 22px;
    top: var(--curs-y);
    left: var(--curs-x);
    border-radius: 100%;
    border: 2px solid #fff;
    --opacityTime: 0.4s;
    --opacityEasing: ease-out;
    transition: ease-out 0.1s, opacity var(--opacityEasing) var(--opacityTime);
    z-index: 500;
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 1;
    &.down {
      width: 25px;
      height: 25px;
    }
    &[data-invis-stage='1'] {
      --opacityTime: 2s;
      opacity: 0.7;
    }
    &[data-invis-stage='2'] {
      --opacityTime: 3s;
      --opacityEasing: ease-in-out;
      opacity: 0;
    }
  }
</style>
