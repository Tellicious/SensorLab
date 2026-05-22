<!--
  FftChart
  ========
  Spectrum display based on uPlot. One or more magnitude/dB series share
  a common frequency axis. Supports linear or logarithmic X (frequency)
  and Y (dB vs linear). Fixed-range or auto-scaled Y.

  Like TimeChart, CSS variable colors are resolved to literal values at
  mount time so canvas2D understands them.

  Inline legend in the top-right corner matches TimeChart's style.
-->
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import uPlot from 'uplot';
  import 'uplot/dist/uPlot.min.css';

  interface SeriesDef { label: string; color: string }

  interface Props {
    freqs: Float32Array;
    spectra: Float32Array[];
    seriesDefs: SeriesDef[];
    logY: boolean;
    logX: boolean;
    yMin?: number;
    yMax?: number;
    autoScale: boolean;
    yLabel?: string;
  }

  let {
    freqs, spectra, seriesDefs, logY, logX,
    yMin = -120, yMax = 0, autoScale = true,
    yLabel = 'dB'
  }: Props = $props();

  let container: HTMLDivElement;
  let plot: uPlot | null = null;
  let frame = 0;

  function resolveColor(value: string): string {
    if (!value.startsWith('var(')) return value;
    const name = value.slice(4, -1).trim();
    return getComputedStyle(document.documentElement).getPropertyValue(name).trim() || '#888';
  }

  let resolvedColors = $derived(seriesDefs.map((s) => resolveColor(s.color)));

  function makeOpts(w: number, h: number): uPlot.Options {
    const axisColor = resolveColor('var(--fg-secondary)');
    const gridColor = resolveColor('var(--grid)');
    const tickColor = resolveColor('var(--separator)');
    void logY; // dB conversion happens in the parent; flag kept for API symmetry
    return {
      width: w,
      height: h,
      pxAlign: 0,
      cursor: { show: true, x: true, y: false },
      legend: { show: false },
      scales: {
        x: { time: false, distr: logX ? 3 : 1 },
        y: { auto: autoScale, distr: 1 }
      },
      axes: [
        {
          stroke: axisColor,
          grid: { stroke: gridColor, width: 1 },
          ticks: { stroke: tickColor, width: 1 },
          values: (_u, splits) =>
            splits.map((v) => (v >= 1000 ? (v / 1000).toFixed(1) + 'k' : v.toFixed(0) + 'Hz'))
        },
        {
          stroke: axisColor,
          grid: { stroke: gridColor, width: 1 },
          ticks: { stroke: tickColor, width: 1 },
          label: yLabel,
          labelGap: 4,
          labelSize: 14
        }
      ],
      series: [
        {},
        ...seriesDefs.map((s) => ({
          label: s.label,
          stroke: resolveColor(s.color),
          width: 1.2,
          points: { show: false }
        }))
      ]
    };
  }

  function buildData(): uPlot.AlignedData {
    return [freqs, ...spectra] as unknown as uPlot.AlignedData;
  }

  onMount(() => {
    plot = new uPlot(makeOpts(container.clientWidth, container.clientHeight), buildData(), container);
    const ro = new ResizeObserver(() => {
      plot?.setSize({ width: container.clientWidth, height: container.clientHeight });
    });
    ro.observe(container);

    let last = 0;
    const loop = (t: number) => {
      frame = requestAnimationFrame(loop);
      if (t - last < 50) return;
      last = t;
      if (!plot) return;
      plot.setData(buildData(), false);
      if (!autoScale) plot.setScale('y', { min: yMin, max: yMax });
    };
    frame = requestAnimationFrame(loop);

    return () => { cancelAnimationFrame(frame); ro.disconnect(); };
  });

  onDestroy(() => { plot?.destroy(); plot = null; });
</script>

<div class="fft-chart-wrap">
  <div bind:this={container} class="chart"></div>
  {#if seriesDefs.length > 0}
    <div class="legend" aria-hidden="true">
      {#each seriesDefs as s, i}
        <span class="chip">
          <span class="chip-dot" style:background={resolvedColors[i]}></span>{s.label}
        </span>
      {/each}
    </div>
  {/if}
</div>

<style>
  .fft-chart-wrap {
    position: relative;
    width: 100%;
    height: 100%;
    min-height: 160px;
  }
  .chart {
    width: 100%;
    height: 100%;
  }
  .legend {
    position: absolute;
    top: 4px;
    right: 6px;
    display: flex;
    gap: 6px;
    pointer-events: none;
    padding: 3px 6px;
    border-radius: 6px;
    background: color-mix(in srgb, var(--bg-elev) 78%, transparent);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    font-family: var(--mono);
    font-size: 10px;
    color: var(--fg-secondary);
    z-index: 2;
  }
  .chip {
    display: inline-flex;
    align-items: center;
    gap: 4px;
  }
  .chip-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    display: inline-block;
  }
  :global(.uplot, .uplot *) {
    font-family: var(--mono) !important;
    font-size: 10px !important;
  }
</style>
