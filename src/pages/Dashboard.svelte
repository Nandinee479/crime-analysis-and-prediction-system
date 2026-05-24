<script>
  import { currentPage } from '../stores/navigation.js'
  import { isAdmin } from '../stores/auth.js'
  import { onMount } from 'svelte'

  let stats = null
  let loading = true
  let loadError = ''
  let autoRefresh = false
  let refreshInterval

  onMount(async () => {
    try {
      await loadStats()
    } catch (err) {
      loadError = err?.message || 'Failed to load dashboard data.'
      console.error('Dashboard load failed:', err)
    }
    return () => { if (refreshInterval) clearInterval(refreshInterval) }
  })

  async function loadStats() {
    loading = true
    loadError = ''
    try {
      if (!window.api?.dashboard?.getStats) {
        throw new Error('Dashboard API is not available.')
      }
      stats = await window.api.dashboard.getStats()
    } catch (err) {
      loadError = err?.message || 'Failed to load dashboard data.'
      console.error('Dashboard load failed:', err)
      stats = null
    } finally {
      loading = false
    }
  }

  function toggleAutoRefresh() {
    autoRefresh = !autoRefresh
    if (autoRefresh) {
      refreshInterval = setInterval(loadStats, 30000)
    } else {
      clearInterval(refreshInterval)
      refreshInterval = null
    }
  }

  function trendColor(count, max) {
    if (max === 0) return '#E0E0E0'
    const ratio = count / max
    if (ratio <= 0.2) return '#C5CAE9'
    if (ratio <= 0.4) return '#9FA8DA'
    if (ratio <= 0.6) return '#7986CB'
    if (ratio <= 0.8) return '#5C6BC0'
    return '#1A237E'
  }

  function severityColor(s) {
    if (!s) return '#9C27B0'
    const l = s.toLowerCase()
    if (l === 'high')   return '#C62828'
    if (l === 'medium') return '#E65100'
    if (l === 'low')    return '#2E7D32'
    return '#9C27B0'
  }

  const statCards = [
    { key: 'totalCrimes',    label: 'Total Crimes',    icon: '🚔', bg: '#EDE7F6', color: '#4527A0', page: 'crime' },
    { key: 'totalSuspects',  label: 'Total Suspects',  icon: '🔍', bg: '#E3F2FD', color: '#01579B', page: 'suspect' },
    { key: 'totalVictims',   label: 'Total Victims',   icon: '👤', bg: '#FCE4EC', color: '#880E4F', page: 'victim' },
    { key: 'totalLocations', label: 'Crime Locations', icon: '📍', bg: '#E8F5E9', color: '#1B5E20', page: 'location' },
  ]

  function maxCount(arr) {
    return arr && arr.length ? Math.max(...arr.map(r => r.count), 1) : 1
  }

  const BAR_COLORS = ['#1A237E','#283593','#303F9F','#3949AB','#3F51B5']

  function pieColors(i) {
    const colors = ['#1A237E','#283593','#3F51B5','#5C6BC0','#7986CB','#9FA8DA']
    return colors[i % colors.length]
  }

  function getPieData() {
    if (!stats || !stats.crimesByType.length) return []
    const total = stats.crimesByType.reduce((s, r) => s + r.count, 0)
    let cumulative = 0
    return stats.crimesByType.map((r, i) => {
      const pct = (r.count / total) * 100
      const startAngle = (cumulative / total) * 360
      cumulative += r.count
      const endAngle = (cumulative / total) * 360
      return { ...r, pct, startAngle, endAngle, color: pieColors(i) }
    })
  }

  function polarToCartesian(angle, radius) {
    const rad = (angle - 90) * Math.PI / 180
    return { x: 100 + radius * Math.cos(rad), y: 100 + radius * Math.sin(rad) }
  }

  function describeArc(startAngle, endAngle, radius) {
    if (endAngle - startAngle < 0.1) return ''
    const start = polarToCartesian(startAngle, radius)
    const end = polarToCartesian(endAngle, radius)
    const largeArcFlag = endAngle - startAngle > 180 ? 1 : 0
    return `M 100,100 L ${start.x},${start.y} A ${radius},${radius} 0 ${largeArcFlag},1 ${end.x},${end.y} Z`
  }
</script>

<div class="page-header">
  <div>
    <h2>Dashboard</h2>
    <p>Overview of crime analysis data</p>
  </div>
  <div class="header-actions">
    <button class="btn-refresh" on:click={loadStats} title="Refresh now">
      ↻ Refresh
    </button>
    <button class="btn-autorefresh" class:active={autoRefresh} on:click={toggleAutoRefresh} title="Auto refresh every 30s">
      ⟳ Auto
    </button>
  </div>
</div>

<div class="content-area">
  {#if loading}
    <p style="color:#888; padding:40px 0; text-align:center;">Loading statistics…</p>
  {:else if loadError}
    <div class="error-box" style="margin:40px auto; max-width:520px; padding:20px; background:#FFEBEE; color:#C62828; border-radius:12px; text-align:center;">
      <p>{loadError}</p>
      <button class="btn-refresh" on:click={loadStats} style="margin-top:16px;">Retry</button>
    </div>
  {:else if stats}

    <!-- Stat cards -->
    <div class="stats-grid">
      {#each statCards as c}
        <button class="stat-card stat-card-interactive" on:click={() => currentPage.set(c.page)} type="button">
          <div class="stat-icon" style="background:{c.bg}; color:{c.color};">{c.icon}</div>
          <div>
            <div class="stat-value" style="color:{c.color};">{stats[c.key]}</div>
            <div class="stat-label">{c.label}</div>
          </div>
          <div class="stat-card-arrow">→</div>
        </button>
      {/each}
    </div>

    <!-- Prediction Card — only for users -->
    {#if !$isAdmin && stats.prediction}
      <div class="prediction-card">
        <div class="prediction-icon">🔮</div>
        <div class="prediction-body">
          <div class="prediction-label">Predicted Crimes for <strong>{stats.prediction.nextLabel}</strong></div>
          <div class="prediction-count">{stats.prediction.nextCount}</div>
          <div class="prediction-trend">
            {#if stats.prediction.trend === 'rising'}
              <span class="trend-up">📈 Rising</span>
            {:else if stats.prediction.trend === 'falling'}
              <span class="trend-down">📉 Falling</span>
            {:else}
              <span class="trend-stable">➡️ Stable</span>
            {/if}
          </div>
        </div>
      </div>
    {/if}

    <!-- Charts row -->
    <div class="charts-grid" style="margin-bottom:20px;">

      <!-- Pie Chart: Crimes by Type -->
      <div class="card" style="padding:20px 24px;">
        <h4 style="font-size:15px; color:var(--md-primary); margin-bottom:16px;">Crimes by Type (Pie)</h4>
        {#if stats.crimesByType.length === 0}
          <p style="color:#aaa; font-size:13px;">No data yet</p>
        {:else}
          <div class="pie-container">
            <svg viewBox="0 0 200 200" class="pie-svg">
              {#each getPieData() as slice}
                <path d={describeArc(slice.startAngle, slice.endAngle, 80)} fill={slice.color}
                  title={`${slice.Type_Name || 'Unknown'}: ${slice.count} (${slice.pct.toFixed(1)}%)`} />
              {/each}
            </svg>
            <div class="pie-legend">
              {#each getPieData() as slice}
                <div class="legend-item">
                  <span class="legend-dot" style="background:{slice.color}"></span>
                  <span class="legend-label">{slice.Type_Name || 'Unknown'}</span>
                  <span class="legend-pct">{slice.pct.toFixed(1)}%</span>
                </div>
              {/each}
            </div>
          </div>
        {/if}
      </div>

      <!-- Monthly Trend Bar Chart -->
      <div class="card" style="padding:20px 24px;">
        <h4 style="font-size:15px; color:var(--md-primary); margin-bottom:16px;">Monthly Crime Trend</h4>
        {#if !stats.monthlyTrend || stats.monthlyTrend.length === 0}
          <p style="color:#aaa; font-size:13px;">No data yet</p>
        {:else}
          <div class="trend-chart">
            {#each stats.monthlyTrend as row}
              <div class="trend-bar-wrap">
                <div class="trend-bar" style="height: {(row.count / maxCount(stats.monthlyTrend)) * 100}%; background:{trendColor(row.count, maxCount(stats.monthlyTrend))};">
                  <span class="trend-bar-label">{row.count}</span>
                </div>
                <div class="trend-month">{row.month}</div>
              </div>
            {/each}
          </div>
        {/if}
      </div>
    </div>

    <!-- Bar Charts: By Type & Severity -->
    <div class="charts-grid" style="margin-bottom:20px;">

      <!-- Crimes by type -->
      <div class="card" style="padding:20px 24px;">
        <h4 style="font-size:15px; color:var(--md-primary); margin-bottom:16px;">Crimes by Type</h4>
        {#if stats.crimesByType.length === 0}
          <p style="color:#aaa; font-size:13px;">No data yet</p>
        {:else}
          <div class="chart-bar-wrap">
            {#each stats.crimesByType as row, i}
              <div class="chart-row">
                <div class="chart-label">
                  <span>{row.Type_Name || 'Unknown'}</span>
                  <span style="font-weight:600;">{row.count}</span>
                </div>
                <div class="chart-track">
                  <div class="chart-fill"
                    style="width:{(row.count / maxCount(stats.crimesByType)) * 100}%; background:{BAR_COLORS[i % BAR_COLORS.length]};">
                  </div>
                </div>
              </div>
            {/each}
          </div>
        {/if}
      </div>

      <!-- Crimes by severity -->
      <div class="card" style="padding:20px 24px;">
        <h4 style="font-size:15px; color:var(--md-primary); margin-bottom:16px;">Crimes by Severity</h4>
        {#if stats.crimesBySeverity.length === 0}
          <p style="color:#aaa; font-size:13px;">No data yet</p>
        {:else}
          <div class="chart-bar-wrap">
            {#each stats.crimesBySeverity as row}
              <div class="chart-row">
                <div class="chart-label">
                  <span>{row.Severity}</span>
                  <span style="font-weight:600;">{row.count}</span>
                </div>
                <div class="chart-track">
                  <div class="chart-fill"
                    style="width:{(row.count / maxCount(stats.crimesBySeverity)) * 100}%; background:{severityColor(row.Severity)};">
                  </div>
                </div>
              </div>
            {/each}
          </div>
        {/if}
      </div>
    </div>

    <!-- Recent crimes -->
    <div class="card">
      <div style="padding:16px 24px; border-bottom:1px solid #ECEFF1;">
        <h4 style="font-size:15px; color:var(--md-primary);">Recent Crimes</h4>
      </div>
      {#if stats.recentCrimes.length === 0}
        <div class="empty-state">
          <div class="icon">🚔</div>
          <p>No crime records yet. Add crimes from the Crimes menu.</p>
        </div>
      {:else}
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>ID</th><th>Date</th><th>Type</th><th>Location</th><th>Severity</th>
              </tr>
            </thead>
            <tbody>
              {#each stats.recentCrimes as r}
                <tr>
                  <td>#{r.Crime_ID}</td>
                  <td>{r.Crime_Date || '—'}</td>
                  <td>{r.Type_Name || '—'}</td>
                  <td>{r.Location_Name || '—'}</td>
                  <td>
                    <span class="badge badge-{(r.Severity || '').toLowerCase()}">
                      {r.Severity || '—'}
                    </span>
                  </td>
                </tr>
              {/each}
            </tbody>
          </table>
        </div>
      {/if}
    </div>

  {/if}
</div>

<style>
  .header-actions { display: flex; gap: 8px; align-items: center; }
  .btn-refresh, .btn-autorefresh {
    padding: 8px 14px; border-radius: 20px; font-size: 13px; font-weight: 500;
    cursor: pointer; border: 1.5px solid #C4C7C5; background: #fff;
    color: #555; transition: all .2s; font-family: inherit;
  }
  .btn-refresh:hover { border-color: var(--md-primary); color: var(--md-primary); }
  .btn-autorefresh:hover { border-color: var(--md-primary); color: var(--md-primary); }
  .btn-autorefresh.active { background: var(--md-primary); color: #fff; border-color: var(--md-primary); }
  .pie-container { display: flex; gap: 20px; align-items: center; flex-wrap: wrap; }
  .pie-svg { width: 160px; height: 160px; flex-shrink: 0; }
  .pie-legend { flex: 1; min-width: 0; }
  .legend-item { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; font-size: 13px; }
  .legend-dot { width: 10px; height: 10px; border-radius: 50%; flex-shrink: 0; }
  .legend-label { flex: 1; min-width: 0; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  .legend-pct { font-weight: 600; color: #555; }
  .trend-chart {
    display: flex; align-items: flex-end; gap: 6px; height: 200px;
    padding: 20px 8px 0; border-bottom: 1px solid #E0E0E0;
  }
  .trend-bar-wrap {
    flex: 1; display: flex; flex-direction: column;
    align-items: center; height: 100%; justify-content: flex-end;
  }
  .trend-bar {
    width: 100%; max-width: 48px;
    border-radius: 6px 6px 2px 2px; position: relative;
    min-height: 6px; transition: height .4s ease;
    box-shadow: 0 1px 3px rgba(0,0,0,.15);
  }
  .trend-bar-label {
    position: absolute; top: -22px; left: 50%;
    transform: translateX(-50%); font-size: 12px;
    font-weight: 700; color: #333; white-space: nowrap;
  }
  .trend-month {
    font-size: 11px; color: #888; margin-top: 8px;
    text-align: center; font-weight: 500;
  }
  .prediction-card {
    display: flex; align-items: center; gap: 20px;
    background: linear-gradient(135deg, #EDE7F6, #E8EAF6);
    border: 2px solid #7986CB; border-radius: 14px;
    padding: 20px 24px; margin-bottom: 20px;
    animation: slideUp .3s ease;
  }
  .prediction-icon { font-size: 40px; }
  .prediction-body { flex: 1; }
  .prediction-label { font-size: 14px; color: #555; margin-bottom: 4px; }
  .prediction-count { font-size: 42px; font-weight: 800; color: #1A237E; line-height: 1; }
  .prediction-trend { margin-top: 6px; font-size: 14px; font-weight: 600; }
  .trend-up { color: #C62828; }
  .trend-down { color: #2E7D32; }
  .trend-stable { color: #E65100; }
</style>
