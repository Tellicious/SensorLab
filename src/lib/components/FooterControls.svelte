<!--
  FooterControls
  ==============
  Bottom toolbar with the per-page actions: Start/Stop · Reset · Log · CSV.

  Bug fix in this revision: Reset is now ALWAYS enabled (previously it
  was disabled when not running, which made the user perceive "reset
  doesn't work"). Reset clears KPIs and chart buffers regardless of
  running state — the parent's resetKpi() decides what to clear.
-->
<script lang="ts">
  import { sessionState, startLogging, stopLogging } from '$lib/stores/session';
  import type { ModuleKind } from '$lib/storage/db';
  import { db } from '$lib/storage/db';
  import { downloadSessionCsv } from '$lib/storage/csv';

  interface Props {
    module: ModuleKind | 'audio';
    running: boolean;
    onStart: () => void | Promise<void>;
    onStop: () => void | Promise<void>;
    onResetKpi: () => void;
  }
  let { module, running, onStart, onStop, onResetKpi }: Props = $props();

  const canLog = $derived(module !== 'audio');
  const logging = $derived($sessionState.active && $sessionState.module === module);

  async function toggleLog() {
    if (logging) {
      await stopLogging();
    } else {
      if (!running) await onStart();
      await startLogging(module as ModuleKind);
    }
  }

  async function exportLast() {
    if (!canLog) return;
    const last = await db.sessions
      .where('module').equals(module)
      .reverse()
      .first();
    if (!last) {
      alert('No sessions for this module yet. Tap Log to record one first.');
      return;
    }
    await downloadSessionCsv(last);
  }
</script>

<div class="toolbar" role="toolbar" aria-label="Module controls">
  <button
    class={running ? 'btn-destructive' : 'btn-filled'}
    onclick={() => running ? onStop() : onStart()}
  >
    {running ? 'Stop' : 'Start'}
  </button>

  <button class="btn-tinted btn-small" onclick={onResetKpi}>
    Reset
  </button>

  <button
    class={logging ? 'btn-warn btn-small' : 'btn-tinted btn-small'}
    onclick={toggleLog}
    disabled={!canLog}
    title={canLog ? '' : 'Logging not available for Audio module'}
  >
    {logging ? '● Rec' : 'Log'}
  </button>

  <button
    class="btn-tinted btn-small"
    onclick={exportLast}
    disabled={!canLog}
  >
    CSV
  </button>
</div>

<style>
  .toolbar {
    display: flex;
    gap: 8px;
    align-items: center;
    padding: 8px 12px;
    background: var(--bg-elev);
    border-top: 0.5px solid var(--separator);
    flex-shrink: 0;
  }
  .toolbar > button:first-child { flex: 2 1 0; }
  .toolbar > button {
    flex: 1 1 0;
    min-width: 0;
    min-height: 36px;
    padding: 8px 12px;
    border-radius: var(--r-control);
    border: none;
    font-weight: 600;
    cursor: pointer;
  }
  .btn-filled       { background: var(--tint);    color: #fff; min-height: 44px; }
  .btn-destructive  { background: var(--danger);  color: #fff; min-height: 44px; }
  .btn-warn         { background: var(--warn);    color: #fff; }
  .btn-tinted       { background: var(--tint-dim); color: var(--tint); }
  .btn-small        { font-size: var(--t-subhead); }
  button:disabled   { opacity: 0.4; cursor: not-allowed; }
  button:active     { opacity: 0.7; }
</style>
