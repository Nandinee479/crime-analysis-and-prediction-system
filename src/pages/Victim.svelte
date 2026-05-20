<script>
  import { onMount } from 'svelte'
  import Modal from '../components/Modal.svelte'
  import ConfirmDialog from '../components/ConfirmDialog.svelte'
  import { showToast } from '../stores/navigation.js'

  let records = [], filtered = [], search = ''
  let crimes = []
  let showModal = false, showConfirm = false
  let editing = null, deleteId = null, saving = false
  let form = { Victim_Name: '', Age: '', Gender: '', Crime_ID: '' }

  onMount(async () => {
    crimes = await window.api.crime.getAll()
    await load()
  })

  async function load() {
    records = await window.api.victim.getAll()
    applyFilter()
  }

  function applyFilter() {
    const q = search.toLowerCase()
    filtered = q
      ? records.filter(r =>
          r.Victim_Name?.toLowerCase().includes(q) ||
          r.Crime_Type?.toLowerCase().includes(q) ||
          r.Gender?.toLowerCase().includes(q) ||
          String(r.Age).includes(q))
      : records
  }

  $: search, applyFilter()

  function openAdd() {
    editing = null; form = { Victim_Name: '', Age: '', Gender: '', Crime_ID: '' }; showModal = true
  }
  function openEdit(r) {
    editing = r
    form = {
      Victim_Name: r.Victim_Name,
      Age: r.Age ?? '',
      Gender: r.Gender || '',
      Crime_ID: r.Crime_ID ?? ''
    }
    showModal = true
  }
  function openDelete(id) { deleteId = id; showConfirm = true }

  async function save() {
    if (!form.Victim_Name.trim()) return
    saving = true
    try {
      const payload = {
        Victim_Name: form.Victim_Name,
        Age: form.Age === '' ? null : Number(form.Age),
        Gender: form.Gender || null,
        Crime_ID: form.Crime_ID === '' ? null : Number(form.Crime_ID)
      }
      if (editing) {
        await window.api.victim.update({ ...payload, Victim_ID: editing.Victim_ID })
        showToast('Victim record updated')
      } else {
        await window.api.victim.create(payload)
        showToast('Victim record added')
      }
      showModal = false; 
      await load()
    } catch (error) {
      showToast(error.message, 'error')
    } finally { saving = false }
  }

  async function confirmDelete() {
    try {
      await window.api.victim.delete(deleteId)
      records = records.filter(r => r.Victim_ID !== deleteId)
      applyFilter()
      showConfirm = false
      showToast('Victim record deleted', 'success')
    } catch (error) {
      showToast(error.message, 'error')
    }
  }

  //Drop down menu label for crimes

  function crimeLabel(c) {
    return `#${c.Crime_ID} — ${c.Type_Name || 'Unknown'} (${c.Crime_Date || 'no date'})`
  }
</script>

<div class="page-header">
  <div><h2>Victims</h2><p>Manage victim records linked to crimes</p></div>
  <button class="btn btn-primary" on:click={openAdd}>＋ Add Victim</button>
</div>

<div class="content-area">
  <div class="card">
    <div class="toolbar">
      <div class="search-wrap">
        <span class="search-icon">🔎</span>
        <input bind:value={search} placeholder="Search victims…" />
      </div>
      <span style="font-size:13px; color:#888;">{filtered.length} record{filtered.length !== 1 ? 's' : ''}</span>
    </div>

    {#if filtered.length === 0}
      <div class="empty-state">
        <div class="icon">👤</div>
        <p>{search ? 'No results found.' : 'No victim records yet. Click "+ Add Victim" to begin.'}</p>
      </div>
    {:else}
      <div class="table-wrap">
        <table>
          <thead>
            <tr><th>#</th><th>Name</th><th>Age</th><th>Gender</th><th>Related Crime</th><th>Crime Date</th><th style="width:100px;">Actions</th></tr>
          </thead>
          <tbody>
            {#each filtered as r, index}
              <tr>
                <td>{index + 1}</td>
                <td><strong>{r.Victim_Name}</strong></td>
                <td>{r.Age ?? '—'}</td>
                <td>
                  {#if r.Gender}
                    <span class="badge badge-default">{r.Gender}</span>
                  {:else}—{/if}
                </td>
                <td>{r.Crime_Type || '—'}</td>
                <td>{r.Crime_Date || '—'}</td>
                <td>
                  <div class="actions-cell">
                    <button class="btn-icon-sm btn-edit" on:click={() => openEdit(r)}>✏️</button>
                    <button class="btn-icon-sm btn-delete" on:click={() => openDelete(r.Victim_ID)}>🗑️</button>
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

<Modal title={editing ? 'Edit Victim' : 'Add Victim'} bind:open={showModal} on:close={() => showModal = false}>
  <svelte:fragment slot="body">
    <div class="form-group">
      <label class="form-label" for="victim-name">Full Name *</label>
      <input id="victim-name" class="form-control" bind:value={form.Victim_Name} placeholder="Victim full name" />
    </div>
    <div class="form-row">
      <div class="form-group">
        <label class="form-label" for="victim-age">Age</label>
        <input id="victim-age" class="form-control" type="number" min="0" max="120" bind:value={form.Age} placeholder="Age" />
      </div>
      <div class="form-group">
        <label class="form-label" for="victim-gender">Gender</label>
        <select id="victim-gender" class="form-control" bind:value={form.Gender}>
          <option value="">— Select —</option>
          <option>Male</option><option>Female</option><option>Other</option>
        </select>
      </div>
    </div>
    <div class="form-group">
      <label class="form-label" for="victim-crime">Related Crime</label>
      <select id="victim-crime" class="form-control" bind:value={form.Crime_ID}>
        <option value="">— Select Crime —</option>
        {#each crimes as c}
          <option value={c.Crime_ID}>{crimeLabel(c)}</option>
        {/each}
      </select>
    </div>
  </svelte:fragment>
  <svelte:fragment slot="footer">
    <button class="btn btn-secondary" on:click={() => showModal = false}>Cancel</button>
    <button class="btn btn-primary" on:click={save} disabled={saving || !form.Victim_Name.trim()}>
      {saving ? 'Saving…' : editing ? 'Update' : 'Add'}
    </button>
  </svelte:fragment>
</Modal>

<ConfirmDialog bind:open={showConfirm} on:cancel={() => showConfirm = false} on:confirm={confirmDelete} />
