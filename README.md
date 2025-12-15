<!-- /app/index.html -->
<!doctype html>
<html lang="no">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Livstracker – Dashboard + Synk</title>
  <style>
    :root { --bg:#0b0d10; --card:#11151a; --text:#e7e7e7; --muted:#9aa4af; --accent:#5aa7ff; --ok:#4ade80; --warn:#f59e0b; --bad:#ef4444; }
    * { box-sizing: border-box; }
    body { margin:0; font:14px/1.4 system-ui,-apple-system,Segoe UI,Roboto,Inter,sans-serif; background:var(--bg); color:var(--text); }
    header { padding:16px 20px; display:flex; gap:12px; align-items:center; border-bottom:1px solid #1d232c; position:sticky; top:0; background:rgba(11,13,16,.9); backdrop-filter: blur(8px); }
    header h1 { font-size:16px; margin:0; }
    .grow { flex:1; }
    .btn { padding:8px 12px; border:1px solid #263040; background:#151922; color:var(--text); border-radius:10px; cursor:pointer; }
    .btn.primary { background:var(--accent); border-color:transparent; color:#05070a; }
    .btn.ghost { background:transparent; border-color:#263040;}
    .btn:disabled { opacity:.6; cursor:not-allowed; }
    main { padding:20px; display:grid; gap:16px; grid-template-columns: repeat(12, 1fr); }
    .card { background:var(--card); border:1px solid #1d232c; border-radius:16px; padding:16px; }
    .kpis { display:grid; gap:12px; grid-template-columns: repeat(4, 1fr); }
    .kpi h3 { margin:0 0 8px; font-weight:600; color:var(--muted); font-size:12px; text-transform:uppercase; letter-spacing:.6px; }
    .kpi .val { font-size:28px; font-weight:700; }
    .muted { color:var(--muted); }
    .row { display:flex; gap:12px; align-items:center; flex-wrap:wrap; }
    .grid-6 { grid-column: span 6; }
    .grid-12 { grid-column: span 12; }
    .select, input[type="checkbox"] { background:#0f131a; border:1px solid #263040; color:var(--text); padding:8px 10px; border-radius:10px; }
    .legend { display:flex; flex-wrap:wrap; gap:8px; }
    .legend .chip { padding:6px 10px; border-radius:999px; border:1px solid #2a3342; cursor:pointer; }
    .heatmap { display:grid; grid-template-columns: repeat(12, 1fr); gap:6px; }
    .heatcell { aspect-ratio:1.2; border-radius:6px; background:#0f131a; border:1px solid #1f2733; }
    .heatcell[data-lvl="1"]{ background:#0d1a26; }
    .heatcell[data-lvl="2"]{ background:#0b253a; }
    .heatcell[data-lvl="3"]{ background:#08314f; }
    .heatcell[data-lvl="4"]{ background:#054065; }
    .flex { display:flex; align-items:center; gap:8px; flex-wrap:wrap; }
    .spacer { flex:1; }
    .small { font-size:12px; }
    @media (max-width: 960px) {
      .kpis { grid-template-columns: repeat(2, 1fr); }
      main { grid-template-columns: repeat(6, 1fr); }
      .grid-6 { grid-column: span 6; }
    }
  </style>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js" defer></script>
  <script src="https://www.gstatic.com/firebasejs/10.13.1/firebase-app-compat.js" defer></script>
  <script src="https://www.gstatic.com/firebasejs/10.13.1/firebase-auth-compat.js" defer></script>
  <script src="https://www.gstatic.com/firebasejs/10.13.1/firebase-firestore-compat.js" defer></script>
</head>
<body>
  <header>
    <h1>Livstracker – Oversikt + Synk</h1>
    <div class="grow"></div>
    <button id="btnImport" class="btn">Importer JSON</button>
    <button id="btnExport" class="btn ghost">Eksporter JSON</button>
    <span id="syncStatus" class="muted small">Ikke pålogget</span>
    <button id="btnLogin" class="btn">Logg inn</button>
    <button id="btnLogout" class="btn ghost" style="display:none">Logg ut</button>
  </header>

  <main>
    <section class="card grid-12">
      <div class="kpis">
        <div class="kpi"><h3>Dager ført (7/30/365)</h3><div class="val" id="kpiDays">0 / 0 / 0</div></div>
        <div class="kpi"><h3>Streak</h3><div class="val" id="kpiStreak">0</div></div>
        <div class="kpi"><h3>I dag – % fullført</h3><div class="val" id="kpiTodayPct">0%</div></div>
        <div class="kpi"><h3>Antall felt</h3><div class="val" id="kpiFields">0</div></div>
      </div>
    </section>

    <section class="card grid-12">
      <div class="row">
        <label>Periode:
          <select id="period" class="select">
            <option value="7d">Uke</option>
            <option value="30d">Måned</option>
            <option value="365d">År</option>
            <option value="all">Alt</option>
          </select>
        </label>
        <label>Gruppering:
          <select id="grouping" class="select">
            <option value="day">Dag</option>
            <option value="week">Uke</option>
            <option value="month">Måned</option>
          </select>
        </label>
        <label>Aggregasjon:
          <select id="agg" class="select">
            <option value="avg">Snitt</option>
            <option value="sum">Sum</option>
          </select>
        </label>
        <label>Glatting:
          <select id="smooth" class="select">
            <option value="none">Av</option>
            <option value="ma7">7d</option>
            <option value="ma28">28d</option>
          </select>
        </label>
        <label>Akse-mapping:
          <select id="axisMap" class="select">
            <option value="auto">Auto (score→100, tid→10)</option>
            <option value="manual">Manuell</option>
          </select>
        </label>
        <div class="spacer"></div>
        <label class="row" style="gap:6px;align-items:center;">
          <input type="checkbox" id="showGoals" /> <span class="small muted">Vis mål-linjer</span>
        </label>
        <div class="spacer"></div>
        <label>Velg felt:
          <select id="fieldSelect" class="select" multiple size="1" style="min-width:220px"></select>
        </label>
      </div>
    </section>

    <section class="card grid-12">
      <canvas id="chart" height="120"></canvas>
      <div class="legend" id="legend"></div>
      <div class="small muted">Venstre akse: 0–100 (score/%) · Høyre akse: 0–10 (timer/min).</div>
    </section>

    <section class="card grid-6">
      <h3 style="margin-top:0">Heatmap (12 uker)</h3>
      <div id="heatmap" class="heatmap"></div>
    </section>

    <section class="card grid-6">
      <h3 style="margin-top:0">Trender</h3>
      <div class="flex small">
        <div>7d endring: <span id="trend7">0</span></div>
        <div>· 28d endring: <span id="trend28">0</span></div>
      </div>
      <div class="muted small" id="trendHint">Velg ett felt for presise trendtall.</div>
    </section>
  </main>

  <input id="fileInput" type="file" accept="application/json" style="display:none" />

  <script>
    // === Firebase config (lim inn dine nøkler) ===
    const FIREBASE_CONFIG = {
      // Fyll inn fra Firebase Console:
      // apiKey: "XXX",
      // authDomain: "XXX.firebaseapp.com",
      // projectId: "XXX",
      // appId: "1:XXXX:web:XXXX"
    };

    // --- Dato helpers ---
    const todayISO = () => new Date().toISOString().slice(0,10);
    const iso = d => new Date(d).toISOString().slice(0,10);
    const addDays = (d, n) => { const x = new Date(d); x.setDate(x.getDate()+n); return x; };

    // --- IndexedDB ---
    const DB_NAME = 'livstracker'; const DB_VERSION = 1; let db;
    function idbOpen(){return new Promise((res,rej)=>{const req=indexedDB.open(DB_NAME,DB_VERSION);
      req.onupgradeneeded=()=>{const d=req.result;
        if(!d.objectStoreNames.contains('meta')) d.createObjectStore('meta');
        if(!d.objectStoreNames.contains('fields')) d.createObjectStore('fields',{keyPath:'id'});
        if(!d.objectStoreNames.contains('entries')) d.createObjectStore('entries',{keyPath:'date'});};
      req.onsuccess=()=>{db=req.result;res();}; req.onerror=()=>rej(req.error);});}
    function idbGetAll(store){return new Promise((res,rej)=>{const tx=db.transaction(store,'readonly');const r=tx.objectStore(store).getAll();r.onsuccess=()=>res(r.result||[]);r.onerror=()=>rej(r.error);});}
    function idbSet(store,value){return new Promise((res,rej)=>{const tx=db.transaction(store,'readwrite');tx.objectStore(store).put(value);tx.oncomplete=()=>res();tx.onerror=()=>rej(tx.error);});}

    // --- State ---
    let state = { fields:[], entries:{}, uid:null, axisMapMode:'auto', selectedFields:[] };

    // --- Firebase ---
    let fb = { enabled:false, app:null, auth:null, db:null };
    function tryInitFirebase(){
      try {
        if (!FIREBASE_CONFIG || !FIREBASE_CONFIG.apiKey) return;
        fb.app = firebase.initializeApp(FIREBASE_CONFIG);
        fb.auth = firebase.auth();
        fb.db = firebase.firestore();
        fb.enabled = true;
      } catch(e){ console.warn('Firebase init feilet', e); fb.enabled=false; }
    }
    async function signIn(){
      if(!fb.enabled) return alert('Firebase ikke konfigurert.');
      try {
        const provider = new firebase.auth.GoogleAuthProvider();
        const res = await fb.auth.signInWithPopup(provider).catch(()=>fb.auth.signInAnonymously());
        return res.user;
      } catch(e){ alert('Innlogging feilet: '+e.message); }
    }
    function signOut(){ if(fb.enabled && fb.auth.currentUser) fb.auth.signOut(); }

    // --- Firestore synk ---
    let unsubFields=null, unsubEntries=null;
    async function startSync(uid){
      if(!fb.enabled) return;
      unsubFields = fb.db.collection('users').doc(uid).collection('fields')
        .onSnapshot(async snap=>{
          const byId=new Map();
          snap.forEach(d=>{const v=d.data(); byId.set(v.id, v);});
          state.fields = [...byId.values()].sort((a,b)=>a.name.localeCompare(b.name));
          for(const f of state.fields) await idbSet('fields', f);
          renderAll();
        });
      unsubEntries = fb.db.collection('users').doc(uid).collection('entries')
        .onSnapshot(async snap=>{
          const all={};
          snap.forEach(d=>{const v=d.data(); all[v.date]=v.values||{};});
          state.entries = all;
          for(const [date,values] of Object.entries(all)) await idbSet('entries',{date,values,updatedAt:Date.now()});
          renderAll();
        });
    }
    async function pushField(f){
      await idbSet('fields', f);
      if(fb.enabled && state.uid){
        await fb.db.collection('users').doc(state.uid).collection('fields').doc(f.id).set(f,{merge:true});
      }
    }
    async function pushEntry(date, values){
      const doc={date,values,updatedAt:Date.now()};
      await idbSet('entries', doc);
      if(fb.enabled && state.uid){
        await fb.db.collection('users').doc(state.uid).collection('entries').doc(date).set(doc,{merge:true});
      }
    }

    // --- Import/Eksport ---
    function exportJSON(){
      const blob=new Blob([JSON.stringify({fields:state.fields,entries:state.entries},null,2)],{type:'application/json'});
      const url=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download='livstracker-export.json'; a.click(); URL.revokeObjectURL(url);
    }
    async function importJSON(file){
      const text=await file.text(); const data=JSON.parse(text);
      // Dedupe: siste vinner
      if(Array.isArray(data.fields)){
        const byId=new Map(state.fields.map(f=>[f.id,f]));
        data.fields.forEach(f=>byId.set(f.id,{...f,updatedAt:Date.now()}));
        state.fields=[...byId.values()];
        for(const f of state.fields) await pushField(f);
      }
      if(data.entries && typeof data.entries==='object'){
        for(const [d,vals] of Object.entries(data.entries)) await pushEntry(d, vals);
      }
      renderAll(); alert('Import fullført');
    }

    // --- Utils / aggregering ---
    function dateRange(period){ const now=new Date(); if(period==='all') return null; const map={ '7d':7,'30d':30,'365d':365 }; return addDays(now,-map[period]); }
    function groupKey(d,mode){ const x=new Date(d);
      if(mode==='week'){const day=(x.getUTCDay()+6)%7; const monday=new Date(Date.UTC(x.getUTCFullYear(),x.getUTCMonth(),x.getUTCDate()-day)); return monday.toISOString().slice(0,10);}
      if(mode==='month') return `${x.getUTCFullYear()}-${String(x.getUTCMonth()+1).padStart(2,'0')}-01`;
      return iso(x);
    }
    function movingAvg(arr,win){ const out=[]; let sum=0,q=[]; for(const v of arr){ q.push(v??0); sum+=v??0; if(q.length>win) sum-=q.shift(); out.push(q.length===win?sum/win:null);} return out; }
    function pickAxis(field){
      const mode = state.axisMapMode;
      if(mode==='manual' && field.axisHint) return field.axisHint;
      const n=(field.name||'').toLowerCase();
      if(n.includes('score')||n.includes('%')) return 'left';
      if(n.includes('time')||n.includes('timer')||n.includes('min')) return 'right';
      if(typeof field.goal==='number' && field.goal>10) return 'left';
      return 'right';
    }

    // --- Chart.js ---
    let chart;
    function buildDatasets(dates, fields, matrix, smoothMode, showGoals){
      const colors=['#5aa7ff','#7dd3fc','#a78bfa','#f472b6','#fca5a5','#f59e0b','#34d399','#4ade80','#22d3ee','#60a5fa','#93c5fd'];
      const ds=[]; const seen=new Set(); // hindrer duplikater
      fields.forEach((f,idx)=>{
        if(seen.has(f.id)) return; seen.add(f.id);
        const raw = dates.map(d => (matrix[d] && typeof matrix[d][f.id] !== 'undefined') ? Number(matrix[d][f.id]) : null);
        const yAxisID = pickAxis(f)==='left' ? 'y' : 'y1';
        const data = smoothMode==='ma7' ? movingAvg(raw,7) : smoothMode==='ma28' ? movingAvg(raw,28) : raw;
        ds.push({ label:f.name, data, borderColor:colors[idx%colors.length], backgroundColor:colors[idx%colors.length]+'33', pointRadius:0, tension:0.25, yAxisID });
        if(showGoals && typeof f.goal==='number'){
          ds.push({ label:f.name+' mål', data:dates.map(()=>f.goal), borderColor:colors[idx%colors.length], borderDash:[6,6], pointRadius:0, tension:0, yAxisID });
        }
      });
      return ds;
    }
    function renderChart(){
      const period=document.getElementById('period').value;
      const grouping=document.getElementById('grouping').value;
      const agg=document.getElementById('agg').value;
      const smooth=document.getElementById('smooth').value;
      const showGoals=document.getElementById('showGoals').checked;

      const since=dateRange(period);
      const datesAll=Object.keys(state.entries).sort();
      const dates=datesAll.filter(d=>!since || new Date(d)>=since);

      const groups={};
      for(const d of dates){
        const g=groupKey(d,grouping); groups[g]??={};
        const vals=state.entries[d]||{};
        for(const f of state.fields){
          const v=vals[f.id]; if(typeof v==='undefined') continue;
          groups[g][f.id]??=[]; groups[g][f.id].push(Number(v));
        }
      }
      const labels=Object.keys(groups).sort();
      const matrix={};
      for(const g of labels){
        matrix[g]={};
        for(const f of state.fields){
          const arr=groups[g][f.id]||[];
          matrix[g][f.id]=arr.length?(agg==='sum'?arr.reduce((a,b)=>a+b,0):arr.reduce((a,b)=>a+b,0)/arr.length):null;
        }
      }
      const fieldIds = state.selectedFields.length ? new Set(state.selectedFields) : new Set(state.fields.map(f=>f.id));
      const fields = state.fields.filter(f=>fieldIds.has(f.id) && f.type==='number');

      const ctx=document.getElementById('chart').getContext('2d');
      const datasets=buildDatasets(labels, fields, matrix, smooth, showGoals);

      if(chart) chart.destroy();
      chart=new Chart(ctx,{ type:'line', data:{labels,datasets},
        options:{
          responsive:true, animation:false, interaction:{mode:'nearest',intersect:false},
          plugins:{ legend:{display:true,position:'bottom'}, tooltip:{enabled:true} },
          scales:{
            y:{type:'linear',position:'left',min:0,max:100,grid:{color:'#1e2633'},ticks:{color:'#9aa4af'}},
            y1:{type:'linear',position:'right',min:0,max:10, grid:{drawOnChartArea:false},ticks:{color:'#9aa4af'}},
            x:{grid:{color:'#1e2633'},ticks:{color:'#9aa4af'}}
          }
        }
      });

      const legend=document.getElementById('legend'); legend.innerHTML='';
      fields.forEach((f)=>{
        const el=document.createElement('div'); el.className='chip'; el.textContent=f.name;
        el.onclick=()=>{
          const i=chart.data.datasets.findIndex(d=>d.label===f.name);
          const gi=chart.data.datasets.findIndex(d=>d.label===f.name+' mål');
          if(i>=0) chart.toggleDataVisibility(i);
          if(gi>=0) chart.toggleDataVisibility(gi);
          chart.update();
        };
        legend.appendChild(el);
      });
    }

    // --- Heatmap / KPI / Trender ---
    function renderHeatmap(){
      const host=document.getElementById('heatmap'); host.innerHTML='';
      const end=new Date(); end.setUTCHours(0,0,0,0);
      const start=addDays(end,-7*12); const days=[]; for(let d=new Date(start); d<=end; d=addDays(d,1)) days.push(iso(d));
      for(let i=0;i<12;i++){ const col=document.createElement('div'); col.style.display='grid'; col.style.gap='6px';
        const week=days.slice(i*7,i*7+7);
        week.forEach(day=>{ const cell=document.createElement('div'); cell.className='heatcell';
          const vals=state.entries[day]||{}; const count=Object.keys(vals).length;
          const lvl=count>=6?4:count>=4?3:count>=2?2:count>=1?1:0;
          cell.dataset.lvl=String(lvl); cell.title=`${day}: ${count} felt`; col.appendChild(cell);});
        host.appendChild(col);
      }
    }
    function renderKPIs(){
      const dates=Object.keys(state.entries).sort(); const now=new Date();
      const counts=(n)=>{const since=addDays(now,-n); return dates.filter(d=>new Date(d)>=since && Object.keys(state.entries[d]).length>0).length;};
      let streak=0; for(let d=new Date(now);;d=addDays(d,-1)){const s=iso(d); if(state.entries[s] && Object.keys(state.entries[s]).length>0) streak++; else break;}
      const today=todayISO(); const todayVals=state.entries[today]||{};
      const maxCount=state.fields.filter(f=>f.includeInProgress && f.type!=='checkbox').length;
      const doneCount=Object.keys(todayVals).filter(fid=>todayVals[fid]!==null && typeof todayVals[fid]!=='undefined').length;
      const pct=maxCount?Math.round(doneCount/maxCount*100):0;
      document.getElementById('kpiDays').textContent=`${counts(7)} / ${counts(30)} / ${counts(365)}`;
      document.getElementById('kpiStreak').textContent=String(streak);
      document.getElementById('kpiTodayPct').textContent=pct+'%';
      document.getElementById('kpiFields').textContent=String(state.fields.length);
    }
    function renderTrends(){
      const sel=state.selectedFields; const target=sel.length===1?state.fields.find(f=>f.id===sel[0]):null;
      const t7=document.getElementById('trend7'); const t28=document.getElementById('trend28'); const hint=document.getElementById('trendHint');
      if(!target){ t7.textContent='0'; t28.textContent='0'; hint.style.display='block'; return; }
      hint.style.display='none';
      const dates=Object.keys(state.entries).sort();
      const values=dates.map(d=>state.entries[d]?.[target.id]).filter(v=>typeof v!=='undefined').map(Number);
      if(!values.length){ t7.textContent='0'; t28.textContent='0'; return; }
      const last=values.at(-1); const avg=a=>a.length?(a.reduce((x,y)=>x+y,0)/a.length):0;
      const avg7=avg(values.slice(-7)); const avg28=avg(values.slice(-28));
      const f=x=>(x>=0?'+':'')+x.toFixed(2); t7.textContent=f(last-avg7); t28.textContent=f(last-avg28);
    }

    function renderFieldSelect(){
      const sel=document.getElementById('fieldSelect'); sel.innerHTML='';
      const byId=new Map(); state.fields.forEach(f=>byId.set(f.id,f)); // dedupe
      const nums=[...byId.values()].filter(f=>f.type==='number');
      nums.forEach(f=>{ const opt=document.createElement('option'); opt.value=f.id; opt.textContent=f.name; sel.appendChild(opt); });
      for(const id of state.selectedFields){ const o=[...sel.options].find(o=>o.value===id); if(o) o.selected=true; }
      sel.onchange=()=>{ state.selectedFields=[...sel.selectedOptions].map(o=>o.value); renderAll(); };
    }

    function renderAll(){ renderFieldSelect(); renderChart(); renderHeatmap(); renderKPIs(); renderTrends(); }

    async function boot(){
      await idbOpen();
      // Load local + dedupe
      const fieldsArr=await idbGetAll('fields'); const byId=new Map(); fieldsArr.forEach(f=>byId.set(f.id,f)); state.fields=[...byId.values()];
      const entriesArr=await idbGetAll('entries'); state.entries=Object.fromEntries(entriesArr.map(x=>[x.date,x.values]));
      tryInitFirebase();

      document.getElementById('btnExport').onclick=exportJSON;
      document.getElementById('btnImport').onclick=()=>document.getElementById('fileInput').click();
      document.getElementById('fileInput').onchange=(e)=>{const f=e.target.files?.[0]; if(f) importJSON(f);};

      ['period','grouping','agg','smooth'].forEach(id=>document.getElementById(id).onchange=()=>renderAll());
      document.getElementById('axisMap').onchange=(e)=>{ state.axisMapMode=e.target.value; renderAll(); };
      document.getElementById('showGoals').onchange=()=>renderAll();

      const syncStatus=document.getElementById('syncStatus'); const btnLogin=document.getElementById('btnLogin'); const btnLogout=document.getElementById('btnLogout');
      if(fb.enabled){
        btnLogin.style.display='inline-block';
        firebase.auth().onAuthStateChanged(async (user)=>{
          if(user){ state.uid=user.uid; syncStatus.textContent='Synk på: '+(user.displayName||'konto'); btnLogin.style.display='none'; btnLogout.style.display='inline-block'; await startSync(user.uid); }
          else { state.uid=null; syncStatus.textContent='Ikke pålogget'; btnLogin.style.display='inline-block'; btnLogout.style.display='none'; if(unsubFields)unsubFields(); if(unsubEntries)unsubEntries(); }
        });
        btnLogin.onclick=signIn; btnLogout.onclick=signOut;
      } else {
        syncStatus.textContent='Kun lokal lagring (legg inn Firebase-nøkler for synk)';
        btnLogin.style.display='inline-block';
        btnLogin.onclick=()=>alert('Legg inn Firebase-nøkler i FIREBASE_CONFIG først.');
      }

      renderAll();
    }

    async function ensureDemo(){
      if(state.fields.length) return;
      const demoFields=[
        {id:'sleep_score',name:'Søvnscore %',type:'number',goal:80,includeInProgress:true,axisHint:'left',updatedAt:Date.now()},
        {id:'sleep_hours',name:'Søvn timer', type:'number',goal:8, includeInProgress:true, axisHint:'right',updatedAt:Date.now()}
      ];
      for(const f of demoFields) await pushField(f);
      const base=todayISO();
      for(let i=0;i<21;i++){const d=iso(addDays(base,-i));
        await pushEntry(d,{sleep_score:Math.round(80+(Math.random()*20-10)),sleep_hours:Math.max(4,Math.min(10,Math.round(7+(Math.random()*4-2))))});
      }
    }

    boot().then(ensureDemo);
  </script>

  <!-- Firestore sikkerhetsregler (kopiér til Firebase Console > Firestore Rules)
  rules_version = '2';
  service cloud.firestore {
    match /databases/{db}/documents {
      match /users/{uid}/{col}/{doc} {
        allow read, write: if request.auth != null && request.auth.uid == uid;
      }
    }
  } -->
</body>
</html>
