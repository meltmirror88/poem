/* Complæxity Exhibition Player
   Clean modular version.

   File structure:
   - index.html: normal version
   - index_inverted.html: inverted version
   - styles.css: exhibition fullscreen layout
   - pageData.js: PDF text coordinates + page metadata
   - assets/pages/page_###.png: high-resolution page images
*/

(() => {
  const pages = (window.PAGE_DATA || []).map(normalizePage);
  const isInverted = document.body.classList.contains("inverted");

  const canvas = document.getElementById("pageCanvas");
  const ctx = canvas.getContext("2d");
  const overlay = document.getElementById("overlay");
  const startBtn = document.getElementById("startBtn");
  const muteBtn = document.getElementById("muteBtn");
  const stage = document.getElementById("stage");

  const config = {
    firstPageMs: 3000,
    emptyPageMs: 3000,
    shortPageMs: 6000,
    longPageMs: 11000,
    extraLongPageMs: 13000,
    lastPageMs: 6000,

    longCharThreshold: 260,
    extraLongCharThreshold: 520,

    minTypeMs: 650,
    maxTypeRatio: 0.84,
    exitMs: 350,
    glitchMs: 180,
    cursorBlinkMs: 500,

    zoomLandscape: 2.0,
    zoomPortrait: 1.6,

    keyVolume: 0.06,
  };

  const state = {
    images: [],
    alienLayers: [],
    pageIndex: 0,
    phase: "idle",
    phaseStart: 0,
    phaseDuration: 1,
    pageStart: 0,
    holdMs: 400,
    lastVisibleChars: 0,
    started: false,
    raf: 0,
    audioEnabled: false,
    audioCtx: null,
    masterGain: null,
    noiseBuffer: null,
    lastKeySoundAt: 0,
  };

  function normalizePage(page) {
    const spans = (page.spans || []).slice().sort((a, b) => {
      const yDiff = a.bbox[1] - b.bbox[1];
      return Math.abs(yDiff) > 8 ? yDiff : a.bbox[0] - b.bbox[0];
    });

    return {
      ...page,
      spans,
      charCount: spans.reduce((sum, span) => {
        return sum + Math.max(1, span.chars || (span.text || "").length || 1);
      }, 0),
    };
  }

  function clamp(value, min, max) {
    return Math.min(max, Math.max(min, value));
  }

  function easeOutCubic(t) {
    return 1 - Math.pow(1 - t, 3);
  }

  function easeInOutQuad(t) {
    return t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2;
  }

  function pageTotalMs(index, page) {
    const chars = page.charCount || 0;

    if (index === 0) return config.firstPageMs;
    if (index === pages.length - 1) return config.lastPageMs;
    if (chars === 0) return config.emptyPageMs;
    if (chars > config.extraLongCharThreshold) return config.extraLongPageMs;
    if (chars > config.longCharThreshold) return config.longPageMs;

    return config.shortPageMs;
  }

  function pageTypingMs(index, page) {
    const chars = page.charCount || 0;

    if (index === 0 || chars === 0) return 0;

    const totalMs = pageTotalMs(index, page);
    const naturalTypingMs = chars * 52 + 260;

    return clamp(naturalTypingMs, config.minTypeMs, totalMs * config.maxTypeRatio);
  }

  function resizeCanvas() {
    const dpr = Math.max(1, Math.min(window.devicePixelRatio || 1, 3));
    const width = Math.round(window.innerWidth * dpr);
    const height = Math.round(window.innerHeight * dpr);

    if (canvas.width !== width || canvas.height !== height) {
      canvas.width = width;
      canvas.height = height;
    }
  }

  function getZoom() {
    return window.innerWidth > window.innerHeight
      ? config.zoomLandscape
      : config.zoomPortrait;
  }

  function getDrawMetrics(page, image) {
    const zoom = getZoom();
    const scale = (canvas.height / image.naturalHeight) * zoom;
    const drawWidth = image.naturalWidth * scale;
    const drawHeight = image.naturalHeight * scale;
    const dx = (canvas.width - drawWidth) / 2;
    const dy = (canvas.height - drawHeight) / 2;

    return {
      dx,
      dy,
      drawWidth,
      drawHeight,
      pdfScaleX: drawWidth / page.pdfWidth,
      pdfScaleY: drawHeight / page.pdfHeight,
    };
  }

  function setPhase(name, duration) {
    state.phase = name;
    state.phaseStart = performance.now();
    state.phaseDuration = Math.max(1, duration);
    state.lastVisibleChars = 0;
  }

  function beginPage(index) {
    state.pageIndex = index;
    state.pageStart = performance.now();
    state.lastKeySoundAt = 0;

    const page = pages[index];
    const totalMs = pageTotalMs(index, page);
    const typingMs = pageTypingMs(index, page);

    state.holdMs = Math.max(240, totalMs - typingMs - config.exitMs);

    if (index === 0) {
      setPhase("cover", totalMs - config.exitMs);
      return;
    }

    if ((page.charCount || 0) === 0) {
      setPhase("blank", totalMs - config.exitMs);
      return;
    }

    setPhase("typing", typingMs);
  }

  async function preloadImages() {
    state.images = await Promise.all(
      pages.map((page) => {
        return new Promise((resolve, reject) => {
          const img = new Image();
          img.onload = () => resolve(img);
          img.onerror = reject;
          img.src = `assets/pages/${page.image}`;
        });
      })
    );

    state.alienLayers = pages.map(buildAlienLayer);
  }

  function drawPage(page, image, visibleChars) {
    resizeCanvas();

    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = "#fff";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    const metrics = getDrawMetrics(page, image);
    ctx.drawImage(image, metrics.dx, metrics.dy, metrics.drawWidth, metrics.drawHeight);

    hideUnrevealedText(page, metrics, visibleChars);

    return metrics;
  }

  function hideUnrevealedText(page, metrics, visibleChars) {
    if (visibleChars >= (page.charCount || 0)) return;

    let remainingChars = visibleChars;

    for (const span of page.spans) {
      const [x1, y1, x2, y2] = span.bbox;
      const charLength = Math.max(1, span.chars || (span.text || "").length || 1);

      let revealRatio = 0;

      if (remainingChars <= 0) revealRatio = 0;
      else if (remainingChars >= charLength) revealRatio = 1;
      else revealRatio = remainingChars / charLength;

      remainingChars -= charLength;

      if (revealRatio < 1) {
        const spanX = metrics.dx + x1 * metrics.pdfScaleX;
        const spanY = metrics.dy + y1 * metrics.pdfScaleY;
        const spanWidth = (x2 - x1) * metrics.pdfScaleX;
        const spanHeight = (y2 - y1) * metrics.pdfScaleY;
        const coverX = spanX + spanWidth * revealRatio;
        const coverWidth = spanWidth * (1 - revealRatio);

        ctx.fillStyle = "#fff";
        ctx.fillRect(coverX - 1, spanY - 1, coverWidth + 2, spanHeight + 2);
      }
    }
  }

  function drawCursor(page, metrics, now) {
    if (!page.spans.length) return;
    if (Math.floor(now / config.cursorBlinkMs) % 2 !== 0) return;

    const lastSpan = page.spans[page.spans.length - 1];
    const [, y1, x2, y2] = lastSpan.bbox;
    const x = metrics.dx + x2 * metrics.pdfScaleX + 2;
    const y = metrics.dy + y1 * metrics.pdfScaleY + 1;
    const height = Math.max(14, (y2 - y1) * metrics.pdfScaleY) - 2;

    ctx.fillStyle = "#000";
    ctx.fillRect(x, y, Math.max(2, canvas.width * 0.0024), height);
  }

  function drawGlitch(page, image, elapsedMs) {
    const t = clamp(elapsedMs / config.glitchMs, 0, 1);
    if (t >= 1) return;

    const punch = Math.sin(Math.PI * t);
    const metrics = getDrawMetrics(page, image);

    ctx.save();

    ctx.globalAlpha = 0.20 + 0.28 * punch;
    ctx.fillStyle = "rgba(255,255,255,.85)";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    const split = 6 + 18 * punch;
    ctx.globalAlpha = 0.30 + 0.18 * punch;
    ctx.drawImage(image, metrics.dx + split, metrics.dy, metrics.drawWidth, metrics.drawHeight);
    ctx.drawImage(image, metrics.dx - split, metrics.dy, metrics.drawWidth, metrics.drawHeight);

    ctx.globalAlpha = 0.52 * punch;
    for (let i = 0; i < 5; i += 1) {
      const y = Math.random() * canvas.height;
      const sliceHeight = 6 + Math.random() * 18;
      const dx = (Math.random() - 0.5) * (54 + 120 * punch);
      ctx.drawImage(canvas, 0, y, canvas.width, sliceHeight, dx, y, canvas.width, sliceHeight);
    }

    ctx.globalAlpha = 0.28 * punch;
    ctx.fillStyle = "rgba(0,0,0,.75)";
    for (let i = 0; i < 3; i += 1) {
      const y = Math.random() * canvas.height;
      ctx.fillRect(0, y, canvas.width, 2 + Math.random() * 5);
    }

    ctx.restore();
  }

  function drawExitEffects(page, image, metrics, exitT) {
    const ghost = Math.max(0, 1 - exitT);

    if (ghost > 0) {
      ctx.save();
      ctx.globalAlpha = 0.10 * ghost;
      ctx.drawImage(image, metrics.dx + 8 + 10 * exitT, metrics.dy, metrics.drawWidth, metrics.drawHeight);
      ctx.globalAlpha = 0.08 * ghost;
      ctx.drawImage(image, metrics.dx - 6 - 9 * exitT, metrics.dy + 1, metrics.drawWidth, metrics.drawHeight);
      ctx.restore();
    }

    drawAlienText(page, metrics, state.alienLayers[state.pageIndex], exitT);

    ctx.save();
    ctx.globalAlpha = 0.58 * exitT;
    ctx.fillStyle = "#000";
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.restore();
  }

  function drawAlienText(page, metrics, layer, exitT) {
    const alpha = (1 - exitT) * 0.78;
    if (!layer || alpha <= 0) return;

    ctx.save();
    ctx.globalAlpha = alpha;
    ctx.fillStyle = "rgba(60,60,60,.62)";
    ctx.textBaseline = "top";

    for (const line of layer.lines) {
      const x =
        metrics.dx +
        line.x * metrics.pdfScaleX +
        Math.sin(performance.now() / 75 + line.jitter) * 3;
      const y = metrics.dy + line.y * metrics.pdfScaleY;
      const fontSize = line.fontSize * ((metrics.pdfScaleX + metrics.pdfScaleY) / 2);

      ctx.font = `${fontSize}px ui-monospace, SFMono-Regular, Menlo, Consolas, monospace`;
      ctx.fillText(line.text, x, y, line.width * metrics.pdfScaleX);

      if (Math.random() < 0.35) {
        ctx.globalAlpha = alpha * 0.28;
        ctx.fillText(
          line.text,
          x + (Math.random() - 0.5) * 18,
          y + (Math.random() - 0.5) * 7,
          line.width * metrics.pdfScaleX
        );
        ctx.globalAlpha = alpha;
      }
    }

    ctx.restore();
  }

  function buildAlienLayer(page, index) {
    const random = seeded((index + 1) * 12107);
    const box = getTextBox(page);
    const lineCount = 5 + Math.floor(random() * 5);
    const lines = [];

    for (let i = 0; i < lineCount; i += 1) {
      const y = box.y + box.height * (0.12 + random() * 0.76);
      const x = box.x + box.width * (0.02 + random() * 0.2);
      const width = box.width * (0.52 + random() * 0.42);
      const fontSize = Math.max(11, Math.min(20, box.height * 0.035 * (0.9 + random() * 0.7)));
      const wordCount = 3 + Math.floor(random() * 6);

      let text = "";
      for (let j = 0; j < wordCount; j += 1) {
        text += `${makeAlienWord(random)}   `;
      }

      lines.push({
        x,
        y,
        width,
        fontSize,
        text: text.trim(),
        jitter: random() * 3.2,
      });
    }

    return { lines };
  }

  function getTextBox(page) {
    if (!page.spans.length) {
      return {
        x: page.pdfWidth * 0.1,
        y: page.pdfHeight * 0.12,
        width: page.pdfWidth * 0.8,
        height: page.pdfHeight * 0.74,
      };
    }

    const xs1 = page.spans.map((span) => span.bbox[0]);
    const ys1 = page.spans.map((span) => span.bbox[1]);
    const xs2 = page.spans.map((span) => span.bbox[2]);
    const ys2 = page.spans.map((span) => span.bbox[3]);

    const minX = Math.min(...xs1);
    const minY = Math.min(...ys1);
    const maxX = Math.max(...xs2);
    const maxY = Math.max(...ys2);
    const padX = (maxX - minX) * 0.04 + 10;
    const padY = (maxY - minY) * 0.03 + 10;

    return {
      x: Math.max(20, minX - padX),
      y: Math.max(20, minY - padY),
      width: Math.min(page.pdfWidth - 40, maxX - minX + 2 * padX),
      height: Math.min(page.pdfHeight - 40, maxY - minY + 2 * padY),
    };
  }

  function makeAlienWord(random) {
    const tokens = [
      "[SYNK]",
      "[NULL]",
      "[ERROR]",
      "[AE]",
      "[POS]",
      "[TRACE]",
      "[RE:]",
      "[MEMORY]",
      `0x${Math.floor(random() * 65535).toString(16).padStart(4, "0")}`,
      "æ",
      "Æ",
      "∆",
      "▯",
      "▒",
      "░",
      "※",
      "⌁",
      "⌬",
      "⟡",
      "⟐",
      "∴",
      "∵",
      "INPUT",
      "MERGE",
      "RESTORE",
      "DREAM",
      "PORT",
      "SELF",
      "UNSTABLE",
      "거리불명",
      "좌표중첩",
      "번역불가",
      "복수인스턴스",
      "동기화실패",
    ];

    const count = 2 + Math.floor(random() * 5);
    const parts = [];

    for (let i = 0; i < count; i += 1) {
      parts.push(tokens[Math.floor(random() * tokens.length)]);
    }

    if (random() < 0.35) parts.push("//");
    if (random() < 0.28) parts.push("...");

    return parts.join(random() < 0.35 ? "" : " ");
  }

  function seeded(seed) {
    let value = seed % 2147483647;
    if (value <= 0) value += 2147483646;

    return () => {
      value = (value * 16807) % 2147483647;
      return value / 2147483647;
    };
  }

  function ensureAudio() {
    if (!state.audioCtx) {
      state.audioCtx = new (window.AudioContext || window.webkitAudioContext)();

      state.masterGain = state.audioCtx.createGain();
      state.masterGain.gain.value = 1.0;
      state.masterGain.connect(state.audioCtx.destination);

      const length = Math.floor(state.audioCtx.sampleRate * 0.11);
      state.noiseBuffer = state.audioCtx.createBuffer(1, length, state.audioCtx.sampleRate);

      const data = state.noiseBuffer.getChannelData(0);
      for (let i = 0; i < length; i += 1) {
        const env = Math.pow(1 - i / length, 4.1);
        data[i] = (Math.random() * 2 - 1) * env;
      }
    }

    if (state.audioCtx.state === "suspended") {
      state.audioCtx.resume();
    }
  }

  function playTypingSound() {
    if (!state.audioEnabled || !state.audioCtx || !state.masterGain || !state.noiseBuffer) return;

    const nowMs = performance.now();
    if (nowMs - state.lastKeySoundAt < 55 + Math.random() * 35) return;

    state.lastKeySoundAt = nowMs;
    const now = state.audioCtx.currentTime;

    const source = state.audioCtx.createBufferSource();
    source.buffer = state.noiseBuffer;

    const lowpass = state.audioCtx.createBiquadFilter();
    lowpass.type = "lowpass";
    lowpass.frequency.value = 520 + Math.random() * 180;
    lowpass.Q.value = 0.35;

    const body = state.audioCtx.createBiquadFilter();
    body.type = "peaking";
    body.frequency.value = 145 + Math.random() * 65;
    body.gain.value = 8.5;
    body.Q.value = 0.7;

    const gain = state.audioCtx.createGain();
    gain.gain.setValueAtTime(0.0001, now);
    gain.gain.linearRampToValueAtTime(config.keyVolume, now + 0.006);
    gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.085);

    source.connect(lowpass).connect(body).connect(gain).connect(state.masterGain);
    source.start(now);
    source.stop(now + 0.11);

    const thump = state.audioCtx.createOscillator();
    const thumpGain = state.audioCtx.createGain();

    thump.type = "triangle";
    thump.frequency.value = 72 + Math.random() * 42;
    thumpGain.gain.setValueAtTime(0.0001, now);
    thumpGain.gain.linearRampToValueAtTime(config.keyVolume * 0.28, now + 0.006);
    thumpGain.gain.exponentialRampToValueAtTime(0.0001, now + 0.075);

    thump.connect(thumpGain).connect(state.masterGain);
    thump.start(now);
    thump.stop(now + 0.09);
  }

  function start(withAudio) {
    state.started = true;
    state.audioEnabled = Boolean(withAudio);

    if (state.audioEnabled) ensureAudio();

    overlay.classList.add("hidden");
    beginPage(0);
    loop();
  }

  function nextPage() {
    const next = state.pageIndex >= pages.length - 1 ? 0 : state.pageIndex + 1;
    beginPage(next);
  }

  function advance() {
    if (!state.started || !overlay.classList.contains("hidden")) return;

    const page = pages[state.pageIndex];

    if (state.phase === "typing") {
      state.lastVisibleChars = page.charCount || 0;
      setPhase("hold", state.holdMs);
      return;
    }

    setPhase("exit", config.exitMs);
  }

  function loop() {
    cancelAnimationFrame(state.raf);
    state.raf = requestAnimationFrame(loop);

    if (!state.started) return;

    const now = performance.now();
    const page = pages[state.pageIndex];
    const image = state.images[state.pageIndex];

    const phaseT = clamp((now - state.phaseStart) / state.phaseDuration, 0, 1);
    const pageElapsed = now - state.pageStart;

    let visibleChars = page.charCount || 0;
    let exitT = 0;

    if (state.phase === "cover" || state.phase === "blank") {
      visibleChars = state.phase === "blank" ? 0 : page.charCount || 0;
      if (phaseT >= 1) setPhase("exit", config.exitMs);
    } else if (state.phase === "typing") {
      visibleChars = Math.floor((page.charCount || 0) * easeOutCubic(phaseT));

      if (visibleChars > state.lastVisibleChars) {
        playTypingSound();
        state.lastVisibleChars = visibleChars;
      }

      if (phaseT >= 1) setPhase("hold", state.holdMs);
    } else if (state.phase === "hold") {
      visibleChars = page.charCount || 0;
      if (phaseT >= 1) setPhase("exit", config.exitMs);
    } else if (state.phase === "exit") {
      visibleChars = page.charCount || 0;
      exitT = easeInOutQuad(phaseT);

      if (phaseT >= 1) {
        nextPage();
        return;
      }
    }

    const metrics = drawPage(page, image, visibleChars);

    if (state.pageIndex > 0 && pageElapsed < config.glitchMs && state.phase !== "exit") {
      drawGlitch(page, image, pageElapsed);
    }

    if (state.phase === "hold" && (page.charCount || 0) > 0) {
      drawCursor(page, metrics, now);
    }

    if (state.phase === "exit") {
      drawExitEffects(page, image, metrics, exitT);
    }
  }

  // iPad exhibition note:
  // Do not call requestFullscreen().
  // On iPadOS, native fullscreen can show a system-level X button in the upper-left corner.
  // Use “Add to Home Screen” instead for a clean exhibition display.
  function enterFullscreen() {
    // intentionally disabled
  }

  startBtn.addEventListener("click", () => {
    start(true);
  });

  muteBtn.addEventListener("click", () => {
    start(false);
  });

  stage.addEventListener("pointerdown", () => {
    advance();
  });

  document.addEventListener("keydown", (event) => {
    if (event.code === "Space") {
      event.preventDefault();
      advance();
    }
  });

  window.addEventListener("resize", () => {
    resizeCanvas();
  });

  preloadImages()
    .then(() => {
      resizeCanvas();
      drawPage(pages[0], state.images[0], pages[0].charCount || 0);
    })
    .catch((error) => {
      console.error("Failed to load assets:", error);
    });
})();
