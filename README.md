<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Mini Geometry Dash (HTML/JS)</title>
<style>
  :root{
    --bg:#0b1020;
    --ground:#161a2b;
    --accent:#ffd166;
    --player:#06d6a0;
    --ob:#ef476f;
    --muted:#96a0b3;
    font-family: Inter, ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
  }
  html,body{height:100%;margin:0;background:linear-gradient(180deg,var(--bg),#071022);color:#fff}
  .wrap{display:flex;flex-direction:column;align-items:center;justify-content:center;height:100%;gap:14px;padding:18px;box-sizing:border-box}
  canvas{background:transparent;border-radius:12px;box-shadow:0 8px 30px rgba(2,6,23,.6);display:block}
  .ui{display:flex;gap:12px;align-items:center;font-size:14px}
  .pill{background:rgba(255,255,255,0.04);padding:6px 10px;border-radius:999px;color:var(--muted)}
  .big{font-weight:700;color:var(--accent)}
  .hint{font-size:13px;color:var(--muted);margin-top:-6px}
  button{background:var(--accent);border:none;padding:8px 12px;border-radius:8px;font-weight:700;color:#06121a;cursor:pointer}
  @media (max-width:640px){canvas{width:100%;height:220px}}
</style>
</head>
<body>
<div class="wrap">
  <div class="ui">
    <div class="pill">Score: <span id="score">0</span></div>
    <div class="pill">High: <span id="high">0</span></div>
    <div style="width:14px"></div>
    <button id="muteBtn">Mute</button>
  </div>
  <canvas id="game" width="900" height="300" role="img" aria-label="Geometry Dash style game"></canvas>
  <div class="hint">Jump with <strong>Space</strong> / <strong>↑</strong> or tap/click. Press Space to restart after death.</div>
</div>

<script>
// --- Settings ---
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d', { alpha: true });
const W = canvas.width, H = canvas.height;
let lastTime = 0;
let dt = 0;

const GROUND_Y = H - 50;
const GRAVITY = 2000; // px/s^2
const JUMP_V = -650; // initial jump velocity px/s
let speed = 300; // obstacle base speed px/s
let difficultyIncreaseRate = 0.005; // per second

// Game state
let player = { x: 120, y: GROUND_Y - 40, w: 42, h: 42, vy: 0, onGround: true, alive: true };
let obstacles = [];
let spawnTimer = 0;
let spawnInterval = 1.2; // seconds initial
let score = 0;
let highScore = Number(localStorage.getItem('gd_high') || 0);
document.getElementById('high').innerText = highScore;
document.getElementById('score').innerText = score.toFixed(0);
let muted = false;
const muteBtn = document.getElementById('muteBtn');
muteBtn.addEventListener('click', () => { muted = !muted; muteBtn.textContent = muted ? 'Unmute' : 'Mute'; });

// --- Audio (small WebAudio helpers) ---
const AudioCtx = window.AudioContext || window.webkitAudioContext;
let audioCtx = null;
function ensureAudioCtx(){
  if(!audioCtx) audioCtx = new AudioCtx();
}
function beep(freq=440, time=0.08, type='sine', vol=0.12){
  if(muted) return;
  ensureAudioCtx();
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = type;
  o.frequency.value = freq;
  g.gain.value = vol;
  o.connect(g); g.connect(audioCtx.destination);
  o.start();
  g.gain.setValueAtTime(vol, audioCtx.currentTime);
  g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + time);
  o.stop(audioCtx.currentTime + time + 0.02);
}

// --- Controls ---
let pressedJump = false;
function doJump(){
  if(!player.alive) return;
  if(player.onGround){
    player.vy = JUMP_V;
    player.onGround = false;
    beep(880, 0.06, 'triangle', 0.08);
  }
}
window.addEventListener('keydown', e => {
  if(e.code === 'Space' || e.code === 'ArrowUp'){
    e.preventDefault();
    if(!player.alive){ restart(); return; }
    doJump();
  }
});
canvas.addEventListener('mousedown', e => {
  if(!player.alive){ restart(); return; }
  doJump();
});
canvas.addEventListener('touchstart', e => {
  e.preventDefault();
  if(!player.alive){ restart(); return; }
  doJump();
}, {passive:false});

// --- Utility ---
function randRange(a,b){ return a + Math.random()*(b-a); }
function rectsOverlap(a,b){
  return !(a.x + a.w <= b.x || a.x >= b.x + b.w || a.y + a.h <= b.y || a.y >= b.y + b.h);
}

// --- Obstacle spawn ---
function spawnObstacle(){
  // Variety: single block, tall block, double spike
  const type = Math.random() < 0.14 ? 'tall' : (Math.random() < 0.18 ? 'double' : 'single');
  let w = 30, h = 40;
  if(type === 'tall'){ h = randRange(80, 140); w = 34; }
  const obs = {
    x: W + 20,
    w: Math.round(w),
    h: Math.round(h),
    y: GROUND_Y - h,
    speedMul: 1 + Math.random()*0.2
  };
  if(type==='double'){
    // create 2 small ones with a gap
    obs.double = true;
    obs.w = 26;
    obs.h = 40;
    obs.gap = 80 + Math.random()*60;
  }
  obstacles.push(obs);
}

// --- Game loop ---
function update(t){
  if(!lastTime) lastTime = t;
  dt = (t - lastTime) / 1000;
  lastTime = t;

  if(dt > 0.08) dt = 0.08; // clamp large frames

  if(player.alive){
    // physics
    player.vy += GRAVITY * dt;
    player.y += player.vy * dt;
    if(player.y + player.h >= GROUND_Y){
      player.y = GROUND_Y - player.h;
      player.vy = 0;
      player.onGround = true;
    }

    // spawn obstacles
    spawnTimer -= dt;
    if(spawnTimer <= 0){
      spawnObstacle();
      // spawn faster as game goes up in difficulty
      spawnInterval = Math.max(0.52, spawnInterval * (0.98 - difficultyIncreaseRate*0.1));
      spawnTimer = spawnInterval + Math.random()*0.5;
    }

    // move obstacles
    for(let i = obstacles.length-1; i >= 0; i--){
      const o = obstacles[i];
      const vx = (speed * o.speedMul) * (1 + score*0.0012); // scale with score
      o.x -= vx * dt;
      // collision boxes: if double, two boxes (left and right)
      if(o.double){
        const left = { x: o.x, y: o.y, w: o.w, h: o.h };
        const right = { x: o.x + o.w + o.gap, y: o.y, w: o.w, h: o.h };
        if(rectsOverlap(player, left) || rectsOverlap(player, right)){
          playerDie();
        }
      } else {
        if(rectsOverlap(player, o)){
          playerDie();
        }
      }
      if(o.x + (o.double ? (o.w*2 + o.gap) : o.w) < -40){
        obstacles.splice(i,1);
      }
    }

    // score
    score += dt * 120; // speed of score growth
    document.getElementById('score').innerText = Math.floor(score);
    // gradually increase base speed
    speed += difficultyIncreaseRate * dt * 600;
    // tiny spawnInterval shrink to amp difficulty
    spawnInterval = Math.max(0.55, spawnInterval - difficultyIncreaseRate * dt);
  }

  draw();
  requestAnimationFrame(update);
}

// --- Draw ---
function draw(){
  ctx.clearRect(0,0,W,H);

  // background: subtle grid
  ctx.fillStyle = 'rgba(255,255,255,0.02)';
  for(let i=0;i<15;i++){
    ctx.fillRect(i*(W/15),0,1, H);
  }

  // ground
  ctx.fillStyle = '#0f1626';
  ctx.fillRect(0, GROUND_Y, W, H - GROUND_Y);

  // decorative ground line
  ctx.fillStyle = 'rgba(255,255,255,0.04)';
  ctx.fillRect(0, GROUND_Y-6, W, 4);

  // obstacles
  for(const o of obstacles){
    if(o.double){
      // left block
      drawRect(o.x, o.y, o.w, o.h, '#ef476f');
      // right block
      drawRect(o.x + o.w + o.gap, o.y, o.w, o.h, '#ef476f');
    } else {
      const color = o.h > 70 ? '#ff9f1c' : '#ef476f';
      drawRect(o.x, o.y, o.w, o.h, color);
    }
  }

  // player (rounded square)
  drawRoundedRect(player.x, player.y, player.w, player.h, 6, '#06d6a0');
  // outline
  ctx.strokeStyle = 'rgba(0,0,0,0.25)';
  ctx.lineWidth = 2;
  ctx.strokeRect(player.x+1, player.y+1, player.w-2, player.h-2);

  // if dead, overlay
  if(!player.alive){
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(0,0,W,H);
    ctx.fillStyle = '#fff';
    ctx.font = '700 28px Inter, Arial';
    ctx.textAlign = 'center';
    ctx.fillText('You crashed!', W/2, H/2 - 10);
    ctx.font = '600 16px Inter, Arial';
    ctx.fillText('Press Space / Tap to restart — Score: ' + Math.floor(score), W/2, H/2 + 22);
  }
}

// small helpers:
function drawRect(x,y,w,h,color){
  ctx.fillStyle = color;
  ctx.fillRect(Math.round(x), Math.round(y), Math.round(w), Math.round(h));
}
function drawRoundedRect(x,y,w,h,r,color){
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  ctx.closePath();
  ctx.fill();
}

// --- Death & Restart ---
function playerDie(){
  if(!player.alive) return;
  player.alive = false;
  beep(120,0.25,'sawtooth',0.18);
  if(Math.floor(score) > highScore){
    highScore = Math.floor(score);
    localStorage.setItem('gd_high', highScore);
    document.getElementById('high').innerText = highScore;
  }
}

function restart(){
  // reset state
  player = { x: 120, y: GROUND_Y - 40, w: 42, h: 42, vy: 0, onGround: true, alive: true };
  obstacles = [];
  spawnTimer = 0.8;
  spawnInterval = 1.2;
  score = 0;
  speed = 300;
  document.getElementById('score').innerText = 0;
  lastTime = 0;
  beep(880,0.05,'sine',0.08);
}

// start
spawnTimer = 0.4;
requestAnimationFrame(update);

// make canvas crisp on HiDPI
function resizeCanvasForHiDPI(){
  const dpr = window.devicePixelRatio || 1;
  if(dpr !== 1){
    canvas.width = W * dpr;
    canvas.height = H * dpr;
    canvas.style.width = W + 'px';
    canvas.style.height = H + 'px';
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  } else {
    canvas.width = W; canvas.height = H;
    canvas.style.width = W + 'px';
    canvas.style.height = H + 'px';
  }
}
resizeCanvasForHiDPI();
window.addEventListener('resize', () => {
  // keep visual size responsive; don't change internal game size
  if(window.innerWidth < 640){
    canvas.style.width = '100%';
    canvas.style.height = '220px';
  } else {
    canvas.style.width = W + 'px';
    canvas.style.height = H + 'px';
  }
});

// accessibility: let user enable audio by clicking anywhere once (mobile autoplay restriction)
document.addEventListener('click', function enableAudioOnce(){
  ensureAudioCtx();
  if(audioCtx && audioCtx.state === 'suspended'){
    audioCtx.resume();
  }
  document.removeEventListener('click', enableAudioOnce);
});
</script>
</body>
</html><!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Mini Geometry Dash (HTML/JS)</title>
<style>
  :root{
    --bg:#0b1020;
    --ground:#161a2b;
    --accent:#ffd166;
    --player:#06d6a0;
    --ob:#ef476f;
    --muted:#96a0b3;
    font-family: Inter, ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
  }
  html,body{height:100%;margin:0;background:linear-gradient(180deg,var(--bg),#071022);color:#fff}
  .wrap{display:flex;flex-direction:column;align-items:center;justify-content:center;height:100%;gap:14px;padding:18px;box-sizing:border-box}
  canvas{background:transparent;border-radius:12px;box-shadow:0 8px 30px rgba(2,6,23,.6);display:block}
  .ui{display:flex;gap:12px;align-items:center;font-size:14px}
  .pill{background:rgba(255,255,255,0.04);padding:6px 10px;border-radius:999px;color:var(--muted)}
  .big{font-weight:700;color:var(--accent)}
  .hint{font-size:13px;color:var(--muted);margin-top:-6px}
  button{background:var(--accent);border:none;padding:8px 12px;border-radius:8px;font-weight:700;color:#06121a;cursor:pointer}
  @media (max-width:640px){canvas{width:100%;height:220px}}
</style>
</head>
<body>
<div class="wrap">
  <div class="ui">
    <div class="pill">Score: <span id="score">0</span></div>
    <div class="pill">High: <span id="high">0</span></div>
    <div style="width:14px"></div>
    <button id="muteBtn">Mute</button>
  </div>
  <canvas id="game" width="900" height="300" role="img" aria-label="Geometry Dash style game"></canvas>
  <div class="hint">Jump with <strong>Space</strong> / <strong>↑</strong> or tap/click. Press Space to restart after death.</div>
</div>

<script>
// --- Settings ---
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d', { alpha: true });
const W = canvas.width, H = canvas.height;
let lastTime = 0;
let dt = 0;

const GROUND_Y = H - 50;
const GRAVITY = 2000; // px/s^2
const JUMP_V = -650; // initial jump velocity px/s
let speed = 300; // obstacle base speed px/s
let difficultyIncreaseRate = 0.005; // per second

// Game state
let player = { x: 120, y: GROUND_Y - 40, w: 42, h: 42, vy: 0, onGround: true, alive: true };
let obstacles = [];
let spawnTimer = 0;
let spawnInterval = 1.2; // seconds initial
let score = 0;
let highScore = Number(localStorage.getItem('gd_high') || 0);
document.getElementById('high').innerText = highScore;
document.getElementById('score').innerText = score.toFixed(0);
let muted = false;
const muteBtn = document.getElementById('muteBtn');
muteBtn.addEventListener('click', () => { muted = !muted; muteBtn.textContent = muted ? 'Unmute' : 'Mute'; });

// --- Audio (small WebAudio helpers) ---
const AudioCtx = window.AudioContext || window.webkitAudioContext;
let audioCtx = null;
function ensureAudioCtx(){
  if(!audioCtx) audioCtx = new AudioCtx();
}
function beep(freq=440, time=0.08, type='sine', vol=0.12){
  if(muted) return;
  ensureAudioCtx();
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = type;
  o.frequency.value = freq;
  g.gain.value = vol;
  o.connect(g); g.connect(audioCtx.destination);
  o.start();
  g.gain.setValueAtTime(vol, audioCtx.currentTime);
  g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + time);
  o.stop(audioCtx.currentTime + time + 0.02);
}

// --- Controls ---
let pressedJump = false;
function doJump(){
  if(!player.alive) return;
  if(player.onGround){
    player.vy = JUMP_V;
    player.onGround = false;
    beep(880, 0.06, 'triangle', 0.08);
  }
}
window.addEventListener('keydown', e => {
  if(e.code === 'Space' || e.code === 'ArrowUp'){
    e.preventDefault();
    if(!player.alive){ restart(); return; }
    doJump();
  }
});
canvas.addEventListener('mousedown', e => {
  if(!player.alive){ restart(); return; }
  doJump();
});
canvas.addEventListener('touchstart', e => {
  e.preventDefault();
  if(!player.alive){ restart(); return; }
  doJump();
}, {passive:false});

// --- Utility ---
function randRange(a,b){ return a + Math.random()*(b-a); }
function rectsOverlap(a,b){
  return !(a.x + a.w <= b.x || a.x >= b.x + b.w || a.y + a.h <= b.y || a.y >= b.y + b.h);
}

// --- Obstacle spawn ---
function spawnObstacle(){
  // Variety: single block, tall block, double spike
  const type = Math.random() < 0.14 ? 'tall' : (Math.random() < 0.18 ? 'double' : 'single');
  let w = 30, h = 40;
  if(type === 'tall'){ h = randRange(80, 140); w = 34; }
  const obs = {
    x: W + 20,
    w: Math.round(w),
    h: Math.round(h),
    y: GROUND_Y - h,
    speedMul: 1 + Math.random()*0.2
  };
  if(type==='double'){
    // create 2 small ones with a gap
    obs.double = true;
    obs.w = 26;
    obs.h = 40;
    obs.gap = 80 + Math.random()*60;
  }
  obstacles.push(obs);
}

// --- Game loop ---
function update(t){
  if(!lastTime) lastTime = t;
  dt = (t - lastTime) / 1000;
  lastTime = t;

  if(dt > 0.08) dt = 0.08; // clamp large frames

  if(player.alive){
    // physics
    player.vy += GRAVITY * dt;
    player.y += player.vy * dt;
    if(player.y + player.h >= GROUND_Y){
      player.y = GROUND_Y - player.h;
      player.vy = 0;
      player.onGround = true;
    }

    // spawn obstacles
    spawnTimer -= dt;
    if(spawnTimer <= 0){
      spawnObstacle();
      // spawn faster as game goes up in difficulty
      spawnInterval = Math.max(0.52, spawnInterval * (0.98 - difficultyIncreaseRate*0.1));
      spawnTimer = spawnInterval + Math.random()*0.5;
    }

    // move obstacles
    for(let i = obstacles.length-1; i >= 0; i--){
      const o = obstacles[i];
      const vx = (speed * o.speedMul) * (1 + score*0.0012); // scale with score
      o.x -= vx * dt;
      // collision boxes: if double, two boxes (left and right)
      if(o.double){
        const left = { x: o.x, y: o.y, w: o.w, h: o.h };
        const right = { x: o.x + o.w + o.gap, y: o.y, w: o.w, h: o.h };
        if(rectsOverlap(player, left) || rectsOverlap(player, right)){
          playerDie();
        }
      } else {
        if(rectsOverlap(player, o)){
          playerDie();
        }
      }
      if(o.x + (o.double ? (o.w*2 + o.gap) : o.w) < -40){
        obstacles.splice(i,1);
      }
    }

    // score
    score += dt * 120; // speed of score growth
    document.getElementById('score').innerText = Math.floor(score);
    // gradually increase base speed
    speed += difficultyIncreaseRate * dt * 600;
    // tiny spawnInterval shrink to amp difficulty
    spawnInterval = Math.max(0.55, spawnInterval - difficultyIncreaseRate * dt);
  }

  draw();
  requestAnimationFrame(update);
}

// --- Draw ---
function draw(){
  ctx.clearRect(0,0,W,H);

  // background: subtle grid
  ctx.fillStyle = 'rgba(255,255,255,0.02)';
  for(let i=0;i<15;i++){
    ctx.fillRect(i*(W/15),0,1, H);
  }

  // ground
  ctx.fillStyle = '#0f1626';
  ctx.fillRect(0, GROUND_Y, W, H - GROUND_Y);

  // decorative ground line
  ctx.fillStyle = 'rgba(255,255,255,0.04)';
  ctx.fillRect(0, GROUND_Y-6, W, 4);

  // obstacles
  for(const o of obstacles){
    if(o.double){
      // left block
      drawRect(o.x, o.y, o.w, o.h, '#ef476f');
      // right block
      drawRect(o.x + o.w + o.gap, o.y, o.w, o.h, '#ef476f');
    } else {
      const color = o.h > 70 ? '#ff9f1c' : '#ef476f';
      drawRect(o.x, o.y, o.w, o.h, color);
    }
  }

  // player (rounded square)
  drawRoundedRect(player.x, player.y, player.w, player.h, 6, '#06d6a0');
  // outline
  ctx.strokeStyle = 'rgba(0,0,0,0.25)';
  ctx.lineWidth = 2;
  ctx.strokeRect(player.x+1, player.y+1, player.w-2, player.h-2);

  // if dead, overlay
  if(!player.alive){
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(0,0,W,H);
    ctx.fillStyle = '#fff';
    ctx.font = '700 28px Inter, Arial';
    ctx.textAlign = 'center';
    ctx.fillText('You crashed!', W/2, H/2 - 10);
    ctx.font = '600 16px Inter, Arial';
    ctx.fillText('Press Space / Tap to restart — Score: ' + Math.floor(score), W/2, H/2 + 22);
  }
}

// small helpers:
function drawRect(x,y,w,h,color){
  ctx.fillStyle = color;
  ctx.fillRect(Math.round(x), Math.round(y), Math.round(w), Math.round(h));
}
function drawRoundedRect(x,y,w,h,r,color){
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  ctx.closePath();
  ctx.fill();
}

// --- Death & Restart ---
function playerDie(){
  if(!player.alive) return;
  player.alive = false;
  beep(120,0.25,'sawtooth',0.18);
  if(Math.floor(score) > highScore){
    highScore = Math.floor(score);
    localStorage.setItem('gd_high', highScore);
    document.getElementById('high').innerText = highScore;
  }
}

function restart(){
  // reset state
  player = { x: 120, y: GROUND_Y - 40, w: 42, h: 42, vy: 0, onGround: true, alive: true };
  obstacles = [];
  spawnTimer = 0.8;
  spawnInterval = 1.2;
  score = 0;
  speed = 300;
  document.getElementById('score').innerText = 0;
  lastTime = 0;
  beep(880,0.05,'sine',0.08);
}

// start
spawnTimer = 0.4;
requestAnimationFrame(update);

// make canvas crisp on HiDPI
function resizeCanvasForHiDPI(){
  const dpr = window.devicePixelRatio || 1;
  if(dpr !== 1){
    canvas.width = W * dpr;
    canvas.height = H * dpr;
    canvas.style.width = W + 'px';
    canvas.style.height = H + 'px';
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  } else {
    canvas.width = W; canvas.height = H;
    canvas.style.width = W + 'px';
    canvas.style.height = H + 'px';
  }
}
resizeCanvasForHiDPI();
window.addEventListener('resize', () => {
  // keep visual size responsive; don't change internal game size
  if(window.innerWidth < 640){
    canvas.style.width = '100%';
    canvas.style.height = '220px';
  } else {
    canvas.style.width = W + 'px';
    canvas.style.height = H + 'px';
  }
});

// accessibility: let user enable audio by clicking anywhere once (mobile autoplay restriction)
document.addEventListener('click', function enableAudioOnce(){
  ensureAudioCtx();
  if(audioCtx && audioCtx.state === 'suspended'){
    audioCtx.resume();
  }
  document.removeEventListener('click', enableAudioOnce);
});
</script>
</body>
</html>
