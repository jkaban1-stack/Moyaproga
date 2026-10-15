<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>✨ Оживлятор рисунков ✨</title>
<style>
  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  html, body {
    margin: 0; padding: 0; height: 100%;
    font-family: -apple-system, 'Segoe UI', system-ui, sans-serif;
    background: #1a1a2e; color: #fff;
    overflow: hidden; user-select: none;
  }

  #app {
    display: flex; flex-direction: column;
    height: 100vh; height: 100dvh;
  }

  /* Заголовок */
  header {
    padding: 12px;
    text-align: center;
    font-size: 20px;
    font-weight: 700;
    background: linear-gradient(135deg, #ff5c8a, #a15cff);
    -webkit-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color: transparent;
    flex-shrink: 0;
  }

  /* Сцена */
  #stage {
    flex: 1;
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
    background: #0f0f1e;
  }

  #stage video, #stage canvas {
    position: absolute;
    max-width: 100%;
    max-height: 100%;
    border-radius: 16px;
  }

  #stage video { display: none; }
  #stage canvas { display: block; }

  /* Кнопки */
  #controls {
    flex-shrink: 0;
    padding: 16px;
    display: flex;
    gap: 10px;
    justify-content: center;
    flex-wrap: wrap;
    background: #1a1a2e;
  }

  button {
    font-size: 18px;
    font-weight: 600;
    padding: 14px 22px;
    border: none;
    border-radius: 14px;
    color: #fff;
    cursor: pointer;
    transition: transform 0.15s, filter 0.15s;
    box-shadow: 0 4px 14px rgba(0,0,0,0.35);
  }
  button:active { transform: scale(0.94); }
  button:disabled { opacity: 0.5; }

  .btn-snap   { background: linear-gradient(135deg, #ff5c8a, #ff8a5c); }
  .btn-again  { background: linear-gradient(135deg, #5cc8ff, #5c8aff); }
  .btn-save   { background: linear-gradient(135deg, #5cff9e, #5cffd6); color:#0a2a1a; }
  .btn-bg     { background: linear-gradient(135deg, #a15cff, #ff5cc8); }

  /* Подсказка */
  #hint {
    position: absolute;
    bottom: 20px;
    left: 50%; transform: translateX(-50%);
    background: rgba(0,0,0,0.65);
    padding: 10px 18px;
    border-radius: 999px;
    font-size: 15px;
    text-align: center;
    pointer-events: none;
    opacity: 0;
    transition: opacity 0.3s;
    white-space: nowrap;
  }
  #hint.show { opacity: 1; }

  /* Экран загрузки */
  #loading {
    position: absolute;
    inset: 0;
    background: rgba(15,15,30,0.85);
    display: none;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    font-size: 18px;
    gap: 12px;
    z-index: 10;
  }
  #loading.show { display: flex; }
  .spinner {
    width: 40px; height: 40px;
    border: 4px solid rgba(255,255,255,0.2);
    border-top-color: #ff5c8a;
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
  }
  @keyframes spin { to { transform: rotate(360deg); } }
</style>
</head>
<body>
<div id="app">
  <header>✨ Оживлятор рисунков ✨</header>

  <div id="stage">
    <video id="cam" autoplay playsinline muted></video>
    <canvas id="out"></canvas>

    <div id="hint">Наведи камеру на рисунок и нажми «Оживить»</div>
    <div id="loading">
      <div class="spinner"></div>
      <div>Оживляю...</div>
    </div>
  </div>

  <div id="controls">
    <button id="snapBtn" class="btn-snap">📸 Оживить</button>
    <button id="againBtn" class="btn-again" style="display:none;">🔄 Ещё раз</button>
    <button id="saveBtn" class="btn-save" style="display:none;">💾 Сохранить</button>
    <button id="bgBtn" class="btn-bg" style="display:none;">🌈 Фон</button>
  </div>
</div>

<script>
// ================ Элементы ================
const video    = document.getElementById('cam');
const canvas   = document.getElementById('out');
const ctx      = canvas.getContext('2d');
const hint     = document.getElementById('hint');
const loading  = document.getElementById('loading');
const snapBtn  = document.getElementById('snapBtn');
const againBtn = document.getElementById('againBtn');
const saveBtn  = document.getElementById('saveBtn');
const bgBtn    = document.getElementById('bgBtn');

// ================ Состояние ================
let spriteImage = null;       // ImageData персонажа с альфой
let spriteCanvas = null;      // Canvas с вырезанным персонажем
let animationId = null;
let animTime = 0;
let bgIndex = 0;

// Фоны
const BACKGROUNDS = [
  { name:'Небо',       colors:['#bfe6ff','#ffe8f0'] },
  { name:'Закат',      colors:['#ffb37a','#ff6a9e'] },
  { name:'Космос',     colors:['#1a0b3d','#3d1a6b'] },
  { name:'Луг',        colors:['#c8f7c5','#f0fff0'] },
  { name:'Радуга',     colors:['#ffd6e0','#d6e4ff'] },
];

// ================ Камера ================
async function startCamera() {
  try {
    const stream = await navigator.mediaDevices.getUserMedia({
      video: {
        facingMode: { ideal: 'environment' },
        width:  { ideal: 1280 },
        height: { ideal: 720 },
      },
      audio: false,
    });
    video.srcObject = stream;
    await video.play();
    showHint('Наведи камеру на рисунок и нажми «Оживить»', 3500);
  } catch (err) {
    console.error(err);
    showHint('Не получилось включить камеру 😔', 5000);
  }
}

// ================ Подсказки ================
let hintTimer = null;
function showHint(text, ms = 2500) {
  hint.textContent = text;
  hint.classList.add('show');
  clearTimeout(hintTimer);
  hintTimer = setTimeout(() => hint.classList.remove('show'), ms);
}

// ================ Вырезание фона ================
function cutBackground(sourceCanvas) {
  const w = sourceCanvas.width;
  const h = sourceCanvas.height;
  const srcCtx = sourceCanvas.getContext('2d');
  const imgData = srcCtx.getImageData(0, 0, w, h);
  const d = imgData.data;

  // Порог: пиксели светлее — фон
  const THRESHOLD = 200;

  for (let i = 0; i < d.length; i += 4) {
    const r = d[i], g = d[i+1], b = d[i+2];
    // Яркость
    const lum = 0.299*r + 0.587*g + 0.114*b;
    // Насыщенность (насколько далёк от серого)
    const max = Math.max(r, g, b);
    const min = Math.min(r, g, b);
    const sat = max - min;

    // Белый/светло-серый фон = высокая яркость и низкая насыщенность
    if (lum > THRESHOLD && sat < 30) {
      // Делаем прозрачным
      d[i+3] = 0;
    } else if (lum > THRESHOLD - 40 && sat < 30) {
      // Плавный край
      const t = (THRESHOLD - lum) / 40;
      d[i+3] = Math.floor(t * 255);
    }
  }

  srcCtx.putImageData(imgData, 0, 0);
  return sourceCanvas;
}

// Обрезать по границам содержимого
function cropToContent(sourceCanvas) {
  const w = sourceCanvas.width;
  const h = sourceCanvas.height;
  const srcCtx = sourceCanvas.getContext('2d');
  const imgData = srcCtx.getImageData(0, 0, w, h);
  const d = imgData.data;

  let minX = w, minY = h, maxX = 0, maxY = 0;
  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      const a = d[(y*w + x) * 4 + 3];
      if (a > 30) {
        if (x < minX) minX = x;
        if (x > maxX) maxX = x;
        if (y < minY) minY = y;
        if (y > maxY) maxY = y;
      }
    }
  }

  if (maxX <= minX || maxY <= minY) return sourceCanvas; // пусто

  const pad = 10;
  minX = Math.max(0, minX - pad);
  minY = Math.max(0, minY - pad);
  maxX = Math.min(w - 1, maxX + pad);
  maxY = Math.min(h - 1, maxY + pad);

  const cw = maxX - minX;
  const ch = maxY - minY;
  const out = document.createElement('canvas');
  out.width = cw;
  out.height = ch;
  out.getContext('2d').drawImage(sourceCanvas, minX, minY, cw, ch, 0, 0, cw, ch);
  return out;
}

// ================ Захват кадра ================
function captureFrame() {
  const vw = video.videoWidth;
  const vh = video.videoHeight;
  if (!vw || !vh) return null;

  const c = document.createElement('canvas');
  // Уменьшаем для скорости (максимум 700 по длинной стороне)
  const maxSide = 700;
  const scale = Math.min(1, maxSide / Math.max(vw, vh));
  c.width  = Math.round(vw * scale);
  c.height = Math.round(vh * scale);

  const cctx = c.getContext('2d');
  cctx.drawImage(video, 0, 0, c.width, c.height);

  // Вырезаем фон и обрезаем
  cutBackground(c);
  return cropToContent(c);
}

// ================ Анимация ================
function setupCanvas() {
  const stage = document.getElementById('stage');
  const size = Math.min(stage.clientWidth, stage.clientHeight);
  canvas.width = size;
  canvas.height = size;
}

function drawBackground() {
  const bg = BACKGROUNDS[bgIndex];
  const grad = ctx.createLinearGradient(0, 0, 0, canvas.height);
  grad.addColorStop(0, bg.colors[0]);
  grad.addColorStop(1, bg.colors[1]);
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // Если космос — добавляем звёздочки
  if (bg.name === 'Космос') {
    ctx.fillStyle = 'rgba(255,255,255,0.8)';
    for (let i = 0; i < 40; i++) {
      const x = (i * 97.3) % canvas.width;
      const y = (i * 53.7) % canvas.height;
      const r = ((i * 13) % 3) + 0.5;
      const twinkle = 0.5 + 0.5 * Math.sin(animTime * 2 + i);
      ctx.globalAlpha = twinkle * 0.9;
      ctx.beginPath();
      ctx.arc(x, y, r, 0, Math.PI * 2);
      ctx.fill();
    }
    ctx.globalAlpha = 1;
  }
}

function drawAnimated() {
  if (!spriteCanvas) return;

  animTime += 0.04;

  // Фон
  drawBackground();

  const W = canvas.width;
  const H = canvas.height;

  // Вписываем персонажа в 70% площади
  const maxW = W * 0.75;
  const maxH = H * 0.75;
  const sAspect = spriteCanvas.width / spriteCanvas.height;
  let dw, dh;
  if (sAspect > maxW / maxH) {
    dw = maxW;
    dh = maxW / sAspect;
  } else {
    dh = maxH;
    dw = maxH * sAspect;
  }

  // Параметры оживления
  const breath = 1 + 0.035 * Math.sin(animTime * 1.6);       // дыхание
  const swayX  = 12 * Math.sin(animTime * 0.7);               // качание влево-вправо
  const bobY   = 8 * Math.sin(animTime * 1.2);                // подпрыгивание
  const rot    = 0.035 * Math.sin(animTime * 0.9);            // покачивание

  ctx.save();
  ctx.translate(W/2 + swayX, H/2 + bobY);
  ctx.rotate(rot);
  ctx.scale(breath, breath);

  // Мягкая тень под персонажем
  ctx.shadowColor = 'rgba(0,0,0,0.25)';
  ctx.shadowBlur = 20;
  ctx.shadowOffsetY = 15;

  ctx.drawImage(spriteCanvas, -dw/2, -dh/2, dw, dh);
  ctx.restore();

  animationId = requestAnimationFrame(drawAnimated);
}

// ================ Кнопка «Оживить» ================
snapBtn.onclick = () => {
  loading.classList.add('show');

  setTimeout(() => {
    try {
      const cropped = captureFrame();
      if (!cropped) {
        showHint('Не вижу рисунок 😔 Попробуй ещё', 3000);
        loading.classList.remove('show');
        return;
      }

      spriteCanvas = cropped;

      setupCanvas();
      window.addEventListener('resize', setupCanvas);

      // Переключаем UI
      video.style.display = 'none';
      canvas.style.display = 'block';
      snapBtn.style.display = 'none';
      againBtn.style.display = 'inline-block';
      saveBtn.style.display = 'inline-block';
      bgBtn.style.display = 'inline-block';

      // Запускаем анимацию
      if (animationId) cancelAnimationFrame(animationId);
      animTime = 0;
      drawAnimated();

      showHint('✨ Смотри, ожил! ✨', 2500);
    } catch (e) {
      console.error(e);
      showHint('Что-то пошло не так 😔', 3000);
    }
    loading.classList.remove('show');
  }, 50);
};

// ================ Кнопка «Ещё раз» ================
againBtn.onclick = () => {
  if (animationId) cancelAnimationFrame(animationId);
  spriteCanvas = null;

  video.style.display = 'block';
  canvas.style.display = 'none';
  snapBtn.style.display = 'inline-block';
  againBtn.style.display = 'none';
  saveBtn.style.display = 'none';
  bgBtn.style.display = 'none';

  showHint('Наведи камеру на новый рисунок 📷', 2500);
};

// ================ Кнопка «Фон» ================
bgBtn.onclick = () => {
  bgIndex = (bgIndex + 1) % BACKGROUNDS.length;
  showHint(`Фон: ${BACKGROUNDS[bgIndex].name}`, 1200);
};

// ================ Кнопка «Сохранить» ================
saveBtn.onclick = () => {
  if (!canvas.captureStream) {
    showHint('Запись видео не поддерживается 😔', 3000);
    return;
  }

  try {
    const stream = canvas.captureStream(30);
    const mimeCandidates = [
      'video/webm;codecs=vp9',
      'video/webm;codecs=vp8',
      'video/webm',
      'video/mp4',
    ];
    const mime = mimeCandidates.find(m => MediaRecorder.isTypeSupported(m)) || '';
    const recorder = new MediaRecorder(stream, mime ? { mimeType: mime } : undefined);
    const chunks = [];

    recorder.ondataavailable = e => { if (e.data.size) chunks.push(e.data); };
    recorder.onstop = () => {
      const blob = new Blob(chunks, { type: mime || 'video/webm' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'ozhivlennyi-' + Date.now() + (mime.includes('mp4') ? '.mp4' : '.webm');
      a.click();
      setTimeout(() => URL.revokeObjectURL(url), 2000);
      showHint('💾 Сохранено!', 2000);
    };

    recorder.start();
    showHint('🎬 Записываю 4 секунды...', 4000);
    setTimeout(() => recorder.stop(), 4000);
  } catch (e) {
    console.error(e);
    showHint('Не получилось сохранить 😔', 3000);
  }
};

// ================ Старт ================
startCamera();
window.addEventListener('resize', () => {
  if (spriteCanvas) setupCanvas();
});
</script>
</body>
</html>