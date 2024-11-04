<!-- ControlButtons.svelte -->
<script lang="ts">
  import { createEventDispatcher } from 'svelte';

  const dispatch = createEventDispatcher();

  function handleStart() {
    dispatch('start');
  }

  function handleStop() {
    dispatch('stop');
  }
</script>

<button on:click={handleStart} class="start-button">赤色の割合を計算</button>
<button on:click={handleStop} class="stop-button">赤色の検知を停止</button>

<style>
  .start-button,
  .stop-button {
    position: absolute;
    font-size: 14px;
    padding: 4px 8px;
    z-index: 2;
  }
  .start-button {
    top: 10px;
    left: 10px;
  }
  .stop-button {
    top: 10px;
    left: 150px;
  }
</style>
