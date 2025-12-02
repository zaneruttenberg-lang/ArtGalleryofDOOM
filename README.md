<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Wine & Gallery — Enhanced Parody Game</title>
<!-- Phaser 3 from CDN -->
<script src="https://cdn.jsdelivr.net/npm/phaser@3.60.0/dist/phaser.min.js"></script>
<style>
  html,body{height:100%;margin:0;background:#111;color:#eee}
  #game-container{width:100%;height:100vh;display:flex;align-items:center;justify-content:center;position:relative}
  /* Modal overlay for painting view */
  .overlay{
    position:fixed;inset:0;display:none;align-items:center;justify-content:center;
    background:rgba(0,0,0,0.75);z-index:50;padding:20px;backdrop-filter:blur(6px);
    transition:opacity 300ms ease;
  }
  .overlay.show{display:flex;opacity:1}
  .overlay.hide{opacity:0}
  .overlay .panel{
    background:#fff;padding:18px;border-radius:10px;max-width:1100px;width:95%;max-height:90vh;overflow:auto;
    box-shadow:0 20px 60px rgba(0,0,0,0.6);
    transform-origin:center center;transition:transform 300ms ease;
  }
  .overlay.show .panel{transform:translateY(0) scale(1)}
  .overlay.hide .panel{transform:translateY(12px) scale(0.98)}
  .overlay img{max-width:100%;height:auto;display:block;margin:0 auto;border-radius:6px}
  .overlay .meta{margin-top:12px;font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial;color:#222}
  .overlay .close{
    margin-top:12px;padding:8px 12px;background:#222;color:#fff;border:none;border-radius:8px;cursor:pointer;
  }
  /* small HUD */
  .hint{
    position:fixed;left:50%;transform:translateX(-50%);bottom:18px;padding:8px 12px;background:rgba(0,0,0,0.6);
    color:#fff;border-radius:8px;font-family:monospace;z-index:5;
  }
  .credits{position:fixed;right:8px;top:8px;color:#ddd;font-size:13px;font-family:system-ui,Arial;z-index:5}
  /* mobile controls */
  .touch-controls{position:fixed;left:12px;bottom:12px;z-index:6;display:flex;gap:8px}
  .dpad{display:grid;grid-template-columns:repeat(3,56px);grid-template-rows:repeat(3,56px);gap:6px}
  .btn{width:56px;height:56px;border-radius:10px;background:rgba(255,255,255,0.06);display:flex;align-items:center;justify-content:center;color:#fff;font-weight:600;border:1px solid rgba(255,255,255,0.06);user-select:none}
  .interact-btn{position:fixed;right:12px;bottom:12px;width:64px;height:64px;border-radius:999px;background:#7b2b2b;display:flex;align-items:center;justify-content:center;color:#fff;font-weight:700;z-index:6;border:none}
  /* minimap */
  #minimap{position:fixed;left:8px;top:8px;width:160px;height:120px;border-radius:6px;background:rgba(0,0,0,0.45);z-index:6;padding:6px;box-sizing:border-box}
</style>
</head>
<body>
<div id="game-container"></div>

<!-- painting modal -->
<div id="overlay" class="overlay" role="dialog" aria-hidden="true">
  <div class="panel" role="document">
    <img id="painting-large" alt="Painting view"/>
    <div class="meta">
      <strong id="painting-title">Title</strong>
      <div id="painting-desc">Description</div>
    </div>
    <button class="close" id="close-overlay">Close</button>
  </div>
</div>

<div class="hint" id="hint" style="display:none">Press <strong>E</strong> to inspect painting</div>
<div class="credits">Use WASD / Arrow keys to move • Tap paintings to inspect</div>

<!-- mobile controls -->
<div class="touch-controls" id="touch-controls" style="display:none" aria-hidden="true">
  <div class="dpad">
    <div></div><div class="btn" data-dir="up">↑</div><div></div>
    <div class="btn" data-dir="left">←</div><div></div><div class="btn" data-dir="right">→</div>
    <div></div><div class="btn" data-dir="down">↓</div><div></div>
  </div>
</div>
<button class="interact-btn" id="touch-interact" style="display:none">E</button>

<!-- minimap -->
<canvas id="minimap" width="160" height="120"></canvas>

<script>
/*
  Enhanced Phaser gallery demo with:
  - ambient synthesized audio + sip sound
  - animated player sprite (procedurally generated frames)
  - NPC visitors wandering
  - camera pan/zoom to painting before modal
  - smooth overlay transitions
  - minimap
  - on-screen mobile controls
*/

// Placeholder paintings: you can replace dataUrl values with real image URLs.
const PAINTINGS = [
  { id: 'p1', title: 'Still Life with Merlot', desc: 'A tasteful study of a bottle, a glass, and a halo of light.', dataUrl: 'data:image/svg+xml;utf8,' + encodeURIComponent(`<svg xmlns="http://www.w3.org/2000/svg" width="1200" height="900"><rect width="100%" height="100%" fill="#f6efe9"/><rect x="70" y="60" width="1060" height="780" fill="#e3d4c7" rx="12" /><g transform="translate(120,120)"><ellipse cx="360" cy="760" rx="520" ry="80" fill="#ead6c6"/><rect x="160" y="40" width="120" height="400" fill="#4b1f1f" rx="20"/><circle cx="320" cy="160" r="60" fill="#9b2b2b"/><path d="M440 80 q60 40 120 0" stroke="#d18b8b" stroke-width="20" fill="none"/></g></svg>`) },
  { id: 'p2', title: 'Abstract Palette', desc: 'A playful abstract composition; imagine expensive provenance.', dataUrl: 'data:image/svg+xml;utf8,' + encodeURIComponent(`<svg xmlns="http://www.w3.org/2000/svg" width="1200" height="900"><rect width="100%" height="100%" fill="#f0f3ff"/><circle cx="300" cy="270" r="120" fill="#ff9a9e"/><rect x="540" y="180" width="440" height="320" rx="30" fill="#b8f2e6"/><path d="M240 840 q240 -320 640 -120" stroke="#6d6dff" stroke-width="56" fill="none" stroke-linecap="round"/><text x="40" y="860" font-size="36" fill="#333">Untitled (Maybe)</text></svg>`) },
  { id: 'p3', title: 'Baroque Impression (With Corkscrew)', desc: 'A dramatic scene in miniature; contains subtle notes of oak.', dataUrl: 'data:image/svg+xml;utf8,' + encodeURIComponent(`<svg xmlns="http://www.w3.org/2000/svg" width="1200" height="900"><rect width="100%" height="100%" fill="#fff9ee"/><g transform="translate(80,60)"><rect x="20" y="20" width="1040" height="700" fill="#efe0d0" rx="12"/><path d="M240 600 q240 -440 720 -120" stroke="#7a3f3f" stroke-width="72" fill="none"/><circle cx="840" cy="240" r="64" fill="#c54b4b"/></g></svg>`) }
];

const config = {
  type: Phaser.AUTO,
  parent: 'game-container',
  width: Math.min(window.innerWidth, 1200),
  height: Math.min(window.innerHeight, 800),
  backgroundColor: '#dad7d2',
  physics: { default: 'arcade', arcade: { debug: false } },
  scene: { preload: preload, create: create, update: update }
};

const game = new Phaser.Game(config);
let player, cursors, interactKey, paintingsGroup, npcGroup, minimapCanvas, minimapCtx;
let hintEl = document.getElementById('hint');
let overlay = document.getElementById('overlay');
let paintingLarge = document.getElementById('painting-large');
let paintingTitle = document.getElementById('painting-title');
let paintingDesc = document.getElementById('painting-desc');
let closeOverlayBtn = document.getElementById('close-overlay');
let currentNearby = null;
let isMobile = /Mobi|Android/i.test(navigator.userAgent);

// Audio synthesis (WebAudio) - ambient drone + sip sound
let audioCtx, ambientGain, sipEnabled = true;
function startAudio() {
  if (audioCtx) return;
  audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  // ambient drone: two detuned oscillators with slow gain LFO
  ambientGain = audioCtx.createGain();
  ambientGain.gain.value = 0.02; // very low
  ambientGain.connect(audioCtx.destination);

  const osc1 = audioCtx.createOscillator(); osc1.type = 'sine'; osc1.frequency.value = 110;
  const osc2 = audioCtx.createOscillator(); osc2.type = 'sine'; osc2.frequency.value = 116.8;
  const mix = audioCtx.createGain(); mix.gain.value = 0.5;
  osc1.connect(mix); osc2.connect(mix); mix.connect(ambientGain);
  osc1.start(); osc2.start();

  // gentle LFO on gain
  const lfo = audioCtx.createOscillator(); lfo.type = 'sine'; lfo.frequency.value = 0.07;
  const lfoGain = audioCtx.createGain(); lfoGain.gain.value = 0.015;
  lfo.connect(lfoGain); lfoGain.connect(ambientGain.gain); lfo.start();
}

function playSip() {
  if (!audioCtx) startAudio();
  if (!sipEnabled) return;
  const now = audioCtx.currentTime;
  const dur = 0.18;
  const noise = audioCtx.createBufferSource();
  const buffer = audioCtx.createBuffer(1, audioCtx.sampleRate * dur, audioCtx.sampleRate);
  const data = buffer.getChannelData(0);
  for (let i=0;i<data.length;i++) data[i] = (Math.random()*2-1) * (1 - i/data.length);
  noise.buffer = buffer;
  const filt = audioCtx.createBiquadFilter(); filt.type = 'lowpass'; filt.frequency.value = 1800;
  const g = audioCtx.createGain(); g.gain.value = 0.18;
  noise.connect(filt); filt.connect(g); g.connect(audioCtx.destination);
  noise.start(now); noise.stop(now + dur);
}

function preload() {
  // preload painting thumbnails (so modal appears instantly)
  PAINTINGS.forEach(p => this.load.image(p.id, p.dataUrl));
}

function create() {
  const scene = this;
  startAudio();

  // world
  const w = this.scale.width; const h = this.scale.height;
  const g = this.add.graphics();
  const wallThickness = 48;
  g.fillStyle(0xf3efe9,1); g.fillRect(0,0,w,h);
  g.fillStyle(0x2b2b2b,1); g.fillRect(0,0,w,wallThickness); g.fillRect(0,h-wallThickness,w,wallThickness);
  g.fillRect(0,0,wallThickness,h); g.fillRect(w-wallThickness,0,wallThickness,h);

  // paintings container
  paintingsGroup = this.physics.add.staticGroup();
  const margin = 140; let pi=0;
  PAINTINGS.forEach((p,i)=>{
    let x,y;
    if (i%3===0){ x = wallThickness + 140; y = margin + i*30; }
    else if (i%3===1){ x = w/2; y = wallThickness + 120 + i*10; }
    else { x = w - wallThickness - 140; y = margin + i*30; }
    const frame = this.add.rectangle(x,y,180,140,0xffffff).setStrokeStyle(6,0x6b4c3b).setDepth(1);
    const thumb = this.add.image(x,y,p.id).setDisplaySize(160,120).setDepth(2).setInteractive();
    const zone = this.add.zone(x,y,180,140); this.physics.world.enable(zone); zone.body.setAllowGravity(false); zone.body.moves=false;
    zone.paintingId = p.id; zone.meta = p; paintingsGroup.add(zone);

    // clicking/tapping thumbnail -> camera pan then modal
    thumb.on('pointerdown', ()=>{
      centerCameraOnPaintingAndOpen(scene, zone.meta);
    });
  });

  // create NPC visitors
  npcGroup = this.physics.add.group();
  for (let n=0;n<6;n++){
    const nx = Phaser.Math.Between(200, w-200); const ny = Phaser.Math.Between(200, h-200);
    const npc = this.add.circle(nx, ny, 12, 0x7a7a7a).setDepth(4);
    this.physics.add.existing(npc); npc.body.setCollideWorldBounds(true); npc.body.setBounce(1); npc.speed = Phaser.Math.Between(30,70);
    npcGroup.add(npc);
    wanderNpc(this, npc);
  }

  // player (animated via generated textures)
  generatePlayerTextures(this);
  player = this.physics.add.sprite(w/2, h/2, 'player_walk_0').setDepth(6);
  player.setSize(28,36); player.setCollideWorldBounds(true);
  this.anims.create({key:'walk', frames: this.anims.generateFrameNames('player', {start:0,end:3,prefix:'walk_'}), frameRate:10, repeat:-1});
  player.play('walk');
  player.speed = 160;

  // collisions / overlap
  this.physics.add.overlap(player, paintingsGroup, (pl, zone)=>{ currentNearby = zone; });

  // camera
  this.cameras.main.startFollow(player, true, 0.08, 0.08);
  this.cameras.main.setBounds(0,0,w,h);

  // keyboard
  cursors = this.input.keyboard.createCursorKeys();
  interactKey = this.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.E);

  // click on world: try to interact if clicking near a painting
  this.input.on('pointerdown', (pointer)=>{
    const px = pointer.worldX, py = pointer.worldY;
    const found = paintingsGroup.getChildren().find(z=> Phaser.Math.Distance.Between(px,py,z.x,z.y) < 100);
    if (found) centerCameraOnPaintingAndOpen(scene, found.meta);
  });

  // overlay handlers
  closeOverlayBtn.addEventListener('click', ()=>{ closeOverlayAndReturnCamera(scene); });
  overlay.addEventListener('click',(ev)=>{ if (ev.target === overlay) closeOverlayAndReturnCamera(scene); });

  // mobile UI
  if (isMobile){ document.getElementById('touch-controls').style.display='block'; document.getElementById('touch-interact').style.display='block'; document.getElementById('touch-controls').setAttribute('aria-hidden','false'); document.getElementById('touch-interact').setAttribute('aria-hidden','false'); }
  document.querySelectorAll('.dpad .btn').forEach(btn => {
    btn.addEventListener('touchstart', (e)=>{ e.preventDefault(); btn.classList.add('active'); startTouchMove(btn.getAttribute('data-dir')); });
    btn.addEventListener('touchend', (e)=>{ e.preventDefault(); btn.classList.remove('active'); stopTouchMove(btn.getAttribute('data-dir')); });
    btn.addEventListener('mousedown', ()=>{ startTouchMove(btn.getAttribute('data-dir')); });
    btn.addEventListener('mouseup', ()=>{ stopTouchMove(btn.getAttribute('data-dir')); });
  });
  document.getElementById('touch-interact').addEventListener('click', ()=>{ if (currentNearby) centerCameraOnPaintingAndOpen(scene, currentNearby.meta); });

  // minimap setup
  minimapCanvas = document.getElementById('minimap'); minimapCtx = minimapCanvas.getContext('2d');
  window.addEventListener('resize', ()=>{ scene.scale.resize(Math.min(window.innerWidth,1200), Math.min(window.innerHeight,800)); });
}

// simple virtual movement state for touch buttons
const touchState = {up:false,down:false,left:false,right:false};
function startTouchMove(dir){ touchState[dir]=true; }
function stopTouchMove(dir){ touchState[dir]=false; }

function update(time,delta){
  // movement
  let vx=0, vy=0;
  if (cursors.left.isDown) vx = -1; if (cursors.right.isDown) vx = 1; if (cursors.up.isDown) vy = -1; if (cursors.down.isDown) vy = 1;
  if (touchState.left) vx = -1; if (touchState.right) vx = 1; if (touchState.up) vy = -1; if (touchState.down) vy = 1;
  if (vx!==0 && vy!==0){ vx *= Math.SQRT1_2; vy *= Math.SQRT1_2; }
  player.setVelocity(vx * player.speed, vy * player.speed);
  if (vx!==0 || vy!==0){ if (!player.anims.isPlaying) player.play('walk'); } else player.anims.stop();

  // proximity check
  const zones = paintingsGroup.getChildren(); let near=null;
  for (let i=0;i<zones.length;i++){ const z=zones[i]; const dist = Phaser.Math.Distance.Between(player.x,player.y,z.x,z.y); if (dist < 90){ near=z; break; } }
  if (near){ hintEl.style.display='block'; currentNearby = near; if (Phaser.Input.Keyboard.JustDown(interactKey)) centerCameraOnPaintingAndOpen(this, near.meta); }
  else { hintEl.style.display='none'; currentNearby=null; }

  // update minimap
  drawMinimap();
}

function centerCameraOnPaintingAndOpen(scene, paintingMeta){
  // first stop following so we can tween camera
  const cam = scene.cameras.main; cam.stopFollow();
  const targetX = paintingMeta._zoneX || paintingMeta.x || paintingMeta.x; // meta.x - we didn't store; but zone has x
  // find zone object
  let zone = paintingsGroup.getChildren().find(z=> z.meta && z.meta.id === paintingMeta.id);
  if (!zone) zone = {x: scene.cameras.main.midPoint.x, y: scene.cameras.main.midPoint.y};
  const tw = scene.tweens.add({ targets: cam, props: { scrollX: { value: zone.x - cam.width/2, ease:'Sine.easeInOut' }, scrollY: { value: zone.y - cam.height/2, ease:'Sine.easeInOut' }, zoom: { value: 1.6, ease:'Sine.easeInOut' } }, duration: 700, onComplete: ()=>{ openPaintingOverlay(paintingMeta); } });
}

function openPaintingOverlay(p){
  // set image src, show overlay after small delay (to sync with camera motion)
  paintingLarge.src = p.dataUrl; paintingTitle.textContent = p.title || 'Untitled'; paintingDesc.textContent = p.desc || '';
  overlay.classList.remove('hide'); overlay.classList.add('show'); overlay.style.display='flex'; overlay.setAttribute('aria-hidden','false');
  // sip sound
  playSip();
}

function closeOverlayAndReturnCamera(scene){
  overlay.classList.remove('show'); overlay.classList.add('hide'); setTimeout(()=>{ overlay.style.display='none'; overlay.classList.remove('hide'); overlay.setAttribute('aria-hidden','true'); }, 250);
  // tween camera back to player
  const cam = scene.cameras.main; scene.tweens.add({ targets: cam, props: { scrollX: { value: player.x - cam.width/2, ease:'Sine.easeInOut' }, scrollY: { value: player.y - cam.height/2, ease:'Sine.easeInOut' }, zoom: { value: 1, ease:'Sine.easeInOut' } }, duration: 700, onComplete: ()=>{ cam.startFollow(player, true, 0.08, 0.08); } });
}

function drawMinimap(){
  if (!minimapCtx) return; const w = game.scale.width; const h = game.scale.height; const mw = minimapCanvas.width; const mh = minimapCanvas.height;
  minimapCtx.clearRect(0,0,mw,mh); minimapCtx.fillStyle = 'rgba(255,255,255,0.04)'; minimapCtx.fillRect(0,0,mw,mh);
  // scale positions (we'll assume world is same as camera bounds used earlier)
  const scaleX = mw / Math.max(w, mw); const scaleY = mh / Math.max(h, mh);
  // draw paintings
  paintingsGroup.getChildren().forEach(z=>{
    const x = Math.round((z.x / w) * mw); const y = Math.round((z.y / h) * mh);
    minimapCtx.fillStyle = 'rgba(203,153,101,0.9)'; minimapCtx.fillRect(x-3,y-3,6,6);
  });
  // draw NPCs
  npcGroup.getChildren().forEach(n=>{
    const x = Math.round((n.x / w) * mw); const y = Math.round((n.y / h) * mh);
    minimapCtx.fillStyle = 'rgba(120,120,120,0.9)'; minimapCtx.fillRect(x-2,y-2,4,4);
  });
  // draw player
  const px = Math.round((player.x / w) * mw); const py = Math.round((player.y / h) * mh);
  minimapCtx.fillStyle = '#ffde59'; minimapCtx.beginPath(); minimapCtx.arc(px,py,4,0,Math.PI*2); minimapCtx.fill();
}

function wanderNpc(scene, npc){
  const w = scene.scale.width; const h = scene.scale.height;
  const tx = Phaser.Math.Between(100, w-100); const ty = Phaser.Math.Between(100, h-100);
  scene.tweens.add({ targets: npc, x: tx, y: ty, duration: Phaser.Math.Between(3000,9000), ease:'Sine.easeInOut', onComplete: ()=>{ wanderNpc(scene, npc); } });
}

// Procedurally generate a small 4-frame player walking sprite using Phaser Graphics
function generatePlayerTextures(scene){
  const baseKey = 'player';
  const rtW = 64, rtH = 64;
  for (let f=0; f<4; f++){
    const rt = scene.add.renderTexture(0,0,rtW,rtH);
    const g = scene.add.graphics();
    // body
    g.fillStyle(0x2e2e2e,1); g.fillCircle(32,22,10);
    g.fillRoundedRect(24,32,16,20,6);
    // wine glass (changes slightly per frame)
    g.fillStyle(0xffffff,1); g.fillRect(40,20,2,12 + (f%2)); g.fillEllipse(41,18,12,8);
    g.fillStyle(0x7b2b2b,1); g.fillEllipse(41,16,8,4);
    // arm movement
    g.lineStyle(3,0x2e2e2e); const ax = 30 - f*1; const ay = 36 + f*1; g.strokeLineShape(new Phaser.Geom.Line(30,36,ax,ay));
    rt.draw(g); rt.saveTexture(`${baseKey}_walk_${f}`);
    g.destroy(); rt.destroy();
  }
  // create animation frames
  scene.textures.addSpriteSheetFromAtlas('player', { atlas: { frame: null }, frameWidth: rtW, frameHeight: rtH });
  // Manually create frame names so animation system in create() can reference them
  scene.anims.create({ key: 'walk', frames: [ { key: `${baseKey}_walk_0` },{ key: `${baseKey}_walk_1` },{ key: `${baseKey}_walk_2` },{ key: `${baseKey}_walk_3` } ], frameRate: 10, repeat: -1 });
}

// allow user gesture to start audio if needed
window.addEventListener('click', ()=>{ if (audioCtx && audioCtx.state === 'suspended') audioCtx.resume(); });

</script>
</body>
</html>
