<!--
  TimeChart
  =========
  Scrolling time-series chart based on uPlot. Hosts one or more series
  drawn against a shared X axis (seconds, relative to acquisition start).

  Inputs are pre-allocated typed arrays whose contents are mutated by the
  parent at sensor rate; the chart only reads them on its RAF tick and
  passes slices [0..count) to uPlot. Throttled to ~30 fps.

  CSS-variable color values can't be used directly in canvas, so we
  resolve them once at mount via getComputedStyle.

  Inline legend: a small floating chip cluster in the top-right corner
  shows one dot+label per series. Pointer-events disabled so it doesn't
  intercept chart interaction.
-->
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import uPlot from 'uplot';
  import 'uplot/dist/uPlot.min.css';

  interface SeriesDef { label: string; color: string }

  interface Props {
    xs: Float64Array;
    ys: Float64Array[];
    seriesDefs: SeriesDef[];
    count: number;
    windowSec: number;
    yMin?: number | null;
    yMax?: number | null;
    yLabel?: string;
  }

  let {
    xs, ys, seriesDefs, count, windowSec,
    yMin = null, yMax = null, yLabel = ''
  }: Props = $props();

  let container: HTMLDivElement;
  let plot: uPlot | null = null;
  let frame = 0;

  function resolveColor(value: string): string {
    if (!value.startsWith('var(')) return value;
    const name = value.slice(4, -1).trim();
    return getComputedStyle(document.documentElement).getPropertyValue(name).trim() || '#888';
  }

  /** Resolved colors (literal hex) for the inline legend dots. */
  let resolvedColors = $derived(seriesDefs.map((s) => resolveColor(s.color)));

  function makeOpts(w: number, h: number): uPlot.Options {
    const axisColor = resolveColor('var(--fg-secondary)');
    const gridColor = resolveColor('var(--grid)');
    const tickColor = resolveColor('var(--separator)');
    return {
      width: w,
      height: h,
      pxAlign: 0,
      cursor: { show: false },
      legend: { show: false },   // we render our own HTML legend overlay
      scales: {
        x: { time: false, auto: false },
        y: { auto: yMin === null || yMax === null }
      },
      axes: [
        {
          stroke: axisColor,
          grid: { stroke: gridColor, width: 1 },
          ticks: { stroke: tickColor, width: 1 },
          values: (_u, splits) => splits.map((v) => v.toFixed(1) + 's')
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
          width: 1.5,
          points: { show: false }
        }))
      ]
    };
  }

  function buildData(): uPlot.AlignedData {
    if (count === 0) {
      return [new Float64Array(1), ...ys.map(() => new Float64Array(1))] as unknown as uPlot.AlignedData;
    }
    return [xs.subarray(0, count), ...ys.map((y) => y.subarray(0, count))] as unknown as uPlot.AlignedData;
  }

  onMount(() => {
    plot = new uPlot(makeOpts(container.clientWidth, container.clientHeight), buildData(), container);

    const ro = new ResizeObserver(() => {
      if (!plot) return;
      plot.setSize({ width: container.clientWidth, height: container.clientHeight });
    });
    ro.observe(container);

    let last = 0;
    const loop = (t: number) => {
      frame = requestAnimationFrame(loop);
      if (t - last < 33) return;
      last = t;
      if (!plot || count === 0) return;
      const xMax = xs[count - 1];
      const xMin = Math.max(0, xMax - windowSec);
      plot.setData(buildData(), false);
      plot.setScale('x', { min: xMin, max: xMax });
      if (yMin !== null && yMax !== null) plot.setScale('y', { min: yMin, max: yMax });
    };
    frame = requestAnimationFrame(loop);

    return () => {
      cancelAnimationFrame(frame);
      ro.disconnect();
    };
  });

  onDestroy(() => { plot?.destroy(); plot = null; });
</script>

<div class="time-chart-wrap">
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
  .time-chart-wrap {
    position: relative;
    width: 100%;
    height: 100%;
    min-height: 140px;
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
