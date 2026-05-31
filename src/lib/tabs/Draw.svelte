<script lang="ts">
  import { onMount } from "svelte";
  import {
    IconPencil, IconEraser, IconSquare, IconCircle,
    IconMinus, IconArrowBack, IconArrowForward,
    IconArrowBadgeRight, IconTrash, IconDownload, IconUpload, IconBrush,
    IconSlash, IconTriangle, IconColorPicker, IconBucketDroplet,
    IconLetterT, IconPointer, IconSelector, IconZoomIn, IconZoomOut,
    IconRefresh, IconMaximize, IconPhoto, IconCopy, IconClipboard,
    IconPaint, IconCircleDot, IconCursorText, IconPencil as IconPenTool,
  } from "@tabler/icons-svelte";

  let { apiKey }: { apiKey: string } = $props();

  // ── Canvas dimensions ────────────────────────────────────────────────
  let svgEl: SVGSVGElement;
  let canvasWrap: HTMLDivElement;
  let w = $state(1200);
  let h = $state(800);

  // ── Zoom & Pan ───────────────────────────────────────────────────────
  let zoom = $state(1);
  let panX = $state(0);
  let panY = $state(0);
  let panning = $state(false);
  let panStart = $state({ x: 0, y: 0 });
  let spaceHeld = $state(false);
  const MIN_ZOOM = 0.1;
  const MAX_ZOOM = 20;

  // ── Cursor position ──────────────────────────────────────────────────
  let cursorX = $state(0);
  let cursorY = $state(0);

  // ── Tool state ───────────────────────────────────────────────────────
  type Tool =
    | "move" | "select" | "pen" | "pencil" | "brush" | "eraser"
    | "line" | "rect" | "ellipse" | "arrow" | "triangle"
    | "eyedropper" | "fill" | "text";

  type ToolGroup = "pointer" | "draw" | "shape" | "misc";

  const TOOL_GROUPS: { group: ToolGroup; tools: { id: Tool; label: string; key: string }[] }[] = [
    { group: "pointer", tools: [
      { id: "move", label: "Move (V)", key: "v" },
      { id: "select", label: "Select (M)", key: "m" },
    ]},
    { group: "draw", tools: [
      { id: "brush", label: "Brush (B)", key: "b" },
      { id: "pencil", label: "Pencil (N)", key: "n" },
      { id: "pen", label: "Pen (P)", key: "p" },
      { id: "eraser", label: "Eraser (E)", key: "e" },
    ]},
    { group: "shape", tools: [
      { id: "line", label: "Line (L)", key: "l" },
      { id: "rect", label: "Rectangle (U)", key: "u" },
      { id: "ellipse", label: "Ellipse (O)", key: "o" },
      { id: "arrow", label: "Arrow (A)", key: "a" },
      { id: "triangle", label: "Triangle", key: "" },
    ]},
    { group: "misc", tools: [
      { id: "eyedropper", label: "Eyedropper (I)", key: "i" },
      { id: "fill", label: "Fill Bucket (G)", key: "g" },
      { id: "text", label: "Text (T)", key: "t" },
    ]},
  ];

  let tool = $state<Tool>("brush");
  let color = $state("#ffffff");
  let bgColor = $state("#1a1a1a");
  let lineWidth = $state(4);
  let opacity = $state(100);
  let hardness = $state(100);
  let fill = $state(false);
  let smoothing = $state(0.5);
  let fontSize = $state(32);

  // ── Brush presets ────────────────────────────────────────────────────
  type BrushPreset = { name: string; icon: string; size: number; hardness: number; opacity: number; smoothing: number };
  const BRUSH_PRESETS: BrushPreset[] = [
    { name: "Hard Round", icon: "●", size: 4, hardness: 100, opacity: 100, smoothing: 0.3 },
    { name: "Soft Round", icon: "◎", size: 20, hardness: 30, opacity: 80, smoothing: 0.5 },
    { name: "Calligraphy", icon: "✎", size: 8, hardness: 90, opacity: 100, smoothing: 0.4 },
    { name: "Fine Pen", icon: "🖊", size: 2, hardness: 100, opacity: 100, smoothing: 0.6 },
    { name: "Marker", icon: "▬", size: 16, hardness: 60, opacity: 60, smoothing: 0.5 },
    { name: "Airbrush", icon: "░", size: 30, hardness: 10, opacity: 30, smoothing: 0.7 },
    { name: "Spray", icon: "⣿", size: 24, hardness: 20, opacity: 50, smoothing: 0.3 },
  ];

  function applyPreset(p: BrushPreset) {
    lineWidth = p.size;
    hardness = p.hardness;
    opacity = p.opacity;
    smoothing = p.smoothing;
    tool = "brush";
  }

  // ── Color state ──────────────────────────────────────────────────────
  let fgColor = $state("#ffffff");
  let bgColor2 = $state("#000000");
  let recentColors = $state<string[]>(["#ff0000","#00ff00","#0000ff","#ffff00","#ff00ff","#00ffff","#ff8800","#8800ff"]);
  let colorMode = $state<"fg" | "bg">("fg");

  function swapColors() {
    const tmp = fgColor;
    fgColor = bgColor2;
    bgColor2 = tmp;
    color = fgColor;
  }

  function selectRecent(c: string) {
    fgColor = c;
    color = c;
  }

  // ── Layers (basic) ───────────────────────────────────────────────────
  type Layer = { id: string; name: string; visible: boolean; opacity: number; strokes: Stroke[] };
  let layers = $state<Layer[]>([{ id: "base", name: "Background", visible: true, opacity: 100, strokes: [] }]);
  let activeLayerIdx = $state(0);

  function addLayer() {
    const id = "L" + Date.now().toString(36);
    layers.push({ id, name: `Layer ${layers.length + 1}`, visible: true, opacity: 100, strokes: [] });
    activeLayerIdx = layers.length - 1;
  }

  function removeLayer() {
    if (layers.length <= 1) return;
    layers.splice(activeLayerIdx, 1);
    if (activeLayerIdx >= layers.length) activeLayerIdx = layers.length - 1;
  }

  // ── Strokes ──────────────────────────────────────────────────────────
  type Point = { x: number; y: number };

  type Stroke = {
    id: string;
    tool: Tool;
    color: string;
    width: number;
    opacity: number;
    fill: boolean;
    d?: string;
    points: Point[];
    pencilPaths?: string[];
    shapeType?: string;
    sx?: number; sy?: number; ex?: number; ey?: number;
    text?: string; fontSize?: number;
    layerId: string;
  };

  let undoStack: Stroke[][] = $state([]);
  let redoStack: Stroke[][] = $state([]);
  let drawing = false;
  let currentStroke: Stroke | null = $state(null);
  let currentPoints: Point[] = $state([]);
  let currentPencilPaths: string[] = $state([]);
  let shiftHeld = $state(false);

  // ── SVG coords (account for zoom/pan) ────────────────────────────────
  function svgPoint(e: PointerEvent): Point {
    const rect = canvasWrap.getBoundingClientRect();
    const offsetX = (rect.width - w * zoom) / 2 + panX;
    const offsetY = (rect.height - h * zoom) / 2 + panY;
    return {
      x: (e.clientX - rect.left - offsetX) / zoom,
      y: (e.clientY - rect.top - offsetY) / zoom,
    };
  }

  // ── Smooth curve (catmull-rom) ───────────────────────────────────────
  function smoothPath(pts: Point[]): string {
    if (pts.length === 0) return "";
    if (pts.length === 1) return `M${pts[0].x},${pts[0].y}`;
    if (pts.length === 2) return `M${pts[0].x},${pts[0].y} L${pts[1].x},${pts[1].y}`;

    let d = `M${pts[0].x},${pts[0].y}`;
    for (let i = 1; i < pts.length; i++) {
      const p0 = pts[i - 1];
      const p1 = pts[i];
      const t = smoothing;
      const cpx = p0.x + (p1.x - p0.x) * t;
      const cpy = p0.y + (p1.y - p0.y) * t;
      const cpx2 = p1.x - (p1.x - p0.x) * t;
      const cpy2 = p1.y - (p1.y - p0.y) * t;
      if (i === 1) {
        d += ` C${cpx},${cpy} ${cpx2},${cpy2} ${p1.x},${p1.y}`;
      } else {
        d += ` S${cpx2},${cpy2} ${p1.x},${p1.y}`;
      }
    }
    return d;
  }

  // ── Pencil paths ─────────────────────────────────────────────────────
  function generatePencilPaths(pts: Point[]): string[] {
    const count = Math.max(2, Math.round(hardness / 20));
    const scatter = Math.max(0.3, (100 - hardness) / 25);
    const paths: string[] = [];
    for (let i = 0; i < count; i++) {
      const offsetPts = pts.map(p => ({
        x: p.x + (Math.random() - 0.5) * scatter,
        y: p.y + (Math.random() - 0.5) * scatter,
      }));
      paths.push(smoothPath(offsetPts));
    }
    return paths;
  }

  // ── Spray dots ───────────────────────────────────────────────────────
  function generateSprayDots(pts: Point[]): Point[] {
    const dots: Point[] = [];
    const radius = lineWidth / 2;
    for (const p of pts) {
      const count = Math.round(lineWidth * 0.8);
      for (let i = 0; i < count; i++) {
        const angle = Math.random() * Math.PI * 2;
        const r = Math.random() * radius;
        dots.push({ x: p.x + Math.cos(angle) * r, y: p.y + Math.sin(angle) * r });
      }
    }
    return dots;
  }

  // ── Brush width from velocity ────────────────────────────────────────
  let lastTime = 0;
  let lastPt: Point | null = null;

  function brushWidth(pt: Point): number {
    const now = performance.now();
    let vel = 0;
    if (lastPt && lastTime) {
      const dx = pt.x - lastPt.x;
      const dy = pt.y - lastPt.y;
      const dist = Math.sqrt(dx * dx + dy * dy);
      const dt = Math.max(1, now - lastTime);
      vel = dist / dt;
    }
    lastPt = pt;
    lastTime = now;
    const minW = lineWidth * 0.2;
    const maxW = lineWidth * 2.5;
    const factor = Math.min(1, vel / 4);
    return maxW - (maxW - minW) * factor;
  }

  // ── Pointer handlers ─────────────────────────────────────────────────
  function getActiveLayer(): Layer {
    return layers[activeLayerIdx] ?? layers[0];
  }

  function pointerDown(e: PointerEvent) {
    if (e.button !== 0) return;

    if (spaceHeld) {
      panning = true;
      panStart = { x: e.clientX - panX, y: e.clientY - panY };
      return;
    }

    if (tool === "move") {
      panning = true;
      panStart = { x: e.clientX - panX, y: e.clientY - panY };
      return;
    }

    if (tool === "eyedropper") {
      pickColor(e);
      return;
    }

    if (tool === "text") {
      const pt = svgPoint(e);
      const text = prompt("Enter text:");
      if (!text) return;
      const id = Date.now().toString(36) + Math.random().toString(36).slice(2, 5);
      const stroke: Stroke = {
        id, tool, color: fgColor, width: lineWidth, opacity: opacity / 100,
        fill: false, points: [pt], text, fontSize,
        layerId: getActiveLayer().id,
      };
      pushStroke(stroke);
      return;
    }

    drawing = true;
    const pt = svgPoint(e);
    currentPoints = [pt];
    lastPt = pt;
    lastTime = performance.now();

    const isShape = ["line", "rect", "ellipse", "arrow", "triangle"].includes(tool);
    const id = Date.now().toString(36) + Math.random().toString(36).slice(2, 5);

    if (isShape) {
      currentStroke = {
        id, tool, color: fgColor, width: lineWidth, opacity: opacity / 100, fill,
        shapeType: tool, points: [pt],
        sx: pt.x, sy: pt.y, ex: pt.x, ey: pt.y,
        layerId: getActiveLayer().id,
      };
    } else {
      const activeColor = tool === "eraser" ? bgColor : fgColor;
      currentStroke = {
        id, tool, color: activeColor, width: lineWidth, opacity: opacity / 100,
        fill: false, points: [pt], d: `M${pt.x},${pt.y}`,
        layerId: getActiveLayer().id,
      };
    }

    svgEl.setPointerCapture(e.pointerId);
  }

  function pointerMove(e: PointerEvent) {
    const pt = svgPoint(e);
    cursorX = Math.round(pt.x);
    cursorY = Math.round(pt.y);

    if (panning) {
      panX = e.clientX - panStart.x;
      panY = e.clientY - panStart.y;
      return;
    }

    if (!drawing || !currentStroke) return;

    currentPoints.push(pt);

    if (["line", "rect", "ellipse", "arrow", "triangle"].includes(tool)) {
      if (shiftHeld) {
        const dx = pt.x - currentStroke.sx!;
        const dy = pt.y - currentStroke.sy!;
        const maxD = Math.max(Math.abs(dx), Math.abs(dy));
        currentStroke.ex = currentStroke.sx! + Math.sign(dx) * maxD;
        currentStroke.ey = currentStroke.sy! + Math.sign(dy) * maxD;
      } else {
        currentStroke.ex = pt.x;
        currentStroke.ey = pt.y;
      }
    } else if (tool === "pencil") {
      currentStroke.points = [...currentPoints];
      const paths = generatePencilPaths(currentPoints);
      currentPencilPaths = paths;
      currentStroke.d = paths[0];
    } else if (tool === "brush") {
      const bw = brushWidth(pt);
      currentStroke.width = bw;
      currentStroke.points = [...currentPoints];
      currentStroke.d = smoothPath(currentPoints);
    } else if (tool === "eraser") {
      currentStroke.points = [...currentPoints];
      currentStroke.d = smoothPath(currentPoints);
    } else {
      currentStroke.points = [...currentPoints];
      currentStroke.d = smoothPath(currentPoints);
    }
  }

  function pointerUp(_e: PointerEvent) {
    if (panning) { panning = false; return; }
    if (!drawing || !currentStroke) return;
    drawing = false;

    if (currentPoints.length < 2 && !currentStroke.shapeType && !currentStroke.text) return;

    if (currentStroke.tool === "pencil") {
      currentStroke.pencilPaths = generatePencilPaths(currentPoints);
    }

    pushStroke(currentStroke);
    currentStroke = null;
    currentPoints = [];
    currentPencilPaths = [];
    lastPt = null;
  }

  function pushStroke(s: Stroke) {
    const layer = layers.find(l => l.id === s.layerId);
    if (layer) {
      layer.strokes = [...layer.strokes, s];
      layers = [...layers];
    }
    undoStack.push([s]);
    if (undoStack.length > 100) undoStack.shift();
    redoStack = [];
  }

  // ── Undo / Redo ──────────────────────────────────────────────────────
  function undo() {
    if (undoStack.length === 0) return;
    const last = undoStack.pop()!;
    for (const s of last) {
      const layer = layers.find(l => l.id === s.layerId);
      if (layer) {
        layer.strokes = layer.strokes.filter(st => st.id !== s.id);
      }
    }
    redoStack.push(last);
    layers = [...layers];
  }

  function redo() {
    if (redoStack.length === 0) return;
    const last = redoStack.pop()!;
    for (const s of last) {
      const layer = layers.find(l => l.id === s.layerId);
      if (layer) {
        layer.strokes = [...layer.strokes, s];
      }
    }
    undoStack.push(last);
    layers = [...layers];
  }

  function clearAll() {
    for (const layer of layers) layer.strokes = [];
    layers = [...layers];
    undoStack = [];
    redoStack = [];
  }

  // ── Eyedropper ───────────────────────────────────────────────────────
  function pickColor(e: PointerEvent) {
    if (!svgEl) return;
    const rect = svgEl.getBoundingClientRect();
    const canvas = document.createElement("canvas");
    canvas.width = w;
    canvas.height = h;
    const ctx = canvas.getContext("2d")!;
    const svgData = renderSvgString(true);
    const img = new Image();
    img.onload = () => {
      ctx.drawImage(img, 0, 0);
      const px = Math.round((e.clientX - rect.left) / rect.width * w);
      const py = Math.round((e.clientY - rect.top) / rect.height * h);
      const data = ctx.getImageData(px, py, 1, 1).data;
      fgColor = `#${[data[0], data[1], data[2]].map(v => v.toString(16).padStart(2, "0")).join("")}`;
      color = fgColor;
      tool = "brush";
    };
    img.src = "data:image/svg+xml;base64," + btoa(unescape(encodeURIComponent(svgData)));
  }

  // ── Shape rendering ──────────────────────────────────────────────────
  function shapeAttrs(s: Stroke) {
    if (s.sx == null || s.sy == null || s.ex == null || s.ey == null) return {};
    const x = Math.min(s.sx, s.ex);
    const y = Math.min(s.sy, s.ey);
    const sw = Math.abs(s.ex - s.sx);
    const sh = Math.abs(s.ey - s.sy);

    switch (s.shapeType) {
      case "line":
        return { x1: s.sx, y1: s.sy, x2: s.ex, y2: s.ey };
      case "rect":
        return { x, y, width: sw, height: sh };
      case "ellipse":
        return { cx: (s.sx + s.ex) / 2, cy: (s.sy + s.ey) / 2, rx: sw / 2, ry: sh / 2 };
      case "arrow": {
        const angle = Math.atan2(s.ey - s.sy, s.ex - s.sx);
        const headLen = Math.min(20, Math.sqrt(sw * sw + sh * sh) * 0.3);
        const a = Math.PI / 6;
        return {
          x1: s.sx, y1: s.sy, x2: s.ex, y2: s.ey,
          headX1: s.ex - headLen * Math.cos(angle - a),
          headY1: s.ey - headLen * Math.sin(angle - a),
          headX2: s.ex - headLen * Math.cos(angle + a),
          headY2: s.ey - headLen * Math.sin(angle + a),
        };
      }
      case "triangle": {
        const topX = (s.sx + s.ex) / 2;
        const topY = Math.min(s.sy, s.ey);
        const botY = Math.max(s.sy, s.ey);
        return { points: `${topX},${topY} ${Math.min(s.sx, s.ex)},${botY} ${Math.max(s.sx, s.ex)},${botY}` };
      }
      default:
        return {};
    }
  }

  // ── Save to cloud ────────────────────────────────────────────────────
  let saving = $state(false);
  let saveError = $state<string | null>(null);
  let saveName = $state("drawing.png");
  let saveOk = $state(false);

  async function saveToCloud() {
    if (!svgEl || saving) return;
    saving = true; saveError = null; saveOk = false;
    try {
      const svgData = renderSvgString(true);
      const canvas = document.createElement("canvas");
      canvas.width = w; canvas.height = h;
      const ctx = canvas.getContext("2d")!;
      const img = new Image();
      const blob = await new Promise<Blob>((res, rej) => {
        img.onload = async () => {
          ctx.drawImage(img, 0, 0);
          canvas.toBlob(b => b ? res(b) : rej(new Error("toBlob failed")), "image/png");
        };
        img.onerror = rej;
        img.src = "data:image/svg+xml;base64," + btoa(unescape(encodeURIComponent(svgData)));
      });
      const fd = new FormData();
      fd.append("file", blob, saveName);
      const up = await fetch("/api/telegram/uploadFile", {
        method: "POST", body: fd,
        headers: { "X-Api-Key": apiKey },
      });
      if (!up.ok) throw new Error(`Upload failed: ${up.status}`);
      saveOk = true;
      setTimeout(() => saveOk = false, 3000);
    } catch (err: any) {
      saveError = err?.message ?? "Failed to save";
    } finally {
      saving = false;
    }
  }

  // ── Render SVG string ────────────────────────────────────────────────
  function renderSvgString(withBg = false): string {
    let inner = "";
    for (const layer of layers) {
      if (!layer.visible) continue;
      const layerOpacity = layer.opacity / 100;
      inner += `<g opacity="${layerOpacity}">`;
      for (const s of layer.strokes) {
        inner += renderStroke(s, false);
      }
      inner += "</g>";
    }

    if (currentStroke) {
      inner += renderStroke(currentStroke, true);
    }

    const bg = withBg ? `<rect width="100%" height="100%" fill="${bgColor}"/>` : "";
    return `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 ${w} ${h}" width="${w}" height="${h}">${bg}${inner}</svg>`;
  }

  function renderStroke(s: Stroke, isPreview: boolean): string {
    const op = isPreview ? 0.5 : s.opacity;
    const sw = isPreview ? s.width / 2 : s.width;

    if (s.shapeType) {
      const attrs = shapeAttrs(s);
      switch (s.shapeType) {
        case "line":
          return `<line x1="${attrs.x1}" y1="${attrs.y1}" x2="${attrs.x2}" y2="${attrs.y2}" stroke="${s.color}" stroke-width="${sw}" stroke-linecap="round" opacity="${op}"/>`;
        case "rect":
          return `<rect x="${attrs.x}" y="${attrs.y}" width="${attrs.width}" height="${attrs.height}" fill="${s.fill ? s.color : 'none'}" stroke="${s.fill ? 'none' : s.color}" stroke-width="${sw}" opacity="${op}"/>`;
        case "ellipse":
          return `<ellipse cx="${attrs.cx}" cy="${attrs.cy}" rx="${attrs.rx}" ry="${attrs.ry}" fill="${s.fill ? s.color : 'none'}" stroke="${s.fill ? 'none' : s.color}" stroke-width="${sw}" opacity="${op}"/>`;
        case "arrow":
          return `<line x1="${attrs.x1}" y1="${attrs.y1}" x2="${attrs.x2}" y2="${attrs.y2}" stroke="${s.color}" stroke-width="${sw}" stroke-linecap="round" opacity="${op}"/><polygon points="${attrs.x2},${attrs.y2} ${attrs.headX1},${attrs.headY1} ${attrs.headX2},${attrs.headY2}" fill="${s.color}" opacity="${op}"/>`;
        case "triangle":
          return `<polygon points="${attrs.points}" fill="${s.fill ? s.color : 'none'}" stroke="${s.fill ? 'none' : s.color}" stroke-width="${sw}" stroke-linejoin="round" opacity="${op}"/>`;
      }
    }

    if (s.text) {
      return `<text x="${s.points[0]?.x}" y="${s.points[0]?.y}" fill="${s.color}" font-size="${s.fontSize ?? 32}" font-family="sans-serif" opacity="${op}">${escapeXml(s.text)}</text>`;
    }

    if (s.tool === "pencil") {
      let r = "";
      for (const pd of s.pencilPaths ?? []) {
        r += `<path d="${pd}" fill="none" stroke="${s.color}" stroke-width="${s.width * 0.35}" stroke-linecap="round" stroke-linejoin="round" opacity="${op * 0.5}"/>`;
      }
      r += `<path d="${s.d}" fill="none" stroke="${s.color}" stroke-width="${s.width * 0.3}" stroke-linecap="round" stroke-linejoin="round" opacity="${op}"/>`;
      return r;
    }

    return `<path d="${s.d}" fill="none" stroke="${s.color}" stroke-width="${sw}" stroke-linecap="round" stroke-linejoin="round" opacity="${op}"/>`;
  }

  function escapeXml(s: string): string {
    return s.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;");
  }

  function attr(obj: any, ...keys: string[]): string {
    return keys.map(k => `${k}="${obj[k] ?? ""}"`).join(" ");
  }

  // ── Download ─────────────────────────────────────────────────────────
  function downloadPng() {
    const svgData = renderSvgString(true);
    const canvas = document.createElement("canvas");
    canvas.width = w; canvas.height = h;
    const ctx = canvas.getContext("2d")!;
    const img = new Image();
    img.onload = () => {
      ctx.drawImage(img, 0, 0);
      const a = document.createElement("a");
      a.download = saveName;
      a.href = canvas.toDataURL("image/png");
      a.click();
    };
    img.src = "data:image/svg+xml;base64," + btoa(unescape(encodeURIComponent(svgData)));
  }

  function downloadSvg() {
    const svgData = renderSvgString(true);
    const a = document.createElement("a");
    a.download = saveName.replace(/\.png$/i, ".svg");
    a.href = "data:image/svg+xml;base64," + btoa(unescape(encodeURIComponent(svgData)));
    a.click();
  }

  // ── Zoom helpers ─────────────────────────────────────────────────────
  function zoomAt(delta: number, cx?: number, cy?: number) {
    const oldZoom = zoom;
    zoom = Math.min(MAX_ZOOM, Math.max(MIN_ZOOM, zoom * (1 + delta)));
    if (cx !== undefined && cy !== undefined) {
      panX = cx - (cx - panX) * (zoom / oldZoom);
      panY = cy - (cy - panY) * (zoom / oldZoom);
    }
  }

  function zoomTo(z: number) {
    zoomAt(z / zoom - 1, wrapW / 2, wrapH / 2);
  }

  function fitToScreen() {
    if (!wrapW || !wrapH) return;
    const pad = 40;
    zoom = Math.min((wrapW - pad) / w, (wrapH - pad) / h);
    panX = 0; panY = 0;
  }

  function onWheel(e: WheelEvent) {
    e.preventDefault();
    if (e.ctrlKey || e.metaKey) {
      const rect = canvasWrap?.getBoundingClientRect();
      if (!rect) return;
      const cx = e.clientX - rect.left;
      const cy = e.clientY - rect.top;
      zoomAt(e.deltaY > 0 ? -0.1 : 0.1, cx, cy);
    } else {
      panX -= e.deltaX;
      panY -= e.deltaY;
    }
  }

  // ── Keyboard shortcuts ───────────────────────────────────────────────
  function onkeydown(e: KeyboardEvent) {
    if (e.target instanceof HTMLInputElement || e.target instanceof HTMLTextAreaElement) return;

    if (e.key === " ") { e.preventDefault(); spaceHeld = true; return; }
    if (e.ctrlKey || e.metaKey) {
      if (e.key === "z" && !e.shiftKey) { e.preventDefault(); undo(); return; }
      if (e.key === "z" && e.shiftKey) { e.preventDefault(); redo(); return; }
      if (e.key === "y") { e.preventDefault(); redo(); return; }
      if (e.key === "0") { e.preventDefault(); fitToScreen(); return; }
      if (e.key === "=" || e.key === "+") { e.preventDefault(); zoomTo(Math.min(MAX_ZOOM, zoom * 1.5)); return; }
      if (e.key === "-") { e.preventDefault(); zoomTo(Math.max(MIN_ZOOM, zoom / 1.5)); return; }
    }

    if (e.shiftKey) { shiftHeld = true; return; }

    const key = e.key.toLowerCase();
    for (const group of TOOL_GROUPS) {
      for (const t of group.tools) {
        if (t.key === key) { tool = t.id as Tool; return; }
      }
    }
    if (key === "[") { lineWidth = Math.max(0.5, lineWidth - 1); return; }
    if (key === "]") { lineWidth = Math.min(100, lineWidth + 1); return; }
  }

  function onkeyup(e: KeyboardEvent) {
    if (e.key === " ") spaceHeld = false;
    if (e.key === "Shift") shiftHeld = false;
  }

  // ── Resize ───────────────────────────────────────────────────────────
  let wrapW = $state(0);
  let wrapH = $state(0);

  let svgOffsetX = $derived((wrapW - w * zoom) / 2 + panX);
  let svgOffsetY = $derived((wrapH - h * zoom) / 2 + panY);

  onMount(() => {
    const ro = new ResizeObserver(() => {
      if (!canvasWrap) return;
      const rect = canvasWrap.getBoundingClientRect();
      wrapW = rect.width;
      wrapH = rect.height;
    });
    ro.observe(canvasWrap);
    const rect = canvasWrap?.getBoundingClientRect();
    if (rect) { wrapW = rect.width; wrapH = rect.height; }
    fitToScreen();
    return () => ro.disconnect();
  });

  $effect(() => {
    color = fgColor;
  });

  let rightPanel = $state<"brush" | "color" | "layers">("brush");
  let showBrushPanel = $state(false);
</script>

<svelte:window onkeydown={onkeydown} onkeyup={onkeyup}/>

<div class="draw-root" role="application" tabindex="-1">
  <!-- ═══ LEFT TOOLBAR ═══ -->
  <div class="tool-sidebar">
    {#each TOOL_GROUPS as group, gi}
      {#if gi > 0}<div class="ts-sep"></div>{/if}
      {#each group.tools as t}
        <button
          class="ts-btn"
          class:active={tool === t.id}
          onclick={() => tool = t.id as Tool}
          title={t.label}
        >
          {#if t.id === "move"}<IconPointer size={16}/>{:else if t.id === "select"}<IconSelector size={16}/>
          {:else if t.id === "brush"}<IconBrush size={16}/>{:else if t.id === "pencil"}<IconPencil size={16}/>
          {:else if t.id === "pen"}<IconPenTool size={16}/>{:else if t.id === "eraser"}<IconEraser size={16}/>
          {:else if t.id === "line"}<IconMinus size={16}/>{:else if t.id === "rect"}<IconSquare size={16}/>
          {:else if t.id === "ellipse"}<IconCircle size={16}/>{:else if t.id === "arrow"}<IconArrowBadgeRight size={16}/>
          {:else if t.id === "triangle"}<IconTriangle size={16}/>{:else if t.id === "eyedropper"}<IconColorPicker size={16}/>
          {:else if t.id === "fill"}<IconBucketDroplet size={16}/>{:else if t.id === "text"}<IconLetterT size={16}/>
          {/if}
        </button>
      {/each}
    {/each}

    <div class="ts-sep"></div>

    <!-- Undo / Redo -->
    <button class="ts-btn" onclick={undo} disabled={undoStack.length === 0} title="Undo (Ctrl+Z)"><IconArrowBack size={16}/></button>
    <button class="ts-btn" onclick={redo} disabled={redoStack.length === 0} title="Redo (Ctrl+Shift+Z)"><IconArrowForward size={16}/></button>
    <button class="ts-btn" onclick={clearAll} title="Clear all"><IconTrash size={16}/></button>

    <div class="ts-spacer"></div>

    <!-- Color swatches -->
    <button class="ts-swatch ts-fg" style="background:{fgColor}" onclick={() => colorMode = 'fg'} title="Foreground color">
      <input type="color" bind:value={fgColor} class="ts-color-hid"/>
    </button>
    <button class="ts-swatch ts-bg" style="background:{bgColor2}" onclick={() => colorMode = 'bg'} title="Background color">
      <input type="color" bind:value={bgColor2} class="ts-color-hid"/>
    </button>
    <button class="ts-swap" onclick={swapColors} title="Swap colors (X)">
      <svg width="10" height="10" viewBox="0 0 10 10"><path d="M1 3L4 0L4 2L9 2L9 4L4 4L4 6Z" fill="currentColor" transform="rotate(180 5 3)"/></svg>
    </button>
  </div>

  <!-- ═══ CENTER AREA ═══ -->
  <div class="center-area">
    <!-- Top options bar -->
    <div class="options-bar">
      <span class="ob-tool">{tool}</span>
      <div class="ob-sep"></div>
      <label class="ob-label">Size
        <input type="range" min="0.5" max="100" step="0.5" bind:value={lineWidth} class="ob-slider"/>
        <span class="ob-val">{lineWidth.toFixed(1)}</span>
      </label>
      <label class="ob-label">Opacity
        <input type="range" min="1" max="100" bind:value={opacity} class="ob-slider"/>
        <span class="ob-val">{opacity}%</span>
      </label>
      {#if tool === "pencil" || tool === "brush"}
        <label class="ob-label">Hardness
          <input type="range" min="1" max="100" bind:value={hardness} class="ob-slider"/>
          <span class="ob-val">{hardness}%</span>
        </label>
      {/if}
      {#if tool === "brush"}
        <label class="ob-label">Smoothing
          <input type="range" min="0" max="1" step="0.05" bind:value={smoothing} class="ob-slider"/>
          <span class="ob-val">{Math.round(smoothing * 100)}%</span>
        </label>
      {/if}
      {#if tool === "text"}
        <label class="ob-label">Font size
          <input type="range" min="8" max="200" bind:value={fontSize} class="ob-slider"/>
          <span class="ob-val">{fontSize}px</span>
        </label>
      {/if}
      <label class="ob-check">
        <input type="checkbox" bind:checked={fill}/>
        Fill
      </label>
      <div class="ob-spacer"></div>

      <!-- Brush presets quick bar -->
      {#if tool === "brush" || tool === "pencil" || tool === "eraser"}
        <div class="ob-presets">
          {#each BRUSH_PRESETS as p}
            <button class="ob-preset" onclick={() => applyPreset(p)} title={p.name}>
              <span style="font-size:{Math.min(16, p.size + 4)}px">{p.icon}</span>
            </button>
          {/each}
        </div>
      {/if}
    </div>

    <!-- Canvas wrapper -->
    <div
      class="canvas-wrap"
      bind:this={canvasWrap}
      onwheel={onWheel}
      class:panning={panning || spaceHeld}
    >
      <svg
        bind:this={svgEl}
        viewBox="0 0 {w} {h}"
        width={w * zoom}
        height={h * zoom}
        class="draw-svg"
        style="touch-action:none; transform: translate({svgOffsetX}px, {svgOffsetY}px)"
        onpointerdown={pointerDown}
        onpointermove={pointerMove}
        onpointerup={pointerUp}
        onpointerleave={pointerUp}
      >
        <!-- Checkerboard transparency -->
        <defs>
          <pattern id="checker" width="16" height="16" patternUnits="userSpaceOnUse">
            <rect width="16" height="16" fill="#2a2a2a"/>
            <rect width="8" height="8" fill="#222"/>
            <rect x="8" y="8" width="8" height="8" fill="#222"/>
          </pattern>
        </defs>
        <rect width="100%" height="100%" fill={bgColor}/>
        <rect width="100%" height="100%" fill="url(#checker)" opacity="0"/>

        {#each layers as layer (layer.id)}
          {#if layer.visible}
            <g opacity={layer.opacity / 100}>
              {#each layer.strokes as s (s.id)}
                {@html renderStroke(s, false)}
              {/each}
            </g>
          {/if}
        {/each}

        {#if currentStroke}
          {@html renderStroke(currentStroke, true)}
        {/if}
      </svg>
    </div>

    <!-- Status bar -->
    <div class="status-bar">
      <span>{w} × {h}px</span>
      <span class="sb-sep">|</span>
      <span>{cursorX}, {cursorY}</span>
      <span class="sb-sep">|</span>
      <span>{Math.round(zoom * 100)}%</span>
      <div class="sb-sep"></div>
      <button class="sb-btn" onclick={() => zoomTo(1)} title="100%">1:1</button>
      <button class="sb-btn" onclick={fitToScreen} title="Fit to screen">Fit</button>
      <button class="sb-btn" onclick={() => zoomTo(Math.min(MAX_ZOOM, zoom * 2))}>+</button>
      <button class="sb-btn" onclick={() => zoomTo(Math.max(MIN_ZOOM, zoom / 2))}>-</button>
      <div class="sb-spacer"></div>
      <span class="sb-stroke-count">{layers.reduce((a, l) => a + l.strokes.length, 0)} strokes</span>
      <span class="sb-sep">|</span>
      <input class="sb-fname" bind:value={saveName} placeholder="drawing.png"/>
      <button class="sb-btn accent" onclick={downloadPng}><IconDownload size={12}/> PNG</button>
      <button class="sb-btn accent" onclick={downloadSvg}><IconDownload size={12}/> SVG</button>
      <button class="sb-btn primary" onclick={saveToCloud} disabled={saving}>
        {#if saving}Saving…{:else}<IconUpload size={12}/> Save{/if}
      </button>
      {#if saveOk}<span class="sb-ok">✓</span>{/if}
      {#if saveError}<span class="sb-err">{saveError}</span>{/if}
    </div>
  </div>

  <!-- ═══ RIGHT PANEL ═══ -->
  <div class="right-panel">
    <div class="rp-tabs">
      <button class="rp-tab" class:active={rightPanel === "brush"} onclick={() => rightPanel = "brush"}>Brush</button>
      <button class="rp-tab" class:active={rightPanel === "color"} onclick={() => rightPanel = "color"}>Color</button>
      <button class="rp-tab" class:active={rightPanel === "layers"} onclick={() => rightPanel = "layers"}>Layers</button>
    </div>

    {#if rightPanel === "brush"}
      <div class="rp-content">
        <p class="rp-heading">Brush Presets</p>
        <div class="rp-presets">
          {#each BRUSH_PRESETS as p}
            <button class="rp-preset" class:active={lineWidth === p.size && hardness === p.hardness} onclick={() => applyPreset(p)}>
              <span class="rp-preset-icon">{p.icon}</span>
              <span class="rp-preset-name">{p.name}</span>
              <span class="rp-preset-size">{p.size}px</span>
            </button>
          {/each}
        </div>

        <p class="rp-heading">Settings</p>
        <label class="rp-slider">Size
          <input type="range" min="0.5" max="100" step="0.5" bind:value={lineWidth}/>
          <span>{lineWidth.toFixed(1)}</span>
        </label>
        <label class="rp-slider">Opacity
          <input type="range" min="1" max="100" bind:value={opacity}/>
          <span>{opacity}%</span>
        </label>
        <label class="rp-slider">Hardness
          <input type="range" min="1" max="100" bind:value={hardness}/>
          <span>{hardness}%</span>
        </label>
        <label class="rp-slider">Smoothing
          <input type="range" min="0" max="1" step="0.05" bind:value={smoothing}/>
          <span>{Math.round(smoothing * 100)}%</span>
        </label>
      </div>
    {:else if rightPanel === "color"}
      <div class="rp-content">
        <p class="rp-heading">Color</p>
        <div class="rp-color-big">
          <div class="rp-fg" style="background:{fgColor}"></div>
          <div class="rp-bg" style="background:{bgColor2}"></div>
        </div>
        <input type="color" bind:value={fgColor} class="rp-color-input"/>
        <label class="rp-label">Hex
          <input type="text" bind:value={fgColor} class="rp-hex" maxlength="7"/>
        </label>
        <p class="rp-heading">Opacity</p>
        <label class="rp-slider">A
          <input type="range" min="1" max="100" bind:value={opacity}/>
          <span>{opacity}%</span>
        </label>

        <p class="rp-heading">Recent</p>
        <div class="rp-recent">
          {#each recentColors as c}
            <button class="rp-recent-swatch" style="background:{c}" onclick={() => selectRecent(c)}></button>
          {/each}
        </div>
      </div>
    {:else if rightPanel === "layers"}
      <div class="rp-content">
        <div class="rp-layer-actions">
          <button class="rp-layer-btn" onclick={addLayer} title="Add layer">+ New</button>
          <button class="rp-layer-btn" onclick={removeLayer} disabled={layers.length <= 1}>- Delete</button>
        </div>
        {#each [...layers].reverse() as layer, ri (layer.id)}
          {@const idx = layers.length - 1 - ri}
          <div
            class="rp-layer"
            class:active={idx === activeLayerIdx}
            onclick={() => activeLayerIdx = idx}
            role="button"
            tabindex="-1"
          >
            <button class="rp-layer-vis" onclick={(e) => { e.stopPropagation(); layer.visible = !layer.visible; layers = [...layers]; }}>
              {layer.visible ? "👁" : "—"}
            </button>
            <span class="rp-layer-name">{layer.name}</span>
            <span class="rp-layer-count">{layer.strokes.length}</span>
          </div>
        {/each}
      </div>
    {/if}
  </div>
</div>

<style>
  /* ═══ LAYOUT ═══ */
  .draw-root {
    display: flex;
    height: 100%;
    width: 100%;
    overflow: hidden;
    background: #1a1a1e;
    user-select: none;
  }

  /* ═══ LEFT TOOLBAR ═══ */
  .tool-sidebar {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 42px;
    background: #222226;
    border-right: 1px solid #333;
    padding: 6px 0;
    gap: 2px;
    flex-shrink: 0;
    overflow-y: auto;
  }
  .ts-sep {
    width: 24px; height: 1px;
    background: #444;
    margin: 4px 0;
  }
  .ts-btn {
    display: flex; align-items: center; justify-content: center;
    width: 32px; height: 32px;
    border-radius: 6px;
    background: none; border: none;
    color: #888; cursor: pointer;
    transition: .12s;
  }
  .ts-btn:hover { background: #333; color: #ddd; }
  .ts-btn.active { background: #6366f1; color: #fff; }
  .ts-btn:disabled { opacity: .25; cursor: default; }
  .ts-spacer { flex: 1; }
  .ts-swatch {
    width: 24px; height: 24px;
    border-radius: 4px;
    border: 2px solid #555;
    cursor: pointer;
    position: relative;
    overflow: hidden;
  }
  .ts-fg { margin-top: 4px; z-index: 2; }
  .ts-bg {
    width: 18px; height: 18px;
    margin-top: -8px; margin-left: 12px;
    z-index: 1;
  }
  .ts-color-hid {
    position: absolute; inset: 0;
    opacity: 0; width: 100%; height: 100%;
    cursor: pointer;
  }
  .ts-swap {
    background: none; border: none;
    color: #666; cursor: pointer;
    padding: 4px; margin-top: 2px;
  }
  .ts-swap:hover { color: #aaa; }

  /* ═══ CENTER ═══ */
  .center-area {
    flex: 1; display: flex; flex-direction: column;
    min-width: 0; overflow: hidden;
  }

  /* Options bar */
  .options-bar {
    display: flex; align-items: center; gap: 10px;
    padding: 4px 12px;
    background: #26262a;
    border-bottom: 1px solid #333;
    min-height: 32px;
    flex-shrink: 0;
    overflow-x: auto;
  }
  .ob-tool {
    font-size: 11px; font-weight: 600;
    color: #aaa; text-transform: uppercase;
    letter-spacing: .05em;
    min-width: 60px;
  }
  .ob-sep { width: 1px; height: 16px; background: #444; flex-shrink: 0; }
  .ob-label {
    display: flex; align-items: center; gap: 4px;
    font-size: 11px; color: #777;
    white-space: nowrap;
  }
  .ob-slider { width: 60px; height: 3px; accent-color: #6366f1; }
  .ob-val { font-family: 'Geist Mono', monospace; font-size: 10px; color: #555; min-width: 28px; }
  .ob-check {
    display: flex; align-items: center; gap: 4px;
    font-size: 11px; color: #777; cursor: pointer;
  }
  .ob-check input { accent-color: #6366f1; }
  .ob-spacer { flex: 1; }
  .ob-presets { display: flex; gap: 3px; }
  .ob-preset {
    width: 24px; height: 24px;
    border-radius: 4px; border: 1px solid #444;
    background: #2a2a2e; color: #aaa;
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; transition: .12s;
  }
  .ob-preset:hover { border-color: #6366f1; color: #fff; }

  /* Canvas */
  .canvas-wrap {
    flex: 1; overflow: hidden;
    position: relative;
    background: #1a1a1e;
    background-image:
      linear-gradient(45deg, #222 25%, transparent 25%),
      linear-gradient(-45deg, #222 25%, transparent 25%),
      linear-gradient(45deg, transparent 75%, #222 75%),
      linear-gradient(-45deg, transparent 75%, #222 75%);
    background-size: 20px 20px;
    background-position: 0 0, 0 10px, 10px -10px, -10px 0;
  }
  .canvas-wrap.panning { cursor: grab !important; }
  .draw-svg {
    display: block;
    position: absolute;
    box-shadow: 0 0 0 1px rgba(255,255,255,.05), 0 4px 24px rgba(0,0,0,.4);
    cursor: crosshair;
  }

  /* Status bar */
  .status-bar {
    display: flex; align-items: center; gap: 6px;
    padding: 3px 12px;
    background: #26262a;
    border-top: 1px solid #333;
    font-size: 11px; color: #666;
    font-family: 'Geist Mono', monospace;
    min-height: 26px;
    flex-shrink: 0;
  }
  .sb-sep { width: 1px; height: 12px; background: #444; }
  .sb-spacer { flex: 1; }
  .sb-btn {
    display: flex; align-items: center; gap: 3px;
    padding: 2px 6px; border-radius: 4px;
    background: #2a2a2e; border: 1px solid #444;
    color: #888; font-size: 10px; cursor: pointer;
    font-family: 'Geist Mono', monospace;
    transition: .12s;
  }
  .sb-btn:hover { border-color: #6366f1; color: #ccc; }
  .sb-btn.accent { color: #aaa; }
  .sb-btn.primary {
    background: #6366f1; border-color: #6366f1;
    color: #fff; font-weight: 600;
  }
  .sb-btn.primary:hover { opacity: .85; }
  .sb-btn.primary:disabled { opacity: .5; cursor: not-allowed; }
  .sb-fname {
    background: #1a1a1e; border: 1px solid #444;
    border-radius: 4px; padding: 2px 6px;
    color: #aaa; font-size: 10px;
    font-family: 'Geist Mono', monospace;
    width: 100px; outline: none;
  }
  .sb-fname:focus { border-color: #6366f1; }
  .sb-stroke-count { color: #555; }
  .sb-ok { color: #4ade80; }
  .sb-err { color: #f87171; }

  /* ═══ RIGHT PANEL ═══ */
  .right-panel {
    width: 220px; flex-shrink: 0;
    background: #222226;
    border-left: 1px solid #333;
    display: flex; flex-direction: column;
    overflow: hidden;
  }
  .rp-tabs {
    display: flex;
    border-bottom: 1px solid #333;
  }
  .rp-tab {
    flex: 1; padding: 6px 0;
    background: none; border: none;
    color: #666; font-size: 11px;
    font-weight: 600; cursor: pointer;
    border-bottom: 2px solid transparent;
    transition: .12s;
  }
  .rp-tab:hover { color: #aaa; }
  .rp-tab.active { color: #fff; border-bottom-color: #6366f1; }

  .rp-content {
    flex: 1; overflow-y: auto;
    padding: 10px;
    display: flex; flex-direction: column; gap: 6px;
  }
  .rp-heading {
    font-size: 10px; font-weight: 700;
    color: #555; text-transform: uppercase;
    letter-spacing: .08em;
    margin: 8px 0 4px;
  }
  .rp-heading:first-child { margin-top: 0; }

  /* Presets */
  .rp-presets { display: flex; flex-direction: column; gap: 3px; }
  .rp-preset {
    display: flex; align-items: center; gap: 8px;
    padding: 6px 8px; border-radius: 6px;
    background: #2a2a2e; border: 1px solid #333;
    color: #aaa; cursor: pointer;
    font-size: 11px; transition: .12s;
  }
  .rp-preset:hover { border-color: #555; background: #333; }
  .rp-preset.active { border-color: #6366f1; background: #2a2a3e; color: #fff; }
  .rp-preset-icon { font-size: 14px; width: 20px; text-align: center; }
  .rp-preset-name { flex: 1; }
  .rp-preset-size { font-size: 10px; color: #555; font-family: 'Geist Mono', monospace; }

  /* Sliders */
  .rp-slider {
    display: flex; align-items: center; gap: 6px;
    font-size: 11px; color: #777;
  }
  .rp-slider input[type="range"] { flex: 1; accent-color: #6366f1; }
  .rp-slider span { font-size: 10px; color: #555; font-family: 'Geist Mono', monospace; min-width: 30px; text-align: right; }

  /* Color */
  .rp-color-big {
    position: relative;
    width: 60px; height: 60px;
    margin: 0 auto;
  }
  .rp-fg {
    position: absolute; top: 0; left: 0;
    width: 44px; height: 44px;
    border-radius: 6px; border: 2px solid #555;
    z-index: 2;
  }
  .rp-bg {
    position: absolute; bottom: 0; right: 0;
    width: 30px; height: 30px;
    border-radius: 4px; border: 2px solid #444;
    z-index: 1;
  }
  .rp-color-input {
    width: 100%; height: 30px;
    border: 1px solid #444; border-radius: 4px;
    background: #2a2a2e; cursor: pointer;
    padding: 2px;
  }
  .rp-label {
    display: flex; align-items: center; gap: 6px;
    font-size: 11px; color: #777;
  }
  .rp-hex {
    flex: 1; background: #2a2a2e; border: 1px solid #444;
    border-radius: 4px; padding: 3px 6px;
    color: #aaa; font-size: 11px;
    font-family: 'Geist Mono', monospace; outline: none;
  }
  .rp-hex:focus { border-color: #6366f1; }
  .rp-recent {
    display: flex; flex-wrap: wrap; gap: 4px;
  }
  .rp-recent-swatch {
    width: 20px; height: 20px;
    border-radius: 4px; border: 1px solid #444;
    cursor: pointer; transition: .12s;
  }
  .rp-recent-swatch:hover { border-color: #888; transform: scale(1.15); }

  /* Layers */
  .rp-layer-actions {
    display: flex; gap: 4px; margin-bottom: 6px;
  }
  .rp-layer-btn {
    flex: 1; padding: 4px; border-radius: 4px;
    background: #2a2a2e; border: 1px solid #444;
    color: #888; font-size: 10px; cursor: pointer;
    transition: .12s;
  }
  .rp-layer-btn:hover { border-color: #6366f1; color: #ccc; }
  .rp-layer-btn:disabled { opacity: .3; cursor: default; }
  .rp-layer {
    display: flex; align-items: center; gap: 6px;
    padding: 6px 8px; border-radius: 6px;
    background: #2a2a2e; border: 1px solid #333;
    color: #aaa; cursor: pointer;
    font-size: 11px; transition: .12s;
  }
  .rp-layer:hover { border-color: #555; }
  .rp-layer.active { border-color: #6366f1; background: #2a2a3e; color: #fff; }
  .rp-layer-vis {
    background: none; border: none;
    font-size: 12px; cursor: pointer;
    padding: 0; color: inherit;
    width: 20px; text-align: center;
  }
  .rp-layer-name { flex: 1; }
  .rp-layer-count { font-size: 10px; color: #555; }
</style>
