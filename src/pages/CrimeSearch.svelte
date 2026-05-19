<script>
  import { onMount } from 'svelte'
  import { showToast } from '../stores/navigation.js'

  let crimes = []
  let filtered = []
  let search = ''
  let selectedCrime = null
  let loading = true

  onMount(async () => {
    try {
      crimes = await window.api.crime.getAll()
      applyFilter()
    } catch (error) {
      showToast('Failed to load crimes', 'error')
    } finally {
      loading = false
    }
  })

  function applyFilter() {
    const q = search.toLowerCase()
    filtered = q
      ? crimes.filter(crime =>
          crime.Type_Name?.toLowerCase().includes(q) ||
          crime.Location_Name?.toLowerCase().includes(q) ||
          crime.Severity?.toLowerCase().includes(q) ||
          crime.Crime_Date?.includes(q) ||
          String(crime.Crime_ID).includes(q))
      : crimes
  }

  $: search, applyFilter()

  function viewCrime(crime) {
    selectedCrime = crime
  }

  function closeModal() {
    selectedCrime = null
  }

  function getSeverityColor(severity) {
    if (!severity) return '#666'
    const s = severity.toLowerCase()
    if (s === 'high') return '#d32f2f'
    if (s === 'medium') return '#f57c00'
    if (s === 'low') return '#388e3c'
    return '#666'
  }

  function getSeverityBadge(severity) {
    if (!severity) return 'badge-default'
    const s = severity.toLowerCase()
    if (s === 'high') return 'badge-high'
    if (s === 'medium') return 'badge-medium'
    if (s === 'low') return 'badge-low'
    return 'badge-default'
  }
</script>

<div class="page-header">
  <div><h2>Crime Search</h2><p>Search and view crime records</p></div>
</div>

<div class="content-area">
  <div class="card">
    <div class="toolbar">
      <div class="search-wrap">
        <span class="search-icon">🔎</span>
        <input bind:value={search} placeholder="Search by crime type, location, severity, date, or ID…" />
      </div>
      <span style="font-size:13px; color:#888;">{filtered.length} record{filtered.length !== 1 ? 's' : ''}</span>
    </div>

    {#if loading}
      <div class="loading">Loading crimes...</div>
    {:else if filtered.length === 0}
      <div class="empty-state">
        <div class="icon">🔍</div>
        <p>{search ? 'No results found.' : 'No crime records available.'}</p>
      </div>
    {:else}
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>ID</th>
              <th>Date</th>
              <th>Type</th>
              <th>Location</th>
              <th>Severity</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {#each filtered as crime}
              <tr>
                <td><strong>#{crime.Crime_ID}</strong></td>
                <td>{crime.Crime_Date || '—'}</td>
                <td>{crime.Type_Name || '—'}</td>
                <td>{crime.Location_Name || '—'}</td>
                <td>
                  {#if crime.Severity}
                    <span class="badge {getSeverityBadge(crime.Severity)}">{crime.Severity}</span>
                  {:else}—{/if}
                </td>
                <td>
                  <button class="btn-view" on:click={() => viewCrime(crime)}>👁️ View</button>
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    {/if}
  </div>
</div>

<svelte:window on:keydown={(e) => selectedCrime && e.key === 'Escape' && closeModal()} />

<!-- Crime Details Modal -->
{#if selectedCrime}
  <div class="modal-overlay" role="dialog" aria-modal="true" aria-labelledby="modal-title">
    <button class="modal-backdrop" type="button" aria-label="Close dialog" on:click={closeModal}></button>
    <div class="modal-content">
      <div class="modal-header">
        <h3 id="modal-title">Crime Details - #{selectedCrime.Crime_ID}</h3>
        <button class="close-btn" on:click={closeModal} aria-label="Close modal">✕</button>
      </div>

      <div class="modal-body">
        <div class="detail-grid">
          <div class="detail-item">
            <strong>Crime Date:</strong>
            <span>{selectedCrime.Crime_Date || 'Not specified'}</span>
          </div>

          <div class="detail-item">
            <strong>Crime Type:</strong>
            <span>{selectedCrime.Type_Name || 'Unknown'}</span>
          </div>

          <div class="detail-item">
            <strong>Location:</strong>
            <span>{selectedCrime.Location_Name || 'Unknown'}</span>
          </div>

          <div class="detail-item">
            <strong>Severity:</strong>
            {#if selectedCrime.Severity}
              <span class="severity-badge {getSeverityBadge(selectedCrime.Severity)}">{selectedCrime.Severity}</span>
            {:else}
              <span>Not specified</span>
            {/if}
          </div>
        </div>

        <!-- Related Data -->
        <div class="related-section">
          <h4>Related Information</h4>
          <p class="info-note">For detailed suspect and victim information, please contact system administrator.</p>
        </div>
      </div>
    </div>
  </div>
{/if}

<style>
  .toolbar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
    padding-bottom: 15px;
    border-bottom: 1px solid #e0e0e0;
  }

  .search-wrap {
    position: relative;
    flex: 1;
    max-width: 400px;
  }

  .search-icon {
    position: absolute;
    left: 12px;
    top: 50%;
    transform: translateY(-50%);
    color: #666;
  }

  .search-wrap input {
    width: 100%;
    padding: 10px 10px 10px 40px;
    border: 2px solid #e0e0e0;
    border-radius: 8px;
    font-size: 14px;
    transition: border-color 0.2s;
  }

  .search-wrap input:focus {
    outline: none;
    border-color: #3949ab;
  }

  .btn-view {
    background: #3949ab;
    color: white;
    border: none;
    padding: 6px 12px;
    border-radius: 6px;
    cursor: pointer;
    font-size: 12px;
    transition: background 0.2s;
  }

  .btn-view:hover {
    background: #283593;
  }

  .loading {
    text-align: center;
    padding: 40px;
    color: #666;
  }

  .empty-state {
    text-align: center;
    padding: 60px 20px;
  }

  .empty-state .icon {
    font-size: 48px;
    margin-bottom: 16px;
    opacity: 0.5;
  }

  .empty-state p {
    color: #666;
    font-size: 16px;
  }

  .badge {
    padding: 4px 8px;
    border-radius: 4px;
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
  }

  .badge-high { background: #ffebee; color: #d32f2f; }
  .badge-medium { background: #fff3e0; color: #f57c00; }
  .badge-low { background: #e8f5e9; color: #388e3c; }
  .badge-default { background: #f5f5f5; color: #666; }

  /* Modal Styles */
  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0,0,0,0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }

  .modal-backdrop {
    position: absolute;
    inset: 0;
    border: none;
    background: transparent;
    padding: 0;
    margin: 0;
    cursor: default;
  }

  .modal-content {
    position: relative;
    background: white;
    border-radius: 12px;
    max-width: 600px;
    width: 90%;
    max-height: 80vh;
    overflow-y: auto;
    box-shadow: 0 10px 30px rgba(0,0,0,0.3);
  }

  .modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px 24px;
    border-bottom: 1px solid #e0e0e0;
  }

  .modal-header h3 {
    margin: 0;
    color: #1a237e;
  }

  .close-btn {
    background: none;
    border: none;
    font-size: 24px;
    cursor: pointer;
    color: #666;
    padding: 4px;
    border-radius: 4px;
    transition: background 0.2s;
  }

  .close-btn:hover {
    background: #f5f5f5;
  }

  .modal-body {
    padding: 24px;
  }

  .detail-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
    margin-bottom: 30px;
  }

  .detail-item {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .detail-item strong {
    font-weight: 600;
    color: #333;
    font-size: 14px;
  }

  .detail-item span {
    color: #666;
    font-size: 16px;
  }

  .severity-badge {
    display: inline-block;
    margin-top: 4px;
  }

  .related-section {
    border-top: 1px solid #e0e0e0;
    padding-top: 20px;
  }

  .related-section h4 {
    margin: 0 0 12px;
    color: #1a237e;
  }

  .info-note {
    color: #666;
    font-style: italic;
    background: #f9f9f9;
    padding: 12px;
    border-radius: 6px;
    border-left: 4px solid #3949ab;
  }

  @media (max-width: 768px) {
    .detail-grid {
      grid-template-columns: 1fr;
    }

    .modal-header {
      padding: 16px 20px;
    }

    .modal-body {
      padding: 20px;
    }
  }
</style>