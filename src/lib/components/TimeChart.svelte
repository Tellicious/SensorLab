<!--
  Inline time-series chart powered by uPlot.

  Features:
  - Auto-follows the head of the data (last `windowSec` of x).
  - Inline HTML legend (top-right) — the uPlot built-in legend is
    suppressed because it breaks layout on narrow phone widths.
  - "Fullscreen" affordance: when `fullscreenTitle` is supplied, a
    small ⛶ button in the top-right opens the ChartFullscreen overlay,
    which mounts a second uPlot instance with touch-driven pan / zoom.
    Both instances read from the same Float64 buffer references so the
    fullscreen view is live too.

  Buffers are mutated in place by the parent; the RAF reader picks up
  the latest values each frame without Svelte reactivity getting
  involved. This is also how the audio waveform now stays live (see
  the audio page's `timeBufF64` for the matching pre-allocated buffer).
-->
<script lang="ts">
  import { onMount, onDestroy, untrack } from 'svelte';
  import uPlot from 'uplot';
  import 'uplot/dist/uPlot.min.css';
  import ChartFullscreen from './ChartFullscreen.svelte';

  interface SeriesDef { label: string; color: string; }
  interface Props {
    xs: Float64Array;
    ys: Float64Array[];
    seriesDefs: SeriesDef[];
    count: number;
    windowSec: number;
    yMin?: number;
    yMax?: number;
    yLabel?: string;
    xLabel?: string;
    fullscreenTitle?: string;
    /** Centered moving-average window in samples. 0 = no smoothing. */
    smoothingSamples?: number;
  }
  let {
    xs, ys, seriesDefs, count, windowSec,
    yMin, yMax, yLabel = '', xLabel = 's',
    fullscreenTitle = '',
    smoothingSamples = 0
  }: Props = $props();

  let host: HTMLDivElement;
  let plot: uPlot | null = null;
  let rafId = 0;
  let showFull = $state(false);

  function resolveColor(c: string): string {
    if (!c.startsWith('var(')) return c;
    const name = c.slice(4, -1).trim();
    return getComputedStyle(document.documentElement).getPropertyValue(name).trim() || '#888';
  }
  let resolvedColors = $state<string[]>([]);

  /**
   * Causal moving-average smoothing over `w` samples. Output length = n.
   * For w<=1 returns the input slice unchanged. The smoothing is per-call
   * (allocates), but only when the user has explicitly enabled it.
   */
  function smooth(src: Float64Array, n: number, w: number): number[] {
    if (w <= 1 || n === 0) return Array.from(src.subarray(0, n)) as number[];
    const out = new Array<number>(n);
    let sum = 0;
    for (let i = 0; i < n; i++) {
      sum += src[i];
      if (i >= w) sum -= src[i - w];
      out[i] = sum / Math.min(i + 1, w);
    }
    return out;
  }

  function buildSeries(): uPlot.Series[] {
    const arr: uPlot.Series[] = [{}];
    for (const def of seriesDefs) {
      arr.push({
        label: def.label,
        stroke: resolveColor(def.color),
        width: 1.4,
        spanGaps: false
      });
    }
    return arr;
  }

  function buildData(): uPlot.AlignedData {
    const n = Math.min(count, xs.length);
    const x = Array.from(xs.subarray(0, n)) as number[];
    const data: uPlot.AlignedData = [x];
    for (const y of ys) {
      data.push(smooth(y, n, smoothingSamples) as unknown as (number | null)[]);
    }
    return data;
  }

  function autoFollow() {
    if (!plot) return;
    const n = Math.min(count, xs.length);
    if (n < 2) return;
    const tEnd = xs[n - 1];
    plot.setScale('x', { min: Math.max(0, tEnd - windowSec), max: tEnd });
  }

  function mount() {
    if (!host) return;
    const tickColor  = getComputedStyle(document.documentElement).getPropertyValue('--separator').trim();
    const labelColor = getComputedStyle(document.documentElement).getPropertyValue('--fg-tertiary').trim();

    resolvedColors = seriesDefs.map(d => resolveColor(d.color));

    const opts: uPlot.Options = {
      // Use whatever the host currently reports, plus a non-zero floor so
      // uPlot doesn't initialize at 0×0 (which produces a stuck chart
      // until something else triggers a setSize). The forceResize() below
      // fixes it after the first layout pass anyway.
      width: Math.max(host.clientWidth, 100),
      height: Math.max(host.clientHeight, 100),
      legend: { show: false },
      cursor: { drag: { x: false, y: false }, points: { show: false } },
      scales: {
        x: { time: false },
        y: {
          auto: yMin === undefined && yMax === undefined,
          range: (yMin !== undefined && yMax !== undefined) ? [yMin, yMax] : undefined
        }
      },
      axes: [
        { stroke: labelColor, grid: { stroke: tickColor, width: 0.5 }, ticks: { stroke: tickColor }, label: xLabel },
        { stroke: labelColor, grid: { stroke: tickColor, width: 0.5 }, ticks: { stroke: tickColor }, label: yLabel }
      ],
      series: buildSeries()
    };
    plot = new uPlot(opts, buildData(), host);
    autoFollow();
  }

  /**
   * Force uPlot to re-read the host's actual size. iOS Safari's initial
   * paint sometimes reports 0×0 for flex children during mount, so the
   * chart appears tiny or invisible until something else triggers a
   * resize. We schedule this on the next two RAF ticks to catch both
   * the immediate post-mount state and the post-first-layout state.
   */
  function forceResize() {
    if (!plot || !host) return;
    const w = host.clientWidth, h = host.clientHeight;
    if (w > 0 && h > 0) plot.setSize({ width: w, height: h });
  }

  function refresh() {
    if (!plot) return;
    plot.setData(buildData(), false);
    autoFollow();
    rafId = requestAnimationFrame(refresh);
  }

  function onResize() {
    forceResize();
  }

  onMount(() => {
    untrack(() => mount());
    rafId = requestAnimationFrame(refresh);
    // Two RAF ticks after mount, force a resize. iOS Safari can report
    // host.clientWidth/Height as 0 during the synchronous mount() call,
    // which strands uPlot at 0×0 until something else triggers a resize
    // (user scroll, orientation change, etc.). This was the root cause of
    // "everything out of place at load, need to scroll to fix".
    requestAnimationFrame(() => requestAnimationFrame(forceResize));
    window.addEventListener('resize', onResize);
    const ro = new ResizeObserver(onResize);
    ro.observe(host);
    return () => ro.disconnect();
  });
  onDestroy(() => {
    if (rafId) cancelAnimationFrame(rafId);
    plot?.destroy();
    plot = null;
    window.removeEventListener('resize', onResize);
  });
</script>

<div class="chart-wrapper">
  <div bind:this={host} class="chart-host"></div>
  <div class="legend">
    {#each seriesDefs as def, i}
      <span class="legend-item">
        <span class="swatch" style="background: {resolvedColors[i] ?? def.color}"></span>
        <span>{def.label}</span>
      </span>
    {/each}
  </div>
  {#if fullscreenTitle}
    <button
      class="fullscreen-btn"
      onclick={() => showFull = true}
      aria-label="Expand to full screen"
    >⛶</button>
  {/if}
</div>

{#if showFull}
  <ChartFullscreen
    kind="time"
    title={fullscreenTitle}
    {xs} {ys} {seriesDefs} {count} {windowSec}
    {yMin} {yMax} {yLabel} {xLabel}
    {smoothingSamples}
    onClose={() => showFull = false}
  />
{/if}

<style>
  .chart-wrapper {
    position: relative;
    width: 100%;
    height: 100%;
  }
  .chart-host { width: 100%; height: 100%; }
  .legend {
    position: absolute;
    top: 4px; right: 36px;
    display: flex; flex-wrap: wrap; gap: 8px;
    pointer-events: none;
  }
  .legend-item {
    display: inline-flex; align-items: center; gap: 4px;
    font-size: 10px;
    color: var(--fg-secondary);
    background: var(--bg-elev);
    padding: 2px 6px;
    border-radius: var(--r-pill);
  }
  .swatch {
    width: 8px; height: 8px;
    border-radius: 50%;
    display: inline-block;
  }
  .fullscreen-btn {
    position: absolute;
    top: 4px; right: 4px;
    width: 28px; height: 28px;
    background: var(--fill-tertiary);
    color: var(--fg-secondary);
    border: none;
    border-radius: var(--r-control);
    font-size: 14px;
    cursor: pointer;
    z-index: 2;
  }
  .fullscreen-btn:active { opacity: 0.6; }
</style>
