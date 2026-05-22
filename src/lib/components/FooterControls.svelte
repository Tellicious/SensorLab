<!--
  FooterControls
  ==============
  Bottom toolbar with the per-module actions: Start/Stop, Reset, Log, CSV.

  Layout: all four buttons share the same height (44 pt — iOS touch
  target). The primary action (Start ↔ Stop) takes a bit more horizontal
  space and uses filled / destructive styling. Secondary actions use the
  tinted style. The Log button switches to a recording style with a
  pulsing dot when a session is active.

  Previously the secondary buttons used `btn-small` (32 pt) which made
  them visibly shorter than the primary — looked unbalanced and was
  under the iOS recommended touch-target. Now all are 44 pt.
-->
<script lang="ts">
  import { sessionState, startLogging, stopLogging } from '$lib/stores/session';
  import type { ModuleKind } from '$lib/storage/db';
  import { db } from '$lib/storage/db';
  import { downloadSessionCsv } from '$lib/storage/csv';

  interface Props {
    /** Module identity. 'audio' has logging disabled. */
    module: ModuleKind | 'audio';
    /** Whether the sensor controller is currently active. */
    running: boolean;
    /** Called when the user taps Start. May be async (e.g. permission flow). */
    onStart: () => void | Promise<void>;
    /** Called when the user taps Stop. */
    onStop: () => void | Promise<void>;
    /** Called when the user taps Reset KPI. */
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
    class="primary {running ? 'btn-destructive' : 'btn-filled'}"
    onclick={() => running ? onStop() : onStart()}
  >
    {running ? 'Stop' : 'Start'}
  </button>

  <button
    class="secondary btn-tinted"
    onclick={onResetKpi}
    disabled={!running}
  >
    Reset
  </button>

  <button
    class="secondary {logging ? 'btn-rec' : 'btn-tinted'}"
    onclick={toggleLog}
    disabled={!canLog}
    title={canLog ? '' : 'Logging not available for Audio module'}
  >
    {#if logging}
      <span class="rec-dot" aria-hidden="true"></span>Rec
    {:else}
      Log
    {/if}
  </button>

  <button
    class="secondary btn-tinted"
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
    align-items: stretch;
    padding: 10px 16px;
    background: var(--bg-elev);
    border-top: 0.5px solid var(--separator);
  }
  .toolbar > button {
    flex: 1 1 0;
    min-width: 0;
    /* Unify height across all buttons (was inconsistent: filled=44, small=32) */
    min-height: 40px;
    padding: 9px 10px;
    font-size: var(--t-subhead);
    font-weight: 600;
    border-radius: var(--r-button);
    /* Allow ellipsis if extremely cramped, but normally text fits */
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  /* Primary action takes more space */
  .toolbar > button.primary {
    flex: 1.6 1 0;
    font-size: var(--t-body);
  }

  .btn-rec {
    background: var(--warn);
    color: #ffffff;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
  }
  .rec-dot {
    display: inline-block;
    width: 8px;
    height: 8px;
    background: #ffffff;
    border-radius: 50%;
    animation: rec-pulse 1.1s ease-in-out infinite;
    flex-shrink: 0;
  }
  @keyframes rec-pulse {
    0%, 100% { opacity: 1; }
    50%      { opacity: 0.35; }
  }
</style>
