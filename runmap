<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>달릴러너</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/leaflet-draw@1.0.4/dist/leaflet.draw.css" />
  <script src="https://cdn.jsdelivr.net/npm/leaflet-draw@1.0.4/dist/leaflet.draw.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@turf/turf@6.5.0/turf.min.js"></script>
  <style>
    html, body { height: 100%; margin: 0; }
    #app { height: 100vh; display: grid; grid-template-columns: 360px 1fr; }
    @media (max-width: 1024px){ #app { grid-template-columns: 320px 1fr; } }
    @media (max-width: 768px){ #app { grid-template-columns: 1fr; grid-template-rows: auto 1fr; } }
    #map { height: 100%; width: 100%; }

    /* 녹색 톤 커스텀 컨트롤 */
    .map-controls { position:absolute; top:16px; left:16px; display:flex; flex-direction:column; gap:10px; z-index:1000; }
    .btn-square { width:44px; height:44px; background:#22c55e; color:#fff; font-size:20px; font-weight:bold; border-radius:8px; display:flex; align-items:center; justify-content:center; box-shadow:0 2px 6px rgba(0,0,0,.15); border:2px solid #16a34a; cursor:pointer; }
    .btn-square:hover { background:#16a34a; }
    .btn-group { display:flex; flex-direction:column; gap:10px; padding:8px; background:rgba(0,0,0,0.06); border-radius:10px; backdrop-filter:saturate(150%) blur(2px); }

    .leaflet-container { background:#f8fafc; }
    /* 선택 하이라이트 */
    .runline-selected { filter: drop-shadow(0 0 6px rgba(34,197,94,.9)); }
  </style>
</head>
<body class="bg-white">
  <div id="app">
    <!-- 좌측 패널 -->
    <aside class="bg-white border-r border-gray-200 p-6 flex flex-col gap-4">
      <header>
        <h1 class="text-4xl font-extrabold text-gray-900">달릴러너</h1>
        <p class="text-sm text-gray-600 mt-1">지도를 클릭해 러닝코스를 만들고, 거리(km)를 확인하며 저장하세요.</p>
      </header>
      <section class="grid grid-cols-2 gap-3">
        <div class="col-span-2 rounded-xl border border-green-200 bg-green-50 p-3">
          <div class="text-[11px] text-green-800 font-semibold">전체 거리</div>
          <div id="distanceTotal" class="text-2xl font-bold text-green-700">0.00 km</div>
        </div>
        <input id="courseName" class="col-span-2 rounded-xl border px-3 py-2 text-sm" placeholder="코스 이름" />
        <button id="saveBtn" class="rounded-xl bg-green-500 hover:bg-green-600 text-white py-2 text-sm font-semibold">코스 저장</button>
        <button id="clearBtn" class="rounded-xl bg-gray-200 hover:bg-gray-300 text-gray-800 py-2 text-sm font-semibold">지우기</button>
      </section>
      <section class="flex-1 overflow-auto">
        <h2 class="text-sm font-semibold mb-2">저장된 코스</h2>
        <ul id="savedList" class="space-y-2"></ul>
      </section>
    </aside>

    <!-- 우측 지도 -->
    <main class="relative">
      <div id="map"></div>

      <!-- 커스텀 컨트롤 -->
      <div class="map-controls">
        <div class="btn-group">
          <button id="zoomIn"  class="btn-square" aria-label="확대">＋</button>
          <button id="zoomOut" class="btn-square" aria-label="축소">－</button>
        </div>
        <div class="btn-group">
          <button id="drawBtn"  class="btn-square" title="그리기" aria-label="그리기">✏</button>
          <button id="editBtn"  class="btn-square" title="수정"  aria-label="수정">✎</button>
          <button id="delBtn"   class="btn-square" title="삭제"  aria-label="삭제">🗑</button>
        </div>
      </div>
    </main>
  </div>

  <script>
  (function(){
    'use strict';

    // 재실행 대비: 기존 인스턴스 정리(전역 충돌 방지)
    if (window.__RUNMAP__ && typeof window.__RUNMAP__.teardown === 'function') {
      try { window.__RUNMAP__.teardown(); } catch(_){}
    }

    // 지도 초기화
    const leafletMap = L.map('map', { zoomControl:false }).setView([37.5665, 126.9780], 13);
    L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
      attribution: '타일: © OpenStreetMap · © CARTO', subdomains:'abcd', maxZoom: 20
    }).addTo(leafletMap);
    leafletMap.whenReady(()=>{ try{ leafletMap.invalidateSize(); }catch(_){} });

    // 드로잉 세트업
    const drawn = new L.FeatureGroup();
    leafletMap.addLayer(drawn);
    const drawOpts = { shapeOptions:{ color:'#22c55e', weight:4 } };
    const drawHandler  = new L.Draw.Polyline(leafletMap, drawOpts);
    const editHandler  = new L.EditToolbar.Edit(leafletMap, { featureGroup: drawn });
    const delHandler   = new L.EditToolbar.Delete(leafletMap, { featureGroup: drawn });

    // 선택 관리 + 하이라이트
    let selectedLayer = null;
    function setSelected(layer, on){
      if (!layer) return;
      try { layer.setStyle({ weight: on ? 6 : 4 }); if (on) layer.bringToFront(); } catch(_){}
      const el = layer._path; if (el && el.classList){ if (on) el.classList.add('runline-selected'); else el.classList.remove('runline-selected'); }
      if (on) selectedLayer = layer; else if (selectedLayer === layer) selectedLayer = null;
    }
    function attachSelectable(layer){
      if (!layer || typeof layer.on !== 'function') return;
      layer.on('click', (e)=>{ if (selectedLayer && selectedLayer!==layer) setSelected(selectedLayer,false); setSelected(layer,true); L.DomEvent.stopPropagation(e); });
      layer.on('mouseover', ()=>{ const el = layer._path; if (el) el.style.cursor = 'pointer'; });
    }

    // 거리 계산/업데이트
    function kmOfLatLngs(latlngs){
      const coords = latlngs.map(ll=>[ll.lng, ll.lat]);
      if (coords.length < 2) return 0;
      const line = turf.lineString(coords);
      return turf.length(line, { units:'kilometers' });
    }
    function updateTotal(){
      let total = 0; drawn.eachLayer(layer => { total += kmOfLatLngs(layer.getLatLngs()); });
      document.getElementById('distanceTotal').textContent = `${total.toFixed(2)} km`;
    }

    leafletMap.on(L.Draw.Event.CREATED, (e) => { drawn.addLayer(e.layer); attachSelectable(e.layer); updateTotal(); });
    leafletMap.on(L.Draw.Event.EDITED, updateTotal);
    leafletMap.on(L.Draw.Event.DELETED, ()=>{ setSelected(selectedLayer,false); updateTotal(); });

    // 커스텀 컨트롤 동작
    document.getElementById('zoomIn').onclick  = () => leafletMap.zoomIn();
    document.getElementById('zoomOut').onclick = () => leafletMap.zoomOut();
    document.getElementById('drawBtn').onclick = () => { setSelected(selectedLayer,false); delHandler.disable(); editHandler.disable(); drawHandler.enable(); };
    document.getElementById('editBtn').onclick = () => { setSelected(selectedLayer,false); drawHandler.disable(); delHandler.disable(); editHandler.enable(); };
    document.getElementById('delBtn').onclick  = () => {
      if (selectedLayer) { try { drawn.removeLayer(selectedLayer); } catch(_){} selectedLayer=null; updateTotal(); }
      else { drawHandler.disable(); editHandler.disable(); delHandler.enable(); }
    };

    // 저장/불러오기 (마지막 선 저장)
    const STORAGE_KEY = 'runningCourses_simple';
    const nameInput = document.getElementById('courseName');
    const saveBtn   = document.getElementById('saveBtn');
    const clearBtn  = document.getElementById('clearBtn');
    const listEl    = document.getElementById('savedList');

    function serializeLast(){ let last=null; drawn.eachLayer(l=> last=l); if(!last) return null; const coords = last.getLatLngs().map(ll=>[ll.lng,ll.lat]); return { type:'Feature', geometry:{ type:'LineString', coordinates: coords } }; }
    function loadFeature(feat){ if(!feat||feat.geometry?.type!=='LineString') return; const latlngs=feat.geometry.coordinates.map(([lng,lat])=>L.latLng(lat,lng)); const layer=L.polyline(latlngs,{color:'#22c55e',weight:4}); drawn.addLayer(layer); attachSelectable(layer); leafletMap.fitBounds(layer.getBounds(),{padding:[24,24]}); updateTotal(); }
    function refreshList(){ listEl.innerHTML=''; const all=JSON.parse(localStorage.getItem(STORAGE_KEY)||'{}'); Object.entries(all).forEach(([name,feat])=>{ const li=document.createElement('li'); li.className='flex items-center justify-between gap-2 bg-green-50 border border-green-200 rounded-xl p-2'; let km='0.00'; try{ const latlngs=feat.geometry.coordinates.map(([lng,lat])=>L.latLng(lat,lng)); km=kmOfLatLngs(latlngs).toFixed(2);}catch(_){} li.innerHTML=`<div class='min-w-0'><div class='text-sm font-semibold text-green-700 truncate'>${name}</div><div class='text-xs text-gray-600'>${km} km</div></div>`; const wrap=document.createElement('div'); wrap.className='flex gap-1'; const b1=document.createElement('button'); b1.className='px-2 py-1 text-xs rounded-lg bg-green-500 text-white'; b1.textContent='불러오기'; b1.onclick=()=>loadFeature(feat); const b2=document.createElement('button'); b2.className='px-2 py-1 text-xs rounded-lg bg-gray-300'; b2.textContent='삭제'; b2.onclick=()=>{ const all2=JSON.parse(localStorage.getItem(STORAGE_KEY)||'{}'); delete all2[name]; localStorage.setItem(STORAGE_KEY, JSON.stringify(all2)); refreshList(); }; wrap.appendChild(b1); wrap.appendChild(b2); li.appendChild(wrap); listEl.appendChild(li); }); }

    saveBtn.onclick = () => {
      const name = (nameInput.value || '').trim();
      if (!name) return alert('코스 이름을 입력해주세요.');
      const feat = serializeLast();
      if (!feat) return alert('먼저 코스를 그려주세요.');
      const all = JSON.parse(localStorage.getItem(STORAGE_KEY) || '{}');
      all[name] = feat; 
      localStorage.setItem(STORAGE_KEY, JSON.stringify(all));
      nameInput.value = '';
      refreshList();
      alert('저장되었습니다!');

      // 저장 후 초기화(새 코스 바로 작성 가능)
      try { drawn.clearLayers(); } catch(_){}
      try { setSelected(selectedLayer,false); } catch(_){}
      selectedLayer = null;
      updateTotal();
      try { leafletMap.setView([37.5665, 126.9780], 13, { animate: true }); } catch(_){}
      setTimeout(()=>{ try{ leafletMap.invalidateSize(); } catch(_){} }, 0);
    };

    clearBtn.onclick = () => { drawn.clearLayers(); setSelected(selectedLayer,false); updateTotal(); };

    refreshList();

    // 재실행 대비 정리 핸들
    window.__RUNMAP__ = {
      teardown(){ try { leafletMap.remove(); } catch(_){} }
    };
  })();
  </script>
</body>
</html>
