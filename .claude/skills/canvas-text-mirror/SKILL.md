---
name: canvas-text-mirror
description: Clone all DOM text from an HTML subtree onto an HTML Canvas with pixel-perfect position, font, color, letter-spacing, and line breaks. Produces a canvas layer that is visually identical to the HTML — same glyphs, same positions — so canvas effects (particles, dissolve, scanlines, shaders) can be layered on top without touching the DOM. Use when the user wants canvas-based text effects, pixel-sampling the page text, or any effect that requires a canvas copy of rendered text.
user-invocable: true
argument-hint: "[target element or page]"
---

## What This Does

Walks every text node in a DOM subtree, reads browser-computed styles, detects exact line breaks using `Range.getClientRects()`, and draws each line to a Canvas 2D context with `fillText`. The result is a canvas that looks identical to the HTML — same fonts, same positions, same colors, same wrapping.

This is the **foundation layer** for canvas text effects: particles, grain dissolve, scanlines, glitch, shaders — anything that needs the text in pixel form.

---

## The Core Algorithm

### 1 — Walk text nodes

```ts
const walker = document.createTreeWalker(root, NodeFilter.SHOW_TEXT)
while (walker.nextNode()) {
  const node = walker.currentNode as Text
  if (!node.textContent?.trim()) continue
  const parent = node.parentElement!
  const cs = getComputedStyle(parent)
  // cache: font, color, letterSpacing, textTransform
}
```

Gate everything behind `await document.fonts.ready` — drawing before fonts load produces wrong widths.

### 2 — Detect browser line breaks with Range

Do NOT reimplement text layout (Pretext, manual wrapping). The browser already laid out the text — ask it where the lines are.

```ts
const range = document.createRange()
for (let i = 0; i < text.length; i++) {
  range.setStart(node, i)
  range.setEnd(node, i + 1)
  const r = range.getClientRects()[0]
  // detect new line: Math.round(r.top) !== prevRoundedTop
  // store: unrounded r.top (for drawing), r.height (for halfLeading), r.left
}
```

**Rounding rule**: use `Math.round(r.top)` ONLY to detect line-break (same line = same rounded top). Store the **unrounded** `r.top` for actual drawing — rounding the draw position causes visible drift on high-DPI screens.

### 3 — Y-position formula

CSS line-box model:
```
line_box_top
  └─ halfLeading   (= (line_box_height - em_height) / 2)
  └─ em_box_top    (= font ascender line)
  └─ baseline      ← canvas draws here with textBaseline='alphabetic'
  └─ em_box_bottom
  └─ halfLeading
line_box_bottom
```

```ts
ctx.textBaseline = 'alphabetic'

const m = ctx.measureText('H')
const fontAscent = m.fontBoundingBoxAscent      // ← CRITICAL: NOT actualBoundingBoxAscent
const emHeight   = fontAscent + m.fontBoundingBoxDescent

// line.top    = line_box_top  (from Range rect, unrounded)
// line.height = line_box_height (from Range rect rects[0].height)
const halfLeading = Math.max(0, (line.height - emHeight) / 2)
const y = line.top + halfLeading + fontAscent

ctx.fillText(text, line.left, y)
```

### Why `fontBoundingBoxAscent`, NOT `actualBoundingBoxAscent`

| Metric | What it measures | Problem |
|--------|-----------------|---------|
| `actualBoundingBoxAscent` | Ink top of drawn glyphs (capHeight of 'H') | Too small — varies per glyph, places text too HIGH |
| `fontBoundingBoxAscent` | Font's designed ascender (top of em box) | Correct — matches what CSS uses for line box layout |

Using `actualBoundingBoxAscent` shifts text upward by `fontAscent - capHeight`, which is font-dependent (visible misalignment across Outfit, Geist Mono, Lora, etc.).

### Why `line.height` for halfLeading, NOT CSS `lineHeight`

- `getComputedStyle(el).lineHeight` returns `"normal"` for elements without explicit `line-height`
- `"normal"` is font-metric-dependent — different fonts produce different pixel values
- `rects[0].height` is the **actual rendered line-box height** — no guessing required

Do not hardcode `fontSize * 1.2` or `fontSize * 1.6` as a "normal" fallback.

### 4 — Canvas setup

```ts
const dpr = window.devicePixelRatio || 1
canvas.width  = window.innerWidth  * dpr
canvas.height = window.innerHeight * dpr
canvas.style.width  = window.innerWidth  + 'px'
canvas.style.height = window.innerHeight + 'px'

// In drawCached:
ctx.setTransform(dpr, 0, 0, dpr, 0, 0)  // scale once, draw in CSS pixels
```

Since canvas is `position: fixed`, `getBoundingClientRect()` / `getClientRects()` positions are already viewport-relative — no scroll math needed.

### 5 — letter-spacing reset (critical)

Canvas `ctx.letterSpacing` **persists between draws**. If one element sets it, the next element inherits it even with a different `ctx.font`.

```ts
if ('letterSpacing' in ctx) {
  (ctx as any).letterSpacing = rec.letterSpacing === 'normal' ? '0px' : rec.letterSpacing
}
```

Always set this for every text record, even if it is `'0px'`.

### 6 — textTransform

Canvas knows nothing about CSS `text-transform`. Apply it manually before drawing:

```ts
function applyTextTransform(text: string, transform: string): string {
  if (transform === 'uppercase') return text.toUpperCase()
  if (transform === 'lowercase') return text.toLowerCase()
  if (transform === 'capitalize') return text.replace(/\b\w/g, c => c.toUpperCase())
  return text
}
```

---

## Complete Implementation (Vue composable)

```ts
// composables/useCanvasMirror.ts

interface TextRecord {
  node: Text
  parent: HTMLElement
  font: string
  fontSize: number
  color: string
  letterSpacing: string
  textTransform: string
}

interface RenderedLine {
  text: string
  left: number
  top: number      // unrounded
  height: number   // actual line-box height from browser
}

function applyTextTransform(text: string, transform: string): string {
  if (transform === 'uppercase') return text.toUpperCase()
  if (transform === 'lowercase') return text.toLowerCase()
  if (transform === 'capitalize') return text.replace(/\b\w/g, c => c.toUpperCase())
  return text
}

function getRenderedLines(node: Text): RenderedLine[] {
  const text = node.textContent || ''
  if (!text.trim()) return []

  const range = document.createRange()
  const lines: RenderedLine[] = []

  let lineStart = 0
  let lineLeft = 0
  let lineTop: number | null = null
  let lineTopKey: number | null = null
  let lineHeight = 0

  for (let i = 0; i < text.length; i++) {
    range.setStart(node, i)
    range.setEnd(node, i + 1)
    const rects = range.getClientRects()
    if (!rects.length) continue

    const r = rects[0]
    const topKey = Math.round(r.top)

    if (lineTopKey === null) {
      lineTopKey = topKey
      lineTop = r.top
      lineLeft = r.left
      lineHeight = r.height
    } else if (topKey !== lineTopKey) {
      lines.push({ text: text.slice(lineStart, i), left: lineLeft, top: lineTop!, height: lineHeight })
      lineStart = i
      lineTopKey = topKey
      lineTop = r.top
      lineLeft = r.left
      lineHeight = r.height
    }
  }

  if (lineTop !== null)
    lines.push({ text: text.slice(lineStart), left: lineLeft, top: lineTop, height: lineHeight })

  range.detach()
  return lines
}

export function useCanvasMirror(
  canvas: Ref<HTMLCanvasElement | null>,
  root: Ref<HTMLElement | null>
) {
  const cache = new Map<Text, TextRecord>()

  async function prepare() {
    await document.fonts.ready
    cache.clear()
    if (!root.value) return

    const walker = document.createTreeWalker(root.value, NodeFilter.SHOW_TEXT)
    while (walker.nextNode()) {
      const node = walker.currentNode as Text
      if (!node.textContent?.trim()) continue
      const parent = node.parentElement
      if (!parent) continue

      const cs = getComputedStyle(parent)
      const fontSize = parseFloat(cs.fontSize)
      const font = `${cs.fontStyle} ${cs.fontWeight} ${cs.fontSize} ${cs.fontFamily}`

      cache.set(node, {
        node, parent, font, fontSize,
        color: cs.color,
        letterSpacing: cs.letterSpacing,
        textTransform: cs.textTransform,
      })
    }
  }

  function drawCached() {
    const c = canvas.value
    if (!c) return
    const ctx = c.getContext('2d')
    if (!ctx) return

    const dpr = window.devicePixelRatio || 1
    ctx.clearRect(0, 0, c.width, c.height)
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
    ctx.textBaseline = 'alphabetic'

    for (const rec of cache.values()) {
      ctx.font = rec.font
      ctx.fillStyle = rec.color
      if ('letterSpacing' in ctx)
        (ctx as any).letterSpacing = rec.letterSpacing === 'normal' ? '0px' : rec.letterSpacing

      const m = ctx.measureText('H')
      const fontAscent = m.fontBoundingBoxAscent
      const emHeight = fontAscent + m.fontBoundingBoxDescent

      for (const line of getRenderedLines(rec.node)) {
        const text = applyTextTransform(line.text, rec.textTransform).trim()
        if (!text) continue
        const halfLeading = Math.max(0, (line.height - emHeight) / 2)
        ctx.fillText(text, line.left, line.top + halfLeading + fontAscent)
      }
    }
  }

  async function draw() {
    if (cache.size === 0) await prepare()
    drawCached()
  }

  function clear() {
    const c = canvas.value
    if (!c) return
    const ctx = c.getContext('2d')
    if (!ctx) return
    ctx.clearRect(0, 0, c.width, c.height)
    cache.clear()
  }

  function update()             { drawCached() }                      // resize, no re-walk
  async function reprepare()    { cache.clear(); await prepare(); drawCached() }  // content change

  return { draw, clear, update, reprepare }
}
```

---

## React port (hook)

```ts
import { useRef, useCallback } from 'react'

export function useCanvasMirror(
  canvasRef: React.RefObject<HTMLCanvasElement>,
  rootRef: React.RefObject<HTMLElement>
) {
  const cacheRef = useRef(new Map())

  // same prepare / drawCached / getRenderedLines logic
  // return { draw, clear, update, reprepare }
}
```

## Vanilla JS port

```ts
export class CanvasMirror {
  constructor(private canvas: HTMLCanvasElement, private root: HTMLElement) {}
  async draw() { ... }
  clear() { ... }
  update() { ... }
}
```

---

## Canvas overlay setup (CSS)

```css
.mirror-canvas {
  position: fixed;
  inset: 0;
  pointer-events: none;   /* HTML links stay clickable */
  z-index: 1000;
}
```

Canvas is additive — HTML text remains visible and interactive underneath. The canvas layer adds visual texture on top.

---

## Effects you can build on this foundation

Once the mirror is pixel-perfect, the canvas layer is the starting point for:

| Effect | How |
|--------|-----|
| **Particle letterform** | Sample pixels from headline on OffscreenCanvas → place particles → Brownian drift |
| **Grain dissolve** | Draw text, then mask with noise texture using `globalCompositeOperation` |
| **Scanlines** | Draw text at low opacity in accent color + horizontal scanline sweep |
| **Chromatic aberration** | Draw text 3× (R, G, B) with small x/y offsets, `globalCompositeOperation: 'screen'` |
| **Glitch** | Random horizontal slice shifts on canvas regions, animated |
| **Blur reveal** | Start blurred (`filter: blur(Npx)`) → animate to sharp on scroll |

For particle effects: draw the text onto an OffscreenCanvas → `getImageData()` → scan filled pixels → subsample → place particles.

---

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Using `actualBoundingBoxAscent` for baseline | Use `fontBoundingBoxAscent` |
| Storing rounded `top` for drawing | Round ONLY for line-break detection; draw with float |
| Computing `halfLeading` from `cs.lineHeight` | Use `rects[0].height` from Range |
| Not resetting `ctx.letterSpacing` | Always reset it, even to `'0px'` |
| Forgetting `textTransform` | Apply it manually before `fillText` |
| Drawing before fonts load | Gate behind `await document.fonts.ready` |
| Using `textBaseline = 'top'` | Font-dependent behavior across typefaces; use `'alphabetic'` |
| Calling `ctx.setTransform` inside the glyph loop | Set once per `drawCached` call |
| Reimplementing line-break detection in JS | Use `Range.getClientRects()` — the browser already did it |