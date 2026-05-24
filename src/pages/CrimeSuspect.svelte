<script>
  import { onMount } from 'svelte'
  import Modal from '../components/Modal.svelte'
  import ConfirmDialog from '../components/ConfirmDialog.svelte'
  import { showToast } from '../stores/navigation.js'

  let records = [], filtered = [], search = ''
  let crimes = [], suspects = []
  let showModal = false, showConfirm = false
  let editing = null, deleteId = null, saving = false
  let form = { Crime_ID: '', Suspect_ID: '' }

  onMount(async () => {
    ;[crimes, suspects] = await Promise.all([
      window.api.crime.getAll(),
      window.api.suspect.getAll()
    ])
    await load()
  })
    //covert number to string for select binding, keep null/undefined as empty string for "no selection" state
  function normalizeSelectionValue(value) {
    return value == null ? '' : String(value)
  }

  async function load() {
    records = await window.api.crimeSuspect.getAll()
    applyFilter()
  }

  function applyFilter() {
    const q = search.toLowerCase()
    filtered = q
      ? records.filter(r =>
          r.Suspect_Name?.toLowerCase().includes(q) ||
          r.Crime_Type?.toLowerCase().includes(q) ||
          r.Crime_Date?.includes(q))
      : records
  }

  $: search, applyFilter()

  function openAdd() {
    editing = null; form = { Crime_ID: '', Suspect_ID: '' }; showModal = true
  }
  function openEdit(r) {
    editing = r
    form = {
      Crime_ID: normalizeSelectionValue(r.Crime_ID),
      Suspect_ID: normalizeSelectionValue(r.Suspect_ID)
    }
    showModal = true
  }
  function openDelete(id) { deleteId = id; showConfirm = true }

  async function save() {
    if (!form.Crime_ID || !form.Suspect_ID) return
    saving = true
    try {
      const payload = { Crime_ID: Number(form.Crime_ID), Suspect_ID: Number(form.Suspect_ID) }
      if (editing) {
        await window.api.crimeSuspect.update({ ...payload, ID: editing.ID })
        showToast('Link updated')
      } else {
        await window.api.crimeSuspect.create(payload)
        showToast('Suspect linked to crime')
      }
      showModal = false; await load()
    } catch (error) {
      showToast(error.message, 'error')
    } finally { saving = false }
  }

  async function confirmDelete() {
    try {
      await window.api.crimeSuspect.delete(deleteId)
      records = records.filter(r => r.ID !== deleteId)
      applyFilter()
      showConfirm = false
      showToast('Link removed', 'success')
    } catch (error) {
      showToast(error.message, 'error')
    }
  }

  function crimeLabel(c) {
    return `#${c.Crime_ID} — ${c.Type_Name || c.Crime_Type || 'Unknown'} (${c.Crime_Date || 'no date'})`
  }
</script>

<div class="page-header">
  <div><h2>Crime ↔ Suspects</h2><p>Link suspects to crime incidents</p></div>
  <button class="btn btn-primary" on:click={openAdd}>＋ Link Suspect</button>
</div>

<div class="content-area">
  <div class="card">
    <div class="toolbar">
      <div class="search-wrap">
        <span class="search-icon">🔎</span>
        <input bind:value={search} placeholder="Search by suspect or crime type…" />
      </div>
      <span style="font-size:13px; color:#888;">{filtered.length} link{filtered.length !== 1 ? 's' : ''}</span>
    </div>

    {#if filtered.length === 0}
      <div class="empty-state">
        <div class="icon">🔗</div>
        <p>{search ? 'No results found.' : 'No links yet. Click "+ Link Suspect" to associate suspects with crimes.'}</p>
      </div>
    {:else}
      <div class="table-wrap">
        <table>
          <thead>
            <tr><th>#</th><th>Crime ID</th><th>Crime Type</th><th>Crime Date</th><th>Suspect</th><th style="width:100px;">Actions</th></tr>
          </thead>
          <tbody>
            {#each filtered as r, index}
              <tr>
                <td>{index + 1}</td>
                <td><span class="badge badge-default">#{ r.Crime_ID}</span></td>
                <td>{r.Crime_Type || '—'}</td>
                <td>{r.Crime_Date || '—'}</td>
                <td><strong>{r.Suspect_Name || '—'}</strong></td>
                <td>
                  <div class="actions-cell">
                    <button class="btn-icon-sm btn-edit" on:click={() => openEdit(r)}>✏️</button>
                    <button class="btn-icon-sm btn-delete" on:click={() => openDelete(r.ID)}>🗑️</button>
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

<Modal title={editing ? 'Edit Crime-Suspect Link' : 'Link Suspect to Crime'} bind:open={showModal} on:close={() => showModal = false}>
  <svelte:fragment slot="body">
    <div class="form-group">
      <label class="form-label" for="crime-select">Crime *</label>
      <select id="crime-select" class="form-control" bind:value={form.Crime_ID}>
        <option value="">— Select Crime —</option>
        {#each crimes as c}
          <option value={normalizeSelectionValue(c.Crime_ID)}>{crimeLabel(c)}</option>
        {/each}
      </select>
    </div>
    <div class="form-group">
      <label class="form-label" for="suspect-select">Suspect *</label>
      <select id="suspect-select" class="form-control" bind:value={form.Suspect_ID}>
        <option value="">— Select Suspect —</option>
        {#each suspects as s}
          <option value={normalizeSelectionValue(s.Suspect_ID)}>{s.Suspect_Name}{s.Age ? `, Age ${s.Age}` : ''}</option>
        {/each}
      </select>
    </div>
    {#if crimes.length === 0 || suspects.length === 0}
      <p style="color:var(--md-warning); font-size:13px; margin-top:8px;">
        ⚠️ {crimes.length === 0 ? 'No crimes found. Add crimes first.' : 'No suspects found. Add suspects first.'}
      </p>
    {/if}
  </svelte:fragment>
  <svelte:fragment slot="footer">
    <button class="btn btn-secondary" on:click={() => showModal = false}>Cancel</button>
    <button class="btn btn-primary" on:click={save} disabled={saving || !form.Crime_ID || !form.Suspect_ID}>
      {saving ? 'Saving…' : editing ? 'Update' : 'Link'}
    </button>
  </svelte:fragment>
</Modal>

<ConfirmDialog
  bind:open={showConfirm}
  message="Remove this suspect-crime link? The suspect and crime records will not be deleted."
  on:cancel={() => showConfirm = false}
  on:confirm={confirmDelete}
/>
