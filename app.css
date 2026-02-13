let audioCtx, analyser, srcNode, mediaStream;
let running = false;

const canvas = document.getElementById("spec");
const ctx = canvas.getContext("2d");

const btnStart = document.getElementById("btnStart");
const btnStop  = document.getElementById("btnStop");

const thr = document.getElementById("thr");
const thrVal = document.getElementById("thrVal");
const speed = document.getElementById("speed");
const speedVal = document.getElementById("speedVal");
const fileInput = document.getElementById("file");

// métricas UI
const mRate = document.getElementById("mRate");
const mPause = document.getElementById("mPause");
const mRms = document.getElementById("mRms");
const mPitch = document.getElementById("mPitch");
const mCentroid = document.getElementById("mCentroid");
const mState = document.getElementById("mState");

// buffers e estado CRS
let fftData, timeData;
let lastFrameTs = performance.now();
let silence = false;
let silenceStart = null;

let pauseDurations = [];      // segundos
let pauseCountWindow = [];    // timestamps de pausas (ms) p/ taxa/min
let pitchHistory = [];        // Hz
let centroidHistory = [];     // relativo

thr.addEventListener("input", () => thrVal.textContent = Number(thr.value).toFixed(3));
speed.addEventListener("input", () => speedVal.textContent = speed.value);

btnStart.addEventListener("click", startMic);
btnStop.addEventListener("click", stopAll);
fileInput.addEventListener("change", handleFile);

// desenho: “scroll” horizontal (tipo spectroid)
function scrollLeft(pixels) {
  const img = ctx.getImageData(pixels, 0, canvas.width - pixels, canvas.height);
  ctx.putImageData(img, 0, 0);
  ctx.fillRect(canvas.width - pixels, 0, pixels, canvas.height);
}

function rmsFromTimeDomain(arr) {
  let sum = 0;
  for (let i=0; i<arr.length; i++) {
    const v = (arr[i] - 128) / 128;
    sum += v*v;
  }
  return Math.sqrt(sum / arr.length);
}

// pitch simples via autocorrelação (funciona bem em voz “limpa”)
function autoCorrelatePitch(timeArr, sampleRate) {
  // converte para float [-1,1]
  const buf = new Float32Array(timeArr.length);
  for (let i=0;i<timeArr.length;i++) buf[i] = (timeArr[i]-128)/128;

  // energia baixa -> sem pitch
  let rms = 0;
  for (let i=0;i<buf.length;i++) rms += buf[i]*buf[i];
  rms = Math.sqrt(rms / buf.length);
  if (rms < 0.01) return null;

  const MIN_HZ = 70, MAX_HZ = 350; // faixa de voz base
  const minLag = Math.floor(sampleRate / MAX_HZ);
  const maxLag = Math.floor(sampleRate / MIN_HZ);

  let bestLag = -1;
  let best = 0;

  for (let lag = minLag; lag <= maxLag; lag++) {
    let corr = 0;
    for (let i=0; i<buf.length - lag; i++) corr += buf[i] * buf[i+lag];
    corr = corr / (buf.length - lag);
    if (corr > best) { best = corr; bestLag = lag; }
  }

  if (bestLag === -1 || best < 0.02) return null;
  return sampleRate / bestLag;
}

function spectralCentroid(fftArr) {
  // fftArr vem em dB [0..255], aproximamos linear
  let num = 0, den = 0;
  for (let i=0;i<fftArr.length;i++) {
    const mag = Math.max(0, fftArr[i]); // 0..255
    num += i * mag;
    den += mag;
  }
  if (den === 0) return null;
  return num / den; // índice (0..N)
}

function updateCRS(nowMs, rms, pitchHz, centroidIdx) {
  const thrValNum = Number(thr.value);

  // detecção de pausa (silêncio operacional)
  const isSilent = rms < thrValNum;

  if (isSilent && !silence) {
    // entrou em pausa
    silence = true;
    silenceStart = nowMs;
  } else if (!isSilent && silence) {
    // saiu da pausa
    silence = false;
    const dur = (nowMs - silenceStart) / 1000;
    silenceStart = null;

    if (dur >= 0.15) { // ignora microburacos
      pauseDurations.push(dur);
      pauseCountWindow.push(nowMs);
      // mantém janela 60s
      pauseCountWindow = pauseCountWindow.filter(t => nowMs - t <= 60000);
      // limita crescimento
      if (pauseDurations.length > 200) pauseDurations.shift();
    }
  } else {
    // mantém janela 60s
    pauseCountWindow = pauseCountWindow.filter(t => nowMs - t <= 60000);
  }

  if (pitchHz) {
    pitchHistory.push(pitchHz);
    if (pitchHistory.length > 120) pitchHistory.shift(); // ~ alguns segundos
  }
  if (centroidIdx != null) {
    centroidHistory.push(centroidIdx);
    if (centroidHistory.length > 120) centroidHistory.shift();
  }

  // métricas
  const pausesPerMin = pauseCountWindow.length;
  const avgPause = pauseDurations.length
    ? pauseDurations.reduce((a,b)=>a+b,0) / pauseDurations.length
    : null;

  const avgPitch = pitchHistory.length
    ? pitchHistory.reduce((a,b)=>a+b,0) / pitchHistory.length
    : null;

  const avgCent = centroidHistory.length
    ? centroidHistory.reduce((a,b)=>a+b,0) / centroidHistory.length
    : null;

  // estado operacional (heurística simples)
  let state = "Estável";
  if (pausesPerMin >= 18) state = "Fragmentado";
  else if (pausesPerMin >= 10) state = "Interrompido";
  else if (rms > thrValNum * 4 && pausesPerMin <= 6) state = "Fluido";

  // UI
  mRate.textContent = `${pausesPerMin}/min`;
  mPause.textContent = avgPause ? `${avgPause.toFixed(2)} s` : "—";
  mRms.textContent = rms.toFixed(3);
  mPitch.textContent = avgPitch ? `${avgPitch.toFixed(0)} Hz` : "—";
  mCentroid.textContent = avgCent ? avgCent.toFixed(1) : "—";
  mState.textContent = state;
}

function drawSpectrogramColumn(fftArr) {
  const colWidth = Number(speed.value); // pixels por frame
  scrollLeft(colWidth);

  // desenha uma coluna vertical: y=agudos/topo, y=graves/base
  for (let x = canvas.width - colWidth; x < canvas.width; x++) {
    for (let y = 0; y < canvas.height; y++) {
      const fftIndex = Math.floor((1 - y / canvas.height) * (fftArr.length - 1));
      const v = fftArr[fftIndex]; // 0..255
      // brilho sem “cor fixa”: usamos escala de cinza (seguro e limpo)
      ctx.fillStyle = `rgb(${v},${v},${v})`;
      ctx.fillRect(x, y, 1, 1);
    }
  }
}

function loop() {
  if (!running || !analyser) return;

  analyser.getByteFrequencyData(fftData);
  analyser.getByteTimeDomainData(timeData);

  const now = performance.now();
  const rms = rmsFromTimeDomain(timeData);
  const pitchHz = autoCorrelatePitch(timeData, audioCtx.sampleRate);
  const cent = spectralCentroid(fftData);

  drawSpectrogramColumn(fftData);
  updateCRS(now, rms, pitchHz, cent);

  requestAnimationFrame(loop);
}

async function startMic() {
  try {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    mediaStream = await navigator.mediaDevices.getUserMedia({ audio: true });
    srcNode = audioCtx.createMediaStreamSource(mediaStream);

    analyser = audioCtx.createAnalyser();
    analyser.fftSize = 2048;
    analyser.smoothingTimeConstant = 0.7;

    srcNode.connect(analyser);

    fftData = new Uint8Array(analyser.frequencyBinCount);
    timeData = new Uint8Array(analyser.fftSize);

    // limpa canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = "#000";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    running = true;
    btnStart.disabled = true;
    btnStop.disabled = false;

    loop();
  } catch (e) {
    alert("Não consegui acessar o microfone. Verifique permissões/HTTPS.");
    console.error(e);
  }
}

function stopAll() {
  running = false;
  btnStart.disabled = false;
  btnStop.disabled = true;

  try {
    if (mediaStream) mediaStream.getTracks().forEach(t => t.stop());
  } catch {}
  mediaStream = null;

  try { if (audioCtx) audioCtx.close(); } catch {}
  audioCtx = null;
  analyser = null;
}

async function handleFile() {
  const file = fileInput.files?.[0];
  if (!file) return;

  // Para tocar/analisar arquivo: cria source de áudio
  stopAll();

  audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  const arrayBuffer = await file.arrayBuffer();
  const audioBuffer = await audioCtx.decodeAudioData(arrayBuffer);

  analyser = audioCtx.createAnalyser();
  analyser.fftSize = 2048;
  analyser.smoothingTimeConstant = 0.7;

  const bufferSource = audioCtx.createBufferSource();
  bufferSource.buffer = audioBuffer;
  bufferSource.connect(analyser);
  analyser.connect(audioCtx.destination); // toca no som (pode remover se quiser)
  bufferSource.start();

  fftData = new Uint8Array(analyser.frequencyBinCount);
  timeData = new Uint8Array(analyser.fftSize);

  // limpa canvas
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.fillStyle = "#000";
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  running = true;
  btnStart.disabled = true;
  btnStop.disabled = false;

  bufferSource.onended = () => stopAll();
  loop();
}