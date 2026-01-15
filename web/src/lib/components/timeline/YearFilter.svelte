<script lang="ts">
  import { TimelineManager } from '$lib/managers/timeline-manager/timeline-manager.svelte';
  import { mobileDevice } from '$lib/stores/mobile-device.svelte';
  import { DateTime } from 'luxon';

  interface Props {
    timelineManager: TimelineManager;
  }

  let { timelineManager }: Props = $props();

  // Get unique years from timeline scrubber months
  const availableYears = $derived(() => {
    const years = new Set(timelineManager.scrubberMonths.map((m) => m.year));
    return Array.from(years).sort((a, b) => b - a); // Sort descending (newest first)
  });

  const isMobile = $derived(mobileDevice.maxMd);

  // Find the first month group for a given year and scroll to it
  const jumpToYear = async (year: number) => {
    // Find the first month of the selected year
    const monthGroup = timelineManager.months.find(
      ({ yearMonth: { year: y, month: m } }) => y === year && m === 1
    );

    if (monthGroup) {
      // Scroll to that month
      timelineManager.scrollTo(monthGroup.top);
    } else {
      // If we don't have data for that year yet, try to find the closest asset
      try {
        const dateTime = DateTime.fromObject({ year, month: 1, day: 1 });
        const asset = await timelineManager.getClosestAssetToDate(dateTime.toObject());
        if (asset) {
          const assetMonthGroup = timelineManager.getMonthGroupByAssetId(asset.id);
          if (assetMonthGroup) {
            timelineManager.scrollTo(assetMonthGroup.top);
          }
        }
      } catch (error) {
        console.error('Failed to jump to year:', error);
      }
    }
  };
</script>

<div class="year-filter-bar">
  <div class="year-filter-container">
    <div class="year-filter-label">快速跳转:</div>

    <div class="year-filter-buttons">
      {#each availableYears as year}
        <button
          type="button"
          class="year-filter-button"
          onclick={() => jumpToYear(year)}
        >
          {year}
        </button>
      {/each}
    </div>
  </div>
</div>

<style>
  .year-filter-bar {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    z-index: 50;
    background-color: rgba(0, 0, 0, 0.9);
    backdrop-filter: blur(10px);
    border-top: 1px solid rgba(255, 255, 255, 0.1);
    padding: 0.75rem 1rem;
    box-shadow: 0 -4px 6px rgba(0, 0, 0, 0.3);
    transition: transform 0.3s ease;
  }

  .year-filter-container {
    max-width: 100%;
    margin: 0 auto;
    display: flex;
    align-items: center;
    gap: 1rem;
    overflow-x: auto;
  }

  .year-filter-label {
    color: white;
    font-size: 0.875rem;
    font-weight: 600;
    white-space: nowrap;
    flex-shrink: 0;
  }

  .year-filter-buttons {
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
    align-items: center;
  }

  .year-filter-button {
    padding: 0.5rem 1rem;
    font-size: 0.875rem;
    font-weight: 500;
    border-radius: 0.5rem;
    border: 1px solid rgba(255, 255, 255, 0.2);
    background-color: rgba(255, 255, 255, 0.1);
    color: rgba(255, 255, 255, 0.8);
    cursor: pointer;
    transition: all 0.2s ease;
    white-space: nowrap;
    flex-shrink: 0;
  }

  .year-filter-button:hover {
    background-color: rgba(255, 255, 255, 0.2);
    color: white;
    border-color: rgba(255, 255, 255, 0.3);
    transform: translateY(-1px);
  }

  .year-filter-button:active {
    transform: translateY(0);
  }

  /* Scrollbar styling for year filter container */
  .year-filter-container::-webkit-scrollbar {
    height: 4px;
  }

  .year-filter-container::-webkit-scrollbar-track {
    background: rgba(255, 255, 255, 0.1);
    border-radius: 2px;
  }

  .year-filter-container::-webkit-scrollbar-thumb {
    background: rgba(255, 255, 255, 0.3);
    border-radius: 2px;
  }

  .year-filter-container::-webkit-scrollbar-thumb:hover {
    background: rgba(255, 255, 255, 0.5);
  }

  /* Mobile responsiveness */
  @media (max-width: 768px) {
    .year-filter-bar {
      padding: 0.5rem 0.75rem;
    }

    .year-filter-button {
      padding: 0.375rem 0.75rem;
      font-size: 0.75rem;
    }
  }
</style>
