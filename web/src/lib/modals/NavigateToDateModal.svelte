<script lang="ts">
  import DateInput from '$lib/elements/DateInput.svelte';
  import { TimelineManager } from '$lib/managers/timeline-manager/timeline-manager.svelte';
  import type { TimelineAsset } from '$lib/managers/timeline-manager/types';
  import { getPreferredTimeZone, getTimezones, toDatetime, type ZoneOption } from '$lib/modals/timezone-utils';
  import { FormModal, HStack, VStack } from '@immich/ui';
  import { mdiNavigationVariantOutline } from '@mdi/js';
  import { DateTime } from 'luxon';
  import { t } from 'svelte-i18n';

  interface Props {
    timelineManager: TimelineManager;
    onClose: (asset?: TimelineAsset) => void;
  }

  let { timelineManager, onClose }: Props = $props();

  const initialDate = DateTime.now();
  let selectedDate = $state(initialDate.toFormat("yyyy-MM-dd'T'HH:mm:ss.SSS"));
  const timezones = $derived(getTimezones(selectedDate));
  // the offsets (and validity) for time zones may change if the date is changed, which is why we recompute the list
  let selectedOption: ZoneOption | undefined = $derived(getPreferredTimeZone(initialDate, undefined, timezones));

  // Get unique years from timeline scrubber months
  const availableYears = $derived(() => {
    const years = new Set(timelineManager.scrubberMonths.map((m) => m.year));
    return Array.from(years).sort((a, b) => b - a); // Sort descending
  });

  const jumpToYear = async (year: number) => {
    const dateTime = DateTime.fromObject({ year, month: 1, day: 1 }, { zone: selectedOption?.value });
    const asset = await timelineManager.getClosestAssetToDate(dateTime.toObject());
    onClose(asset);
  };

  const onSubmit = async () => {
    if (!date.isValid || !selectedOption) {
      onClose();
      return;
    }

    // Get the local date/time components from the selected string using neutral timezone
    const dateTime = toDatetime(selectedDate, selectedOption) as DateTime<true>;
    const asset = await timelineManager.getClosestAssetToDate(dateTime.toObject());
    onClose(asset);
  };

  // when changing the time zone, assume the configured date/time is meant for that time zone (instead of updating it)
  const date = $derived(DateTime.fromISO(selectedDate, { zone: selectedOption?.value, setZone: true }));
</script>

<FormModal
  title={$t('navigate_to_time')}
  icon={mdiNavigationVariantOutline}
  onClose={() => onClose()}
  {onSubmit}
  submitText={$t('confirm')}
  disabled={!date.isValid || !selectedOption}
  size="medium"
>
  <VStack fullWidth>
    <!-- Quick Year Selection -->
    {#if availableYears.length > 0}
      <HStack fullWidth>
        <label class="immich-form-label">快速选择年份</label>
      </HStack>
      <HStack fullWidth class="gap-2 flex-wrap">
        {#each availableYears as year}
          <button
            type="button"
            class="px-3 py-1.5 text-sm rounded-md bg-immich-gray/10 hover:bg-immich-primary hover:text-white dark:bg-immich-dark-gray/20 dark:hover:bg-immich-dark-primary transition-all"
            onclick={() => jumpToYear(year)}
          >
            {year}
          </button>
        {/each}
      </HStack>
      <div class="w-full h-px bg-gray-200 dark:bg-gray-700 my-2"></div>
    {/if}

    <!-- Precise Date/Time Selection -->
    <HStack fullWidth>
      <label class="immich-form-label" for="datetime">{$t('date_and_time')}</label>
    </HStack>
    <HStack fullWidth>
      <DateInput
        class="immich-form-input text-gray-700 w-full"
        id="datetime"
        type="datetime-local"
        bind:value={selectedDate}
      />
    </HStack>
  </VStack>
</FormModal>
