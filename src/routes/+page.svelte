<script>
  import { onMount } from 'svelte';
  import Papa from 'papaparse';

  const URL = '/dog_name_trends_nyc.csv';

  // status
  let errorMsg = '';

  // data
  let rows = [];
  let years = [];
  let names = [];
  let totals = new Map();       // year -> total count

  // charts
  let Plotly = null;
  let lineEl;
  let barsEl;

  // interaction
  let pick = [];
  let topN = 10;
  let barYear = null;    // controls podium + bars
  let timer = null;
  let cursorYear = null; // guideline on trend
  let sceneYear = null;  // scene-specific crown lock

  // toggles
  let showTotal = true;        // draw total licenses line
  let showEvents = true;       // draw event bands/pins

  const EVENTS = [
    { id:'covid',    type:'band', from:2020, to:2021, color:'rgba(0,0,0,.06)', emoji:'🦠', label:'Pandemic' },
    { id:'lockdown', type:'pin',  year:2020, color:'#999', emoji:'🏠', label:'Stay-at-home' },
    { id:'adopt',    type:'pin',  year:2020, color:'#999', emoji:'🐾', label:'Adoption spike' },
    { id:'reopen',   type:'pin',  year:2021, color:'#999', emoji:'🌳', label:'Parks reopen' },
  ];
  let activeEvents = new Set(EVENTS.map(e => e.id));

  // SCROLL STEPS (now dynamic): we specify *years*, not names
  // We’ll compute the top-3 names around these years from the data.
  let steps = []; // filled after data load

  // tiny fallback
  const SAMPLE = [
    { Year: 2014, Name: 'BELLA',  Count: 50 },
    { Year: 2014, Name: 'MAX',    Count: 45 },
    { Year: 2016, Name: 'BELLA',  Count: 90 },
    { Year: 2016, Name: 'MAX',    Count: 80 },
    { Year: 2019, Name: 'LUNA',   Count: 70 },
    { Year: 2020, Name: 'LUNA',   Count: 98 },
    { Year: 2020, Name: 'BELLA',  Count: 88 },
    { Year: 2022, Name: 'LUNA',   Count: 55 },
    { Year: 2023, Name: 'LUNA',   Count: 92 }
  ];

  // ---------- helpers ----------
  const BAD = new Set(['UNKNOWN','NAME NOT PROVIDED']);
  const tidy = (arr) =>
    arr.map(r => ({
      year: +((r.Year ?? r.year) || ''),
      name: (r.Name ?? r.name ?? '').toString().trim().toUpperCase(),
      count: +((r.Count ?? r.count) || '')
    })).filter(r =>
      Number.isFinite(r.year) && r.year > 0 &&
      r.name && !BAD.has(r.name) &&
      Number.isFinite(r.count) && r.count >= 0
    );

  function defaultPick(){
    const byName = new Map();
    for (const r of rows) byName.set(r.name, (byName.get(r.name)||0) + r.count);
    names = Array.from(byName.entries()).sort((a,b)=>b[1]-a[1]).map(d=>d[0]);
    pick = names.slice(0,5);
  }

  function colorFor(name){
    let h = 0;
    for (let i=0;i<name.length;i++) h = (h*31 + name.charCodeAt(i)) % 360;
    return `hsl(${h},70%,55%)`;
  }

  function seriesCounts(name){
    const map = new Map(rows.filter(r=>r.name===name).map(r=>[r.year,r.count]));
    return years.map(y => map.get(y) ?? 0);
  }

  function topKForYear(year, k=3){
    const pool = rows.filter(r=>r.year===year);
    const by = new Map();
    for (const r of pool) by.set(r.name, (by.get(r.name)||0) + r.count);
    return Array.from(by.entries()).sort((a,b)=>b[1]-a[1]).slice(0,k)
                .map(([name,count],i)=>({rank:i+1, name, count}));
  }

  function topNameForYear(year, withinPick=false){
    if (!year) return null;
    const pool = rows.filter(r=>r.year===year);
    const by = new Map();
    for (const r of pool){
      if (!withinPick || (withinPick && pick.includes(r.name))){
        by.set(r.name,(by.get(r.name)||0)+r.count);
      }
    }
    const best = Array.from(by.entries()).sort((a,b)=>b[1]-a[1])[0];
    return best ? {name: best[0], count: best[1]} : null;
  }

  // rolling-window top-k around a target year
  function topKAround(targetYear, k=3, window=1){
    if (!years.length) return [];
    const yr = nearestAvailableYear(targetYear);
    const lo = yr - window, hi = yr + window;
    const pool = rows.filter(r => r.year >= lo && r.year <= hi);
    const by = new Map();
    for (const r of pool) by.set(r.name, (by.get(r.name)||0) + r.count);
    return Array.from(by.entries()).sort((a,b)=>b[1]-a[1]).slice(0, k).map(d=>d[0]);
  }

  function nearestAvailableYear(target){
    if (target == null || !years.length) return null;
    let best = years[0], diff = Math.abs(years[0]-target);
    for (const y of years){
      const d = Math.abs(y - target);
      if (d < diff){ diff = d; best = y; }
    }
    return best;
  }

  // minimal sparkline path for podium tiles
  function sparkPath(values, w=140, h=54, pad=6){
    const vmax = Math.max(1, ...values);
    const xs = values.map((_,i)=> pad + (i*(w-2*pad))/(values.length-1 || 1));
    const ys = values.map(v => h - pad - (v/vmax)*(h-2*pad));
    let d = '';
    for (let i=0;i<xs.length;i++){
      d += (i===0? 'M':'L') + xs[i].toFixed(1) + ' ' + ys[i].toFixed(1) + ' ';
    }
    return d;
  }

  // biggest YoY gainer among picked names for a given year
  function biggestYoYGainer(year) {
    const idx = years.indexOf(year);
    if (idx < 1) return null;
    let best = null;
    for (const nm of pick) {
      const s = seriesCounts(nm);
      const delta = s[idx] - s[idx - 1];
      if (delta > 0 && (!best || delta > best.delta)) {
        best = { name: nm, delta, value: s[idx] };
      }
    }
    return best;
  }

  // ---------- trend (sticky) ----------
  function drawLine(){
    if (!rows.length || !lineEl || !Plotly) return;

    const traces = [];

    // TOTAL line
    if (showTotal && years.length && totals.size) {
      traces.push({
        x: years,
        y: years.map(y => totals.get(y) ?? 0),
        type: 'scatter',
        mode: 'lines',
        name: 'TOTAL LICENSES',
        line: { color: 'rgba(80,80,80,.25)', width: 3 },
        hovertemplate: 'Total: %{y} in %{x}<extra></extra>'
      });
    }

    // selected names
    for (const nm of pick){
      const yvals = seriesCounts(nm);
      const col = colorFor(nm);
      traces.push({
        x: years, y: yvals, type: 'scatter', mode: 'lines+markers',
        name: nm, line:{color:col}, marker:{color:col},
        hovertemplate:`${nm}: %{y} in %{x}<extra></extra>`
      });
    }

    // shapes & annotations
    const shapes = [];
    const ann = [];

    if (showEvents) {
      for (const ev of EVENTS) {
        if (!activeEvents.has(ev.id)) continue;
        if (ev.type === 'band') {
          shapes.push({ type:'rect', xref:'x', yref:'paper', x0:ev.from, x1:ev.to, y0:0, y1:1, fillcolor:ev.color, line:{width:0} });
          const mid = (ev.from + ev.to) / 2;
          ann.push({ x: mid, y: 1.02, xref:'x', yref:'paper', text: ev.emoji, showarrow:false, font:{size:16} });
        } else {
          shapes.push({ type:'line', xref:'x', yref:'paper', x0:ev.year, x1:ev.year, y0:0, y1:1, line:{ color: ev.color, width:1, dash:'dot' }});
          ann.push({ x: ev.year, y: 1.02, xref:'x', yref:'paper', text: ev.emoji, showarrow:false, font:{size:16} });
        }
      }
    }

    if (cursorYear!=null){
      shapes.push({ type:'line', xref:'x', yref:'paper', x0:cursorYear, x1:cursorYear, y0:0, y1:1, line:{dash:'dot', width:1} });
    }

    // crown & Δ arrow
    const crownYear = (cursorYear ?? sceneYear);
    if (crownYear!=null){
      const best = topNameForYear(crownYear,true) || topNameForYear(crownYear,false);
      if (best){
        const xi = years.indexOf(crownYear);
        const trace = traces.find(t=>t.name===best.name);
        const yv = trace?.y?.[xi] ?? null;
        if (yv != null) ann.push({ x:crownYear, y:yv, xref:'x', yref:'y', text:'👑', showarrow:false, font:{size:22} });
      }
      const g = biggestYoYGainer(crownYear);
      if (g) {
        const col = colorFor(g.name);
        ann.push({ x: crownYear, y: g.value, xref:'x', yref:'y', text:`⬆︎ +${g.delta.toLocaleString()}`, showarrow:true, arrowhead:2, ax:0, ay:-28, font:{size:12, color:col}, arrowcolor:col });
      }
    }

    Plotly.newPlot(lineEl, traces, {
      xaxis:{ title:'Year' },
      yaxis:{ title:'Licenses (count)', rangemode:'tozero' },
      shapes, annotations:ann, margin:{l:60,r:20,t:20,b:50}
    }, {displayModeBar:false});
  }

  // ---------- bars (Top-N by year) ----------
  function drawBars(){
    if (!rows.length || !barsEl || !Plotly || !barYear) return;
    const pool = rows.filter(r=>r.year===barYear);
    const by = new Map();
    for (const r of pool) by.set(r.name,(by.get(r.name)||0)+r.count);
    const top = Array.from(by.entries()).sort((a,b)=>b[1]-a[1]).slice(0, Math.max(1,+topN||10));
    Plotly.newPlot(barsEl, [{
      x: top.map(d=>d[1]).reverse(),
      y: top.map(d=>d[0]).reverse(),
      type:'bar', orientation:'h',
      marker:{ color: top.map(d=>colorFor(d[0])).reverse() },
      hovertemplate:'%{y}: %{x}<extra></extra>'
    }], { xaxis:{title:'Licenses (count)', rangemode:'tozero'}, margin:{l:140,r:20,t:20,b:40} }, {displayModeBar:false});
  }

  // ---------- UI handlers ----------
  function addName(e){
    if (e.key!=='Enter') return;
    const v = e.currentTarget.value.trim().toUpperCase();
    if (v && !pick.includes(v)) pick = [...pick, v];
    e.currentTarget.value = '';
    drawLine();
  }

  function playBars(){
    stopBars();
    const idx0 = Math.max(0, years.indexOf(barYear));
    let i = idx0;
    timer = setInterval(()=>{ barYear = years[i]; drawBars(); i=(i+1)%years.length; }, 900);
  }
  function stopBars(){ if (timer){ clearInterval(timer); timer=null; } }

  function inView(node, { onEnter }){
    const obs = new IntersectionObserver(([e])=>{ if (e.isIntersecting) onEnter?.(); }, {threshold:0.6});
    obs.observe(node);
    return { destroy(){ obs.disconnect(); } };
  }

  // ---------- load ----------
  async function fetchCsvText(){ const r = await fetch(URL); if(!r.ok) throw new Error(`HTTP ${r.status}`); return await r.text(); }
  function parseCsvText(text){
    return new Promise((res,rej)=> {
      Papa.parse(text, { header:true, skipEmptyLines:true, complete:o=>res(o.data), error:rej });
    });
  }

  function buildTotals() {
    totals = new Map();
    for (const y of years) {
      totals.set(
        y,
        rows.filter(r => r.year === y).reduce((s, r) => s + r.count, 0)
      );
    }
  }

  function buildDynamicSteps() {
    if (!years.length) { steps = []; return; }

    // choose representative years (data-driven fallbacks)
    const early = years[Math.min(2, Math.max(0, years.length - 1))]; // near start
    const y2019 = nearestAvailableYear(2019);
    const y2020 = nearestAvailableYear(2020);
    const y2023 = nearestAvailableYear(2023);

    steps = [
      { icon: '🐶', year: early,  pick: topKAround(early, 3, 1) },
      { icon: '🌙', year: y2019,  pick: topKAround(y2019, 3, 1) },
      { icon: '🩺', year: y2020,  pick: topKAround(y2020, 3, 1) },
      { icon: '🔁', year: y2023,  pick: topKAround(y2023, 3, 1) },
    ];
  }

  onMount(async ()=>{
    try{
      Plotly = (await import('plotly.js-dist-min')).default;
      const raw = tidy(await parseCsvText(await fetchCsvText()));
      rows = raw.length ? raw : tidy(SAMPLE);
    }catch(e){
      errorMsg = String(e?.message ?? e);
      rows = tidy(SAMPLE);
    }
    years = Array.from(new Set(rows.map(r=>r.year))).sort((a,b)=>a-b);
    buildTotals();
    barYear = years.at(-1);
    defaultPick();
    buildDynamicSteps();      // <— compute scrolly steps from the data
    await Promise.resolve();
    drawLine(); drawBars();
  });

  $: if (Plotly && rows.length && barYear) drawBars();

  // guideline control
  function setCursorLine(y){ cursorYear = y; drawLine(); }
</script>

<div class="container">
  <h1>Rise & Fall of Popular Dog Names (NYC)</h1>
  {#if errorMsg}
    <div class="card warn">Couldn’t load CSV: {errorMsg}. Using sample.</div>
  {/if}

  <!-- Sticky trend + scrolly steps (now data-driven picks) -->
  <section class="scrolly">
    <div class="sticky">
      <div class="card" bind:this={lineEl} style="height:520px;"></div>

      {#if years.length}
        <div class="card ctrl" style="display:flex;gap:12px;align-items:center;flex-wrap:wrap;">
          <div style="display:flex;gap:8px;align-items:center;">
            <label for="year">Highlight year</label>
            <input id="year" type="range" min={years[0]} max={years.at(-1)} step="1"
                   on:input={(e)=> setCursorLine(+e.currentTarget.value)} />
          </div>

          <label style="display:flex;gap:8px;align-items:center;">
            <input type="checkbox" bind:checked={showTotal} on:change={drawLine} />
            Show total licenses
          </label>
          <label style="display:flex;gap:8px;align-items:center;">
            <input type="checkbox" bind:checked={showEvents} on:change={drawLine} />
            Show context events
          </label>

          {#each EVENTS as ev}
            <button
              class="chip"
              aria-pressed={activeEvents.has(ev.id)}
              on:click={() => {
                if (activeEvents.has(ev.id)) activeEvents.delete(ev.id);
                else activeEvents.add(ev.id);
                activeEvents = new Set(activeEvents);
                drawLine();
              }}>
              {ev.emoji} {ev.label}
            </button>
          {/each}
        </div>
      {/if}
    </div>

    <div class="steps">
      {#each steps as s}
        <article class="step card"
          use:inView={{ onEnter: () => {
            pick = s.pick;
            sceneYear = s.year ?? null;
            cursorYear = s.year ?? null;
            drawLine();
          } }}>
          <div class="row">
            <div class="icon">{s.icon}</div>
            {#if s.year}<div class="yearBadge">{s.year}</div>{/if}
            <div class="chips">
              {#each s.pick as nm}
                <span class="chip" style={`--c:${colorFor(nm)}`}>{nm}</span>
              {/each}
            </div>
          </div>
        </article>
      {/each}
    </div>
  </section>

  <!-- YEAR LEADERBOARD (unchanged) -->
  <section class="card" style="margin-top:16px;">
    <h3 style="margin:0 0 6px;">Year Leaderboard</h3>

    <div class="flex">
      <div>
        <label for="yearSel">Year</label>
        <input id="yearSel" type="range"
               min={years[0] ?? 2000} max={years.at(-1) ?? 2024} step="1"
               bind:value={barYear} />
        <div class="mono">{barYear}</div>
      </div>
      <div style="display:flex;gap:8px;align-items:flex-end;">
        <button on:click={playBars} title="Play">▶</button>
        <button on:click={stopBars} title="Stop">⏸</button>
      </div>
    </div>

    {#if barYear}
      {#if topKForYear(barYear, 6).length}
        {#key barYear}
          <div class="podium">
            {#each topKForYear(barYear,3) as d (d.name)}
              <div class={`place r${d.rank}`} style={`--c:${colorFor(d.name)}`}>
                {#if d.rank === 1}<div class="trophy">👑</div>{/if}
                {#if years.length}
                  <svg class="spark" viewBox="0 0 140 54" preserveAspectRatio="none">
                    <path d={sparkPath(seriesCounts(d.name))} />
                  </svg>
                {/if}
                <div class="dog">🐶</div>
                <div class="rank">{d.rank === 1 ? '1st' : d.rank === 2 ? '2nd' : '3rd'}</div>
                <div class="pname">{d.name}</div>
                <div class="pcount">{d.count.toLocaleString()}</div>
              </div>
            {/each}
          </div>

          {#if topKForYear(barYear,6).length > 3}
            <div class="bench">
              {#each topKForYear(barYear,6).slice(3) as d (d.name)}
                <div class="benchItem">
                  <div class="cry">😢</div>
                  <div class="bname" style={`--c:${colorFor(d.name)}`}>{d.rank}. {d.name}</div>
                  <div class="bcount">{d.count.toLocaleString()}</div>
                </div>
              {/each}
            </div>
          {/if}
        {/key}
      {:else}
        <div class="subtle">No data for {barYear}.</div>
      {/if}
    {/if}
  </section>

  <!-- Top-N bars for same year -->
  <section class="card" style="margin-top:12px;">
    <div class="flex">
      <div>
        <label for="topNSel">Top N (bars)</label>
        <input id="topNSel" type="number" min="3" max="20" bind:value={topN} />
      </div>
    </div>
    <div class="card" bind:this={barsEl} style="height:520px;margin-top:10px;"></div>
  </section>

  <!-- Explore -->
  <section class="card" style="margin-top:12px;">
    <div class="flex">
      <div style="flex:1;">
        <label for="nameAdder">Find/add a name</label>
        <input id="nameAdder" placeholder="Type and press Enter…" on:keydown={addName} />
        <div class="subtle" style="margin-top:6px;">{pick.join(', ')}</div>
      </div>
    </div>
  </section>
</div>

<style>
  :global(:root){ --maxw: 1100px; --pad: 12px; font-family: Inter, system-ui, -apple-system, Segoe UI, Roboto, sans-serif; }
  :global(body){ margin:0; color:#111; background:#fff; }
  .container{ max-width:var(--maxw); margin:0 auto; padding:24px var(--pad); }
  h1{ font-size:1.6rem; margin:0 0 8px; }
  .card{ background:#fff; border:1px solid #eee; border-radius:12px; padding:16px; box-shadow:0 1px 2px rgba(0,0,0,.04); }
  .warn{ background:#fff5f5; border-color:#f0cccc; }
  input,button{ font-size:14px; }
  input{ min-height:36px; padding:6px 8px; width:100%; box-sizing:border-box; }
  button{ padding:8px 12px; border-radius:10px; border:1px solid #ddd; background:#fafafa; cursor:pointer; }
  button:hover{ background:#f0f0f0; }
  .mono{ font-family: ui-monospace, Menlo, Consolas, monospace; color:#444; margin-top:4px; }
  .subtle{ color:#555; font-size:.9rem; }

  .scrolly{ display:grid; gap:12px; }
  .sticky{ position:sticky; top:16px; }
  .steps{ display:grid; gap:18px; }
  .row{ display:flex; gap:8px; align-items:center; flex-wrap:wrap; }
  .icon{ font-size:20px; }
  .yearBadge{ background:#111; color:#fff; padding:2px 10px; border-radius:999px; font-size:12px; }
  .chips{ display:flex; gap:6px; flex-wrap:wrap; }
  .chip{ background:#f2f4f7; border:1px solid #e5e7eb; border-radius:999px; padding:2px 10px; font-size:12px; letter-spacing:.2px; color:var(--c); cursor:pointer; }
  .chip[aria-pressed="true"]{ background:#eef2ff; border-color:#c7d2fe; }

  .ctrl{ margin-top:10px; }

  .podium{ display:grid; grid-template-columns: repeat(3, 1fr); gap:14px; align-items:end; margin-top:10px; }
  .place{ position:relative; text-align:center; padding:16px 10px 12px; border-radius:14px; border:1px solid #eee; box-shadow:0 1px 2px rgba(0,0,0,.04); overflow:hidden; }
  .place::after{ content:''; position:absolute; inset:0; background: linear-gradient(180deg, color-mix(in oklab, var(--c) 12%, #fff) 0%, #fff 60%); opacity:.55; pointer-events:none; }
  .r1{ transform: translateY(-6px); }
  .r2{ transform: translateY(6px); }
  .r3{ transform: translateY(12px); }
  .trophy{ font-size:28px; line-height:1; position:absolute; top:-12px; left:50%; transform:translateX(-50%); z-index:2; }
  .spark{ position:absolute; inset:4px 4px auto 4px; height:54px; width:140px; opacity:.25; z-index:1; }
  .spark path{ fill:none; stroke:var(--c); stroke-width:2; }
  .dog{ font-size:46px; margin-top:6px; position:relative; z-index:2; }
  .rank{ font-weight:700; margin-top:6px; letter-spacing:.3px; position:relative; z-index:2; }
  .pname{ font-weight:700; margin-top:2px; position:relative; z-index:2; }
  .pcount{ color:#555; font-size:.9rem; margin-top:2px; position:relative; z-index:2; }

  .bench{ display:grid; grid-template-columns: repeat(auto-fill,minmax(220px,1fr)); gap:10px; margin-top:12px; }
  .benchItem{ display:flex; align-items:center; gap:10px; padding:10px; border:1px dashed #eee; border-radius:10px; }
  .cry{ font-size:22px; }
  .bname{ font-weight:600; color:var(--c); }
  .bcount{ margin-left:auto; color:#555; font-variant-numeric: tabular-nums; }

  .flex{ display:flex; gap:12px; align-items:flex-end; flex-wrap:wrap; }
</style>
