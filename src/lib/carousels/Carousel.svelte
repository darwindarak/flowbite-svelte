<script lang="ts" context="module">
  export type State = {
    images: HTMLImgAttributes[];
    index: number;
  };
</script>

<script lang="ts">
  import { createEventDispatcher, onMount, setContext } from 'svelte';
  import { quintOut } from 'svelte/easing';
  import type { HTMLImgAttributes } from 'svelte/elements';
  import { writable } from 'svelte/store';
  import { fade, type TransitionConfig } from 'svelte/transition';
  import { twMerge } from 'tailwind-merge';
  import Controls from './Controls.svelte';
  import Indicators from './Indicators.svelte';

  type TransitionFunc = (node: HTMLElement, params: any) => TransitionConfig;

  export let images: HTMLImgAttributes[];
  export let index: number = 0;
  export let transition: TransitionFunc = (x) => fade(x, { duration: 700, easing: quintOut });
  export let duration: number = 0;

  // Carousel
  let divClass: string = 'overflow-hidden relative rounded-lg h-56 sm:h-64 xl:h-80 2xl:h-96';

  const dispatch = createEventDispatcher();

  const { set, subscribe, update } = writable<State>({ images, index });

  const state = { set: (s: State) => set({ index: ((s.index % images.length) + images.length) % images.length, images: s.images }), subscribe, update };

  setContext('state', state);

  subscribe((s) => {
    index = s.index;
    dispatch('change', images[index]);
  });

  onMount(() => dispatch('change', images[index]));

  $: state.set({ images, index });

  const nextSlide = () => (index += 1);
  const prevSlide = () => (index -= 1);

  const loop = (node: HTMLElement, duration: number) => {
    carouselDiv = node; // used by DragStart

    // loop timer
    let intervalId: any;

    if (duration > 0) intervalId = setInterval(nextSlide, duration);

    return {
      update: (duration: number) => {
        clearInterval(intervalId);
        if (duration > 0) intervalId = setInterval(nextSlide, duration);
      },
      destroy: () => clearInterval(intervalId)
    };
  };

  type ActiveDragGesture = {
    start: number;
    position: number;
    width: number;
    timestamp: number;
  };

  let activeDragGesture: ActiveDragGesture | undefined;

  let carouselDiv: HTMLElement;
  let percentOffset: number = 0;
  let touchEvent: MouseEvent | TouchEvent | null = null;

  const getPositionFromEvent = (evt: MouseEvent | TouchEvent) => {
    const mousePos = (evt as MouseEvent)?.clientX;
    if (mousePos) return mousePos;

    let touchEvt = evt as TouchEvent;
    if (/^touch/.test(touchEvt?.type)) {
      return touchEvt.touches[0].clientX;
    }
  };

  const onDragStart = (evt: MouseEvent | TouchEvent) => {
    touchEvent = evt;
    evt.preventDefault();
    const start = getPositionFromEvent(evt);
    const width = carouselDiv.getBoundingClientRect().width;
    if (start === undefined || width === undefined) return;
    activeDragGesture = {
      start,
      position: start,
      width,
      timestamp: Date.now()
    };
  };

  $: onDragMove =
    activeDragGesture === undefined
      ? undefined
      : (evt: MouseEvent | TouchEvent) => {
          const position = getPositionFromEvent(evt);
          if (!activeDragGesture || position === undefined) return;
          const { start, width } = activeDragGesture;
          percentOffset = Math.min(100, Math.max(-100, ((position - start) / width) * 100));
          activeDragGesture.position = position;
        };

  $: onDragStop =
    activeDragGesture === undefined
      ? undefined
      : (evt: MouseEvent | TouchEvent) => {
          // These might be exposed one day, keep them safely tucked away as constants.
          const SWIPE_MAX_DURATION = 250;
          const SWIPE_MIN_DISTANCE = 30;
          const DRAG_MIN_PERCENT = 50;

          if (activeDragGesture) {
            const { timestamp, position, start } = activeDragGesture;
            const duration = Date.now() - timestamp;
            const distance = position - start;

            if (Math.abs(distance) >= SWIPE_MIN_DISTANCE && duration <= SWIPE_MAX_DURATION && duration > 0) {
              if (distance > 0) prevSlide();
              else nextSlide();
            } else if (percentOffset > DRAG_MIN_PERCENT) prevSlide();
            else if (percentOffset < -DRAG_MIN_PERCENT) nextSlide();
            else {
              // The gesture is a tap not drag, so manually issue a click event to trigger tap click gestures lost via preventDefault
              touchEvent?.target?.dispatchEvent(
                new Event('click', {
                  bubbles: true
                })
              );
            }
          }
          percentOffset = 0;
          activeDragGesture = undefined;
          touchEvent = null;
        };
</script>

<!-- The move listeners go here, so things keep working if the touch strays out of the element. -->
<svelte:document on:mousemove={onDragMove} on:mouseup={onDragStop} on:touchmove={onDragMove} on:touchend={onDragStop} />
<div bind:this={carouselDiv} class="relative" on:mousedown={onDragStart} on:touchstart={onDragStart} role="button" aria-label="Draggable Carousel" tabindex="0">
  <div style={`transform: translateX(${percentOffset}%)`} {...$$restProps} class={twMerge(divClass, activeDragGesture === undefined ? 'transition-transform' : '', $$props.class)} use:loop={duration}>
    {#key images[index]}
      <img alt="..." {...images[index]} transition:transition={{}} class="absolute block w-full -translate-x-1/2 -translate-y-1/2 top-1/2 left-1/2 object-cover" />
    {/key}
  </div>
  <slot {index} {Controls} {Indicators} />
</div>

<!--
@component
[Go to docs](https://flowbite-svelte.com/)
## Props
@prop export let images: HTMLImgAttributes[];
@prop export let index: number = 0;
@prop export let transition: TransitionFunc = (x) => fade(x, { duration: 700, easing: quintOut });
@prop export let duration: number = 0;
-->
