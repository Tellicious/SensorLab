<!--
  Motion module page.

  Provides the full Motion experience end-to-end:
  - Permission request flow (iOS gates devicemotion behind a user gesture)
  - Live time-domain chart per channel group
  - Live FFT chart with selectable size / window / overlap
  - Six live KPI tiles for |a|: peak, RMS, crest, kurtosis, peak-hold, pk-pk
  - Per-axis KPIs for linear acceleration (Peak / RMS / Avg per visible axis)
  - List of N dominant frequencies from the latest FFT frame
  - Optional 3D orientation cube
  - Bottom toolbar with Start/Stop/Reset/Log/CSV

  Reactivity model:
  - Per-sample work (buffer append, KPI tracker push, channel store) runs at
    full sensor rate (~60 Hz on iOS).
  - UI snapshot ($state objects below) is refreshed at 10 Hz by a single
    interval. KpiCard components read these snapshot values via props and
    re-render their text only — they do NOT get torn down on every update.
  - This replaces an earlier pattern that wrapped the KPI grid in
    {#key tick} which remounted every card every 100 ms — causing visible
    layout jumps, lost click events on the toolbar, and a lot of garbage.
  - FFT is computed by a timer (5 Hz) with zero-padding when the buffer
    isn't yet full, so the spectrum becomes visible immediately rather
    than after ~17 s with default fftSize=1024.
-->
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { settings, updateMotion } from '$lib/stores/settings';
  import { pushMotion } from '$lib/stores/session';
  import {
    createMotionController, requestMotionPermission, type MotionSample
  } from '$lib/sensors/motion';
  import { ChannelStats } from '$lib/dsp/kpi';
  import { FftProcessor, dominantFrequencies, FFT_SIZES } from '$lib/dsp/fft';
  import { WINDOW_NAMES } from '$lib/dsp/windowing';
  import KpiCard from '$lib/components/KpiCard.svelte';
  import TimeChart from '$lib/components/TimeChart.svelte';
  import FftChart from '$lib/components/FftChart.svelte';
  import FooterControls from '$lib/components/FooterControls.svelte';
  import OrientationCube from '$lib/components/OrientationCube.svelte';

  // ---- Buffer config ----------------------------------------------------
  const CAP = 4096;
  const xs = new Float64Array(CAP);
  const ax = new Float64Array(CAP);
  const ay = new Float64Array(CAP);
  const az = new Float64Array(CAP);
  const am = new Float64Array(CAP);
  const axg = new Float64Array(CAP);
  const ayg = new Float64Array(CAP);
  const azg = new Float64Array(CAP);
  const gx = new Float64Array(CAP);
  const gy = new Float64Array(CAP);
  const gz = new Float64Array(CAP);
  const ox = new Float64Array(CAP);
  const oy = new Float64Array(CAP);
  const oz = new Float64Array(CAP);
  let count = $state(0);
  let t0 = 0;

  let orientation = $state({ alpha: 0, beta: 0, gamma: 0 });

  // ---- Sensor controller ------------------------------------------------
  const ctrl = createMotionController();
  let running = $state(false);
  let permission = $state<'unknown' | 'granted' | 'denied' | 'unsupported'>('unknown');
  let hz = $state(0);
  let hzTimer: number | null = null;

  // ---- KPI trackers (plain class instances, not Svelte $state) ---------
  // We snapshot their values into reactive $state objects below.
  function makeStats() {
    const s = $state.snapshot($settings).motion;
    return new ChannelStats({
      capacity: CAP,
      rmsWindowSec: s.rmsWindowSec,
      meanWindowSec: s.meanWindowSec,
      peakHoldDecayDbPerSec: s.peakHoldDecayDbPerSec
    });
  }
  let statsX = makeStats();
  let statsY = makeStats();
  let statsZ = makeStats();
  let statsM = makeStats();

  // ---- KPI snapshot objects --------------------------------------------
  // These are the only thing the template binds to. Updated by the
  // display ticker. No {#key} needed → KpiCard instances persist across
  // updates, so click handlers and layout stay stable.
  let kpiM = $state({ peak: 0, rms: 0, crest: 0, kurt: 0, hold: 0, pkpk: 0 });
  let kpiX = $state({ peak: 0, rms: 0, mean: 0 });
  let kpiY = $state({ peak: 0, rms: 0, mean: 0 });
  let kpiZ = $state({ peak: 0, rms: 0, mean: 0 });

  function refreshKpiSnapshot() {
    kpiM = {
      peak: statsM.peak.peak,
      rms: statsM.rms.rms,
      crest: statsM.crestFactor,
      kurt: statsM.kurt.kurtosis,
      hold: statsM.peakHold.hold,
      pkpk: statsM.peak.peakToPeak
    };
    kpiX = { peak: statsX.peak.peak, rms: statsX.rms.rms, mean: statsX.mean.mean };
    kpiY = { peak: statsY.peak.peak, rms: statsY.rms.rms, mean: statsY.mean.mean };
    kpiZ = { peak: statsZ.peak.peak, rms: statsZ.rms.rms, mean: statsZ.mean.mean };
  }

  // ---- FFT --------------------------------------------------------------
  let fft: FftProcessor | null = null;
  let fftMag = $state<Float32Array>(new Float32Array(1));
  let fftDb = $state<Float32Array>(new Float32Array(1));
  let fftFreqs = $state<Float32Array>(new Float32Array(1));
  let dominant = $state<Array<{ freq: number; mag: number }>>([]);
  let fftTimer: number | null = null;

  function ensureFft() {
    const s = $settings.motion;
    const rate = Math.max(hz || 60, 1);
    if (!fft || fft.size !== s.fftSize || Math.abs(fft.sampleRate - rate) > 1) {
      fft = new FftProcessor({
        size: s.fftSize,
        window: s.fftWindow,
        overlapPct: s.fftOverlapPct,
        sampleRate: rate
      });
      fftMag = new Float32Array(fft.bins);
      fftDb = new Float32Array(fft.bins);
      fftFreqs = fft.freqs;
    }
  }

  /**
   * Compute spectrum from the most recent fft.size samples of |a|.
   * Zero-pads when the buffer isn't yet full so the chart shows
   * something useful from the very first second of acquisition
   * (previously had to wait for the buffer to fill, ~17s @ 60Hz @ N=1024).
   */
  function computeFft() {
    if (!fft || count < 16) return;
    const N = fft.size;
    const available = Math.min(count, N);
    const frame = new Float32Array(N); // zero-filled by spec
    for (let i = 0; i < available; i++) {
      frame[N - available + i] = am[count - available + i];
    }
    fft.compute(frame, fftMag, fftDb);
    const s = $settings.motion;
    dominant = dominantFrequencies(fftMag, fft.freqs, s.dominantFreqCount);
    // Force Svelte reactivity by reassigning. The contents are mutated
    // in-place above, which canvas readers see immediately, but binding
    // updates still need the assignment.
    fftMag = fftMag;
    fftDb = fftDb;
  }

  // ---- Sample callback --------------------------------------------------
  function onSample(s: MotionSample) {
    if (t0 === 0) t0 = s.t;
    const tSec = (s.t - t0) / 1000;
    const mag = Math.hypot(s.ax, s.ay, s.az);
    if (count < CAP) {
      const i = count++;
      xs[i] = tSec;
      ax[i] = s.ax;   ay[i] = s.ay;   az[i] = s.az;   am[i] = mag;
      axg[i] = s.axg; ayg[i] = s.ayg; azg[i] = s.azg;
      gx[i] = s.gx;   gy[i] = s.gy;   gz[i] = s.gz;
      ox[i] = s.ox;   oy[i] = s.oy;   oz[i] = s.oz;
    } else {
      const slots = [xs, ax, ay, az, am, axg, ayg, azg, gx, gy, gz, ox, oy, oz];
      for (const b of slots) b.copyWithin(0, 1);
      const i = CAP - 1;
      xs[i] = tSec;
      ax[i] = s.ax;   ay[i] = s.ay;   az[i] = s.az;   am[i] = mag;
      axg[i] = s.axg; ayg[i] = s.ayg; azg[i] = s.azg;
      gx[i] = s.gx;   gy[i] = s.gy;   gz[i] = s.gz;
      ox[i] = s.ox;   oy[i] = s.oy;   oz[i] = s.oz;
    }

    statsX.push(s.ax, s.t);
    statsY.push(s.ay, s.t);
    statsZ.push(s.az, s.t);
    statsM.push(mag, s.t);

    orientation = { alpha: s.ox, beta: s.oy, gamma: s.oz };

    pushMotion({
      t: Math.floor(s.t - t0),
      ax: s.ax, ay: s.ay, az: s.az,
      axg: s.axg, ayg: s.ayg, azg: s.azg,
      gx: s.gx, gy: s.gy, gz: s.gz,
      ox: s.ox, oy: s.oy, oz: s.oz
    });
  }

  // ---- Start / Stop -----------------------------------------------------
  async function start() {
    permission = await requestMotionPermission();
    if (permission !== 'granted') return;
    ensureFft();
    t0 = 0;
    count = 0;
    statsX.reset(); statsY.reset(); statsZ.reset(); statsM.reset();
    refreshKpiSnapshot();
    ctrl.start(onSample);
    running = true;
    hzTimer = window.setInterval(() => {
      hz = ctrl.hz;
      if (fft && Math.abs(fft.sampleRate - (hz || 60)) > 5) ensureFft();
    }, 500);
    // Independent FFT compute timer — 5 Hz refresh keeps the spectrum
    // responsive regardless of overlap/hop settings, and combined with
    // zero-padding gives an immediate visible result.
    fftTimer = window.setInterval(computeFft, 200);
  }

  function stop() {
    ctrl.stop();
    running = false;
    if (hzTimer !== null) { clearInterval(hzTimer); hzTimer = null; }
    if (fftTimer !== null) { clearInterval(fftTimer); fftTimer = null; }
  }
  function resetKpi() {
    statsX.reset(); statsY.reset(); statsZ.reset(); statsM.reset();
    refreshKpiSnapshot();
  }
  onDestroy(stop);

  // ---- Display ticker --------------------------------------------------
  // Single 10 Hz interval that refreshes the snapshot. Cheap; only does
  // ~12 number reads + one assignment per axis per cycle.
  let displayInterval: number;
  onMount(() => {
    displayInterval = window.setInterval(refreshKpiSnapshot, 100);
    return () => clearInterval(displayInterval);
  });

  // ---- Chart series ----------------------------------------------------
  const COLOR_X = 'var(--series-3)';
  const COLOR_Y = 'var(--series-2)';
  const COLOR_Z = 'var(--series-1)';
  const COLOR_M = 'var(--series-4)';
  const COLOR_SPEC = 'var(--tint)';

  interface ChannelGroup {
    key: string;
    label: string;
    unit: string;
    ys: Float64Array[];
    defs: Array<{ label: string; color: string }>;
  }
  let channelGroups = $derived.by<ChannelGroup[]>(() => {
    const s = $settings.motion;
    const groups: ChannelGroup[] = [];
    if (s.showLinear) {
      const ys: Float64Array[] = []; const defs: Array<{ label: string; color: string }> = [];
      if (s.axisX) { ys.push(ax); defs.push({ label: 'x', color: COLOR_X }); }
      if (s.axisY) { ys.push(ay); defs.push({ label: 'y', color: COLOR_Y }); }
      if (s.axisZ) { ys.push(az); defs.push({ label: 'z', color: COLOR_Z }); }
      if (s.showMagnitude) { ys.push(am); defs.push({ label: '|a|', color: COLOR_M }); }
      // Key includes the series signature so TimeChart fully re-mounts
      // when axis composition changes (uPlot can't change series count
      // after construction).
      if (ys.length) groups.push({
        key: 'linear-' + defs.map(d => d.label).join(''),
        label: 'Linear acceleration', unit: 'm/s²', ys, defs
      });
    }
    if (s.showRaw) {
      const ys: Float64Array[] = []; const defs: Array<{ label: string; color: string }> = [];
      if (s.axisX) { ys.push(axg); defs.push({ label: 'x', color: COLOR_X }); }
      if (s.axisY) { ys.push(ayg); defs.push({ label: 'y', color: COLOR_Y }); }
      if (s.axisZ) { ys.push(azg); defs.push({ label: 'z', color: COLOR_Z }); }
      if (ys.length) groups.push({
        key: 'raw-' + defs.map(d => d.label).join(''),
        label: 'Raw (with gravity)', unit: 'm/s²', ys, defs
      });
    }
    if (s.showGyro) {
      const ys: Float64Array[] = []; const defs: Array<{ label: string; color: string }> = [];
      if (s.axisX) { ys.push(gx); defs.push({ label: 'x', color: COLOR_X }); }
      if (s.axisY) { ys.push(gy); defs.push({ label: 'y', color: COLOR_Y }); }
      if (s.axisZ) { ys.push(gz); defs.push({ label: 'z', color: COLOR_Z }); }
      if (ys.length) groups.push({
        key: 'gyro-' + defs.map(d => d.label).join(''),
        label: 'Gyroscope', unit: 'deg/s', ys, defs
      });
    }
    if (s.showOrientation) {
      const ys: Float64Array[] = []; const defs: Array<{ label: string; color: string }> = [];
      if (s.axisX) { ys.push(ox); defs.push({ label: 'α', color: COLOR_X }); }
      if (s.axisY) { ys.push(oy); defs.push({ label: 'β', color: COLOR_Y }); }
      if (s.axisZ) { ys.push(oz); defs.push({ label: 'γ', color: COLOR_Z }); }
      if (ys.length) groups.push({
        key: 'orient-' + defs.map(d => d.label).join(''),
        label: 'Orientation', unit: 'deg', ys, defs
      });
    }
    return groups;
  });

  // Rebuild FFT / stats when settings change
  $effect(() => {
    void $settings.motion.fftSize;
    void $settings.motion.fftWindow;
    void $settings.motion.fftOverlapPct;
    if (running) ensureFft();
  });
  $effect(() => {
    void $settings.motion.rmsWindowSec;
    void $settings.motion.meanWindowSec;
    void $settings.motion.peakHoldDecayDbPerSec;
    statsX = makeStats(); statsY = makeStats();
    statsZ = makeStats(); statsM = makeStats();
    refreshKpiSnapshot();
  });
</script>

<div class="page">
  {#if permission === 'denied'}
    <div class="banner danger" role="alert">
      Motion permission denied. iOS: Settings → Safari → Motion &amp; Orientation Access → On, then reload.
    </div>
  {/if}
  {#if permission === 'unsupported'}
    <div class="banner danger" role="alert">DeviceMotion is not supported in this browser.</div>
  {/if}

  <div class="status-strip">
    <span class="dot" class:live={running}></span>
    <span class="subhead">
      {running ? 'Acquiring' : 'Idle'} · {hz} Hz · {count} samples
    </span>
  </div>

  <!-- KPI grid -- NOTE: no {#key} wrapper. Cards persist; only values update. -->
  <p class="section-header">Magnitude · |a|</p>
  <section class="kpi-grid">
    <KpiCard label="Peak"   value={kpiM.peak}  unit="m/s²" onReset={() => { statsM.peak.reset(); refreshKpiSnapshot(); }} big accent />
    <KpiCard label="RMS"    value={kpiM.rms}   unit="m/s²" onReset={() => { statsM.rms.reset(); refreshKpiSnapshot(); }} />
    <KpiCard label="Crest"  value={kpiM.crest}             onReset={resetKpi} />
    <KpiCard label="Kurt"   value={kpiM.kurt}              onReset={() => { statsM.kurt.reset(); refreshKpiSnapshot(); }} />
    <KpiCard label="Hold"   value={kpiM.hold}  unit="m/s²" onReset={() => { statsM.peakHold.reset(); refreshKpiSnapshot(); }} />
    <KpiCard label="Pk-Pk"  value={kpiM.pkpk}  unit="m/s²" onReset={() => { statsM.peak.reset(); refreshKpiSnapshot(); }} />
  </section>

  {#if $settings.motion.showLinear && ($settings.motion.axisX || $settings.motion.axisY || $settings.motion.axisZ)}
    <p class="section-header">Per-axis · linear acceleration</p>
    <section class="kpi-grid">
      {#if $settings.motion.axisX}
        <KpiCard label="X · Peak" value={kpiX.peak} unit="m/s²" onReset={() => { statsX.peak.reset(); refreshKpiSnapshot(); }} />
        <KpiCard label="X · RMS"  value={kpiX.rms}  unit="m/s²" onReset={() => { statsX.rms.reset(); refreshKpiSnapshot(); }} />
        <KpiCard label="X · Avg"  value={kpiX.mean} unit="m/s²" onReset={() => { statsX.mean.reset(); refreshKpiSnapshot(); }} />
      {/if}
      {#if $settings.motion.axisY}
        <KpiCard label="Y · Peak" value={kpiY.peak} unit="m/s²" onReset={() => { statsY.peak.reset(); refreshKpiSnapshot(); }} />
        <KpiCard label="Y · RMS"  value={kpiY.rms}  unit="m/s²" onReset={() => { statsY.rms.reset(); refreshKpiSnapshot(); }} />
        <KpiCard label="Y · Avg"  value={kpiY.mean} unit="m/s²" onReset={() => { statsY.mean.reset(); refreshKpiSnapshot(); }} />
      {/if}
      {#if $settings.motion.axisZ}
        <KpiCard label="Z · Peak" value={kpiZ.peak} unit="m/s²" onReset={() => { statsZ.peak.reset(); refreshKpiSnapshot(); }} />
        <KpiCard label="Z · RMS"  value={kpiZ.rms}  unit="m/s²" onReset={() => { statsZ.rms.reset(); refreshKpiSnapshot(); }} />
        <KpiCard label="Z · Avg"  value={kpiZ.mean} unit="m/s²" onReset={() => { statsZ.mean.reset(); refreshKpiSnapshot(); }} />
      {/if}
    </section>
  {/if}

  {#if channelGroups.length > 0}
    <p class="section-header">Time domain</p>
    <div class="card chart-card" style="padding: 0; margin: 0 16px">
      <div class="card-head">
        <span class="headline">Window</span>
        <span class="spacer"></span>
        <select
          value={$settings.motion.timeWindowSec}
          onchange={(e) => updateMotion({ timeWindowSec: +(e.currentTarget as HTMLSelectElement).value })}
        >
          {#each [1, 5, 10, 30, 60] as w}
            <option value={w}>{w}s</option>
          {/each}
        </select>
      </div>
    </div>
    {#each channelGroups as g (g.key)}
      <section class="card chart-card group-card">
        <div class="card-head">
          <span class="headline">{g.label}</span>
          <span class="spacer"></span>
          <span class="caption-1">{g.unit}</span>
        </div>
        <div class="chart-host">
          <TimeChart
            {xs} ys={g.ys} seriesDefs={g.defs}
            {count}
            windowSec={$settings.motion.timeWindowSec}
            yLabel={g.unit}
          />
        </div>
      </section>
    {/each}
  {:else}
    <p class="section-header">Time domain</p>
    <section class="card chart-card">
      <div class="chart-host">
        <div class="placeholder">All channels are hidden — enable at least one in Settings → Motion → channels.</div>
      </div>
    </section>
  {/if}

  <!-- FFT -->
  <p class="section-header">Frequency domain · |a|</p>
  <section class="card chart-card">
    <div class="card-head">
      <span class="headline">Spectrum</span>
      <span class="spacer"></span>
      <select
        value={$settings.motion.fftSize}
        onchange={(e) => updateMotion({ fftSize: +(e.currentTarget as HTMLSelectElement).value as typeof FFT_SIZES[number] })}
      >
        {#each FFT_SIZES.filter(s => s <= 4096) as s}<option value={s}>{s}</option>{/each}
      </select>
      <select
        value={$settings.motion.fftWindow}
        onchange={(e) => updateMotion({ fftWindow: (e.currentTarget as HTMLSelectElement).value as typeof WINDOW_NAMES[number] })}
      >
        {#each WINDOW_NAMES as w}<option value={w}>{w}</option>{/each}
      </select>
    </div>
    <div class="chart-host">
      {#if fft}
        {#key fft.size + ':' + fft.sampleRate}
          <FftChart
            freqs={fftFreqs}
            spectra={[$settings.motion.fftScaleLog ? fftDb : fftMag]}
            seriesDefs={[{ label: '|a|', color: COLOR_SPEC }]}
            logY={$settings.motion.fftScaleLog}
            logX={$settings.motion.fftFreqLog}
            autoScale={$settings.motion.fftAutoScale}
            yMin={$settings.motion.fftYMin}
            yMax={$settings.motion.fftYMax}
            yLabel={$settings.motion.fftScaleLog ? 'dB' : 'magnitude'}
          />
        {/key}
      {:else}
        <div class="placeholder">Tap Start to compute spectrum</div>
      {/if}
    </div>
  </section>

  <p class="section-header">Dominant frequencies · N = {$settings.motion.dominantFreqCount}</p>
  <section class="list-group" style="margin: 0 16px 16px">
    {#each dominant as d, i}
      <div class="list-row">
        <span class="dom-rank caption-1">#{i + 1}</span>
        <span class="list-row-label value-mono">{d.freq.toFixed(2)} <span class="footnote">Hz</span></span>
        <span class="footnote value-mono">{(20 * Math.log10(Math.max(d.mag, 1e-12))).toFixed(1)} dB</span>
      </div>
    {:else}
      <div class="list-row footnote">No signal yet</div>
    {/each}
  </section>

  {#if $settings.motion.showCube}
    <p class="section-header">3D orientation</p>
    <section class="card chart-card">
      <div class="cube-wrap">
        <OrientationCube
          alpha={orientation.alpha}
          beta={orientation.beta}
          gamma={orientation.gamma}
        />
      </div>
      <div class="list-row" style="border-top: 0.5px solid var(--separator)">
        <span class="caption-1" style="width: 56px; color: var(--fg-tertiary)">α / yaw</span>
        <span class="list-row-label value-mono">{orientation.alpha.toFixed(1)}<span class="footnote"> °</span></span>
      </div>
      <div class="list-row">
        <span class="caption-1" style="width: 56px; color: var(--fg-tertiary)">β / pitch</span>
        <span class="list-row-label value-mono">{orientation.beta.toFixed(1)}<span class="footnote"> °</span></span>
      </div>
      <div class="list-row">
        <span class="caption-1" style="width: 56px; color: var(--fg-tertiary)">γ / roll</span>
        <span class="list-row-label value-mono">{orientation.gamma.toFixed(1)}<span class="footnote"> °</span></span>
      </div>
    </section>
  {/if}
</div>

<FooterControls module="motion" {running} onStart={start} onStop={stop} onResetKpi={resetKpi} />

<style>
  .page {
    flex: 1;
    overflow-y: auto;
    padding: 8px 0 12px;
    background: var(--bg-grouped);
  }
  .banner {
    margin: 8px 16px;
    padding: 12px 16px;
    background: var(--bg-elev);
    border-radius: var(--r-card);
    font-size: var(--t-footnote);
    color: var(--danger);
  }
  .status-strip {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 0 16px 12px;
  }
  .section-header {
    font-size: var(--t-footnote);
    color: var(--fg-secondary);
    text-transform: uppercase;
    letter-spacing: 0.05em;
    margin: 16px 16px 6px 20px;
    font-weight: 400;
  }
  .kpi-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 8px;
    padding: 0 16px;
  }
  @media (min-width: 480px) {
    .kpi-grid { grid-template-columns: repeat(3, 1fr); }
  }
  .chart-card {
    margin: 0 16px;
    overflow: hidden;
  }
  .card-head {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 14px;
    border-bottom: 0.5px solid var(--separator);
  }
  .card-head select { padding: 6px 28px 6px 10px; min-height: 32px; font-size: var(--t-subhead); }
  .chart-host {
    height: 220px;
    padding: 8px;
  }
  .placeholder {
    display: flex; align-items: center; justify-content: center;
    height: 100%;
    color: var(--fg-tertiary);
    font-size: var(--t-footnote);
  }
  .dom-rank {
    width: 36px;
    color: var(--fg-tertiary);
  }
  .cube-wrap {
    padding: 8px;
  }
  .group-card {
    margin: 0 16px 8px;
  }
  .group-card:last-of-type {
    margin-bottom: 16px;
  }
</style>
