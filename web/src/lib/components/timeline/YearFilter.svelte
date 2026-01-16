<script lang="ts">
  interface Props {
    years: number[];
    selectedYear?: number;
    onSelectYear?: (year?: number) => void;
  }

  let { years, selectedYear = undefined, onSelectYear = () => {} }: Props = $props();

  const handleChange = (event: Event) => {
    const value = (event.target as HTMLSelectElement).value;
    if (value === '') {
      onSelectYear(undefined);
      return;
    }
    onSelectYear(Number.parseInt(value, 10));
  };
</script>

<div class="year-filter">
  <select class="year-filter-select" value={selectedYear ?? ''} onchange={handleChange} disabled={years.length === 0}>
    <option value=""></option>
    {#each years as year}
      <option value={year}>{year}</option>
    {/each}
  </select>
</div>

<style>
  .year-filter {
    display: flex;
    align-items: center;
    padding: 0.25rem 0.25rem;
  }

  .year-filter-select {
    height: 2.25rem;
    border-radius: 0.5rem;
    border: 1px solid rgba(0, 0, 0, 0.15);
    background: rgba(255, 255, 255, 0.85);
    padding: 0 0.5rem;
    min-width: 5.5rem;
  }

  :global(.dark) .year-filter-select {
    border: 1px solid rgba(255, 255, 255, 0.15);
    background: rgba(0, 0, 0, 0.35);
    color: rgb(var(--immich-dark-fg));
  }
</style>
