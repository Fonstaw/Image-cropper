# 2560×1422 Image Cropper

Pixel-perfect cropper locked to 2560×1422. Local-only canvas export, no upload.

## Features
- Fixed aspect 2560:1422 (1.8:1)
- Drag to pan, wheel/pinch to zoom 50-300%
- Rotate -45° to 45°, flip H/V
- Grid overlay, crosshair
- Export exactly 2560×1422 as JPG (quality control) or PNG
- Keyboard: arrows nudge, +/- zoom, R reset, G grid

## Run locally
```bash
npm install
npm run dev
```

## Deploy to Vercel
1. Push this folder to GitHub
2. Import repo in Vercel dashboard
3. Framework preset: Vite
4. Build command: `npm run build`
5. Output directory: `dist`
6. Deploy

No env vars needed.

## Stack
React + Vite + Tailwind + lucide-react + Canvas API
