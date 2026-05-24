<script>
  import { onMount } from 'svelte'

  let crimes = []
  let loading = true
  let error = ''
  let search = ''
  let filterStatus = 'All'

  let selectedCrime = null
  let showLogModal = false
  let progressLog = []
  let logLoading = false

  const statuses = ['All', 'Ongoing', 'Under Investigation', 'Suspect Caught', 'Case Closed']

  const statusMeta = {
    'Ongoing':             { icon: '🔄', color: '#F57F17', bg: '#FFF8E1' },
    'Under Investigation': { icon: '🔍', color: '#1565C0', bg: '#E3F2FD' },
    'Suspect Caught':      { icon: '🚔', color: '#E65100', bg: '#FFF3E0' },
    'Case Closed':         { icon: '✅', color: '#2E7D32', bg: '#E8F5E9' },
  }

  const progressSteps = ['Ongoing', 'Under Investigation', 'Suspect Caught', 'Case Closed']

  function stepIndex(p) {
    const idx = progressSteps.indexOf(p || 'Ongoing')
    return idx >= 0 ? idx : 0
  }

  function stepColor(i, currentIdx) {
    if (i > currentIdx) return '#E0E0E0'
    const s = progressSteps[i]
    return statusMeta[s]?.color || '#90A4AE'
  }

  onMount(fetchCrimes)

  async function fetchCrimes() {
    loading = true
    error = ''
    try {
      crimes = await window.api.crimeProgress.getAll()
    } catch (e) {
      error = 'Failed to load data: ' + e.message
    } finally {
      loading = false
    }
  }

  $: filtered = crimes.filter(c => {
    const matchSearch =
      !search ||
      (c.Type_Name || '').toLowerCase().includes(search.toLowerCase()) ||
      (c.Location_Name || '').toLowerCase().includes(search.toLowerCase()) ||
      (c.City || '').toLowerCase().includes(search.toLowerCase()) ||
      String(c.Crime_ID).includes(search)
    const matchStatus = filterStatus === 'All' || (c.Progress || 'Ongoing') === filterStatus
    return matchSearch && matchStatus
  })

  $: summary = statuses.slice(1).map(s => ({
    label: s,
    count: crimes.filter(c => (c.Progress || 'Ongoing') === s).length,
    ...statusMeta[s],
  }))

  async function openLogModal(crime) {
    selectedCrime = crime
    showLogModal = true
    logLoading = true
    progressLog = []
    try {
      progressLog = await window.api.crimeProgress.getLog(crime.Crime_ID)
    } catch (e) {
      progressLog = []
    } finally {
      logLoading = false
    }
  }

  function closeLogModal() {
    showLogModal = false
    selectedCrime = null
    progressLog = []
  }
  
  function overlayKey(e) { if (e.key === 'Enter' || e.key === ' ') closeLogModal() }
  function onWindowKey(e) { if (e.key === 'Escape') closeLogModal() }
</script>

<svelte:window on:keydown={onWindowKey} />

<div class="page-header">
  <div>
    <h2>📊 Crime Progress Overview</h2>
    <p>View the investigation status of reported crimes</p>
  </div>
  <button class="btn btn-primary" on:click={fetchCrimes} disabled={loading}>
    🔄 Refresh
  </button>
</div>

<div class="content-area">
  {#if error}
    <div class="toast toast-error" style="margin-bottom:16px;">{error}</div>
  {/if}

  {#if !loading}
    <!-- Summary Cards -->
    <div class="progress-summary">
      {#each summary as s}
        <button
          class="summary-card"
          class:summary-active={filterStatus === s.label}
          style="border-color:{s.color}; {filterStatus === s.label ? `background:${s.bg};` : ''}"
          on:click={() => filterStatus = filterStatus === s.label ? 'All' : s.label}
        >
          <div class="summary-icon">{s.icon}</div>
          <div class="summary-count" style="color:{s.color};">{s.count}</div>
          <div class="summary-label">{s.label}</div>
        </button>
      {/each}
    </div>
  {/if}

  <!-- Search & Filter -->
  <div class="progress-toolbar">
    <div class="search-wrap" style="max-width:400px;">
      <span class="search-icon">🔍</span>
      <input
        type="text"
        bind:value={search}
        placeholder="Search type, location, ID..."
      />
    </div>
    <select class="form-control" style="width:auto; min-width:160px;" bind:value={filterStatus}>
      {#each statuses as s}
        <option value={s}>{s === 'All' ? '📂 All Statuses' : (statusMeta[s]?.icon || '') + ' ' + s}</option>
      {/each}
    </select>
  </div>

  <!-- Loading -->
  {#if loading}
    <div class="loading-state">
      <div class="spinner"></div>
      <p>Loading crime data…</p>
    </div>

  {:else if filtered.length === 0}
    <div class="empty-state">
      <div class="icon">🔎</div>
      <p>{crimes.length === 0 ? 'No crime records found.' : 'No crimes match your search.'}</p>
    </div>

  {:else}
    <!-- Crime Cards -->
    <div class="crime-grid">
      {#each filtered as crime (crime.Crime_ID)}
        <div class="crime-card">
          <!-- Header: ID + Status -->
          <div class="crime-card-header">
            <span class="crime-id">Crime #{crime.Crime_ID}</span>
            <span class="status-badge" style="background:{statusMeta[crime.Progress]?.bg || '#F5F5F5'}; color:{statusMeta[crime.Progress]?.color || '#666'};">
              {statusMeta[crime.Progress]?.icon || '❓'} {crime.Progress || 'Ongoing'}
            </span>
          </div>

          <!-- Title & Severity -->
          <div class="crime-card-body">
            <h3 class="crime-title">{crime.Type_Name || 'Unknown Crime'}</h3>
            <span class="badge badge-{(crime.Severity || 'default').toLowerCase()}">
              {crime.Severity || '—'} Severity
            </span>
          </div>

          <!-- Details -->
          <div class="crime-details">
            <div class="detail-row">
              <span>📍</span>
              <span>{crime.Location_Name || '—'}{crime.City ? `, ${crime.City}` : ''}</span>
            </div>
            <div class="detail-row">
              <span>📅</span>
              <span>{crime.Crime_Date || '—'}</span>
            </div>
            <div class="detail-row">
              <span>👤</span>
              <span class="detail-suspects">{crime.Suspects || 'No suspects linked'}</span>
            </div>
          </div>

          <!-- Progress Bar -->
          <div class="progress-bar-section">
            <div class="progress-track">
              {#each progressSteps as step, i}
                {@const current = stepIndex(crime.Progress)}
                <div
                  class="progress-step"
                  class:progress-done={i <= current}
                  style="background:{stepColor(i, current)};"
                  title={step}
                ></div>
              {/each}
            </div>
            <div class="progress-labels">
              <span>Open</span>
              <span>Closed</span>
            </div>
          </div>

          <!-- View History -->
          <button class="btn btn-text crime-history-btn" on:click={() => openLogModal(crime)}>
            📋 View Update History
          </button>
        </div>
      {/each}
    </div>
  {/if}
</div>

<!-- Progress Log Modal -->
{#if showLogModal && selectedCrime}
  <div class="modal-overlay" role="button" tabindex="0" on:click|self={closeLogModal} on:keydown={overlayKey}>
    <div class="modal" role="dialog" aria-modal="true" tabindex="-1" style="max-width:540px;">
      <div class="modal-header">
        <div>
          <h3>📋 Update History</h3>
          <p style="font-size:13px; color:var(--md-outline); margin-top:4px;">
            Crime #{selectedCrime.Crime_ID} — {selectedCrime.Type_Name}
          </p>
        </div>
        <button class="modal-close" on:click={closeLogModal}>✕</button>
      </div>

      <div class="modal-body">
        {#if logLoading}
          <div class="loading-state" style="padding:32px 0;">
            <div class="spinner"></div>
          </div>
        {:else if progressLog.length === 0}
          <div class="empty-state" style="padding:32px 0;">
            <div class="icon" style="font-size:36px;">📭</div>
            <p>No updates recorded yet for this case.</p>
          </div>
        {:else}
          <div class="timeline">
            <div class="timeline-line"></div>
            {#each progressLog as log (log.Log_ID)}
              <div class="timeline-item">
                <div class="timeline-dot" style="background:{statusMeta[log.Status]?.bg || '#F5F5F5'}; color:{statusMeta[log.Status]?.color || '#666'};">
                  {statusMeta[log.Status]?.icon || '❓'}
                </div>
                <div class="timeline-content">
                  <div class="timeline-header">
                    <span class="timeline-status" style="color:{statusMeta[log.Status]?.color || '#666'};">
                      {log.Status}
                    </span>
                    <span class="timeline-date">{log.Updated_At}</span>
                  </div>
                  {#if log.Note}
                    <p class="timeline-note">{log.Note}</p>
                  {/if}
                  <p class="timeline-author">by {log.Updated_By || 'System'}</p>
                </div>
              </div>
            {/each}
          </div>
        {/if}
      </div>

      <div class="modal-footer">
        <button class="btn btn-secondary" on:click={closeLogModal}>Close</button>
      </div>
    </div>
  </div>
{/if}

<style>
  .progress-summary {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin-bottom: 20px;
  }
  .summary-card {
    background: var(--md-surface);
    border: 2px solid #E0E0E0;
    border-radius: var(--radius);
    padding: 18px 16px;
    text-align: center;
    cursor: pointer;
    transition: all var(--transition);
    font-family: inherit;
  }
  .summary-card:hover {
    box-shadow: var(--md-elevation2);
    transform: translateY(-2px);
  }
  .summary-active {
    box-shadow: var(--md-elevation1);
  }
  .summary-icon { font-size: 28px; margin-bottom: 6px; }
  .summary-count { font-size: 28px; font-weight: 800; line-height: 1.2; }
  .summary-label { font-size: 12px; color: var(--md-outline); margin-top: 4px; font-weight: 500; }

  .progress-toolbar {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 20px;
  }

  .loading-state {
    display: flex; flex-direction: column; align-items: center; justify-content: center;
    padding: 60px 0; color: var(--md-outline); gap: 12px;
  }
  .spinner {
    width: 28px; height: 28px;
    border: 3px solid #E0E0E0; border-top-color: var(--md-primary);
    border-radius: 50%; animation: spin .6s linear infinite;
  }
  @keyframes spin { to { transform: rotate(360deg) } }

  .crime-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: 16px;
  }
  .crime-card {
    background: var(--md-surface);
    border-radius: var(--radius);
    box-shadow: var(--md-elevation1);
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 14px;
    transition: box-shadow var(--transition), transform var(--transition);
  }
  .crime-card:hover {
    box-shadow: var(--md-elevation2);
    transform: translateY(-2px);
  }

  .crime-card-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }
  .crime-id {
    font-size: 12px;
    font-weight: 600;
    color: var(--md-outline);
    font-family: monospace;
  }
  .status-badge {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 12px;
    font-weight: 600;
    white-space: nowrap;
  }

  .crime-card-body {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }
  .crime-title {
    font-size: 16px;
    font-weight: 600;
    color: var(--md-primary);
    margin: 0;
  }

  .crime-details {
    display: flex;
    flex-direction: column;
    gap: 6px;
    font-size: 13px;
    color: #555;
  }
  .detail-row {
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .detail-suspects {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .progress-bar-section { margin-top: 4px; }
  .progress-track {
    display: flex;
    gap: 4px;
    height: 6px;
  }
  .progress-step {
    flex: 1;
    border-radius: 3px;
    transition: background .3s;
  }
  .progress-labels {
    display: flex;
    justify-content: space-between;
    font-size: 10px;
    color: #bbb;
    margin-top: 2px;
  }

  .crime-history-btn {
    width: 100%;
    justify-content: center;
    border: 1.5px solid #E0E0E0;
    border-radius: var(--radius-sm);
    padding: 10px;
    margin-top: auto;
  }
  .crime-history-btn:hover {
    background: #F5F5F5;
    border-color: #ccc;
  }

  /* Timeline */
  .timeline {
    position: relative;
    padding-left: 40px;
  }
  .timeline-line {
    position: absolute;
    left: 20px;
    top: 0;
    bottom: 0;
    width: 2px;
    background: #E0E0E0;
  }
  .timeline-item {
    position: relative;
    padding-bottom: 20px;
  }
  .timeline-item:last-child { padding-bottom: 0; }
  .timeline-dot {
    position: absolute;
    left: -28px;
    width: 36px;
    height: 36px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 16px;
    border: 2px solid #E0E0E0;
    z-index: 1;
  }
  .timeline-content {
    background: #FAFAFA;
    border: 1px solid #ECEFF1;
    border-radius: var(--radius-sm);
    padding: 12px 16px;
  }
  .timeline-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    flex-wrap: wrap;
  }
  .timeline-status {
    font-weight: 700;
    font-size: 14px;
  }
  .timeline-date {
    font-size: 12px;
    color: var(--md-outline);
  }
  .timeline-note {
    font-size: 13px;
    color: #444;
    margin-top: 6px;
    line-height: 1.5;
  }
  .timeline-author {
    font-size: 11px;
    color: var(--md-outline);
    margin-top: 6px;
  }
</style>