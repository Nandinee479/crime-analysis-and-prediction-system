<script>
  import { onMount } from 'svelte'
  import Modal from '../components/Modal.svelte'
  import ConfirmDialog from '../components/ConfirmDialog.svelte'
  import { showToast } from '../stores/navigation.js'

  let records = [], filtered = [], search = ''
  let crimeTypes = [], locations = []
  let showModal = false, showConfirm = false
  let editing = null, deleteId = null, saving = false
  let form = { Type_ID: '', Severity: '', Crime_Date: '', Location_ID: '' }

  const severities = ['Low', 'Medium', 'High']

  onMount(async () => {
    [crimeTypes, locations] = await Promise.all([
      window.api.crimeType.getAll(),
      window.api.location.getAll()
    ])
    await load()
  })

  async function load() {
    records = await window.api.crime.getAll()
    applyFilter()
  }

  function applyFilter() {
    const q = search.toLowerCase()
    filtered = q
      ? records.filter(r =>
          r.Type_Name?.toLowerCase().includes(q) ||
          r.Location_Name?.toLowerCase().includes(q) ||
          r.Severity?.toLowerCase().includes(q) ||
          r.Crime_Date?.includes(q))
      : records
  }

  $: search, applyFilter()

  function openAdd() {
    editing = null
    form = { Type_ID: '', Severity: '', Crime_Date: '', Location_ID: '' }
    showModal = true
  }
  function openEdit(r) {
    editing = r
    form = {
      Type_ID: r.Type_ID ?? '',
      Severity: r.Severity || '',
      Crime_Date: r.Crime_Date || '',
      Location_ID: r.Location_ID ?? ''
    }
    showModal = true
  }
  function openDelete(id) { deleteId = id; showConfirm = true }

  async function save() {
    saving = true
    try {
      const payload = {
        Type_ID: form.Type_ID === '' ? null : Number(form.Type_ID),
        Severity: form.Severity || null,
        Crime_Date: form.Crime_Date || null,
        Location_ID: form.Location_ID === '' ? null : Number(form.Location_ID)
      }
      if (editing) {
        await window.api.crime.update({ ...payload, Crime_ID: editing.Crime_ID })
        showToast('Crime record updated')
      } else {
        await window.api.crime.create(payload)
        showToast('Crime record added')
      }
      showModal = false; await load()
    } catch (error) {
      showToast(error.message, 'error')
    } finally { saving = false }
  }

  async function confirmDelete() {
    try {
      await window.api.crime.delete(deleteId)
      records = records.filter(r => r.Crime_ID !== deleteId)
      applyFilter()
      showConfirm = false
      showToast('Crime record deleted', 'success')
    } catch (error) {
      showToast(error.message, 'error')
    }
  }

  function severityClass(s) {
    if (!s) return 'badge-default'
    const l = s.toLowerCase()
    if (l === 'high')   return 'badge-high'
    if (l === 'medium') return 'badge-medium'
    if (l === 'low')    return 'badge-low'
    return 'badge-default'
  }
</script>

<div class="page-header">
  <div><h2>Crimes</h2>
    <p>Manage crime incident records</p></div>
  <button class="btn btn-primary" on:click={openAdd}>＋ Add Crime</button>
</div>

<div class="content-area">
  <div class="card">
    <div class="toolbar">
      <div class="search-wrap">
        <span class="search-icon">🔎</span>
        <input bind:value={search} placeholder="Search crimes…" />
      </div>
      <span style="font-size:13px; color:#888;">{filtered.length} record{filtered.length !== 1 ? 's' : ''}</span>
    </div>

    {#if filtered.length === 0}
      <div class="empty-state">
        <div class="icon">🚔</div>
        <p>{search ? 'No results found.' : 'No crime records yet. Click "+ Add Crime" to begin.'}</p>
      </div>
    {:else}
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>#</th>
              <th>Date</th>
              <th>Type</th>
              <th>Location</th>
              <th>Severity</th>
              <th style="width:100px;">Actions</th>
            </tr>
          </thead>
          <tbody>
            {#each filtered as r, index}
              <tr>
                <td>{index + 1}</td>
                <td>{r.Crime_Date || '—'}</td>
                <td>{r.Type_Name || '—'}</td>
                <td>{r.Location_Name || '—'}</td>
                <td>
                  {#if r.Severity}
                    <span class="badge {severityClass(r.Severity)}">{r.Severity}</span>
                  {:else}—{/if}
                </td>
                <td>
                  <div class="actions-cell">
                    <button class="btn-icon-sm btn-edit" on:click={() => openEdit(r)}>✏️</button>
                    <button class="btn-icon-sm btn-delete" on:click={() => openDelete(r.Crime_ID)}>🗑️</button>
                  </div>
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    {/if}
  </div>
</div>

<Modal title={editing ? 'Edit Crime Record' : 'Add Crime Record'} bind:open={showModal} on:close={() => showModal = false}>
  <svelte:fragment slot="body">
    <div class="form-row">
      <div class="form-group">
        <label class="form-label" for="crime-type">Crime Type</label>
        <select id="crime-type" class="form-control" bind:value={form.Type_ID}>
          <option value="">— Select Type —</option>
          {#each crimeTypes as t}
            <option value={t.Type_ID}>{t.Type_Name}</option>
          {/each}
        </select>
      </div>
      <div class="form-group">
        <label class="form-label" for="crime-severity">Severity</label>
        <select id="crime-severity" class="form-control" bind:value={form.Severity}>
          <option value="">— Select —</option>
          {#each severities as s}<option value={s}>{s}</option>{/each}
        </select>
      </div>
    </div>
    <div class="form-row">
      <div class="form-group">
        <label class="form-label" for="crime-date">Crime Date</label>
        <input id="crime-date" class="form-control" type="date" bind:value={form.Crime_Date} />
      </div>
      <div class="form-group">
        <label class="form-label" for="crime-location">Location</label>
        <select id="crime-location" class="form-control" bind:value={form.Location_ID}>
          <option value="">— Select Location —</option>
          {#each locations as l}
            <option value={l.Location_ID}>{l.Location_Name}{l.City ? `, ${l.City}` : ''}</option>
          {/each}
        </select>
      </div>
    </div>
  </svelte:fragment>
  <svelte:fragment slot="footer">
    <button class="btn btn-secondary" on:click={() => showModal = false}>Cancel</button>
    <button class="btn btn-primary" on:click={save} disabled={saving}>{saving ? 'Saving…' : editing ? 'Update' : 'Add'}</button>
  </svelte:fragment>
</Modal>

<ConfirmDialog bind:open={showConfirm} on:cancel={() => showConfirm = false} on:confirm={confirmDelete} />
