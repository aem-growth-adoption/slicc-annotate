# Sprinkle HTML reference — annotate

This is the structural reference for the sprinkle at
`/shared/sprinkles/annotate/annotate.shtml`.

Read this when building or modifying the sprinkle. It documents the full
architecture: DOM structure, CSS, JS stroke model, toolbar, and communication protocol.

## Full DOM structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Annotate</title>
  <link rel="icon" href="pencil" />
  <style>
    /* Full-viewport canvas, no scroll */
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { width: 100%; height: 100%; overflow: hidden; background: #1a1a2e; }
    #canvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; cursor: crosshair; }
    #text-input {
      position: absolute; display: none; z-index: 2147483646;
      background: transparent; border: 2px dashed currentColor;
      color: inherit; font: 16px sans-serif; padding: 2px 4px;
      outline: none; min-width: 100px;
    }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  <input id="text-input" type="text" />
  <!-- Toolbar is injected via Shadow DOM in JS -->
  <script>
  // === FULL JS BELOW ===
  </script>
</body>
</html>
```

## Toolbar (Shadow DOM)

The toolbar is created as a `<div>` with a Shadow DOM root, positioned fixed at top center.

```js
(function() {
  var toolbarHost = document.createElement('div');
  toolbarHost.id = 'toolbar-host';
  document.body.appendChild(toolbarHost);
  var shadow = toolbarHost.attachShadow({ mode: 'open' });
  shadow.innerHTML = `
    <style>
      :host {
        position: fixed; top: 12px; left: 50%; transform: translateX(-50%);
        z-index: 2147483647; pointer-events: auto;
      }
      .toolbar {
        display: flex; align-items: center; gap: 6px; padding: 8px 16px;
        background: rgba(30, 30, 50, 0.92); backdrop-filter: blur(12px);
        border-radius: 9999px; box-shadow: 0 4px 24px rgba(0,0,0,0.4);
        font: 13px/1 -apple-system, BlinkMacSystemFont, sans-serif;
        color: #e0e0e0;
      }
      button {
        display: flex; align-items: center; justify-content: center;
        height: 32px; min-width: 32px; padding: 0 10px;
        border: 1px solid rgba(255,255,255,0.15); border-radius: 9999px;
        background: transparent; color: #e0e0e0; cursor: pointer;
        font: inherit; transition: all 0.15s;
      }
      button:hover { background: rgba(255,255,255,0.1); }
      button.active { background: rgba(99,102,241,0.3); border-color: #6366f1; color: #a5b4fc; }
      button.capture { background: #6366f1; border-color: #6366f1; color: #fff; font-weight: 600; }
      button.capture:hover { background: #4f46e5; }
      button.close { border-color: rgba(239,68,68,0.4); color: #fca5a5; }
      button.close:hover { background: rgba(239,68,68,0.15); }
      .sep { width: 1px; height: 20px; background: rgba(255,255,255,0.15); margin: 0 4px; }
      .swatch {
        width: 20px; height: 20px; border-radius: 50%; border: 2px solid transparent;
        cursor: pointer; transition: border-color 0.15s;
      }
      .swatch.active { border-color: #fff; }
      svg { width: 16px; height: 16px; fill: none; stroke: currentColor; stroke-width: 2; stroke-linecap: round; stroke-linejoin: round; }
    </style>
    <div class="toolbar">
      <!-- Tool buttons with inline SVG icons -->
      <button data-tool="pencil" class="active" title="Pencil (1)">
        <svg viewBox="0 0 24 24"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/></svg>
      </button>
      <button data-tool="highlighter" title="Highlighter (2)">
        <svg viewBox="0 0 24 24"><path d="m9 11-6 6v3h9l3-3"/><path d="m22 12-4.6 4.6a2 2 0 0 1-2.8 0l-5.2-5.2a2 2 0 0 1 0-2.8L14 4"/></svg>
      </button>
      <button data-tool="rectangle" title="Rectangle (3)">
        <svg viewBox="0 0 24 24"><rect x="3" y="3" width="18" height="18" rx="2"/></svg>
      </button>
      <button data-tool="ellipse" title="Ellipse (4)">
        <svg viewBox="0 0 24 24"><ellipse cx="12" cy="12" rx="10" ry="8"/></svg>
      </button>
      <button data-tool="arrow" title="Arrow (5)">
        <svg viewBox="0 0 24 24"><line x1="5" y1="12" x2="19" y2="12"/><polyline points="12 5 19 12 12 19"/></svg>
      </button>
      <button data-tool="text" title="Text (6)">
        <svg viewBox="0 0 24 24"><polyline points="4 7 4 4 20 4 20 7"/><line x1="9" y1="20" x2="15" y2="20"/><line x1="12" y1="4" x2="12" y2="20"/></svg>
      </button>
      <div class="sep"></div>
      <!-- Color swatches -->
      <div class="swatch active" data-color="#ef4444" style="background:#ef4444" title="Red"></div>
      <div class="swatch" data-color="#3b82f6" style="background:#3b82f6" title="Blue"></div>
      <div class="swatch" data-color="#22c55e" style="background:#22c55e" title="Green"></div>
      <div class="swatch" data-color="#eab308" style="background:#eab308" title="Yellow"></div>
      <div class="swatch" data-color="#ffffff" style="background:#ffffff" title="White"></div>
      <div class="swatch" data-color="#000000" style="background:#000000; border: 1px solid rgba(255,255,255,0.3)" title="Black"></div>
      <div class="sep"></div>
      <button data-action="undo" title="Undo (Ctrl+Z)">
        <svg viewBox="0 0 24 24"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 2.13-9.36L1 10"/></svg>
      </button>
      <button data-action="clear" title="Clear all">
        <svg viewBox="0 0 24 24"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/></svg>
      </button>
      <div class="sep"></div>
      <button data-action="capture" class="capture" title="Capture">Capture</button>
      <button data-action="close" class="close" title="Close (Esc)">Close</button>
    </div>
  `;
})();
```

## JS Architecture

All code inside a single `<script>` in an IIFE. No globals.

### Constants and state

```js
(function() {
  'use strict';

  var canvas = document.getElementById('canvas');
  var ctx = canvas.getContext('2d');
  var textInput = document.getElementById('text-input');

  // HiDPI scaling
  var dpr = window.devicePixelRatio || 1;

  // State
  var baseImage = null;        // HTMLImageElement or null (blank canvas fallback)
  var strokes = [];            // Array of Stroke objects
  var currentStroke = null;    // In-progress stroke during drag
  var currentTool = 'pencil';
  var currentColor = '#ef4444';
  var isDrawing = false;

  // Stroke object shape:
  // { tool, color, lineWidth, opacity, points?, rect?, start?, end?, text?, position? }
})();
```

### Canvas sizing

```js
function resizeCanvas() {
  canvas.width = window.innerWidth * dpr;
  canvas.height = window.innerHeight * dpr;
  canvas.style.width = window.innerWidth + 'px';
  canvas.style.height = window.innerHeight + 'px';
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  redraw();
}
window.addEventListener('resize', resizeCanvas);
resizeCanvas();
```

### Redraw loop

```js
function redraw() {
  ctx.clearRect(0, 0, canvas.width / dpr, canvas.height / dpr);

  // Draw base image (fit to viewport, centered)
  if (baseImage) {
    var scale = Math.min(
      (canvas.width / dpr) / baseImage.naturalWidth,
      (canvas.height / dpr) / baseImage.naturalHeight
    );
    var w = baseImage.naturalWidth * scale;
    var h = baseImage.naturalHeight * scale;
    var x = ((canvas.width / dpr) - w) / 2;
    var y = ((canvas.height / dpr) - h) / 2;
    ctx.drawImage(baseImage, x, y, w, h);
  }

  // Draw all committed strokes
  for (var i = 0; i < strokes.length; i++) {
    drawStroke(strokes[i]);
  }

  // Draw in-progress stroke
  if (currentStroke) {
    drawStroke(currentStroke);
  }
}
```

### Drawing individual strokes

```js
function drawStroke(s) {
  ctx.save();
  ctx.strokeStyle = s.color;
  ctx.fillStyle = s.color;
  ctx.lineWidth = s.lineWidth;
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';
  ctx.globalAlpha = s.opacity;

  switch (s.tool) {
    case 'pencil':
    case 'highlighter':
      if (s.points && s.points.length > 1) {
        ctx.beginPath();
        ctx.moveTo(s.points[0].x, s.points[0].y);
        for (var i = 1; i < s.points.length; i++) {
          ctx.lineTo(s.points[i].x, s.points[i].y);
        }
        ctx.stroke();
      }
      break;

    case 'rectangle':
      if (s.rect) {
        ctx.strokeRect(s.rect.x, s.rect.y, s.rect.w, s.rect.h);
      }
      break;

    case 'ellipse':
      if (s.rect) {
        var cx = s.rect.x + s.rect.w / 2;
        var cy = s.rect.y + s.rect.h / 2;
        var rx = Math.abs(s.rect.w / 2);
        var ry = Math.abs(s.rect.h / 2);
        ctx.beginPath();
        ctx.ellipse(cx, cy, rx, ry, 0, 0, Math.PI * 2);
        ctx.stroke();
      }
      break;

    case 'arrow':
      if (s.start && s.end) {
        // Line
        ctx.beginPath();
        ctx.moveTo(s.start.x, s.start.y);
        ctx.lineTo(s.end.x, s.end.y);
        ctx.stroke();
        // Arrowhead
        var angle = Math.atan2(s.end.y - s.start.y, s.end.x - s.start.x);
        var headLen = Math.max(12, s.lineWidth * 4);
        ctx.beginPath();
        ctx.moveTo(s.end.x, s.end.y);
        ctx.lineTo(s.end.x - headLen * Math.cos(angle - Math.PI / 6),
                   s.end.y - headLen * Math.sin(angle - Math.PI / 6));
        ctx.moveTo(s.end.x, s.end.y);
        ctx.lineTo(s.end.x - headLen * Math.cos(angle + Math.PI / 6),
                   s.end.y - headLen * Math.sin(angle + Math.PI / 6));
        ctx.stroke();
      }
      break;

    case 'text':
      if (s.text && s.position) {
        ctx.font = s.lineWidth * 5 + 'px sans-serif';
        ctx.fillText(s.text, s.position.x, s.position.y);
      }
      break;
  }
  ctx.restore();
}
```

### Pointer event handlers

```js
function getPos(e) {
  var rect = canvas.getBoundingClientRect();
  return { x: e.clientX - rect.left, y: e.clientY - rect.top };
}

canvas.addEventListener('pointerdown', function(e) {
  if (currentTool === 'text') {
    showTextInput(e);
    return;
  }
  isDrawing = true;
  var pos = getPos(e);
  var lineWidth = currentTool === 'highlighter' ? 20 : 3;
  var opacity = currentTool === 'highlighter' ? 0.4 : 1;

  currentStroke = {
    tool: currentTool,
    color: currentColor,
    lineWidth: lineWidth,
    opacity: opacity
  };

  if (currentTool === 'pencil' || currentTool === 'highlighter') {
    currentStroke.points = [pos];
  } else if (currentTool === 'rectangle' || currentTool === 'ellipse') {
    currentStroke.rect = { x: pos.x, y: pos.y, w: 0, h: 0 };
    currentStroke._startPos = pos;
  } else if (currentTool === 'arrow') {
    currentStroke.start = pos;
    currentStroke.end = pos;
  }
  canvas.setPointerCapture(e.pointerId);
});

canvas.addEventListener('pointermove', function(e) {
  if (!isDrawing || !currentStroke) return;
  var pos = getPos(e);

  if (currentTool === 'pencil' || currentTool === 'highlighter') {
    currentStroke.points.push(pos);
  } else if (currentTool === 'rectangle' || currentTool === 'ellipse') {
    currentStroke.rect.w = pos.x - currentStroke._startPos.x;
    currentStroke.rect.h = pos.y - currentStroke._startPos.y;
  } else if (currentTool === 'arrow') {
    currentStroke.end = pos;
  }
  redraw();
});

canvas.addEventListener('pointerup', function(e) {
  if (!isDrawing || !currentStroke) return;
  isDrawing = false;
  // Clean up internal properties
  delete currentStroke._startPos;
  strokes.push(currentStroke);
  currentStroke = null;
  redraw();
});
```

### Text tool

```js
function showTextInput(e) {
  var pos = getPos(e);
  textInput.style.display = 'block';
  textInput.style.left = pos.x + 'px';
  textInput.style.top = pos.y + 'px';
  textInput.style.color = currentColor;
  textInput.value = '';
  textInput.focus();
  textInput._pos = pos;
}

textInput.addEventListener('keydown', function(e) {
  if (e.key === 'Enter') {
    e.preventDefault();
    if (textInput.value.trim()) {
      strokes.push({
        tool: 'text',
        color: currentColor,
        lineWidth: 3,
        opacity: 1,
        text: textInput.value,
        position: textInput._pos
      });
      redraw();
    }
    textInput.style.display = 'none';
  } else if (e.key === 'Escape') {
    textInput.style.display = 'none';
  }
});
```

### Toolbar event wiring (inside Shadow DOM)

```js
var shadow = document.getElementById('toolbar-host').shadowRoot;

// Tool buttons
shadow.querySelectorAll('[data-tool]').forEach(function(btn) {
  btn.addEventListener('click', function() {
    shadow.querySelectorAll('[data-tool]').forEach(function(b) { b.classList.remove('active'); });
    btn.classList.add('active');
    currentTool = btn.dataset.tool;
  });
});

// Color swatches
shadow.querySelectorAll('.swatch').forEach(function(sw) {
  sw.addEventListener('click', function() {
    shadow.querySelectorAll('.swatch').forEach(function(s) { s.classList.remove('active'); });
    sw.classList.add('active');
    currentColor = sw.dataset.color;
  });
});

// Action buttons
shadow.querySelector('[data-action="undo"]').addEventListener('click', function() {
  strokes.pop();
  redraw();
});

shadow.querySelector('[data-action="clear"]').addEventListener('click', function() {
  strokes = [];
  currentStroke = null;
  redraw();
});

shadow.querySelector('[data-action="capture"]').addEventListener('click', captureAndSend);
shadow.querySelector('[data-action="close"]').addEventListener('click', function() {
  slicc.lick({ action: 'close' });
  slicc.close();
});
```

### Capture and send

```js
function captureAndSend() {
  // Determine output dimensions (cap at 1568px long edge)
  var srcW = canvas.width / dpr;
  var srcH = canvas.height / dpr;
  var maxEdge = 1568;
  var scale = 1;
  if (Math.max(srcW, srcH) > maxEdge) {
    scale = maxEdge / Math.max(srcW, srcH);
  }
  var outW = Math.round(srcW * scale);
  var outH = Math.round(srcH * scale);

  var offscreen = document.createElement('canvas');
  offscreen.width = outW;
  offscreen.height = outH;
  var octx = offscreen.getContext('2d');
  octx.drawImage(canvas, 0, 0, canvas.width, canvas.height, 0, 0, outW, outH);

  var quality = 0.85;
  var dataUrl = offscreen.toDataURL('image/jpeg', quality);

  // If >5MB, downscale further
  while (dataUrl.length > 5 * 1024 * 1024 * 1.37 && quality > 0.3) {
    quality -= 0.1;
    dataUrl = offscreen.toDataURL('image/jpeg', quality);
  }

  slicc.lick({ action: 'capture', dataUrl: dataUrl });
}
```

### Receiving base image from scoop

```js
slicc.on('update', function(data) {
  if (data.baseImage) {
    var img = new Image();
    img.onload = function() {
      baseImage = img;
      redraw();
    };
    img.src = data.baseImage;
  }
});
```

### Request screenshot on load

```js
// On initial load, request a screenshot from the extension
setTimeout(function() {
  slicc.lick({ action: 'request-screenshot' });
}, 300);
```

### Keyboard shortcuts

```js
document.addEventListener('keydown', function(e) {
  // Don't capture when text input is focused
  if (textInput.style.display !== 'none' && document.activeElement === textInput) return;

  if (e.key === 'Escape') {
    slicc.lick({ action: 'close' });
    slicc.close();
  } else if (e.key === '1') { selectTool('pencil'); }
  else if (e.key === '2') { selectTool('highlighter'); }
  else if (e.key === '3') { selectTool('rectangle'); }
  else if (e.key === '4') { selectTool('ellipse'); }
  else if (e.key === '5') { selectTool('arrow'); }
  else if (e.key === '6') { selectTool('text'); }
  else if ((e.ctrlKey || e.metaKey) && e.key === 'z') {
    e.preventDefault();
    strokes.pop();
    redraw();
  }
});

function selectTool(tool) {
  currentTool = tool;
  var shadow = document.getElementById('toolbar-host').shadowRoot;
  shadow.querySelectorAll('[data-tool]').forEach(function(b) {
    b.classList.toggle('active', b.dataset.tool === tool);
  });
}
```

### Lick dedup pattern

```js
var lastLick = {};
function safeLick(payload) {
  var key = payload.action;
  var now = Date.now();
  if (lastLick[key] && now - lastLick[key] < 2000) return;
  lastLick[key] = now;
  slicc.lick(payload);
}
```

Use `safeLick` for `request-screenshot` and `close`. The `capture` action uses its own one-shot guard (disable button after click).

## Communication protocol

### Sprinkle → Scoop (lick events)

| action               | payload                        | meaning                         |
| -------------------- | ------------------------------ | ------------------------------- |
| `request-screenshot` | —                              | Please capture the visible tab  |
| `capture`            | `{ dataUrl: 'data:image/...' }`| Annotated image ready           |
| `close`              | —                              | User dismissed                  |

### Scoop → Sprinkle (sprinkle send)

| payload               | meaning                                    |
| --------------------- | ------------------------------------------ |
| `{ baseImage: '...' }`| Base64 data URL of captured tab screenshot |

## Extension-mode capture (scoop-side)

The owning scoop handles `request-screenshot` by running:

```bash
screencapture --dataurl
```

This returns a base64 data URL. The scoop then pushes it:

```bash
sprinkle send annotate "{\"baseImage\":\"$DATAURL\"}"
```

If `screencapture` is unavailable (CLI fallback), the sprinkle starts with a blank dark canvas — the user can still draw annotations on it.
