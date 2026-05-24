<script>
  import { createEventDispatcher } from 'svelte'
  export let title = ''
  export let open = false

  const dispatch = createEventDispatcher()

  function close() { dispatch('close') }

  function onKey(e) { if (e.key === 'Escape') close() }

  function overlayKey(e) {
    if (e.target !== e.currentTarget) return
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault()
      close()
    }
  }
</script>

<svelte:window on:keydown={onKey} />

{#if open}
  <div class="modal-overlay" role="button" tabindex="0" on:click|self={close} on:keydown={overlayKey}>
    <div class="modal" role="dialog" aria-modal="true" tabindex="-1">
      <div class="modal-header">
        <h3>{title}</h3>
        <button type="button" class="modal-close" on:click={close} aria-label="Close">✕</button>
      </div>
      <div class="modal-body">
        <slot name="body" />
      </div>
      <div class="modal-footer">
        <slot name="footer" />
      </div>
    </div>
  </div>
{/if}
