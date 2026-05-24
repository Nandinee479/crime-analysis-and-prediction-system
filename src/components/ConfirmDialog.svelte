<script>
  import { createEventDispatcher } from 'svelte'
  export let open = false
  export let message = 'Are you sure you want to delete this record? This action cannot be undone.'

  const dispatch = createEventDispatcher()
  
  function onKey(e) {
    if (e.key === 'Enter' || e.key === ' ') dispatch('cancel')
  }

  function onWindowKey(e) {
    if (e.key === 'Escape') dispatch('cancel')
  }
</script>

<svelte:window on:keydown={onWindowKey} />

{#if open}
  <div class="modal-overlay" role="button" tabindex="0" on:click|self={() => dispatch('cancel')} on:keydown={onKey}>
    <div class="modal confirm-modal" role="dialog" aria-modal="true" tabindex="-1">
      <div class="modal-header">
        <h3>⚠️ Confirm Delete</h3>
        <button type="button" class="modal-close" aria-label="Close" on:click={() => dispatch('cancel')}>✕</button>
      </div>
      <div class="confirm-body">{message}</div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" on:click={() => dispatch('cancel')}>Cancel</button>
        <button type="button" class="btn btn-danger" on:click={() => dispatch('confirm')}>Delete</button>
      </div>
    </div>
  </div>
{/if}
