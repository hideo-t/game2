<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
  <title>Color Pop!（カラフルバブル）</title>
  <style>
    :root { color-scheme: dark; }
    html, body { height: 100%; margin: 0; }
    body{
      background:#070b14;
      font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans JP", sans-serif;
      overscroll-behavior: none;
      -webkit-tap-highlight-color: transparent;
      touch-action: none;
    }
    #wrap{
      position: fixed;
      inset: 0;
      display: grid;
      grid-template-rows: auto 1fr auto;
      padding: env(safe-area-inset-top) 10px env(safe-area-inset-bottom) 10px;
      gap: 8px;
    }
    .hud{
      display:flex;
      justify-content:space-between;
      align-items:center;
      gap:8px;
      flex-wrap: wrap;
      padding: 8px 6px 0;
      color:#d7e1ff;
    }
    .left{ display:flex; gap:10px; flex-wrap: wrap; }
    .pill{
      background: rgba(255,255,255,.06);
      border: 1px solid rgba(255,255,255,.10);
      padding: 7px 10px;
      border-radius: 999px;
      font-size: 13px;
      backdrop-filter: blur(6px);
    }
    .btns{ display:flex; gap:8px; }
    .btn{
      user-select:none;
      cursor:pointer;
      background: rgba(120,190,255,.18);
      border: 1px solid rgba(120,190,255,.35);
      color:#eaf3ff;
      padding: 9px 12px;
      border-radius: 12px;
      font-weight: 800;
      letter-spacing: .02em;
    }
    .btn:active{ transform: translateY(1px); }

    canvas{
      width: 100%;
      height: 100%;
      display:block;
      border-radius: 18px;
      background:
        radial-gradient(1200px 700px at 50% 120%, rgba(110,170,255,.16), transparent 60%),
        linear-gradient(180deg, rgba(255,255,255,.06), rgba(255,255,255,.02));
      border: 1px solid rgba(255,255,255,.10);
    }
    .foot{
      color: rgba(215,225,255,.75);
      font-size: 12px;
      padding: 0 6px 10px;
      line-height: 1.6;
      text-align: center;
    }
    .kbd{
      padding:2px 6px;
      border-radius: 6px;
      background: rgba(255,255,255,.08);
      border: 1px solid rgba(255,255,255,.12);
    }
  </style>
</head>
<body>
  <div id="wrap">
    <div class="hud">
      <div class="left">
        <div class="pill">SCORE: <span id="score">0</span></div>
        <div class="pill">COMBO: <span id="combo">0</span></div>
        <div class="pill">TIME: <span id="time">60.0</span>s</div>
        <div class="pill">BEST: <span id="best">0</span></div>
      </div>
      <div class="btns">
        <button class="btn" id="btnStart">START</button>
        <button class="btn" id="btnRestart">RESTART</button>
      </div>
    </div>

    <canvas id="game"></canvas>

    <div class="foot">
      遊び方：カラフルなバブルをタップで割るだけ。連続で割るとコンボで加点。<br>
      PCならクリック / <span class="kbd">Space</span>で開始・一時停止
    </div>
  </div>

<script>
(() => {
  const canvas = document.getElementById("game");
  const ctx = canvas.getContext("2d", { alpha: true });

  const $score = document.getElementById("score");
  const $combo = document.getElementById("combo");
  const $time  = document.getElementById("time");
  const $best  = document.getElementById("best");
  const btnStart = document.getElementById("btnStart");
  const btnRestart = document.getElementById("btnRestart");

  // --- 画面フィット（高DPI対応） ---
  function resize(){
    const dpr = Math.max(1, Math.min(3, window.devicePixelRatio || 1));
    const rect = canvas.getBoundingClientRect();
    canvas.width = Math.floor(rect.width * dpr);
    canvas.height = Math.floor(rect.height * dpr);
    ctx.setTransform(dpr,0,0,dpr,0,0); // 以後の座標はCSSピクセル基準
  }
  window.addEventListener("resize", resize);

  // --- ゲーム状態 ---
  let W = 0, H = 0;

  let running = false;
  let gameOver = false;

  let score = 0;
  let combo = 0;
  let timeLeft = 60.0;

  const bestKey = "colorpop_best_v1";
  let best = Number(localStorage.getItem(bestKey) || 0);
  $best.textContent = String(best);

  // バブル
  const bubbles = [];
  const pops = []; // パーティクル

  // カラーパレット（明るめ）
  const palette = [
    "#ff4d6d", "#ffb703", "#4cc9f0", "#8aff7a", "#b517ff",
    "#ffd6ff", "#00f5d4", "#f72585", "#90be6d", "#f9c74f"
  ];

  function rand(a,b){ return a + Math.random()*(b-a); }
  function clamp(v,min,max){ return Math.max(min, Math.min(max, v)); }

  function syncHUD(){
    $score.textContent = String(score);
    $combo.textContent = String(combo);
    $time.textContent = timeLeft.toFixed(1);
    $best.textContent = String(best);
  }

  function reset(){
    running = false;
    gameOver = false;
    score = 0;
    combo = 0;
    timeLeft = 60.0;
    bubbles.length = 0;
    pops.length = 0;
    syncHUD();
  }

  // バブル生成
  let spawnTimer = 0;
  function spawnBubble(){
    const minDim = Math.min(W,H);

    const r = rand(minDim*0.05, minDim*0.09);
    const x = rand(r+10, W-r-10);
    const y = rand(r+10, H-r-10);

    const life = rand(1.15, 1.85); // 秒
    const grow = rand(0.9, 1.45);  // ふくらむ速度
    const drift = { x: rand(-18, 18), y: rand(-14, 14) };

    bubbles.push({
      x, y, r,
      t: 0,
      life,
      grow,
      drift,
      color: palette[Math.floor(Math.random()*palette.length)],
      ring: rand(0.12, 0.22) // 輪っかの太さ比
    });
  }

  function makePop(x,y,color,baseR){
    const count = 14 + Math.floor(Math.random()*10);
    for (let i=0; i<count; i++){
      const a = Math.random()*Math.PI*2;
      const sp = rand(110, 320);
      pops.push({
        x,y,
        vx: Math.cos(a)*sp,
        vy: Math.sin(a)*sp,
        r: rand(2, 5),
        life: rand(0.35, 0.7),
        t: 0,
        color
      });
    }
    // ふわっと輪
    pops.push({
      x,y,
      vx:0, vy:0,
      r: baseR*0.55,
      life: 0.45,
      t: 0,
      color,
      ring: true
    });
  }

  function toggle(){
    if (gameOver) return;
    running = !running;
  }

  btnStart.addEventListener("click", () => {
    if (gameOver) return;
    running = true;
  });
  btnRestart.addEventListener("click", () => reset());

  window.addEventListener("keydown", (e) => {
    if (e.key === " ") { e.preventDefault(); toggle(); }
  }, { passive:false });

  // タップ判定
  function getPointerPos(e){
    const rect = canvas.getBoundingClientRect();
    const x = (e.clientX - rect.left);
    const y = (e.clientY - rect.top);
    return {x,y};
  }

  function tryHit(x,y){
    // 上にあるやつ優先
    for (let i=bubbles.length-1; i>=0; i--){
      const b = bubbles[i];
      const dx = x - b.x, dy = y - b.y;
      if (dx*dx + dy*dy <= (b.r*b.r)){
        // 命中
        bubbles.splice(i,1);

        combo += 1;
        const base = 10;
        const bonus = Math.floor(base * (1 + Math.min(8, combo)*0.25));
        score += bonus;

        makePop(b.x, b.y, b.color, b.r);

        // ちょい時間回復（上手い人が気持ちよく伸びる）
        timeLeft = clamp(timeLeft + 0.18, 0, 99);

        // BEST更新
        if (score > best){
          best = score;
          localStorage.setItem(bestKey, String(best));
        }

        syncHUD();
        return true;
      }
    }
    // 外したらコンボ終了（潔く）
    combo = 0;
    syncHUD();
    return false;
  }

  canvas.addEventListener("pointerdown", (e) => {
    canvas.setPointerCapture?.(e.pointerId);
    if (!running && !gameOver) running = true;
    const p = getPointerPos(e);
    tryHit(p.x, p.y);
  });

  // --- ループ ---
  let last = performance.now();

  function tick(now){
    const dt = Math.min(0.033, (now-last)/1000);
    last = now;

    // canvas寸法を読む
    const rect = canvas.getBoundingClientRect();
    W = rect.width; H = rect.height;

    update(dt);
    draw();
    requestAnimationFrame(tick);
  }

  function update(dt){
    if (!running || gameOver) return;

    timeLeft -= dt;
    if (timeLeft <= 0){
      timeLeft = 0;
      running = false;
      gameOver = true;
      syncHUD();
      return;
    }
    $time.textContent = timeLeft.toFixed(1);

    // 生成テンポ（残り時間が減ると少し増える）
    const difficulty = 1 + (60 - timeLeft) / 60;
    const interval = clamp(0.38 / difficulty, 0.16, 0.38);

    spawnTimer += dt;
    while (spawnTimer >= interval){
      spawnTimer -= interval;
      spawnBubble();
      // 画面が埋まりすぎないよう上限
      if (bubbles.length > 22) bubbles.shift();
    }

    // バブル更新（ふくらむ + 漂う + 寿命）
    for (let i=bubbles.length-1; i>=0; i--){
      const b = bubbles[i];
      b.t += dt;

      b.x += b.drift.x * dt;
      b.y += b.drift.y * dt;

      // 端で反射
      if (b.x - b.r < 8 || b.x + b.r > W - 8) b.drift.x *= -1;
      if (b.y - b.r < 8 || b.y + b.r > H - 8) b.drift.y *= -1;

      // 成長
      b.r *= (1 + dt * 0.18 * b.grow);

      // 寿命切れ：コンボをリセットして悔しさを作る
      if (b.t >= b.life){
        bubbles.splice(i,1);
        combo = 0;
        syncHUD();
      }
    }

    // パーティクル更新
    for (let i=pops.length-1; i>=0; i--){
      const p = pops[i];
      p.t += dt;
      if (p.t >= p.life){ pops.splice(i,1); continue; }
      if (!p.ring){
        p.vy += 520 * dt; // 重力
        p.x += p.vx * dt;
        p.y += p.vy * dt;
        p.vx *= (1 - dt*2.2);
        p.vy *= (1 - dt*1.6);
      } else {
        p.r *= (1 + dt*2.0);
      }
    }
  }

  function draw(){
    // 背景クリア
    ctx.clearRect(0,0,canvas.width,canvas.height);

    // 星っぽい点（軽い）
    ctx.globalAlpha = 0.22;
    for (let i=0;i<80;i++){
      const x = (i*97) % W;
      const y = (i*173) % H;
      ctx.fillStyle = "white";
      ctx.fillRect(x,y,1,1);
    }
    ctx.globalAlpha = 1;

    // バブル描画
    for (const b of bubbles){
      const life01 = clamp(1 - (b.t / b.life), 0, 1);

      // ふわっと影
      ctx.beginPath();
      ctx.arc(b.x, b.y, b.r*1.08, 0, Math.PI*2);
      ctx.fillStyle = "rgba(0,0,0,.25)";
      ctx.fill();

      // 本体（グラデっぽく見せる：中心を明るく）
      const grad = ctx.createRadialGradient(b.x - b.r*0.35, b.y - b.r*0.35, b.r*0.2, b.x, b.y, b.r);
      grad.addColorStop(0, "rgba(255,255,255,.75)");
      grad.addColorStop(0.25, b.color + "CC");
      grad.addColorStop(1, b.color + "55");

      ctx.beginPath();
      ctx.arc(b.x, b.y, b.r, 0, Math.PI*2);
      ctx.fillStyle = grad;
      ctx.fill();

      // リング
      ctx.lineWidth = Math.max(2, b.r*b.ring);
      ctx.strokeStyle = "rgba(255,255,255,.55)";
      ctx.globalAlpha = 0.35 + 0.45*life01;
      ctx.stroke();
      ctx.globalAlpha = 1;

      // 寿命ゲージ（ちっちゃい弧）
      ctx.beginPath();
      ctx.arc(b.x, b.y, b.r*0.88, -Math.PI/2, -Math.PI/2 + Math.PI*2*life01);
      ctx.lineWidth = 3;
      ctx.strokeStyle = "rgba(255,255,255,.35)";
      ctx.stroke();
    }

    // パーティクル描画
    for (const p of pops){
      const a = 1 - (p.t / p.life);
      if (p.ring){
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.r, 0, Math.PI*2);
        ctx.lineWidth = 4;
        ctx.strokeStyle = hexToRgba(p.color, 0.55*a);
        ctx.stroke();
        continue;
      }
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.r, 0, Math.PI*2);
      ctx.fillStyle = hexToRgba(p.color, 0.9*a);
      ctx.fill();
    }

    // 文字
    if (!running && !gameOver){
      centerText("STARTを押すか 画面をタップ", H*0.53, 18, "rgba(215,225,255,.92)");
      centerText("バブルを割ってコンボを伸ばそう", H*0.61, 13, "rgba(215,225,255,.72)");
    }
    if (gameOver){
      centerText("TIME UP!", H*0.50, 28, "rgba(255,235,235,.95)");
      centerText(`SCORE: ${score}  /  BEST: ${best}`, H*0.60, 14, "rgba(215,225,255,.78)");
      centerText("RESTARTで再挑戦", H*0.68, 14, "rgba(215,225,255,.72)");
    }
  }

  function centerText(text, y, size, color){
    ctx.fillStyle = color;
    ctx.font = `800 ${size}px system-ui, -apple-system, "Segoe UI", "Noto Sans JP", sans-serif`;
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    ctx.fillText(text, W/2, y);
  }

  function hexToRgba(hex, a){
    // "#rrggbb" -> rgba
    const h = hex.replace("#","");
    const r = parseInt(h.slice(0,2),16);
    const g = parseInt(h.slice(2,4),16);
    const b = parseInt(h.slice(4,6),16);
    return `rgba(${r},${g},${b},${a})`;
  }

  // 初期化
  // レイアウト後にリサイズ
  requestAnimationFrame(() => {
    resize();
    reset();
    requestAnimationFrame(tick);
  });
})();
</script>
</body>
</html>