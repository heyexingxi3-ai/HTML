# HTML

<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>岡山県 交通シミュレーター（単一ファイル）</title>
<style>body{font-family:system-ui, -apple-system, "Noto Sans JP", Arial;margin:0}header{padding:12px;border-bottom:1px solid #eee}.wrap{display:flex;gap:12px;height:calc(100vh - 64px)}.side{width:340px;padding:12px;box-sizing:border-box;overflow:auto;border-right:1px solid #eee}.mapwrap{flex:1;display:flex;flex-direction:column}#map{flex:1}label{display:block;margin-top:8px}select,input,button{padding:8px;width:100%;box-sizing:border-box;border-radius:8px;border:1px solid #ccc}.results{padding:8px;border-top:1px solid #eee;background:#fafafa}.small{font-size:13px;color:#444}.legend{display:flex;gap:8px;flex-wrap:wrap;margin-top:8px}.chip{padding:4px 8px;border-radius:999px;background:#fff;border:1px solid #eee;font-size:13px}</style>
</head>
<body>
<header><h2>岡山県 交通シミュレーター（単一ファイル）</h2><div class="small">外部依存を最小化。OSRM が使えない場合は直線近似で表示します。</div></header>
<div class="wrap">
  <div class="side">
    <label>出発地<select id="start"></select></label>
    <label>到着地<select id="goal"></select></label>
    <label><button id="run">ルート計算</button></label>
    <div class="legend"><div class="chip">徒歩</div><div class="chip">自転車</div><div class="chip">車</div><div class="chip">バス</div><div class="chip">電車</div></div>
    <div class="results"><div id="status" class="small">準備中</div>
      <table id="res" style="width:100%;border-collapse:collapse;margin-top:8px"><thead><tr><th style="text-align:left">手段</th><th>距離(km)</th><th>時間(分)</th><th>費用(円)</th><th>CO₂(kg)</th></tr></thead><tbody></tbody></table>
      <div class="small" style="margin-top:8px">注: 電車は駅間直線近似＋徒歩区間。数値は概算。</div></div>
  </div>
  <div class="mapwrap"><div id="map" style="height:100%"></div></div>
</div>

<script>
const PLACE_COORDS = {"岡山市": [34.6618, 133.935], "倉敷市": [34.585, 133.771], "津山市": [35.0694, 134.0046], "玉野市": [34.4939, 133.9459], "笠岡市": [34.5072, 133.5079], "井原市": [34.5966, 133.4637], "総社市": [34.6757, 133.7363], "高梁市": [34.7886, 133.6177], "新見市": [34.9832, 133.4683], "備前市": [34.7532, 134.1952], "瀬戸内市": [34.649, 134.0892], "赤磐市": [34.8282, 134.0868], "真庭市": [35.0734, 133.7099], "美作市": [35.0081, 134.142], "浅口市": [34.5161, 133.5804], "和気町": [34.807, 134.1597], "早島町": [34.602, 133.833], "里庄町": [34.5083, 133.5515], "矢掛町": [34.6221, 133.5943], "新庄村": [35.1995, 133.553], "鏡野町": [35.065, 133.9954], "勝央町": [35.0533, 134.1291], "奈義町": [35.1796, 134.1786], "西粟倉村": [35.172, 134.3514], "久米南町": [34.8916, 133.9334], "美咲町": [34.9734, 133.9978], "吉備中央町": [34.8702, 133.7193]};
const STATIONS = {"岡山駅": [34.6663694, 133.9185889], "倉敷駅": [34.6020694, 133.765675], "津山駅": [35.05475, 134.0030139], "宇野駅": [34.4945028, 133.9538778], "笠岡駅": [34.505, 133.5048333], "井原駅": [34.5929139, 133.4689528], "総社駅": [34.673525, 133.7381], "備中高梁駅": [34.7896111, 133.6175833], "新見駅": [34.9768889, 133.4706111], "伊部駅": [34.7445833, 134.1769722], "邑久駅": [34.6763333, 134.0731111], "熊山駅": [34.7966111, 134.1083056], "中国勝山駅": [35.0793611, 133.6973333], "林野駅": [35.0136596, 134.1509076], "鴨方駅": [34.526936, 133.587068], "和気駅": [34.7972762, 134.1526035], "早島駅": [34.6022444, 133.8332694], "里庄駅": [34.5078472, 133.5509361], "矢掛駅": [34.6292556, 133.5898889], "院庄駅": [35.0590472, 133.9573556], "勝間田駅": [35.0358889, 134.1189833], "あわくら温泉駅": [35.18889, 134.33694], "誕生寺駅": [34.9514667, 133.95945], "弓削駅": [34.9255083, 133.9578389], "足守駅": [34.6982028, 133.8029278]};
function hav(lat1,lon1,lat2,lon2){ const R=6371; const toR=d=>d*Math.PI/180; const dlat=toR(lat2-lat1); const dlon=toR(lon2-lon1); const a=Math.sin(dlat/2)**2 + Math.cos(toR(lat1))*Math.cos(toR(lat2))*Math.sin(dlon/2)**2; return 2*R*Math.asin(Math.sqrt(a)); }

const startSel=document.getElementById('start'), goalSel=document.getElementById('goal');
Object.keys(PLACE_COORDS).forEach(k=>{ startSel.appendChild(new Option(k,k)); goalSel.appendChild(new Option(k,k)); });
startSel.value='岡山市'; goalSel.value='倉敷市';

let map=null, Llib=false;
function loadLeaflet(){ const link=document.createElement('link'); link.rel='stylesheet'; link.href='https://unpkg.com/leaflet@1.9.4/dist/leaflet.css'; document.head.appendChild(link); const s=document.createElement('script'); s.src='https://unpkg.com/leaflet@1.9.4/dist/leaflet.js'; s.onload=()=>{ Llib=true; initMap(); }; document.body.appendChild(s); }
function initMap(){ if(!Llib) return; map=L.map('map').setView([34.7,133.9],9); L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{maxZoom:19, attribution:'© OSM'}).addTo(map); }
loadLeaflet();

function clearMap(){ if(!Llib || !map) return; map.eachLayer(l=>{ if(l.options && l.options.pane!=='tilePane') map.removeLayer(l); }); }

function addLine(coords,color,opts){ if(Llib && map){ return L.polyline(coords,{color:color,weight:4,opacity:0.8,dashArray:opts&&opts.dash?'6 6':null}).addTo(map); } else { return null; } }

async function tryOSRM(mode,a,b){
  const profile = mode==='walking'?'walking': mode==='cycling'?'cycling':'driving';
  const url = `https://router.project-osrm.org/route/v1/${profile}/${a[1]},${a[0]};${b[1]},${b[0]}?overview=full&geometries=geojson`;
  try{ const res = await fetch(url); if(!res.ok) throw ''; const js = await res.json(); if(!js.routes||!js.routes[0]) throw ''; const r = js.routes[0]; return {distance_km:r.distance/1000, duration_min:r.duration/60, coords:r.geometry.coordinates.map(c=>[c[1],c[0]])}; }catch(e){ return null; }
}

function drawTable(rows){ const tbody=document.querySelector('#res tbody'); tbody.innerHTML=''; rows.forEach(r=>{ const tr=document.createElement('tr'); tr.innerHTML=`<td style="text-align:left">${r.label}</td><td>${r.distance.toFixed(1)}</td><td>${Math.round(r.time)}</td><td>${Math.round(r.cost)}</td><td>${r.co2.toFixed(2)}</td>`; tbody.appendChild(tr); }); }

document.getElementById('run').addEventListener('click', async ()=>{
  const s=startSel.value, g=goalSel.value; if(s===g){ alert('出発と到着が同じです'); return; }
  document.getElementById('status').textContent='計算中... (OSRM試行→失敗時は直線近似)'; clearMap();
  if(Llib && map){ L.marker(PLACE_COORDS[s]).addTo(map).bindPopup('出発:'+s); L.marker(PLACE_COORDS[g]).addTo(map).bindPopup('到着:'+g); map.fitBounds([PLACE_COORDS[s], PLACE_COORDS[g]],{padding:[40,40]}); }

  const modes=[{key:'walking',label:'徒歩',color:'#1e9b8a',speed:5,co2:0,costPerKm:0},{key:'cycling',label:'自転車',color:'#3b6ea8',speed:15,co2:0.005,costPerKm:0},{key:'driving',label:'自動車',color:'#d04f4f',speed:40,co2:0.20,costPerKm:20}];
  const results=[];
  for(const m of modes){
    const os = await tryOSRM(m.key, PLACE_COORDS[s], PLACE_COORDS[g]);
    if(os){ addLine(os.coords, m.color, {}); results.push({label:m.label, distance:os.distance_km, time:os.duration_min, cost:os.distance_km*(m.costPerKm||0), co2:os.distance_km*(m.co2||0)}); }
    else{ const straight=hav(PLACE_COORDS[s][0],PLACE_COORDS[s][1], PLACE_COORDS[g][0],PLACE_COORDS[g][1]); const road=straight*1.18; const time = (road/m.speed)*60 * (m.key==='driving'?1.1:1); addLine([[PLACE_COORDS[s][0],PLACE_COORDS[s][1]],[PLACE_COORDS[g][0],PLACE_COORDS[g][1]]], m.color, {dash:true}); results.push({label:m.label, distance:road, time:time, cost:road*(m.costPerKm||0), co2:road*(m.co2||0)}); }
  }
  const driving = results.find(r=>r.label==='自動車'); const busKm = driving.distance; const busTime = busKm/25*60*1.05; const busCost = busKm*30+100; const busCo2 = busKm*0.08; addLine([[PLACE_COORDS[s][0],PLACE_COORDS[s][1]],[PLACE_COORDS[g][0],PLACE_COORDS[g][1]]],'#ff8800',{dash:true}); results.push({label:'バス',distance:busKm,time:busTime,cost:busCost,co2:busCo2});
  function nearestStation(coord){ let best=null,bd=1e9; for(const k in STATIONS){ const s=STATIONS[k]; const d=hav(coord[0],coord[1], s[0],s[1]); if(d<bd){bd=d;best={name:k,lat:s[0],lon:s[1],dist:d};} } return best; }
  const sSt = nearestStation(PLACE_COORDS[s]); const gSt = nearestStation(PLACE_COORDS[g]);
  const sw = await tryOSRM('walking', PLACE_COORDS[s], [sSt.lat,sSt.lon]); const gw = await tryOSRM('walking', [gSt.lat,gSt.lon], PLACE_COORDS[g]);
  if(sw){ addLine(sw.coords,'#222',{}); } else { addLine([[PLACE_COORDS[s][0],PLACE_COORDS[s][1]],[sSt.lat,sSt.lon]],'#222',{dash:true}); }
  if(gw){ addLine(gw.coords,'#222',{}); } else { addLine([[PLACE_COORDS[g][0],PLACE_COORDS[g][1]],[gSt.lat,gSt.lon]],'#222',{dash:true}); }
  addLine([[sSt.lat,sSt.lon],[gSt.lat,gSt.lon]],'#000',{});
  const stStraight = hav(sSt.lat,sSt.lon,gSt.lat,gSt.lon); const railKm = stStraight*1.20; const trainKm = (sw?sw.distance_km: hav(PLACE_COORDS[s][0],PLACE_COORDS[s][1], sSt.lat, sSt.lon)) + railKm + (gw?gw.distance_km: hav(PLACE_COORDS[g][0],PLACE_COORDS[g][1], gSt.lat, gSt.lon)); const trainTime = (railKm/60)*60 + (sw?sw.duration_min: (hav(PLACE_COORDS[s][0],PLACE_COORDS[s][1], sSt.lat, sSt.lon)/5*60)) + (gw?gw.duration_min: (hav(PLACE_COORDS[g][0],PLACE_COORDS[g][1], gSt.lat, gSt.lon)/5*60)); const trainCost = railKm*30 + 120; const trainCo2 = railKm*0.02; results.push({label:'電車',distance:trainKm,time:trainTime,cost:trainCost,co2:trainCo2});
  drawTable(results); document.getElementById('status').textContent='完了（概算）';
});
</script>
</body>
</html>
