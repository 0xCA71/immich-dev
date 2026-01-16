<script lang="ts">
  import { TimelineManager } from '$lib/managers/timeline-manager/timeline-manager.svelte';
  import type { ScrubberDayBucket, ScrubberMonth, ViewportTopMonth } from '$lib/managers/timeline-manager/types';
  import YearFilter from '$lib/components/timeline/YearFilter.svelte';
  import { mobileDevice } from '$lib/stores/mobile-device.svelte';
  import { getTabbable } from '$lib/utils/focus-util';
  import { type ScrubberListener } from '$lib/utils/timeline-util';
  import { AssetOrder } from '@immich/sdk';
  import { Icon } from '@immich/ui';
  import { mdiPlay } from '@mdi/js';
  import { clamp } from 'lodash-es';
  import { onMount } from 'svelte';
  import { fade, fly } from 'svelte/transition';

  interface Props {
    /** Offset from the top of the timeline (e.g., for headers) */
    timelineTopOffset?: number;
    /** Offset from the bottom of the timeline (e.g., for footers) */
    timelineBottomOffset?: number;
    /** Total height of the scrubber component */
    height?: number;
    /** Timeline manager instance that controls the timeline state */
    timelineManager: TimelineManager;
    /** Overall scroll percentage through the entire timeline (0-1), used when no specific month is targeted */
    timelineScrollPercent?: number;
    /** The percentage of scroll through the month that is currently intersecting the top boundary of the viewport */
    viewportTopMonthScrollPercent?: number;
    /** The year/month of the timeline month at the top of the viewport */
    viewportTopMonth?: ViewportTopMonth;

    /** Width of the scrubber component in pixels (bindable for parent component margin adjustments) */
    scrubberWidth?: number;
    /** Callback fired when user interacts with the scrubber to navigate */
    onScrub?: ScrubberListener;
    /** Callback fired when keyboard events occur on the scrubber */
    onScrubKeyDown?: (event: KeyboardEvent, element: HTMLElement) => void;
    /** Callback fired when scrubbing starts */
    startScrub?: ScrubberListener;
    /** Callback fired when scrubbing stops */
    stopScrub?: ScrubberListener;

    years?: number[];
    selectedYear?: number;
    onSelectYear?: (year?: number) => void;
  }

  let {
    timelineTopOffset = 0,
    timelineBottomOffset = 0,
    height = 0,
    timelineManager,
    timelineScrollPercent = 0,
    viewportTopMonthScrollPercent = 0,
    viewportTopMonth = undefined,
    onScrub = undefined,
    onScrubKeyDown = undefined,
    startScrub = undefined,
    stopScrub = undefined,
    years = [],
    selectedYear = undefined,
    onSelectYear = undefined,
    scrubberWidth = $bindable(),
  }: Props = $props();

  let isHover = $state(false);
  let isDragging = $state(false);
  let isHoverOnYearFilter = $state(false);
  let isHoverOnPaddingTop = $state(false);
  let isHoverOnPaddingBottom = $state(false);
  let hoverY = $state(0);
  let clientY = $state(0);
  let windowHeight = $state(0);
  let scrollBar: HTMLElement | undefined = $state();

  const toScrollY = (percent: number) => percent * (height - (PADDING_TOP + PADDING_BOTTOM));
  const toTimelineY = (scrollY: number) => scrollY / (height - (PADDING_TOP + PADDING_BOTTOM));

  const usingMobileDevice = $derived(mobileDevice.pointerCoarse);

  const MOBILE_WIDTH = 20;
  const DESKTOP_WIDTH = 60;
  const DESKTOP_WIDTH_WITH_FILTER = 96;
  const HOVER_DATE_HEIGHT = 31.75;
  const YEAR_FILTER_HEIGHT = 44;
  const showYearFilter = $derived.by(() => !usingMobileDevice && years.length > 0 && !!onSelectYear);
  const PADDING_TOP = $derived(usingMobileDevice ? 25 : HOVER_DATE_HEIGHT + (showYearFilter ? YEAR_FILTER_HEIGHT : 0));
  const PADDING_BOTTOM = $derived(usingMobileDevice ? 25 : 10);
  const MIN_YEAR_LABEL_DISTANCE = 16;
  const MIN_DOT_DISTANCE = 8;

  const width = $derived.by(() => {
    if (isDragging) {
      return '100vw';
    }
    if (usingMobileDevice) {
      if (timelineManager.scrolling) {
        return MOBILE_WIDTH + 'px';
      }
      return '0px';
    }
    return (showYearFilter ? DESKTOP_WIDTH_WITH_FILTER : DESKTOP_WIDTH) + 'px';
  });
  $effect(() => {
    scrubberWidth = usingMobileDevice ? MOBILE_WIDTH : showYearFilter ? DESKTOP_WIDTH_WITH_FILTER : DESKTOP_WIDTH;
  });

  const scrubberLabelMode = $derived.by(() => {
    if (selectedYear !== undefined) {
      return 'month';
    }
    return 'year';
  });

  const toScrollFromMonthGroupPercentage = (
    scrubberMonth: ViewportTopMonth,
    scrubberMonthPercent: number,
    scrubOverallPercent: number,
  ) => {
    if (scrubberMonth === 'lead-in') {
      return relativeTopOffset * scrubberMonthPercent;
    } else if (scrubberMonth === 'lead-out') {
      let offset = relativeTopOffset;
      for (const segment of segments) {
        offset += segment.height;
      }
      return offset + relativeBottomOffset * scrubberMonthPercent;
    } else if (scrubberMonth) {
      let offset = relativeTopOffset;
      let match = false;
      for (const segment of segments) {
        if (segment.month === scrubberMonth.month && segment.year === scrubberMonth.year) {
          offset += scrubberMonthPercent * segment.height;
          match = true;
          break;
        }
        offset += segment.height;
      }
      if (!match) {
        offset += scrubberMonthPercent * relativeBottomOffset;
      }
      return offset;
    } else {
      return scrubOverallPercent * (height - (PADDING_TOP + PADDING_BOTTOM));
    }
  };
  const scrollY = $derived(
    toScrollFromMonthGroupPercentage(viewportTopMonth, viewportTopMonthScrollPercent, timelineScrollPercent),
  );
  const timelineFullHeight = $derived(timelineManager.scrubberTimelineHeight);
  const relativeTopOffset = $derived(toScrollY(timelineTopOffset / timelineFullHeight));
  const relativeBottomOffset = $derived(toScrollY(timelineBottomOffset / timelineFullHeight));

  type Segment = {
    count: number;
    height: number;
    dateFormatted: string;
    year: number;
    month: number;
    dayBuckets?: ScrubberDayBucket[];
    hasLabel: boolean;
    hasDot: boolean;
  };

  const calculateSegments = (months: ScrubberMonth[], labelMode: 'month' | 'year') => {
    let verticalSpanWithoutLabel = 0;
    let verticalSpanWithoutDot = 0;

    let segments: Segment[] = [];
    let previousLabeledSegment: Segment | undefined;

    let top = 0;

    // Process months in reverse order to pick labels, then reverse for display
    const reversed = [...months].reverse();

    for (const scrubMonth of reversed) {
      const scrollBarPercentage = scrubMonth.height / timelineFullHeight;

      const segment = {
        top,
        count: scrubMonth.assetCount,
        height: toScrollY(scrollBarPercentage),
        dateFormatted: scrubMonth.title,
        year: scrubMonth.year,
        month: scrubMonth.month,
        dayBuckets: scrubMonth.dayBuckets,
        hasLabel: false,
        hasDot: false,
      };
      top += segment.height;

      if (labelMode === 'month') {
        segment.hasLabel = true;
        segment.hasDot = false;
        segments.push(segment);
        continue;
      }

      if (previousLabeledSegment) {
        if (previousLabeledSegment.year !== segment.year && verticalSpanWithoutLabel > MIN_YEAR_LABEL_DISTANCE) {
          verticalSpanWithoutLabel = 0;
          segment.hasLabel = true;
          previousLabeledSegment = segment;
        }
        if (segment.height > 5 && verticalSpanWithoutDot > MIN_DOT_DISTANCE) {
          segment.hasDot = true;
          verticalSpanWithoutDot = 0;
        }
      } else {
        segment.hasDot = true;
        segment.hasLabel = true;
        previousLabeledSegment = segment;
      }
      verticalSpanWithoutLabel += segment.height;
      verticalSpanWithoutDot += segment.height;
      segments.push(segment);
    }
    segments.reverse();

    return segments;
  };
  let activeSegment: HTMLElement | undefined = $state();
  const segments = $derived.by(() => calculateSegments(timelineManager.scrubberMonths, scrubberLabelMode));

  const isAscending = $derived.by(() => timelineManager.getAssetOrder() === AssetOrder.Asc);

  const formatScrubberLabel = (segment: Segment) => (scrubberLabelMode === 'month' ? `${segment.month}月` : `${segment.year}`);

  const formatDayFromMonthPercent = (year: number, month: number, percent: number, dayBuckets?: ScrubberDayBucket[]) => {
    const effectivePercent = clamp(isAscending ? percent : 1 - percent, 0, 1);
    if (dayBuckets && dayBuckets.length > 0) {
      const total = dayBuckets.reduce((acc, b) => acc + b.count, 0);
      if (total > 0) {
        const target = effectivePercent * total;
        let cumulative = 0;
        for (const bucket of dayBuckets) {
          cumulative += bucket.count;
          if (cumulative >= target) {
            const date = new Date(Date.UTC(year, month - 1, bucket.day));
            return date.toISOString().slice(0, 10);
          }
        }
        const last = dayBuckets.at(-1);
        if (last) {
          const date = new Date(Date.UTC(year, month - 1, last.day));
          return date.toISOString().slice(0, 10);
        }
      }
    }

    const date = new Date(Date.UTC(year, month - 1, 1));
    const daysInMonth = new Date(Date.UTC(year, month, 0)).getUTCDate();
    const day = Math.min(daysInMonth, Math.max(1, Math.floor(effectivePercent * daysInMonth) + 1));
    date.setUTCDate(day);
    return date.toISOString().slice(0, 10);
  };

  const getMonthSegment = (year: number, month: number) => segments.find((s) => s.year === year && s.month === month);
  const hoverLabel = $derived.by(() => {
    if (isHoverOnPaddingTop) {
      const first = segments.at(0);
      if (scrubberLabelMode === 'month' && first) {
        return formatDayFromMonthPercent(first.year, first.month, 0, first.dayBuckets);
      }
      return first?.dateFormatted;
    }
    if (isHoverOnPaddingBottom) {
      const last = segments.at(-1);
      if (scrubberLabelMode === 'month' && last) {
        return formatDayFromMonthPercent(last.year, last.month, 1, last.dayBuckets);
      }
      return last?.dateFormatted;
    }
    if (scrubberLabelMode === 'month') {
      return dayHoverLabel || activeSegment?.dataset.label;
    }
    return activeSegment?.dataset.label;
  });
  const segmentDate: ViewportTopMonth = $derived.by(() => {
    if (activeSegment?.dataset.id === 'lead-in') {
      return 'lead-in';
    }
    if (activeSegment?.dataset.id === 'lead-out') {
      return 'lead-out';
    }
    if (!activeSegment?.dataset.segmentYearMonth) {
      return undefined;
    }
    const [year, month] = activeSegment.dataset.segmentYearMonth.split('-').map(Number);
    return { year, month };
  });
  const scrollSegment = $derived.by(() => {
    const y = scrollY;
    let cur = relativeTopOffset;
    for (const segment of segments) {
      if (y < cur + segment.height) {
        return segment;
      }
      cur += segment.height;
    }
    return null;
  });
  const scrollDayLabel = $derived.by(() => {
    if (scrubberLabelMode !== 'month') {
      return undefined;
    }
    if (!viewportTopMonth || viewportTopMonth === 'lead-in' || viewportTopMonth === 'lead-out') {
      return undefined;
    }
    const seg = getMonthSegment(viewportTopMonth.year, viewportTopMonth.month);
    return formatDayFromMonthPercent(
      viewportTopMonth.year,
      viewportTopMonth.month,
      viewportTopMonthScrollPercent,
      seg?.dayBuckets,
    );
  });
  const dayHoverLabel = $derived.by(() => {
    if (scrubberLabelMode !== 'month') {
      return undefined;
    }
    if (!scrollBar) {
      return undefined;
    }

    const rect = scrollBar.getBoundingClientRect()!;
    const x = rect.left + rect.width / 2;
    const { monthGroupPercentY } = getActive(x, clientY);
    const segment = segmentDate;
    if (!segment || segment === 'lead-in' || segment === 'lead-out') {
      return undefined;
    }
    const seg = getMonthSegment(segment.year, segment.month);
    return formatDayFromMonthPercent(segment.year, segment.month, monthGroupPercentY, seg?.dayBuckets);
  });
  const scrollHoverLabel = $derived.by(() => {
    if (scrubberLabelMode === 'month') {
      const first = segments.at(0);
      const last = segments.at(-1);

      if (first && scrollY !== undefined && scrollY < relativeTopOffset) {
        return formatDayFromMonthPercent(first.year, first.month, 0, first.dayBuckets);
      }

      if (last && scrollY !== undefined) {
        let offset = relativeTopOffset;
        for (const segment of segments) {
          offset += segment.height;
        }
        if (scrollY > offset) {
          return formatDayFromMonthPercent(last.year, last.month, 1, last.dayBuckets);
        }
      }
    }

    if (scrollY !== undefined) {
      if (scrollY < relativeTopOffset) {
        return segments.at(0)?.dateFormatted;
      } else {
        let offset = relativeTopOffset;
        for (const segment of segments) {
          offset += segment.height;
        }
        if (scrollY > offset) {
          return segments.at(-1)?.dateFormatted;
        }
      }
    }
    const label = scrollSegment?.dateFormatted || '';
    if (scrubberLabelMode === 'month') {
      return scrollDayLabel || label;
    }
    return label;
  });

  const findElementBestY = (elements: Element[], y: number, ...ids: string[]) => {
    if (ids.length === 0) {
      return undefined;
    }
    const filtered = elements.filter((element) => {
      if (element instanceof HTMLElement && element.dataset.id) {
        return ids.includes(element.dataset.id);
      }
      return false;
    }) as HTMLElement[];
    const imperfect = [];
    for (const element of filtered) {
      const boundingClientRect = element.getBoundingClientRect();
      if (boundingClientRect.y > y) {
        imperfect.push({
          element,
          boundingClientRect,
        });
        continue;
      }
      if (y <= boundingClientRect.y + boundingClientRect.height) {
        return {
          element,
          boundingClientRect,
        };
      }
    }
    return imperfect.at(0);
  };

  const getActive = (x: number, y: number) => {
    const elements = document.elementsFromPoint(x, y);
    const bestElement = findElementBestY(elements, y, 'time-segment', 'lead-in', 'lead-out');

    if (bestElement) {
      const segment = bestElement.element;
      const boundingClientRect = bestElement.boundingClientRect;
      const sy = boundingClientRect.y;
      const relativeY = y - sy;
      const monthGroupPercentY = relativeY / boundingClientRect.height;
      return {
        isOnPaddingTop: false,
        isOnPaddingBottom: false,
        segment,
        monthGroupPercentY,
      };
    }

    // check if padding
    const bar = findElementBestY(elements, 0, 'scrubber');
    let isOnPaddingTop = false;
    let isOnPaddingBottom = false;

    if (bar) {
      const sr = bar.boundingClientRect;
      if (y < sr.top + PADDING_TOP) {
        isOnPaddingTop = true;
      }
      if (y > sr.bottom - PADDING_BOTTOM - 1) {
        isOnPaddingBottom = true;
      }
    }

    return {
      isOnPaddingTop,
      isOnPaddingBottom,
      segment: undefined,
      monthGroupPercentY: 0,
    };
  };

  const handleMouseEvent = (event: { clientY: number; isDragging?: boolean }) => {
    const wasDragging = isDragging;

    isDragging = event.isDragging ?? isDragging;
    clientY = event.clientY;

    if (!scrollBar) {
      return;
    }

    const rect = scrollBar.getBoundingClientRect()!;
    const lower = 0;
    const upper = rect?.height - (PADDING_TOP + PADDING_BOTTOM);
    hoverY = clamp(clientY - rect?.top - PADDING_TOP, lower, upper);
    const x = rect!.left + rect!.width / 2;
    const { segment, monthGroupPercentY, isOnPaddingTop, isOnPaddingBottom } = getActive(x, clientY);
    activeSegment = segment;
    isHoverOnPaddingTop = isOnPaddingTop;
    isHoverOnPaddingBottom = isOnPaddingBottom;

    const scrubData = {
      scrubberMonth: segmentDate,
      overallScrollPercent: toTimelineY(hoverY),
      scrubberMonthScrollPercent: monthGroupPercentY,
    };
    if (wasDragging === false && isDragging) {
      void startScrub?.(scrubData);
      void onScrub?.(scrubData);
    }
    if (wasDragging && !isDragging) {
      void stopScrub?.(scrubData);
      return;
    }
    if (!isDragging) {
      return;
    }
    void onScrub?.(scrubData);
  };
  const getTouch = (event: TouchEvent) => {
    // desktop safari does not support this since Apple does not have desktop touch devices
    // eslint-disable-next-line tscompat/tscompat
    if (event.touches.length === 1) {
      // eslint-disable-next-line tscompat/tscompat
      return event.touches[0];
    }
    return null;
  };
  const onTouchStart = (event: TouchEvent) => {
    const touch = getTouch(event);
    if (!touch) {
      isHover = false;
      return;
    }
    // desktop safari does not support this since Apple does not have desktop touch devices
    // eslint-disable-next-line tscompat/tscompat
    const elements = document.elementsFromPoint(touch.clientX, touch.clientY);
    const isHoverScrollbar =
      findElementBestY(elements, 0, 'scrubber', 'time-label', 'lead-in', 'lead-out') !== undefined;

    isHover = isHoverScrollbar;

    if (isHoverScrollbar) {
      handleMouseEvent({
        // eslint-disable-next-line tscompat/tscompat
        clientY: touch.clientY,
        isDragging: true,
      });
    }
  };
  const onTouchEnd = () => {
    if (isHover) {
      isHover = false;
    }
    handleMouseEvent({
      clientY,
      isDragging: false,
    });
  };
  const onTouchMove = (event: TouchEvent) => {
    const touch = getTouch(event);
    if (touch && isDragging) {
      handleMouseEvent({
        // eslint-disable-next-line tscompat/tscompat
        clientY: touch.clientY,
      });
    } else {
      isHover = false;
    }
  };
  onMount(() => {
    document.addEventListener('touchmove', onTouchMove, { capture: true, passive: true });
    return () => {
      document.removeEventListener('touchmove', onTouchMove, true);
    };
  });

  onMount(() => {
    document.addEventListener('touchstart', onTouchStart, { capture: true, passive: true });
    document.addEventListener('touchend', onTouchEnd, { capture: true, passive: true });
    return () => {
      document.removeEventListener('touchstart', onTouchStart, true);
      document.removeEventListener('touchend', onTouchEnd, true);
    };
  });

  const isTabEvent = (event: KeyboardEvent) => event?.key === 'Tab';
  const isTabForward = (event: KeyboardEvent) => isTabEvent(event) && !event.shiftKey;
  const isTabBackward = (event: KeyboardEvent) => isTabEvent(event) && event.shiftKey;
  const isArrowUp = (event: KeyboardEvent) => event?.key === 'ArrowUp';
  const isArrowDown = (event: KeyboardEvent) => event?.key === 'ArrowDown';

  const handleFocus = (event: KeyboardEvent) => {
    const forward = isTabForward(event);
    const backward = isTabBackward(event);
    if (forward || backward) {
      event.preventDefault();

      const focusable = getTabbable(document.body);
      if (scrollBar) {
        const index = focusable.indexOf(scrollBar);
        if (index !== -1) {
          let next: HTMLElement;
          next = forward
            ? (focusable[(index + 1) % focusable.length] as HTMLElement)
            : (focusable[(index - 1) % focusable.length] as HTMLElement);
          next.focus();
        }
      }
    }
  };
  const handleAccessibility = (event: KeyboardEvent) => {
    if (isTabEvent(event)) {
      handleFocus(event);
      return true;
    }
    if (isArrowUp(event)) {
      let next;
      if (scrollSegment) {
        const idx = segments.indexOf(scrollSegment);
        next = idx === -1 ? segments.at(-2) : segments[idx - 1];
      } else {
        next = segments.at(-2);
      }
      if (next) {
        event.preventDefault();
        void onScrub?.({
          scrubberMonth: { year: next.year, month: next.month },
          overallScrollPercent: -1,
          scrubberMonthScrollPercent: 0,
        });
        return true;
      }
    }
    if (isArrowDown(event) && scrollSegment) {
      const idx = segments.indexOf(scrollSegment);
      if (idx !== -1) {
        const next = segments[idx + 1];
        if (next) {
          event.preventDefault();
          void onScrub?.({
            scrubberMonth: { year: next.year, month: next.month },
            overallScrollPercent: -1,
            scrubberMonthScrollPercent: 0,
          });
          return true;
        }
      }
    }
    return false;
  };
  const keydown = (event: KeyboardEvent) => {
    let handled = handleAccessibility(event);
    if (!handled) {
      onScrubKeyDown?.(event, event.currentTarget as HTMLElement);
    }
  };
</script>

<svelte:window
  bind:innerHeight={windowHeight}
  onmousemove={({ clientY }) => (isDragging || isHover) && handleMouseEvent({ clientY })}
  onmousedown={({ clientY }) => isHover && !isHoverOnYearFilter && handleMouseEvent({ clientY, isDragging: true })}
  onmouseup={({ clientY }) => handleMouseEvent({ clientY, isDragging: false })}
/>

<div
  transition:fly={{ x: 50, duration: 250 }}
  tabindex="0"
  role="scrollbar"
  aria-controls="time-label"
  aria-valuetext={hoverLabel}
  aria-valuenow={scrollY + PADDING_TOP}
  aria-valuemax={toScrollY(1)}
  aria-valuemin={toScrollY(0)}
  data-id="scrubber"
  class="absolute end-0 z-1 select-none hover:cursor-row-resize"
  style:padding-top={PADDING_TOP + 'px'}
  style:padding-bottom={PADDING_BOTTOM + 'px'}
  style:width
  style:height={height + 'px'}
  style:background-color={isDragging ? 'transparent' : 'transparent'}
  bind:this={scrollBar}
  onmouseenter={() => (isHover = true)}
  onmouseleave={() => (isHover = false)}
  onkeydown={keydown}
  draggable="false"
>
  {#if showYearFilter}
    <div
      class="absolute end-0 top-0 z-2 pointer-events-auto"
      style:height={YEAR_FILTER_HEIGHT + 'px'}
      onmouseenter={() => (isHoverOnYearFilter = true)}
      onmouseleave={() => (isHoverOnYearFilter = false)}
    >
      <YearFilter years={years} {selectedYear} {onSelectYear} />
    </div>
  {/if}
  {#if !usingMobileDevice && hoverLabel && (isHover || isDragging)}
    <div
      id="time-label"
      class={[
        { 'border-b-2': isDragging },
        { 'rounded-bl-md': !isDragging },
        'bg-light truncate opacity-85 pointer-events-none absolute end-0 min-w-20 max-w-64 w-fit rounded-ss-md border-b-2 border-primary py-1 px-1 text-sm font-medium shadow-[0_0_8px_rgba(0,0,0,0.25)] z-1',
      ]}
      style:top="{hoverY + 2}px"
    >
      {hoverLabel}
    </div>
  {/if}
  {#if usingMobileDevice && ((timelineManager.scrolling && scrollHoverLabel) || isHover || isDragging)}
    <div
      id="time-label"
      class="rounded-s-full w-8 ps-2 text-white bg-immich-primary dark:bg-gray-600 hover:cursor-pointer select-none"
      style:top="{PADDING_TOP + (scrollY - 50 / 2)}px"
      style:height="50px"
      style:right="0"
      style:position="absolute"
      in:fade={{ duration: 200 }}
      out:fade={{ duration: 200 }}
    >
      <Icon icon={mdiPlay} size="20" class="-rotate-90 relative top-2.25 -end-0.5" />
      <Icon icon={mdiPlay} size="20" class="rotate-90 relative top-px -end-0.5" />
      {#if (timelineManager.scrolling && scrollHoverLabel) || isHover || isDragging}
        <p
          transition:fade={{ duration: 200 }}
          style:bottom={50 / 2 - 30 / 2 + 'px'}
          style:right="36px"
          style:width="fit-content"
          class="truncate pointer-events-none absolute text-sm rounded-full w-8 py-2 px-4 text-white bg-immich-primary/90 dark:bg-gray-500 hover:cursor-pointer select-none font-semibold"
        >
          {scrollHoverLabel}
        </p>
      {/if}
    </div>
  {/if}
  <!-- Scroll Position Indicator Line -->
  {#if !usingMobileDevice && !isDragging}
    <div class="absolute end-0 h-0.5 w-10 bg-primary" style:top="{scrollY + PADDING_TOP - 2}px">
      {#if timelineManager.scrolling && scrollHoverLabel && !isHover}
        <p
          transition:fade={{ duration: 200 }}
          class="truncate pointer-events-none absolute end-0 bottom-0 min-w-20 max-w-64 w-fit rounded-tl-md border-b-2 border-immich-primary bg-subtle/90 z-1 py-1 px-1 text-sm font-medium shadow-[0_0_8px_rgba(0,0,0,0.25)] dark:border-immich-dark-primary dark:text-immich-dark-fg"
        >
          {scrollHoverLabel}
        </p>
      {/if}
    </div>
  {/if}
  <div
    class="relative"
    style:height={relativeTopOffset + 'px'}
    data-id="lead-in"
    data-label={segments.at(0)?.dateFormatted}
  ></div>
  <!-- Time Segment -->
  {#each segments as segment (segment.year + '-' + segment.month)}
    <div
      class="relative"
      data-id="time-segment"
      data-segment-year-month={segment.year + '-' + segment.month}
      data-label={formatScrubberLabel(segment)}
      style:height={segment.height + 'px'}
    >
      {#if !usingMobileDevice}
        {#if segment.hasLabel}
          <div
            class="absolute end-5 text-[13px] dark:text-immich-dark-fg font-immich-mono bottom-0 hover:cursor-pointer hover:text-primary transition-colors"
            role="button"
            aria-label={scrubberLabelMode === 'month' ? `Jump to ${segment.month}月` : `Jump to ${segment.year}`}
            onclick={() => {
              const scrubData = {
                scrubberMonth: { year: segment.year, month: segment.month },
                overallScrollPercent: -1,
                scrubberMonthScrollPercent: 0,
              };
              onScrub?.(scrubData);
            }}
          >
            {formatScrubberLabel(segment)}
          </div>
        {/if}
        {#if segment.hasDot && scrubberLabelMode !== 'month'}
          <div class="absolute end-3 bottom-0 h-1 w-1 rounded-full bg-gray-300"></div>
        {/if}
      {/if}
    </div>
  {/each}
  <div
    data-id="lead-out"
    class="relative"
    style:height={relativeBottomOffset + 'px'}
    data-label={segments.at(-1)?.dateFormatted}
  ></div>
</div>
