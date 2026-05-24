<script>
  import { onMount } from 'svelte'

  export let currentUserId = null

  let crimes = []
  let loading = true
  let error = ''
  let search = ''
  let filterStatus = 'All'

  let selectedCrime = null
  let showModal = false
  let newStatus = ''
  let note = ''
  let saving = false
  let saveMsg = ''

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

  onMount(fetchCrimes)

  async function fetchCrimes() {
    loading = true
    error = ''
    try {
      crimes = await window.api.crimeProgress.getAll()
    } catch (e) {
      error = 'Failed to load crimes: ' + e.message
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

  function openUpdateModal(crime) {
    selectedCrime = crime
    newStatus = crime.Progress || 'Ongoing'
    note = ''
    saveMsg = ''
    showModal = true
  }

  function closeModal() {
    showModal = false
    selectedCrime = null
  }

  async function saveProgress() {
    if (!newStatus) return
    saving = true
    saveMsg = ''
    try {
      const result = await window.api.crimeProgress.update({
        crimeId: selectedCrime.Crime_ID,
        status: newStatus,
        note: note.trim(),
        updatedBy: currentUserId
      })
      if (result.success) {
        saveMsg = '✅ Progress updated successfully!'
        await fetchCrimes()
        setTimeout(() => { closeModal() }, 1000)
      } else {
        saveMsg = '❌ Error: ' + (result.error || 'Unknown error')
      }
    } catch (e) {
      saveMsg = '❌ Failed: ' + e.message
    } finally {
      saving = false
    }
  }

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
</script>

<div class="page-header">
  <div>
    <h2>🛡️ Crime Progress Management</h2>
    <p>Update and track the investigation status of each crime</p>
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
    <div class="search-wrap" style="max-width:360px;">
      <span class="search-icon">🔍</span>
      <input
        type="text"
        bind:value={search}
        placeholder="Search type, location, ID..."
      />
    </div>
    <select class="form-control" style="width:auto; min-width:150px;" bind:value={filterStatus}>
      {#each statuses as s}
        <option value={s}>{s === 'All' ? '📂 All Statuses' : (statusMeta[s]?.icon || '') + ' ' + s}</option>
      {/each}
    </select>
    <span class="toolbar-count">{crimes.length} record{crimes.length !== 1 ? 's' : ''}</span>
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
    <!-- Table -->
    <div class="card">
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>ID</th>
              <th>Crime Type</th>
              <th>Severity</th>
              <th>Date</th>
              <th>Location</th>
              <th>Suspects</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {#each filtered as crime (crime.Crime_ID)}
              <tr>
                <td class="cell-mono">#{crime.Crime_ID}</td>
                <td class="cell-type">{crime.Type_Name || '—'}</td>
                <td>
                  <span class="badge badge-{(crime.Severity || 'default').toLowerCase()}">
                    {crime.Severity || '—'}
                  </span>
                </td>
                <td class="cell-date">{crime.Crime_Date || '—'}</td>
                <td>{crime.Location_Name || '—'}{crime.City ? `, ${crime.City}` : ''}</td>
                <td class="cell-suspects">{crime.Suspects || '—'}</td>
                <td>
                  <span class="status-pill" style="background:{statusMeta[crime.Progress]?.bg || '#F5F5F5'}; color:{statusMeta[crime.Progress]?.color || '#666'};">
                    {statusMeta[crime.Progress]?.icon || '❓'} {crime.Progress || 'Ongoing'}
                  </span>
                </td>
                <td>
                  <div class="row-actions">
                    <button class="btn btn-primary" style="padding:6px 14px; font-size:12px;" on:click={() => openUpdateModal(crime)}>
                      ✏️ Update
                    </button>
                    <button class="btn btn-secondary" style="padding:6px 14px; font-size:12px;" on:click={() => openLogModal(crime)}>
                      📋 Log
                    </button>
                  </div>
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </div>
  {/if}
</div>

<!-- ====== Update Progress Modal ====== -->
{#if showModal && selectedCrime}
  <div
    class="modal-overlay"
    role="button"
    aria-label="Close modal"
    tabindex="0"
    on:click|self={closeModal}
    on:keydown={(e) => (e.key === 'Enter' || e.key === ' ') && closeModal()}
  >
    <div class="modal" role="dialog" aria-modal="true">
      <div class="modal-header">
        <div>
          <h3>✏️ Update Crime Progress</h3>
          <p style="font-size:13px; color:var(--md-outline); margin-top:4px;">
            Crime #{selectedCrime.Crime_ID} — {selectedCrime.Type_Name}
          </p>
        </div>
        <button class="modal-close" on:click={closeModal}>✕</button>
      </div>

      <div class="modal-body">
        <div class="current-status-banner" style="background:{statusMeta[selectedCrime.Progress]?.bg || '#F5F5F5'}; color:{statusMeta[selectedCrime.Progress]?.color || '#666'};">
          <span>Current Status:</span>
          <strong>{statusMeta[selectedCrime.Progress]?.icon || '❓'} {selectedCrime.Progress || 'Ongoing'}</strong>
        </div>

        <fieldset class="form-group">
          <legend class="form-label">New Status</legend>
          <div class="status-grid">
            {#each progressSteps as s}
              <button
                type="button"
                class="status-option"
                class:status-option-active={newStatus === s}
                style={newStatus === s ? `border-color:${statusMeta[s]?.color}; background:${statusMeta[s]?.bg};` : ''}
                on:click={() => newStatus = s}
              >
                <span class="status-option-icon">{statusMeta[s]?.icon}</span>
                <span>{s}</span>
              </button>
            {/each}
          </div>
        </fieldset>

        <div class="form-group">
          <label class="form-label" for="progress-note">Note <span style="color:var(--md-outline); font-weight:400;">(optional)</span></label>
          <textarea
            id="progress-note"
            bind:value={note}
            rows="4"
            placeholder="Add a note about this update..."
            class="form-control"
          ></textarea>
        </div>

        {#if saveMsg}
          <div class="save-msg" class:save-success={saveMsg.startsWith('✅')} class:save-error={saveMsg.startsWith('❌')}>
            {saveMsg}
          </div>
        {/if}
      </div>

      <div class="modal-footer">
        <button class="btn btn-secondary" on:click={closeModal}>Cancel</button>
        <button class="btn btn-primary" on:click={saveProgress} disabled={saving || !newStatus}>
          {saving ? 'Saving...' : '💾 Save Progress'}
        </button>
      </div>
    </div>
  </div>
{/if}

<!-- ====== Progress Log Modal ====== -->
{#if showLogModal && selectedCrime}
  <div
    class="modal-overlay"
    role="button"
    aria-label="Close modal"
    tabindex="0"
    on:click|self={closeLogModal}
    on:keydown={(e) => (e.key === 'Enter' || e.key === ' ') && closeLogModal()}
  >
    <div class="modal" role="dialog" aria-modal="true" style="max-width:540px;">
      <div class="modal-header">
        <div>
          <h3>📋 Progress History</h3>
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
  .toolbar-count {
    font-size: 13px;
    color: var(--md-outline);
    margin-left: auto;
    white-space: nowrap;
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

  .cell-mono { font-family: monospace; font-size: 13px; color: var(--md-outline); font-weight: 600; }
  .cell-type { font-weight: 600; color: var(--md-primary); }
  .cell-date { white-space: nowrap; }
  .cell-suspects { max-width: 180px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

  .status-pill {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 12px;
    font-weight: 600;
    white-space: nowrap;
  }
  .row-actions {
    display: flex;
    gap: 6px;
  }

  /* Current status banner */
  .current-status-banner {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 16px;
    border-radius: var(--radius-sm);
    font-size: 14px;
    margin-bottom: 20px;
  }

  /* Status grid */
  .status-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 10px;
  }
  .status-option {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 14px;
    border: 2px solid #E0E0E0;
    border-radius: var(--radius-sm);
    background: var(--md-surface);
    cursor: pointer;
    font-size: 14px;
    font-weight: 500;
    color: #555;
    transition: all var(--transition);
    font-family: inherit;
    text-align: left;
  }
  .status-option:hover {
    border-color: var(--md-primary-lt);
    background: #F5F5F5;
  }
  .status-option-active {
    border-color: var(--md-primary);
    font-weight: 600;
  }
  .status-option-icon { font-size: 18px; }

  .save-msg {
    padding: 10px 14px;
    border-radius: var(--radius-sm);
    font-size: 14px;
    font-weight: 500;
    margin-top: 10px;
  }
  .save-success {
    background: #E8F5E9;
    color: #2E7D32;
  }
  .save-error {
    background: #FFEBEE;
    color: #C62828;
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
  .timeline-status { font-weight: 700; font-size: 14px; }
  .timeline-date { font-size: 12px; color: var(--md-outline); }
  .timeline-note { font-size: 13px; color: #444; margin-top: 6px; line-height: 1.5; }
  .timeline-author { font-size: 11px; color: var(--md-outline); margin-top: 6px; }
</style>