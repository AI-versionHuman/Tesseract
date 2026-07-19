# Tesseract

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Shadow of a Shadow — Tesseract Explorer</title>
<style>
  :root {
    --bg: #0B0E1A;
    --panel: #121629;
    --line: #2A3050;
    --text: #C7CCE0;
    --dim: #6B7394;
    --near: #FFB454;
    --far: #5B8DEF;
  }
  * { margin: 0; padding: 0; box-sizing: border-box; }
  html, body { height: 100%; }
  body {
    background: var(--bg);
    color: var(--text);
    font-family: ui-monospace, "SF Mono", "Cascadia Mono", Menlo, Consolas, monospace;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    -webkit-user-select: none;
    user-select: none;
  }
  header {
    padding: 14px 18px 10px;
    border-bottom: 1px solid var(--line);
    flex-shrink: 0;
  }
  header h1 {
    font-size: 15px;
    font-weight: 600;
    letter-spacing: 0.06em;
    color: #fff;
  }
  header p {
    font-size: 11px;
    color: var(--dim);
    margin-top: 3px;
    line-height: 1.5;
  }
  header .legend {
    display: flex;
    gap: 16px;
    margin-top: 8px;
    font-size: 10px;
    color: var(--dim);
  }
  .legend span { display: flex; align-items: center; gap: 5px; }
  .swatch { width: 14px; height: 3px; border-radius: 2px; display: inline-block; }
  #stage {
    flex: 1;
    position: relative;
    min-height: 0;
    touch-action: none;
    cursor: grab;
  }
  #stage:active { cursor: grabbing; }
  canvas { display: block; width: 100%; height: 100%; }
  #hint {
    position: absolute;
    bottom: 10px;
    left: 0; right: 0;
    text-align: center;
    font-size: 10px;
    color: var(--dim);
    pointer-events: none;
    transition: opacity 0.6s;
  }
  #controls {
    flex-shrink: 0;
    border-top: 1px solid var(--line);
    background: var(--panel);
    padding: 12px 16px 16px;
  }
  .row {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
  }
  .row + .row { margin-top: 10px; }
  .rowlabel {
    font-size: 10px;
    letter-spacing: 0.1em;
    color: var(--dim);
    text-transform: uppercase;
    width: 100%;
  }
  button {
    background: transparent;
    border: 1px solid var(--line);
    color: var(--text);
    font-family: inherit;
    font-size: 12px;
    padding: 8px 14px;
    border-radius: 6px;
    cursor: pointer;
    transition: border-color 0.15s, color 0.15s, background 0.15s;
  }
  button:hover { border-color: var(--far); }
  button:focus-visible { outline: 2px solid var(--near); outline-offset: 2px; }
  button.on {
    border-color: var(--near);
    color: var(--near);
    background: rgba(255, 180, 84, 0.08);
  }
  button.shape { font-size: 12px; }
  button.shape.on {
    border-color: var(--far);
    color: #fff;
    background: rgba(91, 141, 239, 0.15);
  }
  .note {
    font-size: 10px;
    color: var(--dim);
    line-height: 1.5;
    margin-top: 10px;
  }
  .note em { color: var(--near); font-style: normal; }
  @media (max-height: 620px) {
    header p { display: none; }
  }
</style>
</head>
<body>

<header>
  <h1>SHADOW OF A SHADOW</h1>
  <p>A 4D object, projected into 3D, drawn on your 2D screen. Color is the fourth axis.</p>
  <div class="legend">
    <span><span class="swatch" style="background:var(--near)"></span>near in W (toward you, 4th axis)</span>
    <span><span class="swatch" style="background:var(--far)"></span>far in W</span>
  </div>
</header>

<div id="stage">
  <canvas id="c"></canvas>
  <div id="hint">drag to turn the 3D shadow &middot; buttons below turn it through the 4th dimension</div>
</div>

<div id="controls">
  <div class="row">
    <span class="rowlabel">Object</span>
    <button class="shape on" data-shape="tesseract">Tesseract (4D)</button>
    <button class="shape" data-shape="cube">Cube (3D, for comparison)</button>
  </div>
  <div class="row">
    <span class="rowlabel">Rotation planes &mdash; the W ones have no 3D equivalent</span>
    <button class="rot" data-plane="xy">XY</button>
    <button class="rot on" data-plane="xw">XW</button>
    <button class="rot on" data-plane="yw">YW</button>
    <button class="rot" data-plane="zw">ZW</button>
  </div>
  <p class="note" id="noteText">Watch the inner cube: it isn't smaller, it's <em>farther away in W</em>. When the object rotates through a W-plane, inner and outer cubes trade places &mdash; the "inside-out" move that's impossible for any 3D object.</p>
</div>

<script>
(function () {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  const stage = document.getElementById('stage');
  const hint = document.getElementById('hint');
  const noteText = document.getElementById('noteText');
  const reduceMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  let W = 0, H = 0, DPR = 1;
  function resize() {
    DPR = Math.min(window.devicePixelRatio || 1, 2);
    W = stage.clientWidth; H = stage.clientHeight;
    canvas.width = W * DPR; canvas.height = H * DPR;
    ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
  }
  window.addEventListener('resize', resize);
  resize();

  // ---- Geometry -----------------------------------------------------------
  function makeHypercube(dim) {
    const n = 1 << dim;
    const verts = [];
    for (let i = 0; i < n; i++) {
      const v = [];
      for (let d = 0; d < 4; d++) v.push(d < dim ? ((i >> d) & 1 ? 1 : -1) : 0);
      verts.push(v);
    }
    const edges = [];
    for (let i = 0; i < n; i++)
      for (let d = 0; d < dim; d++) {
        const j = i ^ (1 << d);
        if (j > i) edges.push([i, j]);
      }
    return { verts, edges };
  }

  const shapes = {
    tesseract: makeHypercube(4),
    cube: makeHypercube(3)
  };
  let shapeName = 'tesseract';

  // ---- Rotation state -----------------------------------------------------
  // 4D rotation angles per plane
  const angles = { xy: 0, xz: 0, yz: 0, xw: 0, yw: 0, zw: 0 };
  const spinning = { xy: false, xw: true, yw: true, zw: false };
  const SPEED = { xy: 0.5, xw: 0.35, yw: 0.27, zw: 0.4 };
  // 3D view rotation (drag)
  let viewRX = -0.35, viewRY = 0.55;

  const PLANES = {
    xy: [0, 1], xz: [0, 2], yz: [1, 2],
    xw: [0, 3], yw: [1, 3], zw: [2, 3]
  };

  function rotate4(v, plane, a) {
    const [i, j] = PLANES[plane];
    const c = Math.cos(a), s = Math.sin(a);
    const vi = v[i], vj = v[j];
    v[i] = vi * c - vj * s;
    v[j] = vi * s + vj * c;
  }

  // ---- Projection ---------------------------------------------------------
  const CAM_W = 3.2;   // 4D camera distance along w
  const CAM_Z = 4.6;   // 3D camera distance along z

  function project(v) {
    // 4D -> 3D perspective
    let x = v[0], y = v[1], z = v[2], w = v[3];
    const k4 = CAM_W / (CAM_W - w);
    let px = x * k4, py = y * k4, pz = z * k4;
    // 3D view rotation
    let cy = Math.cos(viewRY), sy = Math.sin(viewRY);
    let cx = Math.cos(viewRX), sx = Math.sin(viewRX);
    let x1 = px * cy + pz * sy;
    let z1 = -px * sy + pz * cy;
    let y1 = py * cx - z1 * sx;
    let z2 = py * sx + z1 * cx;
    // 3D -> 2D perspective
    const k3 = CAM_Z / (CAM_Z - z2);
    const scale = Math.min(W, H) * 0.20;
    return {
      x: W / 2 + x1 * k3 * scale,
      y: H / 2 - y1 * k3 * scale,
      w: w,
      depth: z2
    };
  }

  // ---- Color: w encoded as hue -------------------------------------------
  function lerp(a, b, t) { return a + (b - a) * t; }
  function wColor(w, alpha) {
    // w in ~[-1.4, 1.4] after rotations; normalize
    const t = Math.max(0, Math.min(1, (w + 1.45) / 2.9));
    // far (blue) -> near (amber)
    const r = Math.round(lerp(0x5B, 0xFF, t));
    const g = Math.round(lerp(0x8D, 0xB4, t));
    const b = Math.round(lerp(0xEF, 0x54, t));
    return `rgba(${r},${g},${b},${alpha})`;
  }

  // ---- Interaction --------------------------------------------------------
  let dragging = false, lastX = 0, lastY = 0, interacted = false;
  stage.addEventListener('pointerdown', e => {
    dragging = true; lastX = e.clientX; lastY = e.clientY;
    stage.setPointerCapture(e.pointerId);
    if (!interacted) { interacted = true; hint.style.opacity = '0'; }
  });
  stage.addEventListener('pointermove', e => {
    if (!dragging) return;
    viewRY += (e.clientX - lastX) * 0.008;
    viewRX += (e.clientY - lastY) * 0.008;
    viewRX = Math.max(-1.5, Math.min(1.5, viewRX));
    lastX = e.clientX; lastY = e.clientY;
  });
  stage.addEventListener('pointerup', () => dragging = false);
  stage.addEventListener('pointercancel', () => dragging = false);

  document.querySelectorAll('button.rot').forEach(btn => {
    btn.addEventListener('click', () => {
      const p = btn.dataset.plane;
      spinning[p] = !spinning[p];
      btn.classList.toggle('on', spinning[p]);
    });
  });

  document.querySelectorAll('button.shape').forEach(btn => {
    btn.addEventListener('click', () => {
      shapeName = btn.dataset.shape;
      document.querySelectorAll('button.shape').forEach(b =>
        b.classList.toggle('on', b === btn));
      noteText.innerHTML = shapeName === 'tesseract'
        ? 'Watch the inner cube: it isn\'t smaller, it\'s <em>farther away in W</em>. When the object rotates through a W-plane, inner and outer cubes trade places &mdash; the "inside-out" move that\'s impossible for any 3D object.'
        : 'The cube sits entirely at W = 0, so W-rotations tilt it out of your 3D space &mdash; it flattens toward a plane, the way a spinning coin flattens to a line in its 2D shadow.';
    });
  });

  // ---- Render loop --------------------------------------------------------
  let last = performance.now();
  function frame(now) {
    const dt = Math.min((now - last) / 1000, 0.05);
    last = now;

    if (!reduceMotion) {
      for (const p in spinning) if (spinning[p]) angles[p] += SPEED[p] * dt;
    }

    const { verts, edges } = shapes[shapeName];
    const projected = verts.map(v0 => {
      const v = v0.slice();
      rotate4(v, 'xy', angles.xy);
      rotate4(v, 'xw', angles.xw);
      rotate4(v, 'yw', angles.yw);
      rotate4(v, 'zw', angles.zw);
      return project(v);
    });

    ctx.clearRect(0, 0, W, H);

    // faint grid floor line for orientation
    ctx.strokeStyle = 'rgba(42,48,80,0.6)';
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(0, H * 0.92); ctx.lineTo(W, H * 0.92);
    ctx.stroke();

    // edges, sorted far-to-near in w for layering
    const sorted = edges.slice().sort((a, b) => {
      const wa = projected[a[0]].w + projected[a[1]].w;
      const wb = projected[b[0]].w + projected[b[1]].w;
      return wa - wb;
    });

    for (const [i, j] of sorted) {
      const A = projected[i], B = projected[j];
      const wm = (A.w + B.w) / 2;
      const t = Math.max(0, Math.min(1, (wm + 1.45) / 2.9));
      ctx.strokeStyle = wColor(wm, 0.45 + 0.5 * t);
      ctx.lineWidth = 1 + 1.8 * t;
      ctx.beginPath();
      ctx.moveTo(A.x, A.y);
      ctx.lineTo(B.x, B.y);
      ctx.stroke();
    }

    // vertices
    for (const p of projected) {
      const t = Math.max(0, Math.min(1, (p.w + 1.45) / 2.9));
      ctx.fillStyle = wColor(p.w, 0.9);
      ctx.beginPath();
      ctx.arc(p.x, p.y, 1.5 + 2.5 * t, 0, Math.PI * 2);
      ctx.fill();
    }

    requestAnimationFrame(frame);
  }
  requestAnimationFrame(frame);
})();
</script>
</body>
</html>
