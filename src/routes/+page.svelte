<script>
  import { onMount } from 'svelte';
  import Papa from 'papaparse';

  // CSVs
  const URL = '/dog_name_trends_nyc.csv';   // Year, Name, Count  (for trends)
  const ZIP_URL = '/dog_names_cleaned.csv'; // raw licenses with ZipCode + Year (for geography)

  // status
  let errorMsg = '';

  // -------------- base data (names + yearly totals) --------------
  let rows = [];          // {year, name, count}
  let years = [];
  let names = [];
  let totals = new Map(); // year -> total count

  // charts
  let Plotly = null;
  let lineEl;
  let barsEl;

  // interaction (names)
  let pick = [];
  let topN = 10;
  let barYear = null;    // controls podium + bars
  let timer = null;
  let cursorYear = null; // guideline on trend

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

  // ---------- helpers (names CSV) ----------
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
    pick = names.slice(0,4);  // always show top 4 overall
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

  // tiny sparkline path (used in leaderboard cards)
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

  // ---------- TREND (line) ----------
  function drawLine(){
    if (!rows.length || !lineEl || !Plotly) return;

    const traces = [];

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

    for (const nm of pick){
      const yvals = seriesCounts(nm);
      const col = colorFor(nm);
      traces.push({
        x: years, y: yvals, type: 'scatter', mode: 'lines+markers',
        name: nm, line:{color:col}, marker:{color:col},
        hovertemplate:`${nm}: %{y} in %{x}<extra></extra>`
      });
    }

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

    if (cursorYear != null){
      shapes.push({ type:'line', xref:'x', yref:'paper', x0:cursorYear, x1:cursorYear, y0:0, y1:1, line:{dash:'dot', width:1} });
    }

    const crownYear = (cursorYear ?? barYear);
    if (crownYear != null){
      const best = topNameForYear(crownYear, false);
      if (best){
        const xi = years.indexOf(crownYear);
        let yv = null;
        const winnerTrace = traces.find(t => t.name === best.name);
        if (winnerTrace?.y && xi >= 0) yv = winnerTrace.y[xi];
        if (yv == null) {
          const m = new Map(rows.filter(r => r.name === best.name).map(r => [r.year, r.count]));
          yv = m.get(crownYear) ?? best.count;
        }
        if (yv != null) ann.push({ x:crownYear, y:yv, xref:'x', yref:'y', text:'👑', showarrow:false, font:{size:22} });
      }
      const g = biggestYoYGainer(crownYear);
      if (g) {
        const col = colorFor(g.name);
        ann.push({ x: crownYear, y: g.value, xref:'x', yref:'y',
          text:`⬆︎ +${g.delta.toLocaleString()}`, showarrow:true, arrowhead:2, ax:0, ay:-28,
          font:{size:12, color:col}, arrowcolor:col });
      }
    }

    Plotly.newPlot(lineEl, traces, {
      xaxis:{ title:'Year' },
      yaxis:{ title:'Licenses (count)', rangemode:'tozero' },
      shapes, annotations:ann, margin:{l:60,r:20,t:20,b:50}
    }, {displayModeBar:false});
  }

  // ---------- BARS (Top-N by year) ----------
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

  // ---------- ZIP / NEIGHBORHOOD SECTION (TREEMAP ONLY) ----------
  const BORO_COLORS = {
    'Manhattan': '#84cc16',
    'Brooklyn':  '#60a5fa',
    'Queens':    '#f472b6',
    'Bronx':     '#f59e0b',
    'Staten Island': '#34d399',
    'Other': '#a3a3a3'
  };

  // cleaned license rows for geography
  let zipRows = []; // {year:number, zip:number}
  let zipTotalsAll = new Map();          // zip -> total count
  let zipTotalsByYear = new Map();       // year -> Map(zip -> count)
  let zipTreeEl;                         // container for treemap

  // UI state for ZIPs
  let zipTopN = 10;       // top ZIPs per borough to show inside treemap
  let zipYear = null;     // null = overall; otherwise specific year
  let zipOverall = true;  // toggle overall vs per-year
  let zipYears = [];      // distinct years present in zipRows

  function boroughForZip(z){
    const p = Math.floor(z / 100);
    if (p === 112) return 'Brooklyn';
    if (p === 104) return 'Bronx';
    if (p === 103) return 'Staten Island';
    if (p === 111 || p === 113 || p === 114 || p === 116) return 'Queens';
    if (p === 100 || p === 101 || p === 102) return 'Manhattan';
    return 'Other';
  }

  // robust parser for ZIP CSV (auto-detects columns + parses dates)
  function tidyZip(arr){
    const out = [];
    for (const r of arr){
      const lower = Object.fromEntries(Object.entries(r).map(([k,v])=>[k.toLowerCase(), v]));
      // ZIP candidate
      let zipStr = '';
      for (const k of Object.keys(lower)){
        if (k.includes('zip') || k.includes('post code') || k.includes('postcode')){
          const m = String(lower[k] ?? '').match(/\d{5}/);
          if (m){ zipStr = m[0]; break; }
        }
      }
      const zip = +zipStr;
      if (!Number.isFinite(zip)) continue;

      // YEAR
      let year = null;
      for (const key of ['year','extract year','extract_year']){
        if (lower[key] != null){
          const m = String(lower[key]).match(/\d{4}/); if (m){ year = +m[0]; break; }
        }
      }
      if (year == null){
        for (const [k,val] of Object.entries(lower)){
          if (k.includes('date') || k.includes('issued') || k.includes('create')){
            const m = String(val ?? '').match(/\d{4}/);
            if (m){ const y = +m[0]; if (y>=1990 && y<=2100){ year = y; break; } }
            const d = new Date(val);
            if (!isNaN(d.getTime())){ const y = d.getFullYear(); if (y>=1990 && y<=2100){ year = y; break; } }
          }
        }
      }
      if (!Number.isFinite(year)) continue;
      if (zip < 10000 || zip > 11699) continue; // plausible NYC

      out.push({ year, zip });
    }
    return out;
  }

  function buildZipAgg(){
    zipTotalsAll = new Map();
    zipTotalsByYear = new Map();

    for (const r of zipRows){
      // overall zip
      zipTotalsAll.set(r.zip, (zipTotalsAll.get(r.zip) || 0) + 1);
      // by year -> zip
      const m = zipTotalsByYear.get(r.year) || new Map();
      m.set(r.zip, (m.get(r.zip) || 0) + 1);
      zipTotalsByYear.set(r.year, m);
    }
  }

  function getZipCountsFor(yearOrNull){
    if (yearOrNull == null) return zipTotalsAll;
    return zipTotalsByYear.get(yearOrNull) || new Map();
  }

  // ---------- load ----------
  async function fetchCsvText(url){
    const r = await fetch(url);
    if(!r.ok) throw new Error(`HTTP ${r.status} for ${url}`);
    return await r.text();
  }
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

  onMount(async ()=>{
    try{
      Plotly = (await import('plotly.js-dist-min')).default;

      // names/time-series CSV
      const raw1 = tidy(await parseCsvText(await fetchCsvText(URL)));
      rows = raw1;
      if (!rows.length) errorMsg = (errorMsg ? errorMsg + ' | ' : '') + 'Trends CSV loaded but had no rows after cleaning.';

      years = Array.from(new Set(rows.map(r=>r.year))).sort((a,b)=>a-b);
      buildTotals();
      if (years.length){
        barYear = years.at(-1);
        defaultPick();
        drawLine();
        drawBars();
      }

      // ZIP/geo CSV (raw licenses)
      const raw2 = tidyZip(await parseCsvText(await fetchCsvText(ZIP_URL)));
      zipRows = raw2.filter(r => r.zip >= 10000 && r.zip <= 11699);
      if (!zipRows.length){
        errorMsg = (errorMsg ? errorMsg + ' | ' : '') + 'ZIP CSV parsed 0 usable rows (couldn’t find ZIP + Year).';
      }

      buildZipAgg();
      const setYears = Array.from(new Set(zipRows.map(d=>d.year))).sort((a,b)=>a-b);
      zipYears = setYears;
      zipYear = zipYears.at(-1) ?? null;

      // initial treemap
      drawZipTreemap();

    }catch(e){
      errorMsg = (errorMsg ? errorMsg + ' | ' : '') + `Failed to initialize: ${String(e?.message ?? e)}`;
    }
  });

  // redraws
  $: if (Plotly && rows.length && barYear) drawBars();
  // keep trend in sync with the Year Leaderboard selection
  $: if (Plotly && rows.length && barYear != null) drawLine();

  // IMPORTANT: make treemap react to control changes
  $: {
    // reference control state so Svelte tracks these as dependencies
    void zipOverall; void zipYear; void zipTopN;
    if (Plotly && zipTreeEl) drawZipTreemap();
  }

  // guideline control for trend line
  function setCursorLine(y){ cursorYear = y; drawLine(); }

  // autoplay for leaderboard — now updates both bars and line
  function playBars(){
    stopBars();
    if (!years.length) return;
    let i = Math.max(0, years.indexOf(barYear));
    timer = setInterval(()=>{
      barYear = years[i];
      drawBars();
      drawLine();
      i = (i + 1) % years.length;
    }, 900);
  }
  function stopBars(){ if (timer){ clearInterval(timer); timer=null; } }

  // Treemap
  function drawZipTreemap(){
    if (!Plotly || !zipTreeEl) return;
    const counts = getZipCountsFor(zipOverall ? null : zipYear);
    if (!counts.size){ Plotly.purge(zipTreeEl); return; }

    const byBoro = new Map(); // boro -> [ [zip,count], ... ]
    for (const [zip, c] of counts){
      const b = boroughForZip(+zip);
      if (!byBoro.has(b)) byBoro.set(b, []);
      byBoro.get(b).push([zip, c]);
    }
    for (const [b, arr] of byBoro){
      arr.sort((a,b)=>b[1]-a[1]);
      byBoro.set(b, arr.slice(0, Math.max(1, +zipTopN || 10)));
    }

    const labels = ['NYC'];
    const parents = [''];
    const values = [null];
    const markerColors = ['#ddd'];

    for (const [boro, arr] of byBoro){
      const sum = arr.reduce((s,[,c])=>s+c,0);
      labels.push(boro); parents.push('NYC'); values.push(sum);
      markerColors.push(BORO_COLORS[boro] || '#aaa');

      for (const [zip,cnt] of arr){
        labels.push(zip.toString());
        parents.push(boro);
        values.push(cnt);
        markerColors.push(BORO_COLORS[boro] || '#aaa');
      }
    }

    Plotly.newPlot(zipTreeEl, [{
      type: 'treemap',
      labels, parents, values,
      marker: { colors: markerColors },
      branchvalues: 'total',
      hovertemplate: '%{label}: %{value} dogs<extra></extra>'
    }], {
      title: zipOverall ? 'Where are the dogs? (NYC → Borough → ZIP)' :
                          `Where are the dogs in ${zipYear}? (NYC → Borough → ZIP)`,
      margin:{l:10,r:10,t:40,b:10}
    }, {displayModeBar:false});
  }
</script>

<div class="container">
  <h1>Dogs population in NYC</h1>
  {#if errorMsg}
    <div class="card warn">{errorMsg}</div>
  {/if}

  <!-- Trend line + controls -->
  <section>
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
  </section>

  <!-- ZIP / Neighborhoods — TREEMAP ONLY -->
  <section class="card" style="margin-top:16px;">
    <h3 style="margin:0 0 8px;">Where are the dogs? (ZIPs & Boroughs)</h3>

    <div class="flex" style="align-items:center;">
      <label style="display:flex;gap:8px;align-items:center;">
        <input type="checkbox" bind:checked={zipOverall} on:change={drawZipTreemap} />
        Overall<br/>(all years)
      </label>

      <div style="display:flex;gap:8px;align-items:center;">
        <label for="zipYear">Year</label>
        <input id="zipYear" type="range"
               min={(zipYears[0] ?? 2014)} max={(zipYears.at(-1) ?? 2024)} step="1"
               bind:value={zipYear}
               disabled={zipOverall}
               on:input={drawZipTreemap} />
        <div class="mono">{zipOverall ? 'All' : zipYear}</div>
      </div>

      <div style="display:flex;gap:8px;align-items:center;margin-left:auto;">
        <label for="zipTop">Top ZIPs<br/>per borough</label>
        <input id="zipTop" type="number" min="3" max="20" bind:value={zipTopN} on:input={drawZipTreemap} />
      </div>
    </div>

    <div class="card" bind:this={zipTreeEl} style="height:480px;margin-top:10px;"></div>

    <div class="legendBoro">
      {#each Object.entries(BORO_COLORS) as [b,c]}
        <span class="dot" style="--c:{c}"></span>{b}
      {/each}
    </div>
  </section>

  <!-- Year leaderboard (names) -->
  <section class="card" style="margin-top:16px;">
    <h3 style="margin:0 0 6px;">Year Leaderboard</h3>

    <div class="flex">
      <div>
        <label for="yearSel">Year</label>
        <input id="yearSel" type="range"
               min={years[0] ?? 2000} max={years.at(-1) ?? 2024} step="1"
               bind:value={barYear}
               on:input={() => { drawBars(); drawLine(); }} />
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

  <!-- Top-N bars for same year (names) -->
  <section class="card" style="margin-top:12px;">
    <div class="flex">
      <div>
        <label for="topNSel">Top N (bars)</label>
        <input id="topNSel" type="number" min="3" max="20" bind:value={topN} />
      </div>
    </div>
    <div class="card" bind:this={barsEl} style="height:520px;margin-top:10px;"></div>
  </section>
</div>

<style>
  :global(:root){ --maxw: 1100px; --pad: 12px; font-family: Inter, system-ui, -apple-system, Segoe UI, Roboto, sans-serif; 
  }
  :global(body){ 
    margin:0;
     color:#111; 
     background:#fff; 
    }
  .container{ 
    max-width:var(--maxw); 
    margin:0 auto; 
    padding:24px var(--pad); 
  }
  h1{ 
    font-size:1.6rem; 
    margin:0 0 8px;
   }
  .card{ 
    background:#fff; 
    border:1px solid #eee;
    border-radius:12px; 
    padding:16px; 
    box-shadow:0 1px 2px rgba(0,0,0,.04); 
  }
  .warn{ 
    background:#fff5f5; 
    border-color:#f0cccc; 
  }
  input,button{ 
    font-size:14px; 
  }
  input{ 
    min-height:36px; 
    padding:6px 8px; 
    width:100%; 
    box-sizing:border-box; }
  button{ 
    padding:8px 12px;
    border-radius:10px; 
    border:1px solid #ddd; 
    background:#fafafa; 
    cursor:pointer; 
  }
  button:hover{ 
    background:#f0f0f0; 
  }
  .mono{ font-family: 
    ui-monospace, Menlo, Consolas, monospace; 
    color:#444; 
    margin-top:4px; 
  }
  .subtle{ 
    color:#555; 
    font-size:.9rem; 
  }
  .ctrl{ margin-top:10px; }
  .chip{ 
    background:#f2f4f7; 
    border:1px solid #e5e7eb; 
    border-radius:999px; 
    padding:2px 10px; 
    font-size:12px; 
    letter-spacing:.2px; 
    color:var(--c); 
    cursor:pointer; 
  }
  .chip[aria-pressed="true"]{ 
    background:#eef2ff; 
    border-color:#c7d2fe; }

  .legendBoro{ 
    display:flex; 
    gap:14px; 
    align-items:center; 
    color:#444; 
    margin-top:10px; 
    flex-wrap:wrap; 
  }
  .dot{ width:10px; 
    height:10px; 
    border-radius:999px; 
    background:var(--c); 
    display:inline-block;
     margin-right:6px; }

  /* Podium (names) */
  .podium{ 
    display:grid; 
    grid-template-columns: repeat(3, 1fr); 
    gap:14px; align-items:end;
     margin-top:10px; 
    }
  .place{ 
    position:relative; 
    text-align:center; 
    padding:16px 10px 12px; 
    border-radius:14px;
    border:1px solid #eee; 
    box-shadow:0 1px 2px rgba(0,0,0,.04); 
    overflow:hidden; 
  }
  .place::after{ 
    content:''; 
    position:absolute; 
    inset:0; background: linear-gradient(180deg, color-mix(in oklab, var(--c) 12%, #fff) 0%, #fff 60%); 
    opacity:.55; 
    pointer-events:none; 
  }
  .r1{ transform: translateY(-6px); }
  .r2{ transform: translateY(6px); }
  .r3{ transform: translateY(12px); }
  .trophy{ 
    font-size:28px; 
    line-height:1; 
    position:absolute; 
    top:-12px; left:50%; 
    transform:translateX(-50%); 
    z-index:2; 
  }
  .spark{ position:absolute; 
    inset:4px 4px auto 4px; 
    height:54px; 
    width:140px; 
    opacity:.25; 
    z-index:1; 
  }
  .spark path{ 
    fill:none; stroke:var(--c); 
    stroke-width:2;
   }
  .dog{ 
    font-size:46px; 
    margin-top:6px; 
    position:relative; 
    z-index:2; 
  }
  .rank{ font-weight:700;
     margin-top:6px; 
     letter-spacing:.3px; 
     position:relative;
      z-index:2; }
  .pname{ 
    font-weight:700; 
    margin-top:2px; 
    position:relative; 
    z-index:2; 
  }
  .pcount{ 
    color:#555; 
    font-size:.9rem; 
    margin-top:2px; 
    position:relative; 
    z-index:2; 
  }

  .bench{ 
    display:grid; 
    grid-template-columns: repeat(auto-fill,minmax(220px,1fr));
    gap:10px; 
    margin-top:12px; 
    }
  .benchItem{ 
    display:flex; 
    align-items:center; 
    gap:10px; 
    padding:10px; 
    border:1px dashed #eee; 
    border-radius:10px; 
  }
  .cry{ font-size:22px; }
  .bname{ 
    font-weight:600;
     color:var(--c);
    }
  .bcount{ 
    margin-left:auto; 
    color:#555; 
    font-variant-numeric: tabular-nums; }

  .flex{ 
    display:flex;
    gap:12px; 
    align-items:flex-end; 
    flex-wrap:wrap; 
    }
</style>
