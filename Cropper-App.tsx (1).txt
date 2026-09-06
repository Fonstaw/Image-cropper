import React, { useEffect, useRef, useState, useCallback } from "react";
import {
  Upload,
  Image as ImageIcon,
  FlipHorizontal,
  FlipVertical,
  RotateCcw,
  Grid3x3,
  Download,
  X,
  Move,
  ZoomIn,
  ZoomOut,
  RefreshCw,
} from "lucide-react";

const OUT_W = 2560;
const OUT_H = 1422;
const ASPECT = OUT_W / OUT_H;

type FileInfo = { name: string; size: string; w: number; h: number } | null;

export default function App() {
  const [imageSrc, setImageSrc] = useState<string | null>(null);
  const [imgObj, setImgObj] = useState<HTMLImageElement | null>(null);
  const [fileInfo, setFileInfo] = useState<FileInfo>(null);
  const [zoom, setZoom] = useState(1);
  const [rotation, setRotation] = useState(0);
  const [pan, setPan] = useState({ x: 0, y: 0 });
  const [flipH, setFlipH] = useState(false);
  const [flipV, setFlipV] = useState(false);
  const [showGrid, setShowGrid] = useState(true);
  const [format, setFormat] = useState<"jpeg" | "png">("jpeg");
  const [quality, setQuality] = useState(0.92);
  const [isDragOver, setIsDragOver] = useState(false);
  const [exporting, setExporting] = useState(false);
  const [picking, setPicking] = useState(false);

  const workspaceRef = useRef<HTMLDivElement>(null);
  const cropBoxRef = useRef<HTMLDivElement>(null);
  const [cropSize, setCropSize] = useState({ w: 800, h: 800 / ASPECT });
  const fileInputRef = useRef<HTMLInputElement>(null);

  const dragRef = useRef<{ x: number; y: number; panX: number; panY: number; active: boolean; pointerId: number | null }>({
    x: 0, y: 0, panX: 0, panY: 0, active: false, pointerId: null,
  });
  const pinchRef = useRef<{ dist: number; zoom: number } | null>(null);

  // measure crop box
  useEffect(() => {
    const el = cropBoxRef.current;
    if (!el) return;
    const ro = new ResizeObserver(() => {
      const r = el.getBoundingClientRect();
      setCropSize({ w: r.width, h: r.height });
    });
    ro.observe(el);
    const r = el.getBoundingClientRect();
    if (r.width) setCropSize({ w: r.width, h: r.height });
    return () => ro.disconnect();
  }, [imageSrc]);

  // load image
  const handleFile = useCallback((file: File) => {
    if (!file) return;
    if (!["image/jpeg", "image/png", "image/webp"].includes(file.type)) return;
    const url = URL.createObjectURL(file);
    const img = new Image();
    img.onload = () => {
      setImgObj(img);
      setImageSrc(url);
      setFileInfo({
        name: file.name,
        size: `${(file.size / 1024 / 1024).toFixed(2)} MB`,
        w: img.naturalWidth,
        h: img.naturalHeight,
      });
      // reset
      setZoom(1);
      setRotation(0);
      setPan({ x: 0, y: 0 });
      setFlipH(false);
      setFlipV(false);
    };
    img.src = url;
  }, []);

  const onDrop = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    setIsDragOver(false);
    const f = e.dataTransfer.files?.[0];
    if (f) handleFile(f);
  }, [handleFile]);

  const coverScale = React.useMemo(() => {
    if (!imgObj || !cropSize.w) return 1;
    return Math.max(cropSize.w / imgObj.naturalWidth, cropSize.h / imgObj.naturalHeight);
  }, [imgObj, cropSize]);

  const coverScaleOut = React.useMemo(() => {
    if (!imgObj) return 1;
    return Math.max(OUT_W / imgObj.naturalWidth, OUT_H / imgObj.naturalHeight);
  }, [imgObj]);

  // panning handlers
  const onPointerDown = (e: React.PointerEvent) => {
    if (!imgObj) return;
    (e.target as Element).setPointerCapture(e.pointerId);
    dragRef.current = {
      x: e.clientX,
      y: e.clientY,
      panX: pan.x,
      panY: pan.y,
      active: true,
      pointerId: e.pointerId,
    };
  };
  const onPointerMove = (e: React.PointerEvent) => {
    if (!dragRef.current.active) return;
    const dx = e.clientX - dragRef.current.x;
    const dy = e.clientY - dragRef.current.y;
    setPan({ x: dragRef.current.panX + dx, y: dragRef.current.panY + dy });
  };
  const onPointerUp = (e: React.PointerEvent) => {
    dragRef.current.active = false;
    try { (e.target as Element).releasePointerCapture(e.pointerId); } catch {}
  };

  // touch pinch
  const getTouchDist = (touches: React.TouchList) => {
    const dx = touches[0].clientX - touches[1].clientX;
    const dy = touches[0].clientY - touches[1].clientY;
    return Math.hypot(dx, dy);
  };
  const onTouchStart = (e: React.TouchEvent) => {
    if (e.touches.length === 2) {
      e.preventDefault();
      pinchRef.current = { dist: getTouchDist(e.touches), zoom };
    }
  };
  const onTouchMove = (e: React.TouchEvent) => {
    if (e.touches.length === 2 && pinchRef.current) {
      e.preventDefault();
      const dist = getTouchDist(e.touches);
      const factor = dist / pinchRef.current.dist;
      const newZoom = Math.min(3, Math.max(0.5, pinchRef.current.zoom * factor));
      setZoom(newZoom);
    }
  };
  const onTouchEnd = () => {
    pinchRef.current = null;
  };

  // wheel zoom
  const onWheel = (e: React.WheelEvent) => {
    if (!imgObj) return;
    e.preventDefault();
    const delta = -e.deltaY * 0.001;
    setZoom((z) => Math.min(3, Math.max(0.5, z + delta)));
  };

  // keyboard
  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (!imgObj) return;
      if (e.key === "ArrowUp") { e.preventDefault(); setPan(p => ({ ...p, y: p.y + (e.shiftKey ? 20 : 5) })); }
      if (e.key === "ArrowDown") { e.preventDefault(); setPan(p => ({ ...p, y: p.y - (e.shiftKey ? 20 : 5) })); }
      if (e.key === "ArrowLeft") { e.preventDefault(); setPan(p => ({ ...p, x: p.x + (e.shiftKey ? 20 : 5) })); }
      if (e.key === "ArrowRight") { e.preventDefault(); setPan(p => ({ ...p, x: p.x - (e.shiftKey ? 20 : 5) })); }
      if (e.key === "+" || e.key === "=") setZoom(z => Math.min(3, z + 0.1));
      if (e.key === "-" || e.key === "_") setZoom(z => Math.max(0.5, z - 0.1));
      if (e.key.toLowerCase() === "r" && !e.metaKey && !e.ctrlKey) {
        setZoom(1); setRotation(0); setPan({ x: 0, y: 0 }); setFlipH(false); setFlipV(false);
      }
      if (e.key.toLowerCase() === "g") setShowGrid(s => !s);
    };
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [imgObj]);

  const resetAll = () => {
    setZoom(1); setRotation(0); setPan({ x: 0, y: 0 }); setFlipH(false); setFlipV(false);
  };

  const handleExport = async () => {
    if (!imgObj || !imageSrc) return;
    setExporting(true);
    try {
      const canvas = document.createElement("canvas");
      canvas.width = OUT_W;
      canvas.height = OUT_H;
      const ctx = canvas.getContext("2d");
      if (!ctx) return;
      ctx.imageSmoothingEnabled = true;
      ctx.imageSmoothingQuality = "high";

      // background
      if (format === "jpeg") {
        ctx.fillStyle = "#000000";
        ctx.fillRect(0, 0, OUT_W, OUT_H);
      } else {
        ctx.clearRect(0, 0, OUT_W, OUT_H);
      }

      const k = OUT_W / cropSize.w; // scale from preview to output
      // pan scaled
      const outPanX = pan.x * k;
      const outPanY = pan.y * k;

      ctx.save();
      ctx.translate(OUT_W / 2 + outPanX, OUT_H / 2 + outPanY);
      ctx.rotate((rotation * Math.PI) / 180);
      ctx.scale(flipH ? -1 : 1, flipV ? -1 : 1);
      ctx.scale(zoom, zoom);

      const baseW = imgObj.naturalWidth * coverScaleOut;
      const baseH = imgObj.naturalHeight * coverScaleOut;

      ctx.drawImage(imgObj, -baseW / 2, -baseH / 2, baseW, baseH);
      ctx.restore();

      const mime = format === "jpeg" ? "image/jpeg" : "image/png";
      canvas.toBlob((blob) => {
        if (!blob) { setExporting(false); return; }
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = `crop-${OUT_W}x${OUT_H}.${format === "jpeg" ? "jpg" : "png"}`;
        a.click();
        URL.revokeObjectURL(url);
        setExporting(false);
      }, mime, format === "jpeg" ? quality : undefined);
    } catch (e) {
      console.error(e);
      setExporting(false);
    }
  };

  const clearImage = () => {
    if (imageSrc) URL.revokeObjectURL(imageSrc);
    setImageSrc(null);
    setImgObj(null);
    setFileInfo(null);
  };

  return (
    <div className="min-h-screen bg-[#09090b] text-zinc-100 selection:bg-white selection:text-black flex flex-col font-[Inter,ui-sans-serif,system-ui] antialiased">
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Geist:wght@400;500;600&family=Geist+Mono:wght@400;500&display=swap');
        *{font-family: 'Geist', ui-sans-serif, system-ui}
        .mono{font-family: 'Geist Mono', monospace}
        input[type=range]{-webkit-appearance:none; appearance:none; height:4px; background:#27272a; border-radius:999px; outline:none}
        input[type=range]::-webkit-slider-thumb{-webkit-appearance:none; width:16px; height:16px; border-radius:999px; background:white; border:2px solid #09090b; box-shadow:0 1px 4px rgba(0,0,0,.4); cursor:pointer; transition:transform .15s}
        input[type=range]::-webkit-slider-thumb:active{transform:scale(1.2)}
        input[type=range]::-moz-range-thumb{width:16px; height:16px; border-radius:999px; background:white; border:2px solid #09090b; cursor:pointer}
      `}</style>

      {/* Top bar */}
      <header className="h-[56px] shrink-0 flex items-center justify-between px-4 md:px-6 border-b border-zinc-800/80 bg-[#0f0f10] sticky top-0 z-30 backdrop-blur-xl">
        <div className="flex items-center gap-3">
          <div className="h-7 w-7 rounded-[8px] bg-white text-black grid place-items-center font-semibold mono text-[11px] tracking-tight">2560</div>
          <div className="flex items-baseline gap-2">
            <h1 className="text-[14px] font-[600] tracking-tight">2560×1422 Cropper</h1>
            <span className="hidden md:inline text-[11px] text-zinc-500 mono px-2 py-0.5 rounded-full bg-zinc-900 border border-zinc-800">{ASPECT.toFixed(3)}:1 locked</span>
          </div>
        </div>
        <div className="flex items-center gap-2">
          {imageSrc && (
            <button
              onClick={clearImage}
              className="h-8 w-8 grid place-items-center rounded-full bg-zinc-900 border border-zinc-800 text-zinc-400 hover:text-zinc-100 hover:bg-zinc-800 transition"
            >
              <X className="w-4 h-4" />
            </button>
          )}
          <button
            onClick={handleExport}
            disabled={!imageSrc || exporting}
            className="h-9 px-4 rounded-full bg-white text-black text-[13px] font-medium flex items-center gap-2 disabled:opacity-40 disabled:pointer-events-none hover:bg-zinc-100 transition shadow-[0_0_0_1px_rgba(255,255,255,.08),0_2px_10px_rgba(0,0,0,.3)]"
          >
            <Download className="w-4 h-4" />
            {exporting ? "Exporting..." : `Export ${OUT_W}×${OUT_H}`}
          </button>
        </div>
      </header>

      <div className="flex-1 flex flex-col lg:flex-row min-h-0">
        {/* Workspace */}
        <div className="flex-1 flex flex-col min-h-[50vh] lg:min-h-0 bg-[#0a0a0b] relative">
          <div
            ref={workspaceRef}
            className="flex-1 relative flex items-center justify-center p-4 md:p-10 overflow-hidden"
            onDragOver={(e) => { e.preventDefault(); setIsDragOver(true); }}
            onDragLeave={() => setIsDragOver(false)}
            onDrop={onDrop}
            onWheel={onWheel}
          >
            {/* subtle grid bg */}
            <div className="absolute inset-0 opacity-[0.03]" style={{
              backgroundImage: `linear-gradient(to right, white 1px, transparent 1px), linear-gradient(to bottom, white 1px, transparent 1px)`,
              backgroundSize: '32px 32px'
            }} />

            {!imageSrc ? (
              <div className="relative w-full max-w-[720px]">
                <div
                  className={`group relative rounded-[24px] border border-dashed transition-all duration-300 ${isDragOver ? "border-white bg-white/[0.04] scale-[1.01]" : "border-zinc-700/60 bg-[#121214] hover:bg-[#151517] hover:border-zinc-600"}`}
                >
                  <div className="px-8 py-16 md:py-20 flex flex-col items-center text-center">
                    {/* illustration */}
                    <div className="relative mb-8">
                      <div className="h-[88px] w-[88px] rounded-[20px] bg-gradient-to-b from-zinc-800 to-zinc-900 border border-zinc-700/50 grid place-items-center shadow-[inset_0_1px_0_0_rgba(255,255,255,.08),0_20px_40px_rgba(0,0,0,.5)]">
                        <ImageIcon className="w-8 h-8 text-zinc-300" strokeWidth={1.5} />
                      </div>
                      <div className="absolute -bottom-2 -right-2 h-7 w-7 rounded-full bg-white text-black grid place-items-center shadow-lg">
                        <Upload className="w-4 h-4" />
                      </div>
                    </div>

                    <h2 className="text-[20px] md:text-[22px] font-semibold tracking-tight leading-tight">Drop image or click to upload</h2>
                    <p className="mt-2 text-[13px] text-zinc-400 max-w-[36ch] leading-relaxed">
                      Will export at exactly <span className="text-zinc-100 font-medium mono">{OUT_W} × {OUT_H}</span> — locked {ASPECT.toFixed(2)}:1 ratio. JPG, PNG, WEBP supported.
                    </p>

                    <div className="mt-6 flex flex-col items-center gap-2">
                      <div className="flex items-center gap-2">
                        <button
                          onClick={() => { setPicking(true); fileInputRef.current?.click(); setTimeout(()=>setPicking(false), 1800); }}
                          className="h-9 px-5 rounded-full bg-white text-black text-[13px] font-medium hover:bg-zinc-100 transition"
                        >
                          {picking ? "Opening…" : "Browse files"}
                        </button>
                        <span className="text-[11px] text-zinc-500 mono">or drag & drop</span>
                      </div>
                      {picking && <span className="text-[11px] mono text-zinc-300 animate-pulse">Choose JPG, PNG or WEBP</span>}
                    </div>

                    <div className="mt-10 flex items-center gap-2 text-[11px] mono text-zinc-500">
                      <span className="px-2.5 py-1 rounded-full bg-zinc-900 border border-zinc-800">2560×1422</span>
                      <span className="px-2.5 py-1 rounded-full bg-zinc-900 border border-zinc-800">1.8:1</span>
                      <span className="px-2.5 py-1 rounded-full bg-zinc-900 border border-zinc-800">High quality canvas</span>
                    </div>
                  </div>
                </div>

                <p className="mt-4 text-center text-[11px] mono text-zinc-500">
                  No external libraries • Custom pan/zoom logic • Exact pixel export
                </p>
              </div>
            ) : (
              <div className="relative w-full max-w-[1100px] flex items-center justify-center">
                {/* crop area */}
                <div
                  ref={cropBoxRef}
                  className="relative w-full aspect-[2560/1422] max-h-[68vh] lg:max-h-[72vh] rounded-[12px] overflow-hidden bg-black shadow-[0_0_0_1px_rgba(255,255,255,.08),0_30px_80px_rgba(0,0,0,.7)]"
                  style={{ touchAction: "none" }}
                  onPointerDown={onPointerDown}
                  onPointerMove={onPointerMove}
                  onPointerUp={onPointerUp}
                  onTouchStart={onTouchStart}
                  onTouchMove={onTouchMove}
                  onTouchEnd={onTouchEnd}
                >
                  {/* image */}
                  {imageSrc && imgObj && (
                    <img
                      src={imageSrc}
                      alt="source"
                      draggable={false}
                      className="absolute left-1/2 top-1/2 max-w-none select-none will-change-transform"
                      style={{
                        width: imgObj.naturalWidth * coverScale,
                        height: imgObj.naturalHeight * coverScale,
                        transform: `translate(calc(-50% + ${pan.x}px), calc(-50% + ${pan.y}px)) rotate(${rotation}deg) scaleX(${flipH ? -1 : 1}) scaleY(${flipV ? -1 : 1}) scale(${zoom})`,
                        transformOrigin: "center center",
                      }}
                    />
                  )}

                  {/* dark outside using shadow */}
                  <div className="pointer-events-none absolute inset-0 shadow-[0_0_0_9999px_rgba(0,0,0,0.62)]" />

                  {/* border */}
                  <div className="pointer-events-none absolute inset-0 rounded-[12px] border border-white/20" />
                  <div className="pointer-events-none absolute inset-0 rounded-[12px] border border-white/10 mix-blend-overlay" />

                  {/* handles */}
                  <div className="pointer-events-none absolute inset-0">
                    {/* corners */}
                    <div className="absolute -top-1 -left-1 h-5 w-5 border-l-2 border-t-2 border-white rounded-tl-[8px]" />
                    <div className="absolute -top-1 -right-1 h-5 w-5 border-r-2 border-t-2 border-white rounded-tr-[8px]" />
                    <div className="absolute -bottom-1 -left-1 h-5 w-5 border-l-2 border-b-2 border-white rounded-bl-[8px]" />
                    <div className="absolute -bottom-1 -right-1 h-5 w-5 border-r-2 border-b-2 border-white rounded-br-[8px]" />
                    {/* center crosshair */}
                    <div className="absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2">
                      <div className="relative h-6 w-6">
                        <div className="absolute left-1/2 top-0 bottom-0 w-px bg-white/60 -translate-x-1/2" />
                        <div className="absolute top-1/2 left-0 right-0 h-px bg-white/60 -translate-y-1/2" />
                        <div className="absolute left-1/2 top-1/2 h-1.5 w-1.5 bg-white rounded-full -translate-x-1/2 -translate-y-1/2 shadow" />
                      </div>
                    </div>
                  </div>

                  {/* grid */}
                  {showGrid && (
                    <div className="pointer-events-none absolute inset-0">
                      <div className="absolute inset-0 grid grid-cols-3 grid-rows-3">
                        {Array.from({ length: 9 }).map((_, i) => (
                          <div key={i} className="border border-white/[0.12]" />
                        ))}
                      </div>
                    </div>
                  )}

                  {/* dimensions badge inside crop */}
                  <div className="pointer-events-none absolute left-3 top-3 flex items-center gap-2">
                    <div className="mono text-[11px] px-2.5 py-1 rounded-full bg-black/70 backdrop-blur text-white border border-white/15 shadow">
                      {OUT_W} × {OUT_H}
                    </div>
                    <div className="hidden md:flex mono text-[10px] px-2 py-1 rounded-full bg-white text-black font-medium">
                      {(zoom * 100).toFixed(0)}%
                    </div>
                  </div>

                  {/* drag hint */}
                  <div className="pointer-events-none absolute bottom-3 left-1/2 -translate-x-1/2 flex items-center gap-1.5 text-[11px] mono px-2.5 py-1 rounded-full bg-black/60 backdrop-blur border border-white/10 text-white/70">
                    <Move className="w-3 h-3" /> drag to reposition • scroll to zoom
                  </div>
                </div>
              </div>
            )}

            {/* drag over overlay */}
            {isDragOver && (
              <div className="absolute inset-4 rounded-[20px] border-2 border-dashed border-white bg-white/[0.04] backdrop-blur-sm grid place-items-center pointer-events-none">
                <div className="text-[14px] font-medium">Drop to load</div>
              </div>
            )}
          </div>

          {/* bottom info */}
          <div className="h-[44px] shrink-0 border-t border-zinc-800/80 bg-[#0f0f10] flex items-center justify-between px-4 md:px-6">
            <div className="flex items-center gap-3 text-[11px] mono text-zinc-400">
              <span className="flex items-center gap-1.5"><ZoomIn className="w-3 h-3" />{Math.round(zoom * 100)}%</span>
              <span className="h-3 w-px bg-zinc-800" />
              <span className="flex items-center gap-1.5"><RotateCcw className="w-3 h-3" />{rotation}°</span>
              {fileInfo && (
                <>
                  <span className="hidden md:block h-3 w-px bg-zinc-800" />
                  <span className="hidden md:inline truncate max-w-[28ch]">{fileInfo.name} • {fileInfo.w}×{fileInfo.h} • {fileInfo.size}</span>
                </>
              )}
            </div>
            <div className="flex items-center gap-2 text-[11px] mono text-zinc-500">
              <span className="hidden md:inline">←→↑↓ nudge • +/- zoom • R reset • G grid</span>
              <span className="md:hidden">pinch to zoom</span>
            </div>
          </div>
        </div>

        {/* Controls panel */}
        <aside className="w-full lg:w-[340px] shrink-0 bg-[#121214] border-t lg:border-t-0 lg:border-l border-zinc-800 flex flex-col">
          <div className="p-5 flex-1 overflow-auto">
            <div className="flex items-center justify-between mb-5">
              <h3 className="text-[13px] font-semibold tracking-tight">Controls</h3>
              <button onClick={resetAll} disabled={!imageSrc} className="h-7 px-2.5 rounded-full bg-zinc-900 border border-zinc-800 text-[11px] mono flex items-center gap-1.5 hover:bg-zinc-800 disabled:opacity-40">
                <RefreshCw className="w-3 h-3" /> Reset
              </button>
            </div>

            {!imageSrc ? (
              <div className="rounded-[12px] bg-zinc-900/60 border border-zinc-800 p-4 text-[12px] leading-relaxed text-zinc-400">
                Upload an image to unlock precise crop controls. Exact canvas export at 2560×1422 is guaranteed via offscreen canvas draw.
              </div>
            ) : (
              <div className="space-y-6">
                {/* Zoom */}
                <div className="space-y-3">
                  <div className="flex items-center justify-between">
                    <label className="text-[12px] font-medium flex items-center gap-1.5"><ZoomIn className="w-3.5 h-3.5 text-zinc-400" /> Zoom</label>
                    <span className="mono text-[11px] px-2 py-0.5 rounded-full bg-zinc-900 border border-zinc-800">{Math.round(zoom * 100)}%</span>
                  </div>
                  <div className="flex items-center gap-2">
                    <button onClick={() => setZoom(z => Math.max(0.5, z - 0.1))} className="h-8 w-8 rounded-full bg-zinc-900 border border-zinc-800 grid place-items-center hover:bg-zinc-800"><ZoomOut className="w-4 h-4" /></button>
                    <input type="range" min={0.5} max={3} step={0.01} value={zoom} onChange={e => setZoom(parseFloat(e.target.value))} className="flex-1" />
                    <button onClick={() => setZoom(z => Math.min(3, z + 0.1))} className="h-8 w-8 rounded-full bg-zinc-900 border border-zinc-800 grid place-items-center hover:bg-zinc-800"><ZoomIn className="w-4 h-4" /></button>
                  </div>
                  <div className="flex justify-between mono text-[10px] text-zinc-500 px-1"><span>50%</span><span>100%</span><span>300%</span></div>
                </div>

                {/* Rotation */}
                <div className="space-y-3">
                  <div className="flex items-center justify-between">
                    <label className="text-[12px] font-medium">Rotation</label>
                    <div className="flex items-center gap-1">
                      <span className="mono text-[11px] px-2 py-0.5 rounded-full bg-zinc-900 border border-zinc-800">{rotation}°</span>
                      <button onClick={() => setRotation(0)} className="h-6 px-2 rounded-full bg-zinc-800 text-[10px] mono hover:bg-zinc-700">Reset</button>
                    </div>
                  </div>
                  <input type="range" min={-45} max={45} step={1} value={rotation} onChange={e => setRotation(parseInt(e.target.value))} className="w-full" />
                  <div className="flex justify-between mono text-[10px] text-zinc-500 px-1"><span>-45°</span><span>0°</span><span>45°</span></div>
                </div>

                {/* Flip & Grid */}
                <div className="grid grid-cols-3 gap-2">
                  <button onClick={() => setFlipH(v => !v)} className={`h-10 rounded-[10px] border text-[12px] font-medium flex flex-col items-center justify-center gap-0.5 transition ${flipH ? "bg-white text-black border-white" : "bg-zinc-900 border-zinc-800 text-zinc-300 hover:bg-zinc-800"}`}>
                    <FlipHorizontal className="w-4 h-4" /> <span className="text-[10px] mono">Flip H</span>
                  </button>
                  <button onClick={() => setFlipV(v => !v)} className={`h-10 rounded-[10px] border text-[12px] font-medium flex flex-col items-center justify-center gap-0.5 transition ${flipV ? "bg-white text-black border-white" : "bg-zinc-900 border-zinc-800 text-zinc-300 hover:bg-zinc-800"}`}>
                    <FlipVertical className="w-4 h-4" /> <span className="text-[10px] mono">Flip V</span>
                  </button>
                  <button onClick={() => setShowGrid(v => !v)} className={`h-10 rounded-[10px] border text-[12px] font-medium flex flex-col items-center justify-center gap-0.5 transition ${showGrid ? "bg-white text-black border-white" : "bg-zinc-900 border-zinc-800 text-zinc-300 hover:bg-zinc-800"}`}>
                    <Grid3x3 className="w-4 h-4" /> <span className="text-[10px] mono">Grid</span>
                  </button>
                </div>

                <div className="h-px bg-zinc-800/80" />

                {/* Export settings */}
                <div className="space-y-4">
                  <h4 className="text-[12px] font-medium">Export</h4>

                  <div className="grid grid-cols-2 gap-2 p-1 rounded-[12px] bg-zinc-900 border border-zinc-800">
                    {(["jpeg", "png"] as const).map((f) => (
                      <button
                        key={f}
                        onClick={() => setFormat(f)}
                        className={`h-8 rounded-[8px] text-[12px] font-medium mono transition ${format === f ? "bg-white text-black shadow" : "text-zinc-400 hover:text-zinc-100"}`}
                      >
                        {f.toUpperCase()}
                      </button>
                    ))}
                  </div>

                  {format === "jpeg" && (
                    <div className="space-y-2">
                      <div className="flex items-center justify-between">
                        <label className="text-[11px] mono text-zinc-400">Quality</label>
                        <span className="mono text-[11px] px-2 py-0.5 rounded-full bg-zinc-900 border border-zinc-800">{quality.toFixed(2)}</span>
                      </div>
                      <input type="range" min={0.7} max={1} step={0.01} value={quality} onChange={e => setQuality(parseFloat(e.target.value))} className="w-full" />
                      <div className="flex justify-between mono text-[10px] text-zinc-500"><span>0.70</span><span>High fidelity</span><span>1.00</span></div>
                    </div>
                  )}

                  <div className="rounded-[12px] bg-[#0a0a0b] border border-zinc-800 p-3 space-y-2">
                    <div className="flex items-center justify-between text-[11px] mono">
                      <span className="text-zinc-500">Output</span>
                      <span className="text-zinc-100 font-medium">{OUT_W} × {OUT_H} px</span>
                    </div>
                    <div className="flex items-center justify-between text-[11px] mono">
                      <span className="text-zinc-500">Aspect</span>
                      <span className="text-zinc-300">{ASPECT.toFixed(4)} — 2560:1422</span>
                    </div>
                    <div className="flex items-center justify-between text-[11px] mono">
                      <span className="text-zinc-500">Method</span>
                      <span className="text-zinc-300">Canvas • high-quality</span>
                    </div>
                  </div>

                  <button
                    onClick={handleExport}
                    disabled={!imageSrc || exporting}
                    className="w-full h-11 rounded-[12px] bg-white text-black text-[13px] font-semibold flex items-center justify-center gap-2 hover:bg-zinc-100 disabled:opacity-40 transition shadow-[0_0_0_1px_rgba(255,255,255,.1),0_8px_24px_rgba(0,0,0,.4)]"
                  >
                    <Download className="w-4 h-4" /> {exporting ? "Rendering..." : `Download ${format.toUpperCase()}`}
                  </button>
                  <p className="text-[11px] mono text-zinc-500 text-center leading-relaxed">
                    Renders offscreen {OUT_W}×{OUT_H} canvas with current pan / zoom / rotation / flip. Guaranteed exact size.
                  </p>
                </div>
              </div>
            )}
          </div>

          <div className="p-4 border-t border-zinc-800 bg-[#0f0f10]">
            <div className="flex items-center gap-2 text-[11px] mono text-zinc-500">
              <div className="h-2 w-2 rounded-full bg-emerald-400 shadow-[0_0_8px_rgba(16,185,129,.6)]" />
              Canvas export ready • No upload • Local only
            </div>
          </div>
        </aside>
      </div>

      <input ref={fileInputRef} type="file" accept="image/jpeg,image/png,image/webp" className="hidden" onChange={e => { const f = e.target.files?.[0]; if (f) handleFile(f); }} />
    </div>
  );
}
