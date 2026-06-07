# Rich Text Editor — Project Documentation

> **Single source of truth** for the custom Notion-like block-based rich text editor built for Bubble.io.  
> Last updated: June 7–8, 2026 · Current version: **v8**

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Technology Stack & Constraints](#2-technology-stack--constraints)
3. [Architecture](#3-architecture)
4. [Version History & Changelog](#4-version-history--changelog)
5. [Feature Reference](#5-feature-reference)
6. [CSS Custom Properties & Design Tokens](#6-css-custom-properties--design-tokens)
7. [Layout Geometry Reference](#7-layout-geometry-reference)
8. [State Machine — Height & Scroll Behavior](#8-state-machine--height--scroll-behavior)
9. [Key JavaScript Functions](#9-key-javascript-functions)
10. [Bubble.io Integration](#10-bubbleio-integration)
11. [Known Bug Fixes & Root Cause Log](#11-known-bug-fixes--root-cause-log)
12. [Conventions & Code Style](#12-conventions--code-style)
13. [Bubble Setup Checklist](#13-bubble-setup-checklist)

---

## 1. Project Overview

A **self-contained, single-file** HTML/CSS/JS snippet intended to be pasted directly into a Bubble.io **HTML Element**. It functions as a Notion-inspired block-based rich text editor with:

- A floating **Plus (+) button** that tracks the active cursor line and opens a contextual block-formatting menu.
- A **horizontal icon-only toolbar** (expandable from the `+` button) with CSS tooltips.
- Full **Bubble.io Toolbox plugin** integration for real-time data sync.
- **Secure initial content hydration** from Bubble dynamic data via a `data-initial-content` attribute.
- **State-based height behavior**: compact on load, auto-expands on focus, restores on blur.

---

## 2. Technology Stack & Constraints

| Concern | Decision |
|---|---|
| Framework | Vanilla JavaScript only — no React, Vue, or heavy frameworks |
| Styling | Plain CSS with custom properties; no preprocessors |
| Font | `'DM Sans', system-ui, sans-serif` — inherited from Bubble's global load; **no `@import`** |
| Editor core | `<div contenteditable="true">` |
| Bubble sync | Toolbox plugin — `bubble_fn_saveEditorData(html)` |
| External deps | None |
| Target environment | Bubble.io HTML Element (iframe sandbox) |

---

## 3. Architecture

### 3.1 HTML Structure

```
.huz-wrap                        ← outer resizable frame (owns overflow + resize handle)
  └── .huz-editor-body           ← inner flex row container
        ├── .huz-plus-col        ← left gutter column (44px wide)
        │     ├── #huz-plus-btn  ← floating + / × toggle button (position: absolute)
        │     └── #huz-block-menu← horizontal toolbar pill (position: fixed in viewport)
        └── #huz-editor          ← contenteditable div (the typing canvas)
```

### 3.2 Two-Layer Scroll / Resize Model

The wrapper (`.huz-wrap`) owns both `resize: vertical` and `overflow`. This is the only correct way to show a native browser resize grip alongside a scrollbar — both must live on the same element.

```
.huz-wrap         → overflow: auto  + resize: vertical   (default / blur state)
                  → overflow: visible + resize: none       (active / focus state)

#huz-editor       → height: auto + overflow-y: visible always
                    (grows naturally; wrapper does the clipping/scrolling)
```

### 3.3 Block Menu Position

`#huz-block-menu` uses `position: fixed` (viewport coordinates) rather than `position: absolute`. This escapes the wrapper's `overflow: auto` clipping context, so the menu is never cut off regardless of wrapper scroll state. `setMenuPosition()` reads `getBoundingClientRect()` from `#huz-plus-btn` at open-time to anchor the menu.

### 3.4 Cursor Rendering Pipeline (FOUC prevention)

To prevent Flash of Unstyled Content on cursor color:

1. Critical cursor CSS block is placed **above** any `@import` in `<style>` (parsed synchronously on first layout pass).
2. `style="cursor:text;color-scheme:light;"` is inlined on both `.huz-wrap` and `#huz-editor` (applied before CSS engine starts).
3. No `@import` for Google Fonts — font is inherited from Bubble's global stylesheet.

---

## 4. Version History & Changelog

### v1 — Initial Build
**Snapshot:** `Notion-like-block-editor-for-Bubble-io_snapshot_1.md`

**Features introduced:**
- `contenteditable` div editor with CSS `resize: vertical`
- Floating `+` button tracking cursor line position
- Block menu: Text, H1–H3, Bullet, Numbered, Quote, Code block, Divider
- Inline formatting toolbar (Bold, Italic, Underline, Strike, Headings, Lists, Align, Clear)
- `bubble_fn_saveEditorData(html)` debounced 500ms + immediate on blur
- Safe hydration from `data-initial-content` attribute
- Word/char count footer
- DM Sans font, `#534AB7` purple accent

---

### v2 — Toolbar Refactor & Menu Animation
**Snapshot:** `Notion-like-block-editor-for-Bubble-io_snapshot_1.md` (turn 2)

**Changes:**
- Removed word/char count footer (`.huz-footer`, `#huz-wc`, `#huz-cc`, `updateCount()`)
- Removed fixed top toolbar (`.huz-toolbar`)
- Merged all formatting options into the floating `+` menu
- `+` morphs to `×` via CSS `transform: rotate(45deg)` on `.open` class (no JS string swap)
- Menu animates left→right: `transform-origin: left center` + `scaleX(0→1)` with `cubic-bezier(0.22,1,0.36,1)`
- Menu uses `visibility: hidden` (not `display: none`) so exit animation plays fully
- All menu interactions use `mousedown` + `e.preventDefault()` to preserve editor focus
- Blur guard: editor blur won't close menu if focus moves into the menu (`e.relatedTarget`)

---

### v3 — Horizontal Icon Menu, Plus Button First-Line Fix, Resize Handle Deduplication
**Snapshot:** `Notion-like-block-editor-for-Bubble-io_snapshot_1.md` (turn 3)

**Bug fixes:**
1. **Plus button on first line** — `positionPlusBtn(null)` now checks `document.activeElement === editor` before hiding; if focused, calls `showOnFirstLine()` instead. Both `focus` and `click` listeners use `setTimeout(..., 0)` to let the browser commit cursor position first.
2. **Horizontal icon-only menu** — `#huz-block-menu` changed to `flex-direction: row`. Buttons show only glyphs (`H1`, `•`, `1.`, `❝`, `B`, `I`). Tooltips are pure CSS via `::after`/`::before` reading `data-tip` attribute; slide up with opacity fade.
3. **Double resize handle** — Changed `.huz-wrap` from `overflow: auto` to `overflow: hidden`. Removed `.huz-resize-hint` SVG div entirely.

---

### v4 — 5 Critical Bug Fixes (List Items, Menu Centering, Tooltips, Scrollbar, Padding)
**Snapshot:** `Fix-critical-bugs-in-custom-block-based-rich-text-editor_snapshot_2.md` (turn 1)

**Bug fixes:**
1. **Plus button on list items** — `positionPlusBtn()` now checks if block is `<ul>/<ol>`; if so, walks selection anchor to nearest `<li>` and uses *that* element's `getBoundingClientRect()`.
2. **Menu vertical centering** — `setButtonTop(px)` sets button `top` to `px`, and menu `top` to `px + 11` (button's vertical midpoint, since button is 22px tall). Menu's `translateY(-50%)` centers its own midpoint on that coordinate.
3. **Tooltips below** — `top: calc(100% + 7px)` + `border-bottom-color` arrow pointing up.
4. **Scrollbar restored** — `overflow-y: auto` on `.huz-editor-body`; uniform padding standardized.
5. **Uniform padding** — `16px` on top, right, and bottom; left `4px` + 38px column = balanced optical weight.

**Architecture note:** `showOnFirstLine()` now accounts for `editor.scrollTop` to keep plus button correct when scrolled.

---

### v5 — Static Frame / Internal Scroll Architecture
**Snapshot:** `Fix-critical-bugs-in-custom-block-based-rich-text-editor_snapshot_2.md` (turn 2)

**Core architectural change:**

| Layer | Default state | Active state |
|---|---|---|
| `.huz-wrap` | `height: 220px; overflow: hidden; resize: vertical` | `height: auto; overflow: visible; resize: none` |
| `.huz-editor-body` | `position: absolute; inset: 0` | `position: relative; min-height: 220px` |
| `#huz-editor` | `height: 100%; overflow-y: auto` | `height: auto; overflow: visible` |

JS `activate()` adds `.is-active` to wrapper on first focus. `isActive` flag ensures it runs exactly once per session (no blur reset in v5).

**Additional fixes:**
- Plus button measures against `.huz-plus-col` bounding rect (`getPlusColRect()`) in both layout modes
- `#huz-editor` padding: `16px` (shorthand, all sides)

---

### v6 — Static Border, Auto-Grow + Blur Reset, Uniform Padding
**Snapshot:** `Fix-critical-bugs-in-custom-block-based-rich-text-editor_snapshot_2.md` (turn 3) & `Refactor-rich-text-editor-height-focus-and-padding-behaviors_snapshot_3.md` (turn 1)

**Changes:**

**Fix 1 — Static border on focus:**
```css
.huz-wrap:focus-within {
  border-color: var(--border);  /* identical to default */
  box-shadow: none;
  outline: none;
}
```
`transition` removed from `.huz-wrap` entirely — no animation on any interaction.

**Fix 2 — Smart auto-grow with blur reset:**
- `.huz-is-active` class toggled by JS on focus/click → removes on blur
- Compact state: `overflow: hidden` + `position: absolute` body fills fixed wrapper height
- Active state: `overflow: visible`, body in normal flow, wrapper grows with content
- 100ms blur defer prevents accidental collapse when clicking menu buttons

**Fix 3 — Uniform 16px padding:**
- `#huz-editor`: `padding: 16px` on all four sides
- `.huz-plus-col`: `width: 0` — button and menu are `position: absolute`, column takes no layout space

> **Bubble note:** Set HTML element height to **"Fit content"** in Bubble's element settings.

---

### v7 — Resize Handle + Scrollbar Restored, `position: fixed` Menu, 20px Padding
**Snapshot:** `Refactor-rich-text-editor-height-focus-and-padding-behaviors_snapshot_3.md` (turn 2)

**Root cause of missing resize handle:**  
Browser only shows native resize grip when `resize: vertical` + `overflow` is **not** `visible`. In v5/v6, `overflow: hidden` on the wrapper clipped the scrollbar inside its boundary.

**Fix:** Collapse `.huz-wrap` into a direct scroll host:

| State | `.huz-wrap` | `#huz-editor` |
|---|---|---|
| Default / blur | `height: 220px; overflow: auto; resize: vertical` | `height: auto; overflow-y: visible` |
| Active (focus/click) | `height: auto; overflow: visible; resize: none` | unchanged |
| Blur reset | Revert to default (100ms defer, cancellable) | unchanged |

**Menu escape hatch:** `#huz-block-menu` changed to `position: fixed`. `setMenuPosition()` uses `getBoundingClientRect()` on `#huz-plus-btn` for viewport-relative anchoring. This prevents the wrapper's `overflow: auto` from clipping the menu.

**Other changes:**
- `#huz-editor` padding: `20px` on all four sides
- Blur defer timer cancellable via `clearTimeout(blurTimer)` in `mousedown` handlers

---

### v8 — Left Gutter Geometry, Caret Color, Cursor FOUC Fix, Font Import Removal, Focus Bug Fix
**Snapshots:** `Refactor-rich-text-editor-height-focus-and-padding-behaviors_snapshot_3.md` (turns 3–4) & `Refactor-block-editor-layout-and-cursor-styling_snapshot_4.md` (all turns)

#### Left Gutter Geometry (CSS-only)

Restored `.huz-plus-col` to `44px` wide as a true reserved gutter:

| Measurement | Value |
|---|---|
| Column width | 44 px |
| Button `left` offset | 11 px (`(44−22)/2`) |
| Button right edge | 33 px from container wall |
| Button shadow right edge | 36 px (+ 3px box-shadow) |
| `#huz-editor` `padding-left` | 5 px |
| Text cursor start | 49 px from container wall |
| **Gap (button → text)** | **16 px** ✓ |
| `#huz-block-menu` `left` | 49 px (button right edge 33px + 16px gap) |

#### Caret & Cursor Color
```css
#huz-editor {
  caret-color: #1a1a18;      /* solid dark blinking caret */
  cursor: text;               /* locks hover pointer to I-beam */
  color-scheme: light;        /* prevents OS dark-mode cursor inversion */
}
```

**Why `color-scheme: light` is needed:** When a Bubble page or OS-level dark mode sets a dark `color-scheme`, browsers invert system UI cursors — turning the black I-beam white/invisible. Pinning `color-scheme: light` on the editor element forces the OS cursor renderer to always use the dark-on-light variant.

#### FOUC Prevention for Cursor

Critical rules hoisted **above** any `@import`:
```css
/* Parsed synchronously — before any network fetches */
.huz-wrap, .huz-editor-body, #huz-editor { cursor: text; color-scheme: light; }
#huz-plus-btn, .huz-menu-btn, #huz-clear-fmt { cursor: pointer; }
```

Inline styles on markup as absolute-earliest fallback:
```html
<div class="huz-wrap" style="cursor:text;color-scheme:light;">
<div id="huz-editor" ... style="cursor:text;color-scheme:light;">
```

#### Font Import Removal
- Deleted `@import url('https://fonts.googleapis.com/...')` — Bubble already loads DM Sans globally.
- `font-family: 'DM Sans', system-ui, sans-serif` retained in all selectors (inherits pre-cached font).
- This eliminates the render-block on cursor CSS that caused the FOUC.

#### Rogue Focus Bug Fix
`document.addEventListener('click', ...)` scoped to check `wrap.contains(e.target)` — clicks outside the component no longer trigger any component logic. Previously the global listener ran on every page click.

---

## 5. Feature Reference

### 5.1 Floating Plus (+) Button

| Behavior | Implementation |
|---|---|
| Tracks active line | `positionPlusBtn(block)` reads `getBoundingClientRect()` of active block element |
| List item precision | If active block is `<ul>/<ol>`, walks to nearest `<li>` for rect |
| First-line on empty editor | `showOnFirstLine()` reads caret rect from live selection; falls back to `padding-top + lineHeight/2` |
| `+` → `×` morph | CSS `transform: rotate(45deg)` on `.open` class — no JS string swap |
| Menu open animation | `scaleX(0→1)`, `transform-origin: left center`, `cubic-bezier(0.22,1,0.36,1)` |
| Focus preservation | `mousedown` + `e.preventDefault()` on all menu interactions |

### 5.2 Block Menu

| Property | Value |
|---|---|
| Layout | `display: flex; flex-direction: row` (horizontal pill) |
| Position | `position: fixed` (escapes overflow clipping) |
| Anchoring | `setMenuPosition()` reads `#huz-plus-btn` `getBoundingClientRect()` at open-time |
| Icons | Glyph-only (`H1`, `H2`, `•`, `1.`, `❝`, `B`, `I`, `U`, `S`, etc.) |
| Tooltips | Pure CSS `::after`/`::before` reading `data-tip` attribute; appear **below** icon |
| Separators | Thin `1px` vertical `.huz-menu-sep` dividers between groups |
| Visibility trick | `visibility: hidden` (not `display: none`) so exit animation completes |
| Blur guard | Checks `e.relatedTarget` — editor blur won't close menu if focus moves into menu |

### 5.3 Supported Block Types

| Icon | Block type | Format applied |
|---|---|---|
| `¶` | Paragraph | `<p>` |
| `H1` | Heading 1 | `<h1>` |
| `H2` | Heading 2 | `<h2>` |
| `H3` | Heading 3 | `<h3>` |
| `•` | Bullet list | `<ul><li>` |
| `1.` | Numbered list | `<ol><li>` |
| `❝` | Blockquote | `<blockquote>` |
| `</>` | Code block | `<pre><code>` |
| `—` | Divider | `<hr>` |

### 5.4 Inline Formatting (inside menu)

Bold (`B`), Italic (`I`), Underline (`U`), Strikethrough (`S`), and a Clear Format button — operated via `document.execCommand` on the saved selection range.

### 5.5 Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl/Cmd + B` | Bold |
| `Ctrl/Cmd + I` | Italic |
| `Ctrl/Cmd + U` | Underline |
| `Tab` | Indent list item |
| `Enter` in heading | Inserts new `<p>` block |

---

## 6. CSS Custom Properties & Design Tokens

```css
:root {
  --accent:  #534AB7;          /* purple — used on + button glow only */
  --border:  #e5e7eb;          /* default wrapper border (never changes on focus) */
  --text:    #1a1a18;          /* primary text color */
  --bg:      #ffffff;          /* editor background */
  --t:       180ms ease;       /* general transition duration (not used on wrapper) */
  --pad:     16px;             /* base padding unit */
}
```

> **Rule:** `.huz-wrap` has **no transition** property — its border and shadow are completely static. The `--accent` color appears only on `#huz-plus-btn` box-shadow ring.

---

## 7. Layout Geometry Reference

### 7.1 Horizontal Left-Side Layout (v8 values)

```
[container wall]
|← 44px .huz-plus-col →|← #huz-editor (padding-left: 5px) →|
         |← 11px →|← 22px btn →|← 16px gap →|← text cursor
         btn left   btn width    exact gap     text start = 49px
```

### 7.2 Key Measurements

| Element | Property | Value | Rationale |
|---|---|---|---|
| `.huz-plus-col` | `width` | `44px` | True reserved gutter; breathing room around 22px button |
| `#huz-plus-btn` | `left` | `11px` | `(44−22)/2` — optically centered in column |
| `#huz-plus-btn` | `width/height` | `22px` | Touch-friendly minimum |
| `#huz-block-menu` | `left` | `49px` | Button right edge (33px) + 16px gap |
| `#huz-editor` | `padding-left` | `5px` | Text start at 49px = exactly 16px from button right edge |
| `#huz-editor` | `padding` (other sides) | `16px` | Uniform visual breathing room |
| `.huz-wrap` | `min-height` | `220px` (default) | Compact initial frame height |

### 7.3 Safe Clearance Proof

| Measurement | Value |
|---|---|
| Button right edge | `11 + 22 = 33px` from container |
| Button shadow right edge | `33 + 3 = 36px` |
| Text cursor start | `44 + 5 = 49px` |
| **Safe clearance** | **`49 − 36 = 13px`** — no character, cursor, or shadow touches text |

---

## 8. State Machine — Height & Scroll Behavior

```
Page Load
    │
    ▼
[DEFAULT STATE]
.huz-wrap: height: 220px; overflow: auto; resize: vertical
#huz-editor: height: auto; overflow-y: visible
→ Scrollbar appears natively when content > 220px
→ Native resize grip visible (overflow: auto satisfies browser requirement)
    │
    │  user clicks or focuses editor
    ▼
[ACTIVE STATE]  ← JS activateWrap()
.huz-wrap: height: auto; overflow: visible; resize: none
#huz-editor: unchanged
→ Wrapper auto-expands to fit all content
→ No internal scroll; no resize grip (height: auto + resize: none)
→ Bubble HTML element must be set to "Fit content" height
    │
    │  user blurs editor (100ms defer, cancellable)
    ▼
[DEFAULT STATE]  ← JS deactivateWrap()
```

**Blur defer mechanism:** A `blurTimer` is set on `blur` events. It is cleared by `clearTimeout(blurTimer)` in the `mousedown` handlers of `#huz-plus-btn` and `#huz-block-menu`, preventing wrapper collapse when the user clicks a menu item (which briefly removes focus from the editor).

---

## 9. Key JavaScript Functions

| Function | Purpose |
|---|---|
| `init()` | Reads `data-initial-content`, injects into `#huz-editor` if not placeholder token |
| `positionPlusBtn(block)` | Positions `+` button vertically aligned to active block (or `<li>` inside list) |
| `showOnFirstLine()` | Shows `+` button on first/empty line using caret rect; falls back to `padding-top + lineHeight/2` |
| `getActiveBlock()` | Returns the top-level block element containing the cursor |
| `setButtonTop(px)` | Sets button `top: px`; sets menu `top: px + 11` (button midpoint for `translateY(-50%)` centering) |
| `setMenuPosition()` | Reads `getBoundingClientRect()` from `#huz-plus-btn`; positions `#huz-block-menu` in fixed viewport coords |
| `openMenu()` / `closeMenu()` | Toggle menu open/close; morph `+`→`×` via `.open` class |
| `applyBlock(type)` | Applies selected block format to active line; restores cursor focus |
| `activateWrap()` | Switches wrapper to `height: auto; overflow: visible; resize: none` |
| `deactivateWrap()` | Restores wrapper to `height: 220px; overflow: auto; resize: vertical` (100ms defer) |
| `saveDebounced()` | Debounced 500ms call to `bubble_fn_saveEditorData(html)` |
| `saveFn()` | Immediate (non-debounced) save — called on `blur` |

---

## 10. Bubble.io Integration

### 10.1 Data Sync

```javascript
// Called automatically on every input (debounced 500ms) and immediately on blur
if (typeof bubble_fn_saveEditorData === 'function') {
  bubble_fn_saveEditorData(activeContent); // activeContent = editor.innerHTML
}
```

Configure in Bubble Toolbox plugin: expose a function named `saveEditorData`.

### 10.2 Initial Content Hydration

```html
<!-- Replace #BUBBLE_DYNAMIC_DATA# with your Bubble dynamic expression -->
<div id="huz-editor"
     data-initial-content="[Current page Note's Body]"
     contenteditable="true">
</div>
```

In `init()`:
```javascript
const raw = editor.getAttribute('data-initial-content');
if (raw && raw !== '#BUBBLE_DYNAMIC_DATA#') {
  editor.innerHTML = raw; // safe injection — Bubble renders server-side
}
```

### 10.3 Bubble Element Settings

| Setting | Required value |
|---|---|
| Element height | **Fit content** (so Bubble container grows with auto-expanding editor) |
| Toolbox plugin | Must be installed and `saveEditorData` function exposed |
| HTML Element | Paste entire single-file snippet |

---

## 11. Known Bug Fixes & Root Cause Log

| # | Bug | Root Cause | Fix Applied (version) |
|---|---|---|---|
| 1 | `+` button invisible on first empty line | `getActiveBlock()` returned `null` for empty editor; code treated `null` as "hide button" | `showOnFirstLine()` + focus/click `setTimeout(..., 0)` (v3) |
| 2 | `+` button centers on full `<ul>/<ol>` not active `<li>` | `getBoundingClientRect()` called on parent list, not current list item | Walk selection anchor to nearest `<li>` (v4) |
| 3 | Menu opens at top edge of button, not centered | Menu `top` set to same value as button `top` | `setButtonTop` feeds `px + 11` to menu; menu uses `translateY(-50%)` (v4) |
| 4 | Tooltips appearing above icons | CSS `top` / `bottom` placement inverted | `top: calc(100% + 7px)` + bottom-arrow (v4) |
| 5 | Scrollbar disappeared / resize handle missing | `overflow: hidden` on wrapper clipped scrollbar; resize grip hidden by child elements | Wrapper owns both `overflow: auto` and `resize: vertical` (v7) |
| 6 | Block menu clipped when wrapper has `overflow: auto` | Absolutely-positioned menu clipped by overflow context | Menu changed to `position: fixed`; `setMenuPosition()` uses viewport coords (v7) |
| 7 | Wrapper border glows/changes on focus | `:focus-within` rule applied accent color + box-shadow | Explicit static override in `:focus-within`; removed `transition` (v6) |
| 8 | Caret invisible (white) while typing | `caret-color` inherited incorrect color | `caret-color: #1a1a18` on `#huz-editor` (v8) |
| 9 | I-beam cursor white/invisible on hover | OS dark-mode `color-scheme` inverted system cursor | `cursor: text; color-scheme: light` on editor (v8) |
| 10 | Cursor FOUC — I-beam slow to appear on page load | Cursor CSS rules written after `@import` were render-blocked by font network fetch | Hoisted cursor rules above `@import`; added inline `style` attributes; removed `@import` entirely (v8) |
| 11 | Clicking outside editor steals focus | `document.addEventListener('click', ...)` was globally unscoped | Scoped to `wrap.contains(e.target)` check (v8) |
| 12 | Text overlaps `+` button / menu | Left gutter too narrow; editor left padding too small | `.huz-plus-col: 44px`, button `left: 11px`, editor `padding-left: 5px` → 16px safe gap (v8) |
| 13 | Double resize handle in bottom-right corner | `overflow: auto` caused scrollbar track to visually overlap native resize grip | Use `overflow: auto` only on wrapper (not inner `#huz-editor`); removed `.huz-resize-hint` SVG (v3/v7) |

---

## 12. Conventions & Code Style

- All CSS class names are prefixed with `huz-` to avoid conflicts with Bubble's global stylesheet.
- All IDs are prefixed with `huz-`.
- JavaScript is entirely vanilla — no external libraries.
- `execCommand` is used for inline formatting (Bold, Italic, Underline, Strikethrough). While deprecated in spec, it remains the most reliable cross-browser method in `contenteditable` contexts as of 2026.
- `mousedown` + `e.preventDefault()` is used (not `click`) for all menu button interactions. This prevents the editor from losing focus before the saved selection range is restored.
- The save function is debounced at **500ms** to prevent excessive Bubble workflow triggers during fast typing.
- Version numbers are tracked in a comment block at the top of the source file.

---

## 13. Bubble Setup Checklist

- [ ] Paste the complete single-file snippet into a Bubble **HTML Element**
- [ ] Set the HTML Element's height to **"Fit content"** in Bubble element settings
- [ ] Install the **Toolbox plugin** in Bubble
- [ ] Create a Bubble workflow exposing a function named `saveEditorData`
- [ ] Replace `data-initial-content="#BUBBLE_DYNAMIC_DATA#"` with your Bubble dynamic expression (e.g. `[Current page Note's Body]`)
- [ ] Verify DM Sans is loaded globally in Bubble (prevents font flash)
- [ ] Test content hydration on page load
- [ ] Test save trigger fires on input and blur

---

*This file is the canonical knowledge base for the rich-text-editor project. All implementation decisions, architectural choices, and bug resolutions should be recorded here.*
