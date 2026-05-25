---
name: cuiyutian-cyt--figma-d2c
description: Unified end-to-end workflow for converting Figma Export Plugin output to pixel-perfect HTML/CSS. Replaces old workflow and separated skills.
---

# Figma Plugin D2C: Unified Pipeline

Converts the ZIP output of the custom Figma Export Plugin directly into pixel-perfect HTML/CSS using a **skeleton-first** process. This single workflow contains all rules for extraction, building, and validation — no external skills required.

> [!CAUTION]
> **CRITICAL WORKFLOW RULES — VIOLATIONS WILL PRODUCE BAD OUTPUT**
>
> 1. **NO SCRIPTS**: Do NOT write scripts (Python, Node.js) to parse `design.json`. You must read the data and directly write `index.html`.
> 2. **NO INTERMEDIATE FILES**: The only output file is `index.html`. Do not create `tokens.json`, `skeleton.html`, etc.
> 3. **NO HALLUCINATION**: Every CSS property you write must map to `design.json`. Do not invent borders, shadows, margins, or animations.
> 4. **ALL VALUES ARE 1x CSS SCALE**: Do not apply 0.5x or 2x conversion to numeric values.
> 5. **COLORS ARE HEX STRINGS**: Use `background-color` and `color` with HEX string output directly.
> 6. **ASSET PATHS AS-IS**: Use `node.asset` directly (e.g. `"./assets/icon.svg"`).

---

## Export Contract

The Figma Export Plugin is responsible for geometry fidelity. D2C must consume `design.json` exactly as exported, without visual compensation.

- **Coordinates**: `x` and `y` are relative to the JSON immediate parent. Do not apply manual offsets from screenshots or Figma.
- **Layout authority**: If a parent has `layoutMode: "HORIZONTAL"` or `"VERTICAL"`, render its children as flex items using `padding`, `itemSpacing`, and alignment fields. Child `x/y` values may exist as exported Figma placement data, but do not absolute-position normal flex children unless that child has `layoutPositioning: "ABSOLUTE"`.
- **Absolute containers**: If a parent has no `layoutMode`, render it as `position: relative` and place children with `position: absolute` using resolved `x/y` values. Missing `x` or `y` after `_ref` resolution means `0`.
- **Asset placement**: For nodes with `asset`, `x`, `y`, `width`, and `height` are final CSS placement bounds. Complex image crops, rotations, and flips may already be baked into the exported PNG.
- **Asset transform**: If an asset node has no `rotation`, do not add CSS `transform`. Only emit `transform: rotate(...)` when `rotation` is present in that same node.
- **Images**: Render `asset` directly as `<img src="node.asset">` and map `objectFit` to `object-fit`. Do not reinterpret Figma image crop data; the exporter has already resolved it.
- **Templates / `_ref`**: Treat `_ref` as deterministic delta compression: resolve it as `template + overrides` before generating HTML. Missing fields inherit from the matching `_templateId`; fields present on the `_ref` node override the template. Do not infer values from screenshots.
- **Validation tools**: Export-plugin audit or regression scripts are allowed during plugin development, but they are not part of this D2C skill. The `NO SCRIPTS` rule still applies when generating `index.html`.

---

## Phase 1: Preparation & Mental Model

### 1. Unzip (if needed)

```bash
// turbo
unzip ~/Downloads/figma_export.zip -d <project_dir>
```
Confirm the directory contains `design.json`, `skeleton.txt`, `assets/`, and `assets/reference.png`.

### 2. Read `skeleton.txt`
Read `<project_dir>/skeleton.txt`. It gives you the UI tree structure, dimensions (`WxH`), and layout types (`[H]`, `[V]`, `[ABS]`, `[SVG]`, `[TXT]`). Understand the general flow here first.

### 3. Read Extractor Field Guide (Mental Model)
Before looking at `design.json`, note these data conventions:
- **Colors**: Are pre-computed as HEX (`"#RRGGBB"`). If alpha exists (`"#RRGGBBAA"`), convert to `rgba(R, G, B, A/255)` for CSS.
- **Removed fields**: `svgContent`, `assetType`, `{r,g,b,a}` objects, and non-root `absoluteX`/`absY` are no longer in the payload. Do not look for them!
- **Deduplication**: Nodes with `_templateId` are source templates. Nodes with `_ref` are compressed nodes. Resolve them mechanically:
  1. Find the source node whose `_templateId` equals `_ref`.
  2. Deep-copy the template node.
  3. Apply every field present on the `_ref` node as an override, including explicit `x: 0` or `y: 0`, plus `id`, `name`, `content`, `width`, `height`, `color`, `segments`, `asset`, `objectFit`, `children`, and style fields.
  4. Use the resolved node as the only source for HTML/CSS.
  If the template is missing, skip the node and record a deviation. If the resolved node still lacks fields required for its type, record an export issue instead of guessing.

### 4. Read `design.json`
Read `design.json` (section by section if large), matching it against the skeleton. Build a mechanical mental model from `design.json` only. `assets/reference.png` is for final validation, not for deriving missing positions, sizes, colors, or layout rules.

---

## Phase 2: HTML/CSS Generation (Two-Pass Approach)

Start writing `index.html`. Output it to the same directory. You must act as a precise UI assembler with **exact 1:1 fidelity**.

### Mandatory CSS Reset
Include this in `<style>`. These are the ONLY non-data-driven styles allowed:
```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
div, img, span, p, a { flex-shrink: 0; }
img { display: block; }
body { -webkit-font-smoothing: antialiased; }
::-webkit-scrollbar { display: none; }
```
Add Google Fonts `<link>` tags for `fontName.family` values used.

### Pass A: Skeleton Framework
Convert the skeleton into HTML containers.
- Write the hierarchy, dimension (`width`, `height`), and `display: flex`.
- `[H]` → `display:flex; flex-direction:row`
- `[V]` → `display:flex; flex-direction:column`
- `[ABS]` / nodes without `layoutMode` → `position:relative` (children get `position:absolute`)
- `[H]` / `[V]` children stay in flex flow unless their own `layoutPositioning` is `ABSOLUTE`
- Address scrolling: Fixed headers (`top:0; z-index:10`), Scrollable bodies (`overflow-y:auto`), Fixed Footers (`bottom:0`).

### Pass B: Style Population
Add visual properties from `design.json` onto the framework:
- **Sizing**: `layoutSizingHorizontal/Vertical` mapping (`FILL` -> `flex: 1` / `stretch`, `FIXED` -> px dimensions).
- **Flex Align**: Maps `primaryAxisAlignItems` & `counterAxisAlignItems`.
- **Spacing**: `itemSpacing` -> `gap`. `padding...` -> `padding`.
- **Colors**: `backgroundColor`, `color`.
- **Borders/Radius**: `strokeColor` + `strokeWeight`. `cornerRadius` -> `border-radius`. Type `ELLIPSE` -> `50%`.
- **Gradients**: Use the exact `gradient.angle`. Format into `linear-gradient` / `radial-gradient` using `stops[].color` & `stops[].position`.
- **Typography & Segments**: Map `fontSize`, `lineHeight`, `fontName.style` (`400/500/600/700`). Render `segments[]` as `<span>` tags with inline overrides.
- **Images**: Map `asset` to `<img src="...">` and `objectFit` to `object-fit`. Use the node's exported `x/y/width/height` directly. If `rotation` is absent, do not add CSS transforms.
- **Interactive Elements**: Only add `cursor: pointer` & `:hover` states if an `interactions` object explicitly exists.
- **Templates / Overrides (`_ref`)**: First resolve `_ref` nodes using `template + overrides`, then generate HTML/CSS from the resolved node. Inherited template values are allowed only through this deterministic resolution step; never choose values by visual estimation.
- **Child placement**: In absolute containers, apply resolved `left/top`. In flex containers, do not apply `left/top` to normal children; use flex ordering, padding, gap, and alignment instead.

---

## Phase 3: Code Audit (Parameter-Level Self-Check)

> [!IMPORTANT]
> Catch numerical bugs (wrong gap values, missing opacity, font-weights). This step always runs!

Read `index.html` via `view_file` and compare its CSS values directly against `design.json` by sampling:
1. Root frame, major sections.
2. Nodes with `gradient`, `effects` (shadow), `opacity`.
3. Typography nodes with unique sizes.

Output a Markdown **Audit Table** evaluating them:
| Node name   | Field            | JSON value            | HTML value            | Status     |
|-------------|------------------|-----------------------|-----------------------|------------|
| CTA Button  | gradient.angle   | 135                   | 135deg                | ✅          |
| Avatar      | strokeColor      | #FF4433               | missing               | ❌          |

For every `❌`, immediately apply the fix to `index.html` and update the table row to `✅ (fixed)`.

---

## Phase 4: Pixel Validation Snapshot

> [!IMPORTANT]
> A single-pass visual capture loop to ensure total fidelity.

1. Serve the UI: `python3 -m http.server 8080 &`
2. Wait 2 seconds. Use the browser subagent to visit `http://localhost:8080`.
3. Wait for network idle. Capture a screenshot of the root container.
4. Visually compare the screenshot against `assets/reference.png`. Use this only to validate the result, not to invent missing source values. Look strictly for:
   - Flex direction / Wrapping / Gap mismatches.
   - Wrong Z-indexes / Overlaps.
   - Assets failing to load / Incorrect scale.
   - Text rendering issues.
5. Provide a **Deviation Checklist** of everything observed.
6. Fix only items that trace to a clear mapping mistake between `design.json` and `index.html` in a **single editing pass**. If the mismatch is caused by missing/incorrect export data, record it as an export issue instead of adding magic offsets or inferred CSS.
7. End the workflow. Do NOT take another screenshot or loop.
