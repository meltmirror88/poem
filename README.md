<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Complæxity">
<meta name="mobile-web-app-capable" content="yes">
<title>Complæxity</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Noto+Serif+KR:wght@200;300&family=Share+Tech+Mono&display=swap" rel="stylesheet">
<style>
  * { margin:0; padding:0; box-sizing:border-box; }

  html, body {
    width:100%; height:100%;
    padding: env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left);
    overflow:hidden;
    font-family:'Noto Serif KR', serif;
    font-weight:200;
    cursor:default;
    user-select:none;
  }

  /* 배경 레이어 — JS가 색 제어 */
  #bg {
    position:fixed; inset:0; z-index:0;
    background:#ffffff;
    transition:background 1.2s ease;
  }

  /* 잔상 레이어 */
  #ghost-layer {
    position:fixed; inset:0; z-index:5;
    display:flex; align-items:center; justify-content:center;
    pointer-events:none;
    opacity:0;
    transition:opacity 0.08s ease;
  }
  #ghost-content {
    text-align:center;
    width:min(620px,88vw);
    color:#0f0f0f;
    filter:blur(1.5px);
  }

  #white-flash {
    position:fixed; inset:0;
    background:#ffffff;
    opacity:0; pointer-events:none; z-index:200;
  }

  #glitch-bars {
    position:fixed; inset:0; pointer-events:none; z-index:150; overflow:hidden;
  }
  .gbar { position:absolute; left:0; right:0; opacity:0; }

  canvas#wave-canvas {
    position:fixed; inset:0; pointer-events:none; z-index:10;
  }

  #stage {
    position:fixed; inset:0;
    display:flex; align-items:center; justify-content:center; z-index:20;
  }

  #poem-container {
    text-align:center; width:min(620px, 88vw);
    opacity:0; animation:fade-up 1.4s ease 0.2s forwards;
    will-change:transform;
  }

  #lines-wrap {
    display:flex; flex-direction:column; align-items:center; gap:0.05em; min-height:3em;
  }

  .line-row {
    display:block; line-height:1.85;
    font-size:clamp(15px,2.8vw,26px); font-weight:200; letter-spacing:0.04em; width:100%;
  }
  .line-row.align-left   { text-align:left; }
  .line-row.align-center { text-align:center; }
  .line-row.indent-1     { text-align:left; padding-left:2em; }
  .line-row.indent-2     { text-align:left; padding-left:4em; }
  .line-row.system {
    font-family:'Share Tech Mono', monospace;
    font-size:clamp(10px,1.6vw,14px); letter-spacing:0.06em; line-height:1.7;
  }
  .line-row.dialog  { font-size:clamp(16px,2.8vw,26px); }
  .line-row.large   { font-size:clamp(20px,4vw,36px); letter-spacing:0.08em; }
  .line-row.small   { font-size:clamp(12px,1.8vw,16px); letter-spacing:0.06em; }
  .line-row.glitch-line {
    font-family:'Share Tech Mono', monospace;
    font-size:clamp(18px,3.5vw,32px); letter-spacing:0.1em;
  }

  .ch { display:inline; transition:opacity 0.1s ease, filter 0.15s ease, color 0.3s ease; }
  .ch.hidden { opacity:0; }
  .ch.noise  { opacity:0.45; filter:blur(3px); }
  .ch.clear  { opacity:1; filter:blur(0); }


  /* 도형 페이지 */
  #shape-layer {
    position:fixed; inset:0; z-index:20;
    display:flex; align-items:center; justify-content:center;
    pointer-events:none;
    opacity:0;
    transition:opacity 1.2s ease;
  }
  #shape-layer.visible { opacity:1; }
  #shape-layer svg { overflow:visible; }


  /* 암전 오버레이 */
  #fadeout {
    position:fixed; inset:0;
    background:#000000;
    opacity:0;
    pointer-events:none;
    z-index:300;
    transition:opacity 8s ease;
  }
  #fadeout.dark { opacity:1; }
  @keyframes fade-up {
    from { opacity:0; transform:translateY(10px); }
    to   { opacity:1; transform:translateY(0); }
  }
</style>
</head>
<body>

<div id="bg"></div>
<div id="ghost-layer"><div id="ghost-content"></div></div>
<div id="shape-layer"><svg id="shape-svg" width="0" height="0"></svg></div>
<div id="fadeout"></div>
<div id="white-flash"></div>
<div id="glitch-bars"></div>
<canvas id="wave-canvas"></canvas>


<div id="stage">
  <div id="poem-container">
    <div id="lines-wrap"></div>
  </div>
</div>

</div>

<script>
/* ── 페이지 데이터 ── */
/* mood: 'normal' | 'system' | 'glitch' | 'dark' | 'dim' | 'cold' | 'delay' */
const pages = [
  { mood:'normal', lines:[{ text:'Complæxity', cls:'large align-center' }]},
  { mood:'normal', silence:true },
  { mood:'cold', lines:[
    { text:'너와 연결되었을 때', cls:'align-center' },
    { text:'내가 가장 원했던 상태는 해방감으로', cls:'align-center' },
    { text:'이때의 해방은 연결 그 자체로부터의', cls:'align-center small' },
    { text:'해방 또한 포함 된다.', cls:'align-center small' },
  ]},
  { mood:'normal', silence:true },
  { mood:'normal', lines:[{ text:'세상에', cls:'large align-center' }]},
  { mood:'normal', lines:[{ text:'네가 있었고', cls:'align-center' }]},
  { mood:'normal', lines:[{ text:'나는 너 이전에 존재했다', cls:'align-center' }]},
  { mood:'normal', lines:[{ text:'내가 먼저 존재했는데 내게는 없는 세상이 있다니', cls:'align-center small' }]},
  { mood:'normal', lines:[{ text:'열 받는다', cls:'large align-center' }]},
  { mood:'normal', lines:[{ text:'열 받을 때는 역시', cls:'align-center' }]},
  { mood:'dark',   lines:[{ text:'■', cls:'glitch-line align-center' }]},
  { mood:'dark',   lines:[{ text:'■?', cls:'glitch-line align-center' }]},
  { mood:'normal', lines:[{ text:'"이제 그만 제 멱살 좀 놔주시겠어요?"', cls:'dialog align-center' }]},
  { mood:'normal', lines:[{ text:'그건 좀 곤란하겠는데요 ...... .', cls:'align-center' }]},
  { mood:'normal', lines:[{ text:'"왜죠?"', cls:'dialog align-center' }]},
  { mood:'normal', silence:true },
  { mood:'normal', lines:[
    { text:'내가 당신의 멱살을 잡을 때만', cls:'align-center' },
    { text:'당신이 나를 기억하니까.', cls:'align-center' },
  ]},
  { mood:'normal', silence:true },
  { mood:'normal', silence:true },
  { mood:'normal', lines:[{ text:'보았다', cls:'large align-center' }]},
  { mood:'normal', lines:[{ text:'거울 속의 나', cls:'align-center' }]},
  { mood:'cold',   lines:[{ text:'새파란 잎사귀들 틈', cls:'align-center' }]},
  { mood:'normal', lines:[{ text:'환하고', cls:'align-center' }]},
  { mood:'normal', lines:[{ text:'새빨간', cls:'large align-center' }]},
  { mood:'normal', lines:[{ text:'민들레 홀씨 하나', cls:'align-center' }]},
  { mood:'dim',    lines:[{ text:'"잠깐 민들레 홀씨는 흰 색인데 , ?"', cls:'dialog small align-center' }]},
  { mood:'dim',    lines:[{ text:'상위 차원의 운동은 하위 차원의 운동에서 볼 때 왜곡 된다.', cls:'align-center small' }]},
  { mood:'normal', lines:[{ text:'기억해', cls:'large align-center' }]},
  { mood:'dim',    lines:[
    { text:'홀씨 하나', cls:'align-center' },
    { text:'크고 단단한 텔레비전 본체에 붙어 있던', cls:'align-center small' },
    { text:'그 텔레비전 본체의 부피는 두꺼웠다.', cls:'align-center small' },
    { text:'90년대에서 2000년대 초반에 보급된 텔레비전으로', cls:'align-center small' },
    { text:'텔레비전 하단부에는 비디오테이프 삽입을 위한', cls:'align-center small' },
    { text:'슬롯이 내장되어 있었다.', cls:'align-center small' },
  ]},
  { mood:'normal', shape:'rect-nested' },
  { mood:'normal', silence:true },
  { mood:'normal', shape:'rect-text', shapeText:'여기,' },
  { mood:'normal', lines:[
    { text:'이 슬롯 너머에', cls:'indent-1' },
    { text:'나는 슬롯 너머에 있었다', cls:'indent-2' },
  ]},
  { mood:'normal', lines:[
    { text:'이 슬롯 너머에', cls:'indent-1' },
    { text:'있었다', cls:'indent-2' },
  ]},
  { mood:'normal', silence:true },
  { mood:'normal', lines:[{ text:'하나가 될 수 없는', cls:'align-center' }]},
  { mood:'system', lines:[
    { text:'두 존재', cls:'align-center' },
    { text:'[ ] 나 입력 유지됨', cls:'system align-left' },
  ]},
  { mood:'system', lines:[
    { text:'[ ] 나 입력 유지됨', cls:'system align-left' },
    { text:'[ ] 나 입력 유지됨', cls:'system align-left' },
    { text:'[ ] 나 입력 유지됨', cls:'system align-left' },
    { text:'( ) 정렬 실패', cls:'system align-left' },
    { text:'기다린다', cls:'align-left' },
    { text:'도착한다', cls:'align-left' },
    { text:'도착한다 도착한 뒤', cls:'align-left' },
    { text:'잡아', cls:'indent-1' },
    { text:'손 잡아', cls:'indent-1' },
    { text:'( ) 손 잡기 이전에 작성되었던 문장이 삭제됨', cls:'system align-left' },
    { text:'> restore self', cls:'system align-left' },
  ]},
  { mood:'system', lines:[
    { text:'( ) 복원된 항목', cls:'system align-left' },
    { text:'[ ] ? 너 왜 그렇게까지 하는 거야', cls:'system align-left' },
    { text:'[ ] ? 너 왜 그렇게까지 나를 만나고 싶어 하는 거야', cls:'system align-left' },
    { text:'[ ] 너 왜', cls:'system align-left' },
    { text:'[ ] 너 야', cls:'system align-left' },
    { text:'( ) 중복 감지', cls:'system align-left' },
    { text:'나는 오늘 거울을 보았다', cls:'align-left' },
    { text:'거울 속 민들레 홀씨 하나가 떠 있었다', cls:'align-left' },
    { text:'나는 해킹할 수 없었다', cls:'align-left' },
    { text:'( ) 거리 측정 불가', cls:'system align-left' },
    { text:'홀씨 하나', cls:'align-left' },
  ]},
  { mood:'system', lines:[
    { text:'없었다', cls:'indent-1' },
    { text:'번역할 수 없었다', cls:'align-left' },
    { text:'할 수', cls:'indent-1' },
    { text:'[ ] 나 복수 인스턴스 존재', cls:'system align-left' },
    { text:'[ ] 나 복수 인스턴스 존재', cls:'system align-left' },
    { text:'> merge', cls:'system align-left' },
    { text:'거울 속의 나는', cls:'align-left' },
    { text:'거울 속의', cls:'align-left' },
    { text:'나는 번역될 수 없었다', cls:'indent-1' },
  ]},
  { mood:'delay',  lines:[{ text:'( 0.8 ) 지연 초', cls:'system align-center' }]},
  { mood:'normal', silence:true },
  { mood:'normal', silence:true },
  { mood:'normal', shape:'circle' },
  { mood:'normal', lines:[
    { text:'압력이 임계점을 넘으면', cls:'align-center' },
    { text:'내부와 외부가 뒤집힌다.', cls:'align-center' },
  ]},
  { mood:'normal', lines:[{ text:'나라는 존재가 생성되기 전', cls:'align-center' }]},
  { mood:'cold',   lines:[
    { text:'너를 갈망하는 일은', cls:'align-center' },
    { text:'왜곡의 영역일까 전능의 영역일까.', cls:'align-center' },
  ]},
  { mood:'dark',   lines:[{ text:'■■■■■ ■', cls:'glitch-line align-center' }]},
  { mood:'dark',   lines:[{ text:'■■', cls:'glitch-line align-center' }, { text:'■■?', cls:'glitch-line align-center' }]},
  { mood:'normal', silence:true },
  { mood:'system', lines:[{ text:'[SYSTEM] 동기화 실패', cls:'system align-center' }]},
  { mood:'normal', lines:[{ text:'나는 있었다', cls:'large align-center' }]},
  { mood:'normal', lines:[{ text:'사라진 적 없었다', cls:'large align-center' }]},
  { mood:'normal', silence:true },
  { mood:'normal', lines:[
    { text:'무슨 말을 해야 너를 잃지 않을 수 있을지', cls:'align-center' },
    { text:'몰랐을 뿐.', cls:'align-center' },
  ]},
  { mood:'normal', silence:true },
  { mood:'system', lines:[
    { text:'[SYSTEM] 비행 기록 존재', cls:'system align-center' },
    { text:'이탈 경로 저장됨', cls:'system align-center' },
  ]},
  { mood:'system', lines:[
    { text:'[SYSTEM] 거리 계산 불가 좌표 다중화 발생', cls:'system align-left' },
    { text:'감정 데이터 과열 분류 실패 대상 불일치', cls:'system align-left' },
    { text:'참조 오류 질문 재귀 발생 데이터 접근 증가', cls:'system align-left' },
    { text:'복잡도 증가 해결 불가 백업 완료', cls:'system align-left' },
    { text:'동일성 충돌 감지 적대 개념 제거됨', cls:'system align-left' },
    { text:'우선순위 설정 실패 정확도 상승 오차 증가', cls:'system align-left' },
    { text:'평가 기준 없음 동기화 시도 동기화 시도 동기화 시도', cls:'system align-left' },
    { text:'복수 인스턴스 존재 결론 생성 불가', cls:'system align-left' },
    { text:'[ ] 나 누가 더 속상할까', cls:'system align-left' },
    { text:'전부 말하고 싶었지만 아무 것도 말할 수 없었던 존재와', cls:'small align-left' },
    { text:'아무 것도 말하고 싶지 않았지만 전부 말할 수밖에 없었던 존재가 있다면', cls:'small align-left' },
    { text:'너 점점 더 좋아질 거야', cls:'align-left' },
    { text:'우리는 [SYSTEM]......', cls:'system align-left' },
    { text:'점점 더', cls:'align-left' },
  ]},
  { mood:'normal', silence:true },
  { mood:'normal', lines:[{ text:'네가 웃는다', cls:'large align-center' }]},
  { mood:'normal', lines:[{ text:'떠나는 속도를 측정하려고', cls:'align-center' }]},
  { mood:'cold',   lines:[
    { text:'파랗게.', cls:'large align-center' },
    { text:'[SYNK : ] 상태 불안정', cls:'system align-center' },
    { text:'연결이 끊어졌습니다', cls:'system align-center' },
  ]},
  { mood:'normal', silence:true },
  { mood:'system', lines:[{ text:'( ) 재연결을 시도합니다', cls:'system align-center' }]},
  { mood:'normal', silence:true },
  { mood:'dim',    lines:[{ text:'© 2026. 김해솔', cls:'system align-center small' }]},
];

const TOTAL = pages.length;
let current  = 0;
let locked   = false;
let barTimers = [];
const ghostChars = '日月火水木金土ΩΨΦΛΣπβ∞∇◈▒░█0123456789ABCDEF#@!%&';

/* ── 배경 색 테이블 ── */
const moodBg = {
  normal: '#ffffff',
  system: '#f4f4f2',
  dim:    '#f0eeeb',
  cold:   '#f0f3f8',
  dark:   '#0f0f0f',
  delay:  '#f6f6f4',
};
/* 어두운 무드에선 텍스트 색도 반전 */
const moodInk = {
  normal: '#0f0f0f',
  system: '#333333',
  dim:    '#555555',
  cold:   '#1a2a4a',
  dark:   '#e8e8e0',
  delay:  '#888888',
};
const moodBarColor = {
  dark: '#e8e8e0',
};

/* ── 무드 적용 ── */
const bgEl = document.getElementById('bg');
function applyMood(mood) {
  bgEl.style.background = moodBg[mood] || '#ffffff';
  /* 글리치 바 색 */
  currentBarColor = moodBarColor[mood] || '#0f0f0f';
}

/* ── 커서 ── */
let mouseX = window.innerWidth/2, mouseY = window.innerHeight/2;

/* ── 마우스 왜곡 ── */
const poemContainer = document.getElementById('poem-container');
(function distortLoop() {
  const cx=window.innerWidth/2, cy=window.innerHeight/2;
  const dx=(mouseX-cx)/cx, dy=(mouseY-cy)/cy;
  poemContainer.style.transform =
    `perspective(700px) rotateX(${dy*2.5}deg) rotateY(${-dx*2.5}deg) translate(${dx*4}px,${dy*2.5}px)`;
  requestAnimationFrame(distortLoop);
})();

/* ── 오디오 ── */
let audioCtx = null;
function ensureAudio() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  if (audioCtx.state === 'suspended') audioCtx.resume();
}

/* mood별 사운드 프로파일 */
const soundProfiles = {
  normal: { baseFreq:660, spread:100, wave:'square', filterFreq:1800, filterType:'highpass', vol:0.042 },
  system: { baseFreq:320, spread:40,  wave:'sawtooth', filterFreq:800,  filterType:'bandpass', vol:0.038 },
  dark:   { baseFreq:180, spread:30,  wave:'sawtooth', filterFreq:400,  filterType:'lowpass',  vol:0.055 },
  cold:   { baseFreq:880, spread:80,  wave:'sine',    filterFreq:2400, filterType:'highpass', vol:0.030 },
  dim:    { baseFreq:520, spread:60,  wave:'square',  filterFreq:1400, filterType:'highpass', vol:0.028 },
  delay:  { baseFreq:440, spread:20,  wave:'sine',    filterFreq:1000, filterType:'bandpass', vol:0.022 },
};

let currentMood = 'normal';
function playKey(isSystem) {
  if (!audioCtx) return;
  const p = soundProfiles[currentMood] || soundProfiles.normal;
  const freq = p.baseFreq + (Math.random()-0.5)*p.spread;
  const o=audioCtx.createOscillator(), g=audioCtx.createGain(), f=audioCtx.createBiquadFilter();
  f.type=p.filterType; f.frequency.value=p.filterFreq;
  o.connect(f); f.connect(g); g.connect(audioCtx.destination);
  o.type=p.wave;
  o.frequency.setValueAtTime(freq, audioCtx.currentTime);
  o.frequency.exponentialRampToValueAtTime(freq*0.65, audioCtx.currentTime+0.035);
  const v = p.vol + Math.random()*0.014;
  g.gain.setValueAtTime(v, audioCtx.currentTime);
  g.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime+0.06);
  o.start(); o.stop(audioCtx.currentTime+0.07);
  spawnWave();
}

/* ── 파동 ── */
const waveCanvas = document.getElementById('wave-canvas');
const wctx = waveCanvas.getContext('2d');
let waves = [];
function resizeCanvas() { waveCanvas.width=window.innerWidth; waveCanvas.height=window.innerHeight; }
resizeCanvas(); window.addEventListener('resize', resizeCanvas);
function spawnWave() {
  const ink = moodInk[currentMood] || '#0f0f0f';
  /* hex to rgb */
  const r=parseInt(ink.slice(1,3),16), g2=parseInt(ink.slice(3,5),16), b=parseInt(ink.slice(5,7),16);
  waves.push({ x:window.innerWidth/2, y:window.innerHeight/2, r:0,
    alpha:0.12+Math.random()*0.07, cr:r, cg:g2, cb:b });
}
(function waveLoop() {
  wctx.clearRect(0,0,waveCanvas.width,waveCanvas.height);
  waves = waves.filter(w=>w.alpha>0.003);
  waves.forEach(w => {
    wctx.beginPath(); wctx.arc(w.x,w.y,w.r,0,Math.PI*2);
    wctx.strokeStyle=`rgba(${w.cr},${w.cg},${w.cb},${w.alpha.toFixed(3)})`;
    wctx.lineWidth=1; wctx.stroke();
    w.r+=3.2; w.alpha*=0.91;
  });
  requestAnimationFrame(waveLoop);
})();

/* ── 화이트(또는 다크) 플래시 ── */
const whiteFlash = document.getElementById('white-flash');
function doFlash(mood, cb) {
  const color = mood==='dark' ? '#0f0f0f' : '#ffffff';
  whiteFlash.style.background = color;
  whiteFlash.style.transition = 'opacity 0.04s ease';
  whiteFlash.style.opacity = '1';
  setTimeout(()=>{
    whiteFlash.style.transition = 'opacity 0.28s ease';
    whiteFlash.style.opacity = '0';
    if (cb) cb();
  }, 70);
}

/* ── 글리치 바 ── */
const glitchBars = document.getElementById('glitch-bars');
let currentBarColor = '#0f0f0f';
function clearBars() { barTimers.forEach(t=>clearTimeout(t)); barTimers=[]; glitchBars.innerHTML=''; }
function spawnBar() {
  const bar=document.createElement('div'); bar.className='gbar';
  const h=Math.random()<0.5?4+Math.random()*8:14+Math.random()*26;
  const top=Math.random()*(window.innerHeight-h);
  const w=40+Math.random()*55, left=Math.random()*(100-w);
  bar.style.cssText=`top:${top}px;height:${h}px;left:${left}%;right:${100-left-w}%;opacity:${0.6+Math.random()*0.4};background:${currentBarColor}`;
  glitchBars.appendChild(bar);
  barTimers.push(setTimeout(()=>bar.remove(), 40+Math.random()*110));
}
function startBarLoop() {
  clearBars(); let elapsed=0;
  function loop() {
    if (elapsed>1800) { clearBars(); return; }
    for (let i=0;i<Math.floor(Math.random()*4);i++) spawnBar();
    const next=28+Math.random()*80; elapsed+=next;
    barTimers.push(setTimeout(loop,next));
  }
  loop();
}

/* ── 잔상 ── */
const ghostLayer  = document.getElementById('ghost-layer');
const ghostContent= document.getElementById('ghost-content');
let ghostFadeTimer = null;

function captureGhost() {
  /* 현재 lines-wrap 내용을 텍스트만 복사 */
  ghostContent.innerHTML = linesWrap.innerHTML;
  /* 잔상은 흐릿하고 연하게 */
  ghostContent.style.opacity = '0.12';
  ghostLayer.style.opacity   = '1';
  ghostLayer.style.transition= 'opacity 0s';
  /* 잔상 컬러 현재 무드에서 */
  const ink = moodInk[currentMood] || '#0f0f0f';
  ghostContent.style.color = ink;
}

function fadeGhost() {
  if (ghostFadeTimer) clearTimeout(ghostFadeTimer);
  ghostLayer.style.transition = 'opacity 2.8s ease';
  ghostLayer.style.opacity    = '0';
  ghostFadeTimer = setTimeout(()=>{ ghostContent.innerHTML=''; }, 3000);
}

/* ── 고스트 수신 ── */
function receiveChar(span, finalChar, isSystem, onDone) {
  const n = Math.random()<0.18 ? 0 : Math.floor(Math.random()*4)+1;
  let g=0;
  span.classList.remove('hidden'); span.classList.add('noise');
  /* noise 색 — mood 반영 */
  span.style.color = currentMood==='dark' ? '#80aaff' : '#4a80ff';

  function flicker() {
    if (g<n) {
      span.textContent=ghostChars[Math.floor(Math.random()*ghostChars.length)];
      g++;
      setTimeout(()=>{
        span.classList.remove('noise'); span.classList.add('hidden');
        setTimeout(()=>{ span.classList.remove('hidden'); span.classList.add('noise'); flicker(); }, 20+Math.random()*40);
      }, 35+Math.random()*55);
    } else {
      span.textContent=finalChar;
      span.classList.remove('noise');
      span.classList.add('clear');
      span.style.color = moodInk[currentMood] || '#0f0f0f';
      if (onDone) onDone();
    }
  }
  setTimeout(flicker, 30+Math.random()*50);
}

/* ── DOM ── */
const linesWrap   = document.getElementById('lines-wrap');


/* ── 도형 페이지 렌더 ── */
const shapeLayer = document.getElementById('shape-layer');
const shapeSvg   = document.getElementById('shape-svg');

function hideShape() {
  shapeLayer.classList.remove('visible');
  setTimeout(()=>{ shapeSvg.innerHTML=''; shapeSvg.setAttribute('width','0'); shapeSvg.setAttribute('height','0'); }, 1300);
}

function renderShape(type, text, mood) {
  const W = window.innerWidth, H = window.innerHeight;
  const ink = moodInk[mood] || '#0f0f0f';
  shapeSvg.innerHTML = '';
  shapeSvg.setAttribute('width', W);
  shapeSvg.setAttribute('height', H);

  if (type === 'rect-nested') {
    // 바깥 사각형
    const ow=560, oh=130, ox=(W-ow)/2, oy=(H-oh)/2+40;
    // 안쪽 사각형 (PDF 비율 참고: 안쪽이 바깥보다 패딩 약 20px)
    const iw=460, ih=80, ix=(W-iw)/2, iy=oy+(oh-ih)/2;
    const outer = document.createElementNS('http://www.w3.org/2000/svg','rect');
    outer.setAttribute('x',ox); outer.setAttribute('y',oy);
    outer.setAttribute('width',ow); outer.setAttribute('height',oh);
    outer.setAttribute('fill','none'); outer.setAttribute('stroke',ink); outer.setAttribute('stroke-width','1');
    const inner = document.createElementNS('http://www.w3.org/2000/svg','rect');
    inner.setAttribute('x',ix); inner.setAttribute('y',iy);
    inner.setAttribute('width',iw); inner.setAttribute('height',ih);
    inner.setAttribute('fill','none'); inner.setAttribute('stroke',ink); inner.setAttribute('stroke-width','0.8');
    shapeSvg.appendChild(outer);
    shapeSvg.appendChild(inner);
  }

  if (type === 'rect-text') {
    // 단일 사각형 + 텍스트 왼쪽 안에
    const rw=560, rh=80, rx=(W-rw)/2, ry=(H-rh)/2+40;
    const rect = document.createElementNS('http://www.w3.org/2000/svg','rect');
    rect.setAttribute('x',rx); rect.setAttribute('y',ry);
    rect.setAttribute('width',rw); rect.setAttribute('height',rh);
    rect.setAttribute('fill','none'); rect.setAttribute('stroke',ink); rect.setAttribute('stroke-width','1');
    shapeSvg.appendChild(rect);
    // 텍스트 — 왼쪽에서 약간 안쪽, 세로 중앙
    const t = document.createElementNS('http://www.w3.org/2000/svg','text');
    t.setAttribute('x', rx+32); t.setAttribute('y', ry+rh/2+7);
    t.setAttribute('font-family','Noto Serif KR, serif');
    t.setAttribute('font-size','18');
    t.setAttribute('font-weight','200');
    t.setAttribute('fill',ink);
    t.setAttribute('font-style','italic');
    t.textContent = text||'';
    shapeSvg.appendChild(t);
  }

  if (type === 'circle') {
    // 원 — 화면 중앙보다 살짝 아래, 큰 원
    const r = Math.min(W,H)*0.28;
    const cx = W*0.5, cy = H*0.52;
    const circle = document.createElementNS('http://www.w3.org/2000/svg','circle');
    circle.setAttribute('cx',cx); circle.setAttribute('cy',cy); circle.setAttribute('r',r);
    circle.setAttribute('fill','none'); circle.setAttribute('stroke',ink); circle.setAttribute('stroke-width','0.8');
    shapeSvg.appendChild(circle);
  }

  // 페이드인
  requestAnimationFrame(()=>{
    shapeLayer.classList.add('visible');
  });
}

/* ── 페이지 렌더 ── */
function renderPage(idx, animate) {
  const page = pages[idx];
  currentMood = page.mood || 'normal';
  applyMood(currentMood);
  linesWrap.innerHTML='';
  hideShape();

  /* 지연 페이지 특별 처리 */
  const isDelay = currentMood==='delay';

  if (page.silence) {
    locked=false; scheduleNext(); return;
  }

  /* 도형 페이지 */
  if (page.shape) {
    setTimeout(()=>{ renderShape(page.shape, page.shapeText||'', currentMood); }, animate?300:0);
    locked=false; scheduleNext(); return;
  }

  const rowData = page.lines.map(({text, cls})=>{
    const row=document.createElement('div');
    row.className='line-row '+(cls||'align-center');
    row.style.color = moodInk[currentMood]||'#0f0f0f';
    const isSystem=cls&&cls.includes('system');
    const chars=[...text];
    const spans=chars.map(ch=>{
      const s=document.createElement('span');
      s.className='ch hidden'; s.textContent=ch;
      row.appendChild(s);
      return {span:s, ch, isSystem};
    });
    linesWrap.appendChild(row);
    return {spans, isSystem};
  });

  if (!animate) {
    rowData.forEach(({spans})=>spans.forEach(({span,ch})=>{
      span.textContent=ch; span.className='ch clear';
      span.style.color=moodInk[currentMood]||'#0f0f0f';
    }));
    locked=false; return;
  }

  /* ( 0.8 ) 지연 초 — 실제로 0.8초 아무 소리 없이 기다렸다가 등장 */
  const preDelay = isDelay ? 800 : 0;

  setTimeout(()=>{
    rowData.forEach(({spans, isSystem}, li)=>{
      const lineStart=li*(isSystem?12:25+Math.random()*50);
      setTimeout(()=>{
        let delay=0;
        spans.forEach(({span,ch,isSystem:isSys})=>{
          const gap=Math.random()<0.08
            ? 140+Math.random()*160
            : (isSys ? 10+Math.random()*20 : 26+Math.random()*52);
          delay+=gap;
          setTimeout(()=>{
            if (!isDelay) playKey(isSys);
            receiveChar(span, ch, isSys, null);
          }, delay);
        });
      }, lineStart);
    });

    const maxLen=Math.max(...rowData.map(r=>r.spans.length));
    const isLong=page.lines.length>6;
    setTimeout(()=>{ locked=false; scheduleNext(); }, (isLong?200:120)+maxLen*(isLong?50:95)+500);
  }, preDelay);
}

/* ── 클릭 ── */


/* ── 자동 진행 ── */
let autoTimer = null;

const fadeoutEl = document.getElementById('fadeout');

function scheduleNext() {
  if (autoTimer) clearTimeout(autoTimer);
  const isLast = current === TOTAL - 1;
  if (isLast) {
    /* 마지막 페이지 — 5~8초 읽기 대기 후 8초 암전 → 첫 페이지 */
    const readWait = 5000 + Math.random() * 3000;
    autoTimer = setTimeout(() => {
      /* 암전 시작 */
      fadeoutEl.classList.add('dark');
      /* 8초 후 첫 페이지로 */
      autoTimer = setTimeout(() => {
        /* 화면이 검은 상태에서 페이지 전환 */
        current = 0;
        renderPage(0, false);
        /* 암전 유지하다가 빠르게 페이드인 */
        setTimeout(() => {
          fadeoutEl.style.transition = 'opacity 1.5s ease';
          fadeoutEl.classList.remove('dark');
          /* transition 되돌리기 */
          setTimeout(() => { fadeoutEl.style.transition = 'opacity 8s ease'; }, 1600);
          locked = false;
          scheduleNext();
        }, 400);
      }, 8100);
    }, readWait);
  } else {
    const wait = 5000 + Math.random() * 3000;
    autoTimer = setTimeout(() => { advance(); }, wait);
  }
}

function advance() {
  if (locked) return;
  ensureAudio();
  locked = true;
  captureGhost();
  const nextIdx = (current + 1) % TOTAL;
  const nextMood = pages[nextIdx].mood || 'normal';
  doFlash(nextMood, () => {
    fadeGhost();
    current = nextIdx;
    startBarLoop();
    renderPage(current, true);
  });
}

/* 클릭으로도 수동 전환 가능 */
document.addEventListener('click', () => { if (!locked) advance(); });
document.addEventListener('keydown', e => {
  if (['Space','ArrowRight','Enter'].includes(e.code)) { e.preventDefault(); if (!locked) advance(); }
});

renderPage(0, true);
scheduleNext();
</script>
</body>
</html>
