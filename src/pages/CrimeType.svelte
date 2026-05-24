<script>
  import { onMount } from 'svelte'
  import Modal from '../components/Modal.svelte'
  import ConfirmDialog from '../components/ConfirmDialog.svelte'
  import { showToast } from '../stores/navigation.js'

  let records = []
  let filtered = []
  let search = ''
  let showModal = false
  let showConfirm = false
  let editing = null
  let deleteId = null
  let form = { Type_Name: '', Description: '' }
  let saving = false

  onMount(load)

  async function load() {
    records = await window.api.crimeType.getAll()
    applyFilter()
  }

  function applyFilter() {
    const q = search.toLowerCase()
    filtered = q
      ? records.filter(r => r.Type_Name?.toLowerCase().includes(q) || 
      r.Description?.toLowerCase().includes(q))
      : records
  }

  $: search, applyFilter()

  function openAdd() {
    editing = null
    form = { Type_Name: '', Description: '' }
    showModal = true
  }

  function openEdit(r) {
    editing = r
    form = { Type_Name: r.Type_Name, Description: r.Description || '' }
    showModal = true
  }

  function openDelete(id) { deleteId = id; showConfirm = true }

  async function save() {
    if (!form.Type_Name.trim()) return
    saving = true
    try {
      if (editing) {
        await window.api.crimeType.update({ ...form, Type_ID: editing.Type_ID })
        showToast('Crime type updated')
      } else {
        await window.api.crimeType.create(form)
        showToast('Crime type added')
      }
      showModal = false
      await load()
    } catch (error) {
      showToast(error.message, 'error')
    } finally { saving = false }
  }

  async function confirmDelete() {
    try {
      await window.api.crimeType.delete(deleteId)
      // Remove from local array immediately for instant UI update
      records = records.filter(r => r.Type_ID !== deleteId)
      applyFilter()
      showConfirm = false
      showToast('Crime type deleted', 'success')
    } catch (error) {
      showToast(error.message, 'error')
    }
  }
</script>

<div class="page-header">
  <div><h2>Crime Types</h2><p>Manage crime categories and descriptions</p></div>
  <button class="btn btn-primary" on:click={openAdd}>＋ Add Crime Type</button>
</div>

<div class="content-area">
  <div class="card">
    <div class="toolbar">
      <div class="search-wrap">
        <span class="search-icon">🔎</span>
        <input bind:value={search} placeholder="Search crime types…" />
      </div>
      <span style="font-size:13px; color:#888;">{filtered.length} record{filtered.length !== 1 ? 's' : ''}</span>
    </div>

    {#if filtered.length === 0}
      <div class="empty-state">
        <div class="icon">🏷️</div>
        <p>{search ? 'No results found.' : 'No crime types yet. Click "+ Add Crime Type" to begin.'}</p>
      </div>
    {:else}
      <div class="table-wrap">
        <table>
          <thead>
            <tr><th>#</th><th>Type Name</th><th>Description</th><th style="width:100px;">Actions</th></tr>
          </thead>
          <tbody>
            {#each filtered as r, index}
              <tr>
                <td>{index + 1}</td>
                <td><strong>{r.Type_Name}</strong></td>
                <td style="color:#555;">{r.Description || '—'}</td>
                <td>
                  <div class="actions-cell">
                    <button class="btn-icon-sm btn-edit" title="Edit" on:click={() => openEdit(r)}>✏️</button>
                    <button class="btn-icon-sm btn-delete" title="Delete" on:click={() => openDelete(r.Type_ID)}>🗑️</button>
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

<Modal title={editing ? 'Edit Crime Type' : 'Add Crime Type'} bind:open={showModal} on:close={() => showModal = false}>
  <svelte:fragment slot="body">
    <div class="form-group">
      <label class="form-label" for="crime-type-name">Type Name *</label>
      <input id="crime-type-name" class="form-control" bind:value={form.Type_Name} placeholder="e.g. Robbery, Assault…" />
    </div>
    <div class="form-group">
      <label class="form-label" for="crime-type-description">Description</label>
      <textarea id="crime-type-description" class="form-control" bind:value={form.Description} placeholder="Optional description…" rows="3"></textarea>
    </div>
  </svelte:fragment>
  <svelte:fragment slot="footer">
    <button class="btn btn-secondary" on:click={() => showModal = false}>Cancel</button>
    <button class="btn btn-primary" on:click={save} disabled={saving || !form.Type_Name.trim()}>
      {saving ? 'Saving…' : editing ? 'Update' : 'Add'}
    </button>
  </svelte:fragment>
</Modal>

<ConfirmDialog bind:open={showConfirm} on:cancel={() => showConfirm = false} on:confirm={confirmDelete} />
