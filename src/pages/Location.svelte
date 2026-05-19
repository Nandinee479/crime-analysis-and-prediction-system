<script>
  import { onMount } from 'svelte'
  import Modal from '../components/Modal.svelte'
  import ConfirmDialog from '../components/ConfirmDialog.svelte'
  import { showToast } from '../stores/navigation.js'

  let records = [], filtered = [], search = ''
  let showModal = false, showConfirm = false
  let editing = null, deleteId = null, saving = false
  let form = { Location_Name: '', City: '', Area_Type: '' }

  const areaTypes = ['Urban', 'Suburban', 'Rural', 'Commercial', 'Industrial', 'Residential', 'Other']

  onMount(load)

  async function load() {
    records = await window.api.location.getAll()
    applyFilter()
  }

  function applyFilter() {
    const q = search.toLowerCase()
    filtered = q
      ? records.filter(r => r.Location_Name?.toLowerCase().includes(q) || r.City?.toLowerCase().includes(q) || r.Area_Type?.toLowerCase().includes(q))
      : records
  }

  $: search, applyFilter()

  function openAdd() {
    editing = null; form = { Location_Name: '', City: '', Area_Type: '' }; showModal = true
  }
  function openEdit(r) {
    editing = r; form = { Location_Name: r.Location_Name, City: r.City || '', Area_Type: r.Area_Type || '' }; showModal = true
  }
  function openDelete(id) { deleteId = id; showConfirm = true }

  async function save() {
    if (!form.Location_Name.trim()) return
    saving = true
    try {
      if (editing) {
        await window.api.location.update({ ...form, Location_ID: editing.Location_ID })
        showToast('Location updated')
      } else {
        await window.api.location.create(form)
        showToast('Location added')
      }
      showModal = false; await load()
    } catch (error) {
      showToast(error.message, 'error')
    } finally { saving = false }
  }

  async function confirmDelete() {
    try {
      await window.api.location.delete(deleteId)
      records = records.filter(r => r.Location_ID !== deleteId)
      applyFilter()
      showConfirm = false
      showToast('Location deleted', 'success')
    } catch (error) {
      showToast(error.message, 'error')
    }
  }
</script>

<div class="page-header">
  <div><h2>Locations</h2><p>Manage crime scene locations</p></div>
  <button class="btn btn-primary" on:click={openAdd}>＋ Add Location</button>
</div>

<div class="content-area">
  <div class="card">
    <div class="toolbar">
      <div class="search-wrap">
        <span class="search-icon">🔎</span>
        <input bind:value={search} placeholder="Search locations…" />
      </div>
      <span style="font-size:13px; color:#888;">{filtered.length} record{filtered.length !== 1 ? 's' : ''}</span>
    </div>

    {#if filtered.length === 0}
      <div class="empty-state">
        <div class="icon">📍</div>
        <p>{search ? 'No results found.' : 'No locations yet. Click "+ Add Location" to begin.'}</p>
      </div>
    {:else}
      <div class="table-wrap">
        <table>
          <thead>
            <tr><th>#</th><th>Location Name</th><th>City</th><th>Area Type</th><th style="width:100px;">Actions</th></tr>
          </thead>
          <tbody>
            {#each filtered as r, index}
              <tr>
                <td>{index + 1}</td>
                <td><strong>{r.Location_Name}</strong></td>
                <td>{r.City || '—'}</td>
                <td>
                  {#if r.Area_Type}
                    <span class="badge badge-default">{r.Area_Type}</span>
                  {:else}—{/if}
                </td>
                <td>
                  <div class="actions-cell">
                    <button class="btn-icon-sm btn-edit" on:click={() => openEdit(r)}>✏️</button>
                    <button class="btn-icon-sm btn-delete" on:click={() => openDelete(r.Location_ID)}>🗑️</button>
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

<Modal title={editing ? 'Edit Location' : 'Add Location'} bind:open={showModal} on:close={() => showModal = false}>
  <svelte:fragment slot="body">
    <div class="form-group">
      <label class="form-label" for="location-name">Location Name *</label>
      <input id="location-name" class="form-control" bind:value={form.Location_Name} placeholder="e.g. Main Street, Central Park…" />
    </div>
    <div class="form-row">
      <div class="form-group">
        <label class="form-label" for="location-city">City</label>
        <input id="location-city" class="form-control" bind:value={form.City} placeholder="City name" />
      </div>
      <div class="form-group">
        <label class="form-label" for="location-area">Area Type</label>
        <select id="location-area" class="form-control" bind:value={form.Area_Type}>
          <option value="">— Select —</option>
          {#each areaTypes as t}<option value={t}>{t}</option>{/each}
        </select>
      </div>
    </div>
  </svelte:fragment>
  <svelte:fragment slot="footer">
    <button class="btn btn-secondary" on:click={() => showModal = false}>Cancel</button>
    <button class="btn btn-primary" on:click={save} disabled={saving || !form.Location_Name.trim()}>
      {saving ? 'Saving…' : editing ? 'Update' : 'Add'}
    </button>
  </svelte:fragment>
</Modal>

<ConfirmDialog bind:open={showConfirm} on:cancel={() => showConfirm = false} on:confirm={confirmDelete} />
