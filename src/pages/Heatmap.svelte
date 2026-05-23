<script>
  import { onMount } from 'svelte'

  let locations = []
  let loading = true
  let maxCount = 1

  onMount(load)

  async function load() {
    try {
      const data = await window.api.heatmap.getData()
      locations = data || []
      maxCount = Math.max(...locations.map(l => l.crime_count), 1)
    } catch (err) {
      console.error('Heatmap load failed:', err)
      locations = []
    } finally {
      loading = false
    }
  }
             // Heat color based on crime count, scaled to maxCount
  function heatColor(count) {
    const ratio = maxCount > 0 ? count / maxCount : 0
    if (ratio === 0) return '#F5F5F5'
    const r = Math.min(255, Math.round(ratio * 255))
    const g = Math.min(255, Math.round((1 - ratio) * 200))
    const b = Math.min(255, Math.round((1 - ratio) * 100 + 55))
    return `rgb(${r},${g},${b})`
  }
              // For better text contrast on darker tiles
  function heatTextColor(count) {
    const ratio = maxCount > 0 ? count / maxCount : 0
    return ratio > 0.5 ? '#fff' : '#333'
  }

  function getIntensityLabel(count) {
    const ratio = maxCount > 0 ? count / maxCount : 0
    if (ratio === 0) return 'No Crime'
    if (ratio < 0.25) return 'Low'
    if (ratio < 0.5) return 'Moderate'
    if (ratio < 0.75) return 'High'
    return 'Very High'
  }
</script>

<div class="page-header">
  <div>
    <h2>🗺️ Crime Heatmap</h2>
    <p>Crime density by location — darker red = more crimes</p>
  </div>
</div>

<div class="content-area">
  {#if loading}
    <p style="color:#888; padding:40px 0; text-align:center;">Loading heatmap data…</p>
  {:else if locations.length === 0}
    <div class="empty-state">
      <div class="icon">🗺️</div>
      <p>No locations found.</p>
    </div>
  {:else}
    <!-- Legend -->
    <div class="heatmap-legend">
      <span>Low</span>
      <div class="legend-bar">
        {#each Array(10) as _, i}
          <div class="legend-step" style="background:{heatColor(Math.round((i + 1) * maxCount / 10))}"></div>
        {/each}
      </div>
      <span>High</span>
    </div>

    <!-- Heatmap Grid -->
    <div class="heatmap-grid">
      {#each locations as loc}
        <div
          class="heatmap-tile"
          style="background:{heatColor(loc.crime_count)}; color:{heatTextColor(loc.crime_count)};"
          title="{loc.Location_Name}: {loc.crime_count} crimes"
        >
          <div class="tile-count">{loc.crime_count}</div>
          <div class="tile-name">{loc.Location_Name}</div>
          <div class="tile-city">{loc.City || '—'}</div>
          {#if loc.crime_count > 0}
            <div class="tile-badge" style="background:{heatColor(loc.crime_count)}; border-color:{heatTextColor(loc.crime_count)};">
              {getIntensityLabel(loc.crime_count)}
            </div>
          {/if}
        </div>
      {/each}
    </div>
  {/if}
</div>

<style>
  .heatmap-legend {
    display: flex; align-items: center; gap: 12px;
    margin-bottom: 24px; padding: 14px 18px;
    background: #fff; border-radius: 12px; box-shadow: 0 1px 3px rgba(0,0,0,.08);
    font-size: 13px; font-weight: 500; color: #666;
  }
  .legend-bar { display: flex; flex: 1; height: 20px; border-radius: 6px; overflow: hidden; }
  .legend-step { flex: 1; }
  .heatmap-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 14px;
  }
  .heatmap-tile {
    padding: 20px; border-radius: 14px; text-align: center;
    transition: transform .2s, box-shadow .2s;
    cursor: default; position: relative; overflow: hidden;
    box-shadow: 0 1px 4px rgba(0,0,0,.08);
  }
  .heatmap-tile:hover {
    transform: translateY(-4px); box-shadow: 0 6px 20px rgba(0,0,0,.15);
  }
  .tile-count { font-size: 32px; font-weight: 800; line-height: 1; margin-bottom: 6px; }
  .tile-name { font-size: 15px; font-weight: 600; margin-bottom: 2px; }
  .tile-city { font-size: 12px; opacity: .7; margin-bottom: 8px; }
  .tile-badge {
    display: inline-block; padding: 3px 12px; border-radius: 20px;
    font-size: 11px; font-weight: 600; border: 2px solid;
    background: rgba(255,255,255,.3); backdrop-filter: blur(2px);
  }
</style>
