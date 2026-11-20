<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>МОТОЛЕНД — стантишь, пока не купил новый байк</title>
<style>
  * { margin:0; padding:0; box-sizing:border-box; user-select:none; -webkit-user-select:none; touch-action:manipulation; }
  html,body { width:100%; height:100%; overflow:hidden; background:#111; font-family:'Segoe UI', Arial, sans-serif; }
  canvas { display:block; width:100vw; height:100vh; }
  .hud { position:fixed; top:12px; left:14px; color:#fff; text-shadow:0 2px 4px rgba(0,0,0,.6); pointer-events:none; z-index:5; }
  .hud .money { font-size:26px; font-weight:800; color:#ffd24a; }
  .hud .bike { font-size:14px; opacity:.85; margin-top:2px; }
  .hud .combo { font-size:15px; color:#7dffb0; height:20px; font-weight:700; }
  .toast { position:fixed; right:20px; top:90px; color:#ffe98a; font-weight:700; font-size:20px; pointer-events:none; opacity:0; transition:all .5s; z-index:6; text-shadow:0 2px 6px #000; }
  .btn { position:fixed; bottom:18px; left:50%; transform:translateX(-50%); z-index:7; }
  .btn button {
    background:linear-gradient(180deg,#ffcf3d,#f5a623); border:none; border-radius:12px;
    padding:14px 26px; font-size:17px; font-weight:800; color:#3a2600; cursor:pointer;
    box-shadow:0 4px 0 #a5680a, 0 6px 14px rgba(0,0,0,.45);
  }
  .btn button:active { transform:translateY(3px); box-shadow:0 1px 0 #a5680a; }
  .panel {
    position:fixed; inset:0; background:rgba(10,12,18,.88); backdrop-filter:blur(4px);
    display:none; align-items:center; justify-content:center; z-index:10;
  }
  .panel.open { display:flex; }
  .shop { background:#1b2030; border:1px solid #2e3752; border-radius:16px; padding:24px; width:min(480px,94vw); max-height:88vh; overflow:auto; color:#e8ecf5; }
  .shop h2 { margin-bottom:4px; letter-spacing:.5px; }
  .shop .sub { color:#8b94ad; font-size:13px; margin-bottom:14px; }
  .bike-row {
    display:flex; align-items:center; gap:12px; padding:12px; border-radius:12px;
    margin-bottom:8px; background:#232a3e; border:1px solid transparent;
  }
  .bike-row.owned { border-color:#3fae6a; }
  .bike-row.active { border-color:#f5a623; background:#2a3148; }
  .bike-row .info { flex:1; }
  .bike-row .name { font-weight:800; font-size:16px; }
  .bike-row .stats { font-size:12px; color:#9aa4bf; margin-top:2px; }
  .bike-row .dot { width:14px; height:14px; border-radius:50%; flex:none; box-shadow:inset 0 -3px 5px rgba(0,0,0,.4); }
  .bike-row button {
    border:none; border-radius:8px; padding:9px 14px; font-weight:800; cursor:pointer; font-size:13px; min-width:96px;
  }
  .buy { background:linear-gradient(180deg,#ffd24a,#e89b18); color:#3a2600; }
  .buy:disabled { background:#3a415a; color:#7b8299; cursor:default; box-shadow:none; }
  .ride { background:#3fae6a; color:#08331c; }
  .riding-tag { color:#8b94ad; font-size:13px; min-width:96px; text-align:center; }
  .tip { color:#9aa4bf; font-size:12px; margin-top:12px; line-height:1.5; }
  .crash {
    position:fixed; inset:0; display:none; align-items:center; justify-content:center; z-index:9; flex-direction:column; color:#fff;
    background:rgba(20,5,5,.55); text-align:center;
  }
  .crash.show { display:flex; }
  .crash h1 { font-size:40px; color:#ff6b5e; }
  .crash p { margin-top:8px; opacity:.9; }
</style>
</head>
<body>
<canvas id="cv"></canvas>
<div class="hud">
  <div class="money" id="hudMoney">$0</div>
  <div class="bike" id="hudBike"></div>
  <div class="combo" id="hudCombo"></div>
</div>
<div class="toast" id="toast"></div>
<div class="btn"><button id="shopBtn">🏪 Мотосалон</button></div>
<div class="crash" id="crash">
  <h1>ЖЁСТКОЕ ПАДЕНИЕ</h1>
  <p id="crashTxt"></p>
  <p style="opacity:.6;font-size:13px;margin-top:14px">нажми, чтобы продолжить</p>
</div>
<div class="panel" id="panel">
  <div class="shop">
    <h2>МОТОСАЛОН «У Ашота»</h2>
    <div class="sub">Трюки приносят копейки. Байки — как жильё в Москве. Выбирай.</div>
    <div id="bikeList"></div>
    <div class="tip">
      💡 Трюки в воздухе дают деньги только за <b>чистое приземление</b>. Падение — трюки сгорают.<br>
      Комбо: несколько трюков подряд без падения умножают выплату.
    </div>
    <div style="text-align:center;margin-top:14px"><button id="closeShop" class="ride" style="border:none;border-radius:8px;padding:10px 30px;font-weight:800;cursor:pointer">В путь</button></div>
  </div>
</div>

<script>
const cv = document.getElementById('cv'), ctx = cv.getContext('2d');
let W, H;
function resize(){ W = cv.width = innerWidth * devicePixelRatio; H = cv.height = innerHeight * devicePixelRatio; }
addEventListener('resize', resize); resize();

// ---------- Сохранение ----------
const SAVE_KEY = 'motolend_save_v1';
let save = { money: 0, owned: ['STARTER'], active: 'STARTER', best: 0 };
try { const s = localStorage.getItem(SAVE_KEY); if (s) save = Object.assign(save, JSON.parse(s)); } catch(e){}
function persist(){ try { localStorage.setItem(SAVE_KEY, JSON.stringify(save)); } catch(e){} }

// ---------- Байки ----------
const BIKES = [
  { id:'STARTER', name:'Педалька 50',    price:0,    accel:0.055, maxSpd:7.5,  jump:12.5, color:'#8a8f98', accent:'#4a4f58' },
  { id:'SCOUT',   name:'Скаут 125',      price:600,  accel:0.07,  maxSpd:9,    jump:13.5, color:'#d94f3d', accent:'#7a2620' },
  { id:'BAHA',    name:'Бэха 250',       price:2200, accel:0.085, maxSpd:10.5, jump:15,   color:'#2f7fd4', accent:'#1a4a80' },
  { id:'RACER',   name:'Гонец 600',      price:7500, accel:0.11,  maxSpd:12.5, jump:16.5, color:'#2bb673', accent:'#14603c' },
  { id:'KING',    name:'Дрейф-Кинг 1000',price:20000,accel:0.13,  maxSpd:14.5, jump:18,   color:'#9b59f5', accent:'#5a2b9e' },
];
const bikeById = id => BIKES.find(b => b.id === id) || BIKES[0];
let bike = bikeById(save.active);

// ---------- Мир ----------
const G = 0.45;
let terrainSeed = Math.random()*1000;
function groundY(x){
  return H*0.72 + Math.sin(x*0.004 + terrainSeed)*46 + Math.sin(x*0.011+terrainSeed*2)*22 + Math.sin(x*0.027)*9;
}
const RAMP_SPACING = 1400;
function rampAt(x){
  const k = Math.floor(x / RAMP_SPACING);
  const cx = k * RAMP_SPACING + 500;
  return { cx, h: 70 + (k*7919 % 60), up:220 };
}
function terrainAt(x){
  const r = rampAt(x);
  if (x > r.cx - r.up && x < r.cx){
    const t = (x - (r.cx - r.up)) / r.up;
    const flatY = groundY(r.cx - r.up);
    return flatY - r.h * t * t; // трамплин-рампа
  }
  if (x >= r.cx && x < r.cx + 260){
    const t = (x - r.cx) / 260;
    return groundY(r.cx - r.up) - r.h * (1 - t) * (1 - t) * 0; // обрыв — полёт!
  }
  return groundY(x);
}

// ---------- Игрок ----------
const P = {
  x: 200, y: 0, vx: 3, vy: 0, rot: 0, rotV: 0, onGround: true,
  crashTimer: 0, pendingTrick: 0, airTime: 0, combo: 0, distance: 0
};
P.y = groundY(P.x) - 30;

let trickQueue = [];      // накопленные трюки за полёт
let fullFlips = 0;        // полные обороты за полёт

// ---------- Ввод ----------
let keys = {}, touching = false;
addEventListener('keydown', e => { keys[e.code] = true; if (e.code==='Space') e.preventDefault(); });
addEventListener('keyup',   e => { keys[e.code] = false; });
cv.addEventListener('touchstart', e => { touching = true; });
cv.addEventListener('touchend',   e => { touching = false; });

// ---------- UI ----------
const hudMoney = document.getElementById('hudMoney');
const hudBike  = document.getElementById('hudBike');
const hudCombo = document.getElementById('hudCombo');
const toastEl  = document.getElementById('toast');
function fmt(n){ return '$' + n.toLocaleString('ru-RU'); }
function updateHUD(){
  hudMoney.textContent = fmt(save.money);
  hudBike.textContent = '🛞 ' + bike.name;
  hudCombo.textContent = P.combo > 1 ? 'КОМБО x' + P.combo : '';
}
let toastTimer;
function toast(text){
  toastEl.textContent = text;
  toastEl.style.opacity = 1; toastEl.style.transform = 'translateY(-8px)';
  clearTimeout(toastTimer);
  toastTimer = setTimeout(()=>{ toastEl.style.opacity = 0; toastEl.style.transform = 'none'; }, 1200);
}

// ---------- Магазин ----------
const panel = document.getElementById('panel');
document.getElementById('shopBtn').onclick = () => { panel.classList.add('open'); renderShop(); };
document.getElementById('closeShop').onclick = () => panel.classList.remove('open');
function renderShop(){
  const list = document.getElementById('bikeList');
  list.innerHTML = '';
  BIKES.forEach(b => {
    const owned = save.owned.includes(b.id);
    const active = save.active === b.id;
    const row = document.createElement('div');
    row.className = 'bike-row' + (owned ? ' owned' : '') + (active ? ' active' : '');
    const afford = save.money >= b.price;
    row.innerHTML = `
      <div class="dot" style="background:${b.color}"></div>
      <div class="info">
        <div class="name">${b.name}</div>
        <div class="stats">Скорость ${Math.round(b.maxSpd*10)} · Прыжок ${b.jump} · Разгон ${b.accel>0.1?'отличный':b.accel>0.07?'хороший':'так себе'}</div>
      </div>`;
    const btn = document.createElement('button');
    if (active){ btn.className='riding-tag'; btn.textContent='Еду сейчас'; }
    else if (owned){ btn.className='ride'; btn.textContent='Кататься'; btn.onclick = ()=>{ save.active=b.id; bike=bikeById(b.id); persist(); renderShop(); updateHUD(); }; }
    else { btn.className='buy'; btn.textContent=fmt(b.price); btn.disabled=!afford; btn.title = afford? '':'Не хватает денег — крути трюки'; btn.onclick = ()=>{ save.money-=b.price; save.owned.push(b.id); save.active=b.id; bike=bikeById(b.id); persist(); renderShop(); updateHUD(); }; }
    row.appendChild(btn);
    list.appendChild(row);
  });
}

// ---------- Крэш ----------
const crashEl = document.getElementById('crash');
crashEl.onclick = () => { crashEl.classList.remove('show'); respawn(); };
function doCrash(){
  if (P.crashTimer) return;
  P.crashTimer = 90;
  const lost = trickQueue.reduce((s,t)=>s+t.val,0);
  if (lost > 0) crashEl.querySelector('#crashTxt').textContent = 'Незасчитанные трюки сгорели: ' + fmt(lost);
  trickQueue = []; fullFlips = 0; P.combo = 0;
  setTimeout(()=>{ if (P.crashTimer) crashEl.classList.add('show'); }, 1400);
}
function respawn(){
  P.crashTimer = 0; P.rot = 0; P.rotV = 0; P.vx = 3; P.vy = 0;
  P.x += 400; P.y = terrainAt(P.x) - 30; P.onGround = true;
}

// ---------- Трюки ----------
function grantMoney(){
  if (!trickQueue.length && !fullFlips) return;
  let sum = trickQueue.reduce((s,t)=>s+t.val,0) + fullFlips * 25;
  // комбо-множитель растёт без падений, но скромно
  const mult = 1 + Math.min(P.combo * 0.15, 0.75);
  const total = Math.max(1, Math.round(sum * mult));
  save.money += total;
  persist();
  toast('+' + fmt(total) + (mult>1 ? ' (x' + mult.toFixed(2) + ')' : ''));
  updateHUD();
  trickQueue = []; fullFlips = 0;
}
function addTrick(label, val){
  trickQueue.push({ label, val });
  hudCombo.textContent = trickQueue.map(t=>t.label).join(' + ') + (P.combo>0 ? '  x'+P.combo : '');
}

// ---------- Рисование ----------
function drawSky(){
  const g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0,'#2c3e6b'); g.addColorStop(0.5,'#7a86b8'); g.addColorStop(1,'#d9a06b');
  ctx.fillStyle = g; ctx.fillRect(0,0,W,H);
  // солнце
  ctx.fillStyle='rgba(255,220,150,.9)';
  ctx.beginPath(); ctx.arc(W*0.78, H*0.22, H*0.05, 0, 7); ctx.fill();
  // горы-параллакс
  ctx.fillStyle = 'rgba(60,72,110,.55)';
  for (let layer=0; layer<2; layer++){
    const px = -P.x * (0.15 + layer*0.12);
    ctx.beginPath(); ctx.moveTo(0,H);
    for (let i=0;i<=20;i++){
      const sx = i*W/10;
      const mh = H*(0.35 - layer*0.08) + Math.sin((sx-px)*0.001+layer*3)*H*0.06;
      ctx.lineTo(sx+px%W, mh);
    }
    ctx.lineTo(W,H); ctx.closePath(); ctx.fill();
  }
}
function drawTerrain(){
  ctx.beginPath(); ctx.moveTo(0,H);
  for (let sx=0; sx<=W; sx+=8){
    ctx.lineTo(sx, terrainAt(sx + P.x - W/2) - 0);
  }
  // правильный маппинг: terrain рисуем относительно P.x
  ctx.lineTo(W,H); ctx.closePath();
  const g = ctx.createLinearGradient(0,H*0.6,0,H);
  g.addColorStop(0,'#7a5b3a'); g.addColorStop(1,'#4a3421');
  ctx.fillStyle = g; ctx.fill();
  // разметка-тропа
  ctx.strokeStyle='rgba(255,255,255,.15)'; ctx.lineWidth=3; ctx.setLineDash([24,20]);
  ctx.beginPath();
  for (let sx=0; sx<=W; sx+=8){
    const y = terrainAt(sx + P.x - W/2) - 6;
    sx===0 ? ctx.moveTo(sx,y) : ctx.lineTo(sx,y);
  }
  ctx.stroke(); ctx.setLineDash([]);
  // трамплины подсветим
  for (let k=Math.floor((P.x-W)/RAMP_SPACING); k<Math.floor((P.x+W)/RAMP_SPACING)+1; k++){
    const r = rampAt(k*RAMP_SPACING+400);
    const sx = r.cx - P.x + W/2;
    if (sx > -r.up-20 && sx < W+300){
      ctx.fillStyle='rgba(255,80,60,.5)';
      ctx.fillRect(sx-6, terrainAt(r.cx-1)-r.h-14, 12, 14);
    }
  }
}
function drawBike(){
  const bx = W/2, by = P.y;
  ctx.save(); ctx.translate(bx, by); ctx.rotate(P.rot);
  const wheelR = 13;
  function wheel(dx, spin){
    ctx.save(); ctx.translate(dx,0);
    ctx.fillStyle = '#15161a';
    ctx.beginPath(); ctx.arc(0,0,wheelR,0,7); ctx.fill();
    ctx.strokeStyle = '#3d4048'; ctx.lineWidth = 3;
    ctx.beginPath(); ctx.arc(0,0,wheelR-1.5,0,7); ctx.stroke();
    ctx.strokeStyle = '#8a8f98'; ctx.lineWidth = 2;
    const a = spin;
    for (let i=0;i<3;i++){
      const ang = a + i*Math.PI/1.5;
      ctx.beginPath(); ctx.moveTo(Math.cos(ang)*(wheelR-4), Math.sin(ang)*(wheelR-4));
      ctx.lineTo(Math.cos(ang+Math.PI)*(wheelR-4), Math.sin(ang+Math.PI)*(wheelR-4));
      ctx.stroke();
    }
    ctx.restore();
  }
  const spin = P.x * 0.06;
  wheel(-19, spin); wheel(19, spin);
  // рама
  ctx.strokeStyle = bike.color; ctx.lineWidth = 5; ctx.lineCap='round';
  ctx.beginPath(); ctx.moveTo(-19,0); ctx.lineTo(-4,-6); ctx.lineTo(8,-2); ctx.lineTo(19,0); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(-4,-6); ctx.lineTo(4,-16); ctx.stroke();
  // сиденье/бак
  ctx.fillStyle = bike.accent;
  ctx.beginPath(); ctx.ellipse(0,-14,13,6,-0.15,0,7); ctx.fill();
  // райдер
  ctx.fillStyle = '#22262e';
  ctx.beginPath(); ctx.arc(2,-30,6,0,7); ctx.fill();          // голова
  ctx.strokeStyle = '#2f3440'; ctx.lineWidth = 4.5;
  ctx.beginPath(); ctx.moveTo(1,-24); ctx.lineTo(-2,-13); ctx.stroke(); // тело
  ctx.beginPath(); ctx.moveTo(-2,-13); ctx.lineTo(-10,-4); ctx.stroke(); // нога
  ctx.beginPath(); ctx.moveTo(1,-24); ctx.lineTo(12,-14); ctx.stroke();  // рука к рулю
  // выхлоп при газе
  if (P.onGround && P.vx > bike.maxSpd*0.7){
    ctx.fillStyle = 'rgba(200,200,210,.35)';
    ctx.beginPath(); ctx.ellipse(-30,-4,8+Math.random()*4,3+Math.random()*2,0,0,7); ctx.fill();
  }
  ctx.restore();
  // тень
  const gy = terrainAt(P.x);
  if (gy - P.y < 200){
    ctx.fillStyle = 'rgba(0,0,0,.25)';
    ctx.beginPath(); ctx.ellipse(W/2, gy-2, 26*(1-(gy-P.y)/400), 6, 0, 0, 7); ctx.fill();
  }
}

// ---------- Игровой цикл ----------
let last = performance.now(), acc = 0;
function step(){
  if (P.crashTimer) { P.crashTimer--; if (!P.crashTimer) { /* ждём клика */ } return; }

  const gas = keys['Space'] || touching;
  const spinF = keys['ArrowLeft'] || false, spinB = keys['ArrowRight'] || false;

  // физика
  if (P.onGround){
    P.vx += bike.accel * (gas ? 1 : -0.5);
    P.vx = Math.max(1.5, Math.min(P.vx, bike.maxSpd));
    P.vy = 0;
    // прыжок с рампы — высота зависит от скорости и рампы
    const r = rampAt(P.x);
    const nearRamp = (P.x > r.cx - r.up - 40 && P.x < r.cx + 30);
    if (gas && nearRamp){
      P.vy = -(bike.jump + P.vx*0.9);
      P.onGround = false; P.airTime = 0;
      fullFlips = 0; trickQueue = [];
    }
    P.y = terrainAt(P.x) - 26;
  } else {
    // воздух: трюки
    P.airTime++;
    if (spinB){ P.rotV += 0.13; }
    if (spinF){ P.rotV -= 0.13; }
    if (Math.abs(P.rotV) > 0.02) addTrickOnce();
    P.vy += G;
    P.y += P.vy;
    P.rot += P.rotV;
    P.rotV *= 0.985;
    // приземление
    const gy = terrainAt(P.x) - 26;
    if (P.y >= gy){
      P.y = gy; P.onGround = true; P.rotV = 0;
      const norm = Math.abs(((P.rot % (Math.PI*2)) + Math.PI*2 + Math.PI) % (Math.PI*2) - Math.PI);
      if (norm > Math.PI/3){ // приземлился не на колёса
        doCrash();
      } else {
        if (trickQueue.length || fullFlips){ P.combo++; grantMoney(); }
        P.rot = Math.round(P.rot / (Math.PI*2)) * Math.PI*2;
        fullFlips = 0;
      }
    }
  }

  P.x += P.vx;
  P.distance = Math.max(P.distance, P.x);
  if (P.onGround) P.rot = Math.sin(P.x*0.02)*0.03;
  updateHUD();
}

let lastTrickCount = 0;
function addTrickOnce(){
  // каждые ~пол-оборота — «трюк»
  const flips = Math.floor(Math.abs(P.rot) / Math.PI);
  while (lastTrickCount < flips){ lastTrickCount++; fullFlips++; addTrickLabel(); }
}
function addTrickLabel(){ /* трюки копим в очередь */ }

// Упростим: трюки считаем по оборотам в grantMoney — очередь не нужна отдельно,
// но для честности в гринде добавим «простые трюки» за долгий полёт:
function longAirBonus(){
  if (!P.onGround && P.airTime === 90) { trickQueue.push({label:'Airtime', val:15}); toastEl && 0; }
}

function frame(now){
  const dt = Math.min(50, now - last); last = now;
  acc += dt;
  while (acc > 16.6){ step(); acc -= 16.6; if (!P.crashTimer) { /* ok */ } }
  drawSky(); drawTerrain(); drawBike();
  if (P.crashTimer) { /* картинка уже нарисована */ }
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
updateHUD();
</script>
</body>
</html>