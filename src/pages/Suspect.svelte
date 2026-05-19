<script>
  import { onMount } from 'svelte'
  import Modal from '../components/Modal.svelte'
  import ConfirmDialog from '../components/ConfirmDialog.svelte'
  import { showToast } from '../stores/navigation.js'

  let records = [], filtered = [], search = ''
  let showModal = false, showConfirm = false
  let editing = null, deleteId = null, saving = false
  let form = { Suspect_Name: '', Age: '', Gender: '' }

  onMount(load)

  async function load() {
    records = await window.api.suspect.getAll()
    applyFilter()
  }

  function applyFilter() {
    const q = search.toLowerCase()
    filtered = q
      ? records.filter(r => r.Suspect_Name?.toLowerCase().includes(q) || String(r.Age).includes(q) || r.Gender?.toLowerCase().includes(q))
      : records
  }

  $: search, applyFilter()

  function openAdd() {
    editing = null; form = { Suspect_Name: '', Age: '', Gender: '' }; showModal = true
  }
  function openEdit(r) {
    editing = r; form = { Suspect_Name: r.Suspect_Name, Age: r.Age ?? '', Gender: r.Gender || '' }; showModal = true
  }
  function openDelete(id) { deleteId = id; showConfirm = true }

  async function save() {
    if (!form.Suspect_Name.trim()) return
    saving = true
    try {
      const payload = { ...form, Age: form.Age === '' ? null : Number(form.Age) }
      if (editing) {
        await window.api.suspect.update({ ...payload, Suspect_ID: editing.Suspect_ID })
        showToast('Suspect updated')
      } else {
        await window.api.suspect.create(payload)
        showToast('Suspect added')
      }
      showModal = false; await load()
    } catch (error) {
      showToast(error.message, 'error')
    } finally { saving = false }
  }

  async function confirmDelete() {
    try {
      await window.api.suspect.delete(deleteId)
      records = records.filter(r => r.Suspect_ID !== deleteId)
      applyFilter()
      showConfirm = false
      showToast('Suspect deleted', 'success')
    } catch (error) {
      showToast(error.message, 'error')
    }
  }
</script>

<div class="page-header">
  <div><h2>Suspects</h2><p>Manage suspect profiles</p></div>
  <button class="btn btn-primary" on:click={openAdd}>＋ Add Suspect</button>
</div>

<div class="content-area">
  <div class="card">
    <div class="toolbar">
      <div class="search-wrap">
        <span class="search-icon">🔎</span>
        <input bind:value={search} placeholder="Search suspects…" />
      </div>
      <span style="font-size:13px; color:#888;">{filtered.length} record{filtered.length !== 1 ? 's' : ''}</span>
    </div>

    {#if filtered.length === 0}
      <div class="empty-state">
        <div class="icon">🔍</div>
        <p>{search ? 'No results found.' : 'No suspects yet. Click "+ Add Suspect" to begin.'}</p>
      </div>
    {:else}
      <div class="table-wrap">
        <table>
          <thead>
            <tr><th>#</th><th>Name</th><th>Age</th><th>Gender</th><th style="width:100px;">Actions</th></tr>
          </thead>
          <tbody>
            {#each filtered as r, index}
              <tr>
                <td>{index + 1}</td>
                <td><strong>{r.Suspect_Name}</strong></td>
                <td>{r.Age ?? '—'}</td>
                <td>
                  {#if r.Gender}
                    <span class="badge badge-default">{r.Gender}</span>
                  {:else}—{/if}
                </td>
                <td>
                  <div class="actions-cell">
                    <button class="btn-icon-sm btn-edit" on:click={() => openEdit(r)}>✏️</button>
                    <button class="btn-icon-sm btn-delete" on:click={() => openDelete(r.Suspect_ID)}>🗑️</button>
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

<Modal title={editing ? 'Edit Suspect' : 'Add Suspect'} bind:open={showModal} on:close={() => showModal = false}>
  <svelte:fragment slot="body">
    <div class="form-group">
      <label class="form-label" for="suspect-name">Full Name *</label>
      <input id="suspect-name" class="form-control" bind:value={form.Suspect_Name} placeholder="Suspect full name" />
    </div>
    <div class="form-row">
      <div class="form-group">
        <label class="form-label" for="suspect-age">Age</label>
        <input id="suspect-age" class="form-control" type="number" min="1" max="120" bind:value={form.Age} placeholder="Age" />
      </div>
      <div class="form-group">
        <label class="form-label" for="suspect-gender">Gender</label>
        <select id="suspect-gender" class="form-control" bind:value={form.Gender}>
          <option value="">— Select —</option>
          <option>Male</option><option>Female</option><option>Other</option>
        </select>
      </div>
    </div>
  </svelte:fragment>
  <svelte:fragment slot="footer">
    <button class="btn btn-secondary" on:click={() => showModal = false}>Cancel</button>
    <button class="btn btn-primary" on:click={save} disabled={saving || !form.Suspect_Name.trim()}>
      {saving ? 'Saving…' : editing ? 'Update' : 'Add'}
    </button>
  </svelte:fragment>
</Modal>

<ConfirmDialog bind:open={showConfirm} on:cancel={() => showConfirm = false} on:confirm={confirmDelete} />
