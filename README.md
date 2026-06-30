## IN PROGRESS!

# basic view till now
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>FLORA</title>
<style>
*{margin:0;padding:0;box-sizing:border-box;}
html,body{width:100%;height:100%;background:#0a0608;overflow:hidden;}

#webcam{
  position:fixed;inset:0;width:100%;height:100%;
  object-fit:cover;transform:scaleX(-1);
  opacity:0;transition:opacity 2s ease;
  pointer-events:none;z-index:0;
}
#webcam.on{opacity:1;}

#overlay{
  position:fixed;inset:0;z-index:1;pointer-events:none;
  background:rgba(6,3,8,0.52);
}

#canvas{
  position:fixed;inset:0;z-index:2;pointer-events:none;
}

/* vignette */
#vignette{
  position:fixed;inset:0;z-index:3;pointer-events:none;
  background:radial-gradient(ellipse 80% 80% at 50% 50%,
    transparent 35%, rgba(4,2,6,0.65) 100%);
}

/* UI */
#ui{position:fixed;inset:0;z-index:10;pointer-events:none;}

#wordmark{
  position:absolute;top:28px;left:36px;
}
#wordmark-title{
  font-family:'Didact Gothic',Georgia,serif;
  font-size:22px;letter-spacing:0.35em;
  color:rgba(255,210,140,0.82);text-transform:uppercase;
  font-weight:400;
}
#wordmark-sub{
  margin-top:5px;font-size:10px;letter-spacing:0.22em;
  color:rgba(255,180,80,0.32);font-style:italic;
  font-family:Georgia,serif;
}

#status{
  position:absolute;top:30px;right:36px;
  text-align:right;
  font-family:Georgia,serif;font-style:italic;
  font-size:11px;letter-spacing:0.16em;
  color:rgba(255,200,100,0.35);
  transition:color 0.6s;
}
#status.active{color:rgba(255,200,100,0.7);}

#seed-ring{
  position:fixed;pointer-events:none;z-index:8;
  width:48px;height:48px;
  border-radius:50%;
  border:1px solid rgba(255,180,60,0);
  transform:translate(-50%,-50%);
  transition:none;
}

#controls{
  position:absolute;bottom:26px;right:32px;
  display:flex;gap:12px;pointer-events:all;
}
.btn{
  background:rgba(255,180,60,0.07);
  border:0.5px solid rgba(255,180,60,0.25);
  color:rgba(255,200,120,0.55);
  font-family:Georgia,serif;font-style:italic;
  font-size:11px;letter-spacing:0.1em;
  padding:7px 18px;border-radius:20px;cursor:pointer;
  backdrop-filter:blur(6px);transition:all 0.3s;
}
.btn:hover{
  background:rgba(255,180,60,0.15);
  color:rgba(255,210,140,0.9);
  border-color:rgba(255,180,60,0.55);
}

#hint{
  position:absolute;bottom:30px;left:36px;
  font-family:Georgia,serif;font-style:italic;
  font-size:11px;letter-spacing:0.13em;
  color:rgba(255,180,60,0.28);
  transition:opacity 1.2s;
}
#hint.gone{opacity:0;}

/* Permission screen */
#gate{
  position:fixed;inset:0;z-index:100;
  display:flex;flex-direction:column;align-items:center;justify-content:center;
  background:radial-gradient(ellipse at center,#110a0e 0%,#060208 100%);
  transition:opacity 1.4s ease;
}
#gate.out{opacity:0;pointer-events:none;}
#gate-title{
  font-family:Georgia,serif;font-size:42px;
  letter-spacing:0.38em;text-transform:uppercase;
  color:rgba(255,210,140,0.88);margin-bottom:12px;font-weight:400;
}
#gate-sub{
  font-size:12px;letter-spacing:0.2em;
  color:rgba(255,180,80,0.32);font-style:italic;
  font-family:Georgia,serif;margin-bottom:52px;
}
#gate-btn{
  background:transparent;
  border:0.5px solid rgba(255,180,60,0.4);
  color:rgba(255,200,120,0.72);
  font-family:Georgia,serif;font-style:italic;
  font-size:13px;letter-spacing:0.18em;
  padding:14px 44px;border-radius:30px;cursor:pointer;
  transition:all 0.4s;
}
#gate-btn:hover{
  background:rgba(255,180,60,0.1);
  color:rgba(255,220,150,1);
  border-color:rgba(255,180,60,0.75);
}
#gate-err{
  margin-top:18px;font-size:11px;
  color:rgba(255,80,80,0.55);letter-spacing:0.1em;display:none;
}
</style>
</head>
<body>

<div id="gate">
  <div id="gate-title">Flora</div>
  <div id="gate-sub">Hold still. Let things grow.</div>
  <button id="gate-btn" onclick="boot()">Allow Camera &amp; Enter</button>
  <div id="gate-err">Camera access denied — please allow and reload.</div>
</div>

<video id="webcam" autoplay playsinline muted></video>
<div id="overlay"></div>
<canvas id="canvas"></canvas>
<div id="vignette"></div>

<div id="ui">
  <div id="wordmark">
    <div id="wordmark-title">Flora</div>
    <div id="wordmark-sub">Hold still. Let things grow.</div>
  </div>
  <div id="status">waiting for hands</div>
  <div id="hint">linger to plant · pinch to bloom · open palm to release</div>
  <div id="controls">
    <button class="btn" id="btn-cam">Hide Camera</button>
    <button class="btn" id="btn-clear">Clear Garden</button>
    <button class="btn" id="btn-fs">Fullscreen</button>
  </div>
</div>
<div id="seed-ring"></div>

<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script>
/* ═══════════════════════════════════════════════════════════
   FLORA — Hand-tracked generative flower garden
   Canvas 2D · Sharp bezier petals · L-system stems · MediaPipe
   ═══════════════════════════════════════════════════════════ */

const canvas  = document.getElementById('canvas');
const ctx     = canvas.getContext('2d');
let W = canvas.width  = window.innerWidth;
let H = canvas.height = window.innerHeight;
window.addEventListener('resize', () => {
  W = canvas.width  = window.innerWidth;
  H = canvas.height = window.innerHeight;
});

/* ─── palette ─── */
const PAL = {
  petalInner : '#ffe8a0',
  petalMid   : '#f4a040',
  petalOuter : '#b83010',
  petalEdge  : '#ff9030',
  stemBlue   : '#4488ff',
  stemGlow   : '#2255cc',
  centerCore : '#fff5c0',
  centerGlow : '#ffcc40',
};

/* ─── hand state ─── */
const hands = [];        // processed hand objects
let   handsPresent = false;

/* ─── garden ─── */
const flowers = [];      // all planted flowers
const MAX_FLOWERS = 14;

/* ─── dwell tracking (for planting) ─── */
const dwells = [];       // { x, y, t, id } one per fingertip

/* ─── mouse fallback ─── */
let mousePos = null;
let mouseFallback = false;

/* ═══════════════════════════════════════════════════════════
   FLOWER CLASS
   ═══════════════════════════════════════════════════════════ */
let _flowerUID = 0;

class Flower {
  constructor(rx, ry) {
    this.id         = ++_flowerUID;
    this.rootX      = rx;
    this.rootY      = ry;
    this.openness   = 0;       // 0=bud, 1=full bloom
    this.targetOpen = 0;
    this.age        = 0;
    this.petalCount = 7 + Math.floor(Math.random() * 4);
    this.petalLen   = 38 + Math.random() * 28;
    this.petalWid   = 0.38 + Math.random() * 0.18;
    this.rotation   = Math.random() * Math.PI * 2;
    this.rotSpeed   = (Math.random() - 0.5) * 0.003;
    this.stemHeight = 90 + Math.random() * 120;
    this.stemNodes  = this._buildStem();
    this.branchFlowers = []; // child flowers on branches
    this.pulse      = Math.random() * Math.PI * 2;
    this.pulseSpeed = 0.018 + Math.random() * 0.012;
    this.petalHueShift = (Math.random() - 0.5) * 20;
    this.born       = performance.now();
    this.dying      = false;
    this.opacity    = 0;
  }

  /* ─── L-system stem ─── */
  _buildStem() {
    const nodes = [];
    // Main stem: grows upward with slight sway
    const segments = 4 + Math.floor(Math.random() * 3);
    let cx = this.rootX, cy = this.rootY;
    const segH = this.stemHeight / segments;
    nodes.push({ x: cx, y: cy, branch: false, w: 2.8 });
    for (let i = 1; i <= segments; i++) {
      const sway = Math.sin(i * 1.2 + this.rotation) * 14;
      cx += sway;
      cy -= segH;
      const isBranch = i > 1 && i < segments && Math.random() < 0.55;
      nodes.push({ x: cx, y: cy, branch: isBranch, w: 2.8 - i * 0.38, branchDir: Math.random() < 0.5 ? 1 : -1 });
    }
    this.headX = cx;
    this.headY = cy;
    return nodes;
  }

  update(dt, nearestHandDist, pinching, openPalm) {
    this.age   += dt;
    this.pulse += this.pulseSpeed;
    this.rotation += this.rotSpeed;
    this.opacity = Math.min(1, this.opacity + dt * 1.2);

    // Proximity-based openness
    const proxOpen = Math.max(0, 1 - nearestHandDist / 220);
    this.targetOpen = Math.max(proxOpen, this.age > 1.5 ? 0.25 : 0);
    if (pinching) this.targetOpen = 1;
    if (this.dying) this.targetOpen = 0;

    this.openness += (this.targetOpen - this.openness) * dt * 2.2;
    this.openness  = Math.max(0, Math.min(1, this.openness));
  }

  draw(ctx) {
    const alpha = this.opacity * (this.dying ? Math.max(0, 1 - this.age * 0.6) : 1);
    if (alpha < 0.01) return;
    ctx.save();
    ctx.globalAlpha = alpha;

    this._drawStem(ctx);
    this._drawFlowerHead(ctx, this.headX, this.headY, 1);

    ctx.restore();
  }

  _drawStem(ctx) {
    const nodes = this.stemNodes;
    if (nodes.length < 2) return;

    // Glow pass
    ctx.save();
    ctx.shadowColor = PAL.stemGlow;
    ctx.shadowBlur  = 12;
    ctx.strokeStyle = PAL.stemBlue;
    ctx.lineWidth   = nodes[0].w * 1.4;
    ctx.lineCap     = 'round';
    ctx.lineJoin    = 'round';
    ctx.beginPath();
    ctx.moveTo(nodes[0].x, nodes[0].y);
    for (let i = 1; i < nodes.length; i++) {
      const prev = nodes[i - 1], curr = nodes[i];
      const mx   = (prev.x + curr.x) / 2;
      const my   = (prev.y + curr.y) / 2;
      ctx.quadraticCurveTo(prev.x, prev.y, mx, my);
    }
    ctx.lineTo(nodes[nodes.length-1].x, nodes[nodes.length-1].y);
    ctx.stroke();
    ctx.restore();

    // Crisp inner line
    ctx.save();
    ctx.strokeStyle = `rgba(100,160,255,0.6)`;
    ctx.lineWidth   = nodes[0].w * 0.5;
    ctx.lineCap     = 'round';
    ctx.beginPath();
    ctx.moveTo(nodes[0].x, nodes[0].y);
    for (let i = 1; i < nodes.length; i++) {
      const prev = nodes[i - 1], curr = nodes[i];
      ctx.quadraticCurveTo(prev.x, prev.y, (prev.x+curr.x)/2, (prev.y+curr.y)/2);
    }
    ctx.lineTo(nodes[nodes.length-1].x, nodes[nodes.length-1].y);
    ctx.stroke();
    ctx.restore();

    // Branch sub-flowers
    for (let i = 1; i < nodes.length - 1; i++) {
      const n = nodes[i];
      if (!n.branch) continue;
      const prev   = nodes[i - 1];
      const branchLen = 28 + Math.random() * 22;
      const dx     = n.x - prev.x, dy = n.y - prev.y;
      const ang    = Math.atan2(dy, dx) + n.branchDir * (Math.PI * 0.38);
      const bx     = n.x + Math.cos(ang) * branchLen * this.openness;
      const by     = n.y + Math.sin(ang) * branchLen * this.openness;

      // Branch line
      ctx.save();
      ctx.shadowColor = PAL.stemGlow;
      ctx.shadowBlur  = 8;
      ctx.strokeStyle = PAL.stemBlue;
      ctx.lineWidth   = n.w * 0.7;
      ctx.lineCap     = 'round';
      ctx.beginPath();
      ctx.moveTo(n.x, n.y);
      ctx.quadraticCurveTo(
        n.x + Math.cos(ang) * branchLen * 0.5,
        n.y + Math.sin(ang) * branchLen * 0.5,
        bx, by
      );
      ctx.stroke();
      ctx.restore();

      // Small flower at branch tip
      if (this.openness > 0.15) {
        this._drawFlowerHead(ctx, bx, by, 0.52);
      }
    }
  }

  _drawFlowerHead(ctx, hx, hy, scale) {
    const open  = this.openness;
    const plen  = this.petalLen * scale;
    const pulse = 1 + 0.04 * Math.sin(this.pulse);
    const rot   = this.rotation;
    const n     = this.petalCount;

    if (open < 0.02) return;

    /* ── petals ── */
    for (let p = 0; p < n; p++) {
      const ang    = rot + (p / n) * Math.PI * 2;
      const spread = open * Math.PI * 0.18; // how wide petals fan out
      const tiltAng = ang;

      ctx.save();
      ctx.translate(hx, hy);
      ctx.rotate(tiltAng);

      // Each petal: bezier teardrop shape
      const pl  = plen * open * pulse;
      const pw  = pl * this.petalWid;

      // Glow behind petal
      ctx.save();
      ctx.shadowColor = PAL.petalEdge;
      ctx.shadowBlur  = 14 * open;
      this._drawPetal(ctx, pl, pw, p, n, open);
      ctx.restore();

      // Sharp petal on top
      ctx.shadowBlur = 0;
      this._drawPetal(ctx, pl, pw, p, n, open);

      // Inner vein line
      if (open > 0.3) {
        ctx.beginPath();
        ctx.moveTo(0, 0);
        ctx.quadraticCurveTo(pw * 0.1, -pl * 0.5, 0, -pl * 0.95);
        ctx.strokeStyle = `rgba(255,240,160,${open * 0.55})`;
        ctx.lineWidth   = 0.7;
        ctx.shadowColor = '#fff8c0';
        ctx.shadowBlur  = 4;
        ctx.stroke();
      }

      ctx.restore();
    }

    /* ── center ── */
    this._drawCenter(ctx, hx, hy, scale * open);
  }

  _drawPetal(ctx, pl, pw, petalIdx, total, open) {
    // Teardrop shape: start at center, bulge out, come to point
    const t   = petalIdx / total;
    // Color gradient across petals (slight variation)
    const warm = 0.15 + t * 0.25;

    // Filled petal
    ctx.beginPath();
    ctx.moveTo(0, 0);
    ctx.bezierCurveTo(
      -pw,  -pl * 0.35,
      -pw * 0.7, -pl * 0.75,
       0,   -pl
    );
    ctx.bezierCurveTo(
       pw * 0.7, -pl * 0.75,
       pw,  -pl * 0.35,
       0,    0
    );
    ctx.closePath();

    // Radial fill: bright inner, dark outer
    const grad = ctx.createLinearGradient(0, 0, 0, -pl);
    grad.addColorStop(0,   `rgba(255, 210, 90, ${0.85 * open})`);
    grad.addColorStop(0.35,`rgba(240, 130, 30, ${0.9  * open})`);
    grad.addColorStop(0.75,`rgba(180,  55, 12, ${0.88 * open})`);
    grad.addColorStop(1,   `rgba(120,  20,  5, ${0.6  * open})`);
    ctx.fillStyle = grad;
    ctx.fill();

    // Sharp outline
    ctx.strokeStyle = `rgba(255, 160, 40, ${0.55 * open})`;
    ctx.lineWidth   = 0.8;
    ctx.stroke();
  }

  _drawCenter(ctx, hx, hy, scale) {
    if (scale < 0.05) return;
    const r     = 7 * scale;
    const pulse = 1 + 0.08 * Math.sin(this.pulse * 2);

    // Outer glow
    ctx.save();
    ctx.shadowColor = PAL.centerGlow;
    ctx.shadowBlur  = 22 * scale;
    const og = ctx.createRadialGradient(hx, hy, 0, hx, hy, r * 2.5 * pulse);
    og.addColorStop(0,   `rgba(255,250,200,${0.95 * scale})`);
    og.addColorStop(0.4, `rgba(255,200, 60,${0.75 * scale})`);
    og.addColorStop(1,   `rgba(255,140, 10,0)`);
    ctx.beginPath();
    ctx.arc(hx, hy, r * 2.5 * pulse, 0, Math.PI * 2);
    ctx.fillStyle = og;
    ctx.fill();
    ctx.restore();

    // Hard core
    ctx.save();
    ctx.shadowColor = '#fff';
    ctx.shadowBlur  = 8;
    const cg = ctx.createRadialGradient(hx, hy, 0, hx, hy, r * pulse);
    cg.addColorStop(0,   `rgba(255,255,240,1)`);
    cg.addColorStop(0.6, `rgba(255,220,100,0.9)`);
    cg.addColorStop(1,   `rgba(255,160, 30,0.5)`);
    ctx.beginPath();
    ctx.arc(hx, hy, r * pulse, 0, Math.PI * 2);
    ctx.fillStyle = cg;
    ctx.fill();
    ctx.restore();
  }
}

/* ═══════════════════════════════════════════════════════════
   AMBIENT SPORES (tiny floating motes)
   ═══════════════════════════════════════════════════════════ */
const spores = Array.from({ length: 80 }, () => ({
  x    : Math.random() * W,
  y    : Math.random() * H,
  vx   : (Math.random() - 0.5) * 0.3,
  vy   : -0.1 - Math.random() * 0.25,
  r    : 0.6 + Math.random() * 1.4,
  life : Math.random(),
  maxL : 0.4 + Math.random() * 0.4,
}));

function updateSpores(dt) {
  for (const s of spores) {
    s.x    += s.vx + Math.sin(s.life * 4) * 0.15;
    s.y    += s.vy;
    s.life -= dt * 0.04;
    if (s.life < 0 || s.y < -10) {
      s.x    = Math.random() * W;
      s.y    = H + 10;
      s.life = s.maxL;
      s.vx   = (Math.random() - 0.5) * 0.3;
    }
  }
}

function drawSpores() {
  for (const s of spores) {
    const a = s.life * 0.6;
    if (a < 0.01) continue;
    ctx.save();
    ctx.shadowColor = 'rgba(255,200,80,0.5)';
    ctx.shadowBlur  = 4;
    ctx.beginPath();
    ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2);
    ctx.fillStyle = `rgba(255,210,120,${a})`;
    ctx.fill();
    ctx.restore();
  }
}

/* ═══════════════════════════════════════════════════════════
   DWELL / PLANTING SYSTEM
   ═══════════════════════════════════════════════════════════ */
const DWELL_MS     = 900;   // ms to hold before planting
const DWELL_RADIUS = 28;    // px movement tolerance

function updateDwells(fingertips, dt, pinching) {
  const now = performance.now();

  // Update or create dwell points
  for (const tip of fingertips) {
    let matched = false;
    for (const d of dwells) {
      const dist = Math.hypot(tip.x - d.x, tip.y - d.y);
      if (dist < DWELL_RADIUS) {
        d.x = tip.x * 0.1 + d.x * 0.9; // smooth
        d.y = tip.y * 0.1 + d.y * 0.9;
        matched = true;
        const elapsed = now - d.t;
        const progress = Math.min(1, elapsed / DWELL_MS);

        // Draw ring progress indicator
        drawSeedRing(d.x, d.y, progress, pinching);

        // Plant!
        if (elapsed > DWELL_MS && !d.planted) {
          d.planted = true;
          plantFlower(d.x, d.y);
        }
        break;
      }
    }
    if (!matched) {
      dwells.push({ x: tip.x, y: tip.y, t: now, planted: false });
    }
  }

  // Remove stale dwells
  for (let i = dwells.length - 1; i >= 0; i--) {
    const age = now - dwells[i].t;
    if (age > DWELL_MS * 2.5) dwells.splice(i, 1);
  }
}

/* ═══════════════════════════════════════════════════════════
   SEED RING — elegant circular progress
   ═══════════════════════════════════════════════════════════ */
function drawSeedRing(x, y, progress, pinching) {
  if (progress <= 0 || progress >= 1) return;
  ctx.save();
  ctx.translate(x, y);

  // Background ring
  ctx.beginPath();
  ctx.arc(0, 0, 20, 0, Math.PI * 2);
  ctx.strokeStyle = 'rgba(255,180,60,0.15)';
  ctx.lineWidth   = 1;
  ctx.stroke();

  // Progress arc
  ctx.beginPath();
  ctx.arc(0, 0, 20, -Math.PI / 2, -Math.PI / 2 + progress * Math.PI * 2);
  ctx.strokeStyle = `rgba(255,200,80,${0.5 + progress * 0.5})`;
  ctx.lineWidth   = 1.5;
  ctx.shadowColor = 'rgba(255,180,60,0.8)';
  ctx.shadowBlur  = 8;
  ctx.stroke();

  // Center dot
  ctx.beginPath();
  ctx.arc(0, 0, 2, 0, Math.PI * 2);
  ctx.fillStyle = `rgba(255,220,120,${progress * 0.8})`;
  ctx.fill();

  ctx.restore();
}

/* ═══════════════════════════════════════════════════════════
   PLANT A FLOWER
   ═══════════════════════════════════════════════════════════ */
function plantFlower(x, y) {
  // Don't plant too close to existing flowers
  for (const f of flowers) {
    if (Math.hypot(f.rootX - x, f.rootY - y) < 60) return;
  }
  if (flowers.length >= MAX_FLOWERS) {
    // Remove oldest
    flowers.shift();
  }
  flowers.push(new Flower(x, y));
}

/* ═══════════════════════════════════════════════════════════
   MEDIAPIPE HAND TRACKING
   ═══════════════════════════════════════════════════════════ */
let mpLoaded = false;

function initMediaPipe(video) {
  const mp = new Hands({
    locateFile: f => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${f}`
  });
  mp.setOptions({
    maxNumHands        : 2,
    modelComplexity    : 1,
    minDetectionConfidence : 0.72,
    minTrackingConfidence  : 0.65,
  });
  mp.onResults(onResults);

  const cam = new Camera(video, {
    onFrame : async () => { await mp.send({ image: video }); },
    width   : 640,
    height  : 480,
  });
  cam.start().then(() => { mpLoaded = true; });
}

const SMOOTH = 0.15;
const prevPalms = {};

function onResults(results) {
  const lms = results.multiHandLandmarks;
  handsPresent = lms && lms.length > 0;
  hands.length = 0;
  if (!handsPresent) return;

  document.getElementById('status').classList.add('active');
  document.getElementById('status').textContent = lms.length === 1 ? 'one hand' : 'two hands';

  lms.forEach((lm, hi) => {
    // Mirror x (CSS mirrors video, so we flip landmark x)
    const mx = x => 1 - x;

    const palmRaw = { x: mx(lm[9].x) * W, y: lm[9].y * H };
    const prev    = prevPalms[hi] || palmRaw;
    const palm    = {
      x: prev.x + (palmRaw.x - prev.x) * SMOOTH,
      y: prev.y + (palmRaw.y - prev.y) * SMOOTH,
    };
    prevPalms[hi] = palm;

    // Velocity
    const vel = Math.hypot(palm.x - prev.x, palm.y - prev.y);

    // Fingertips: landmarks 4,8,12,16,20
    const tips = [4,8,12,16,20].map(i => ({
      x: mx(lm[i].x) * W,
      y: lm[i].y * H,
    }));

    // Index tip (8) only for planting
    const indexTip = { x: mx(lm[8].x) * W, y: lm[8].y * H };

    // Pinch: thumb(4) <-> index(8)
    const t4 = { x: mx(lm[4].x)*W, y: lm[4].y*H };
    const t8 = { x: mx(lm[8].x)*W, y: lm[8].y*H };
    const pinchDist     = Math.hypot(t4.x-t8.x, t4.y-t8.y);
    const pinching      = pinchDist < 42;
    const pinchStrength = Math.max(0, 1 - pinchDist / 55);

    // Open palm: all 4 fingertips above their MCPs
    let extended = 0;
    [8,12,16,20].forEach((tipI, ki) => {
      const mcpI = [5,9,13,17][ki];
      if (lm[tipI].y < lm[mcpI].y - 0.03) extended++;
    });
    const openPalm = extended >= 3;

    hands.push({ palm, tips, indexTip, pinching, pinchStrength, openPalm, vel });
  });
}

/* ═══════════════════════════════════════════════════════════
   MAIN LOOP
   ═══════════════════════════════════════════════════════════ */
let lastTime = performance.now();
let hintTimer;

function loop() {
  requestAnimationFrame(loop);
  const now = performance.now();
  const dt  = Math.min((now - lastTime) / 1000, 0.05);
  lastTime  = now;

  ctx.clearRect(0, 0, W, H);

  const activePts  = getActivePoints();
  const pinching   = hands.some(h => h.pinching);
  const openPalm   = hands.some(h => h.openPalm);

  updateSpores(dt);
  drawSpores();

  // Update & draw flowers
  for (let i = flowers.length - 1; i >= 0; i--) {
    const f    = flowers[i];
    const dist = nearestHandDist(f.headX, f.headY);
    f.update(dt, dist, pinching, openPalm);

    if (f.dying && f.age > 4) { flowers.splice(i, 1); continue; }
    f.draw(ctx);
  }

  // Dwell / planting (only index fingertip of each hand)
  if (handsPresent || mouseFallback) {
    const plantTips = mouseFallback && mousePos
      ? [mousePos]
      : hands.map(h => h.indexTip);
    updateDwells(plantTips, dt, pinching);
  }

  // Hand skeleton overlay (optional, very subtle)
  if (handsPresent) drawHandDots();

  // Status
  if (!handsPresent && !mouseFallback) {
    document.getElementById('status').classList.remove('active');
    document.getElementById('status').textContent = 'waiting for hands';
  }
}

function getActivePoints() {
  if (mouseFallback && mousePos) return [mousePos];
  return hands.map(h => h.palm);
}

function nearestHandDist(x, y) {
  if (mouseFallback && mousePos) return Math.hypot(mousePos.x - x, mousePos.y - y);
  if (!hands.length) return 9999;
  return Math.min(...hands.flatMap(h =>
    [h.palm, ...h.tips].map(p => Math.hypot(p.x - x, p.y - y))
  ));
}

/* subtle fingertip dots */
function drawHandDots() {
  for (const h of hands) {
    for (const tip of h.tips) {
      ctx.save();
      ctx.beginPath();
      ctx.arc(tip.x, tip.y, 3.5, 0, Math.PI * 2);
      ctx.fillStyle = 'rgba(255,200,100,0.22)';
      ctx.shadowColor = 'rgba(255,180,60,0.5)';
      ctx.shadowBlur  = 8;
      ctx.fill();
      ctx.restore();
    }
    // Palm dot
    ctx.save();
    ctx.beginPath();
    ctx.arc(h.palm.x, h.palm.y, 5, 0, Math.PI * 2);
    ctx.fillStyle = 'rgba(255,180,60,0.12)';
    ctx.fill();
    ctx.restore();
  }
}

/* ═══════════════════════════════════════════════════════════
   BOOT
   ═══════════════════════════════════════════════════════════ */
async function boot() {
  const video = document.getElementById('webcam');
  const err   = document.getElementById('gate-err');
  try {
    const stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: 'user', width: 1280, height: 720 }, audio: false
    });
    video.srcObject = stream;
    await new Promise(r => video.onloadedmetadata = r);
    video.classList.add('on');
    document.getElementById('gate').classList.add('out');
    setTimeout(() => document.getElementById('gate').remove(), 1600);

    initMediaPipe(video);
    initControls();
    loop();

    // Hint hide
    hintTimer = setTimeout(() => {
      document.getElementById('hint').classList.add('gone');
    }, 7000);

  } catch(e) {
    console.error(e);
    err.style.display = 'block';
    // Fall back to mouse
    mouseFallback = true;
    document.getElementById('gate').classList.add('out');
    setTimeout(() => document.getElementById('gate').remove(), 1600);
    initControls();
    loop();
  }
}

/* ─── Mouse fallback ─── */
window.addEventListener('mousemove', e => {
  if (!mouseFallback) return;
  mousePos = { x: e.clientX, y: e.clientY };
  handsPresent = true;
});

/* ─── Controls ─── */
function initControls() {
  let camOn = true;
  document.getElementById('btn-cam').addEventListener('click', () => {
    camOn = !camOn;
    document.getElementById('webcam').style.opacity = camOn ? '1' : '0';
    document.getElementById('btn-cam').textContent  = camOn ? 'Hide Camera' : 'Show Camera';
  });
  document.getElementById('btn-clear').addEventListener('click', () => {
    flowers.forEach(f => { f.dying = true; f.age = 0; });
  });
  document.getElementById('btn-fs').addEventListener('click', () => {
    if (!document.fullscreenElement) document.documentElement.requestFullscreen?.();
    else document.exitFullscreen?.();
  });
}
</script>
</body>
</html>
