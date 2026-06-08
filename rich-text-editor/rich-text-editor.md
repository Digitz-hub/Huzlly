# Rich Text Editor — Project Documentation

> **Single source of truth** for the custom Notion-like block-based rich text editor built for Bubble.io.  
> Last updated: June 8, 2026 · Current source file: **huzlly_rich_editor_v7.html**

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
9. [Key JavaScript Functions & State Variables](#9-key-javascript-functions--state-variables)
10. [Bubble.io Integration](#10-bubbleio-integration)
11. [Known Bug Fixes & Root Cause Log](#11-known-bug-fixes--root-cause-log)
12. [Conventions & Code Style](#12-conventions--code-style)
13. [Bubble Setup Checklist](#13-bubble-setup-checklist)

---

## 1. Project Overview

A **self-contained, single-file** HTML/CSS/JS snippet (`huzlly_rich_editor_v7.html`) intended to be pasted directly into a Bubble.io **HTML Element**. It functions as a Notion-inspired block-based rich text editor with:

- A floating **Plus (+) button** that tracks the active cursor line and opens a contextual block-formatting menu.
- A **horizontal icon-only toolbar** (expandable from the `+` button) with CSS-only tooltips.
- Full **Bubble.io Toolbox plugin** integration for real-time data sync.
- **Secure initial content hydration** from Bubble dynamic data via a `data-initial-content` attribute.
- **Borderless appearance** in all states — no wrapper border, no focus glow.
- **Compact left gutter**: plus button sits flush to the left edge; 10px gaps separate it from both the block menu and the text canvas.

---

## 2. Technology Stack & Constraints

| Concern | Decision |
|---|---|
| Framework | Vanilla JavaScript only — no React, Vue, or heavy frameworks |
| Styling | Plain CSS with custom properties; no preprocessors |
| Font | `'DM Sans', system-ui, sans-serif` via `--font-body`; Google Fonts `@import` is present but can be removed if Bubble loads DM Sans globally |
| Monospace font | `'JetBrains Mono', 'Fira Mono', monospace` via `--font-mono` (used in `<code>` and `<pre>` blocks) |
| Editor core | `<div contenteditable="true">` |
| Bubble sync | Toolbox plugin — `bubble_fn_saveEditorData(html)` |
| External deps | None (Google Fonts import only) |
| Target environment | Bubble.io HTML Element (iframe sandbox) |
| Accessibility | `aria-label`, `aria-expanded`, `role="toolbar"` on interactive elements |

---

## 3. Architecture

### 3.1 HTML Structure

```
.huz-wrap  #huz-wrap                ← outer frame (owns overflow + resize handle; no border)
  └── .huz-editor-body              ← inner flex row (display: flex)
        ├── .huz-plus-col           ← left gutter column (22px wide, flex-shrink: 0)
        │     ├── #huz-plus-btn     ← floating + / × toggle (position: absolute, left: 0)
        │     └── #huz-block-menu   ← horizontal toolbar pill (position: absolute, left: 32px)
        └── #huz-editor             ← contenteditable div (the typing canvas)
```

### 3.2 Scroll / Resize Model

The wrapper (`.huz-wrap`) owns both `resize: vertical` and `overflow`. This is the only correct way to show a native browser resize grip alongside a scrollbar — both must live on the same element.

```
Default state (page load / blur):
  .huz-wrap  → border: none; overflow-x: hidden; overflow-y: auto; resize: vertical; min-height: 180px
  #huz-editor → height: auto; min-height: 180px (grows to content, wrapper clips/scrolls)

Active state (focus — CSS :focus-within driven, no JS class toggle):
  .huz-wrap  → :focus-within applies NO additional border or glow (borderless in all states)
  Layout:     same overflow/resize — wrapper still scrolls/clips if needed
```

> **Note on overflow-x/overflow-y split:** `overflow-x: hidden` suppresses horizontal scroll. `overflow-y: auto` enables the vertical scrollbar. Browsers only remove the native resize grip when `overflow` is `hidden` on **both** axes — so pairing `hidden`/`auto` keeps the resize handle alive.

### 3.3 Block Menu Positioning

`#huz-block-menu` uses `position: absolute` within `.huz-plus-col`. It is anchored at `left: 32px` (fixed in CSS — button right edge 22px + 10px gap) and its `top` is set dynamically by `setButtonTop()`. The menu uses `translateY(-50%)` in CSS so its vertical midpoint aligns with the `+` button's center.

### 3.4 Cursor Rendering Pipeline (FOUC prevention)

To prevent Flash of Unstyled Content on cursor color:

1. Critical cursor CSS is placed **above** the `@import` in `<style>` — parsed synchronously before any network fetch starts.
2. `style="cursor:text;color-scheme:light;"` is inlined on `.huz-wrap` and `#huz-editor` — applied by the HTML parser before the CSS engine runs.
3. If Bubble loads DM Sans globally, the `@import` line can be deleted entirely to eliminate the render-block dependency.

---

## 4. Version History & Changelog

### v1 — Initial Build

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

**Changes:**
- Removed word/char count footer (`.huz-footer`, `#huz-wc`, `#huz-cc`, `updateCount()`)
- Removed fixed top toolbar (`.huz-toolbar`)
- Merged all formatting options into the floating `+` menu
- `+` morphs to `×` via CSS `transform: rotate(45deg)` on `.open` class (no JS string swap)
- Menu animates left→right: `transform-origin: left center` + `scaleX(0→1)` with `cubic-bezier(0.22,1,0.36,1)`
- Menu uses `visibility: hidden` (not `display: none`) so exit animation plays fully
- All menu interactions use `mousedown` + `e.preventDefault()` to preserve editor focus
- Blur guard: editor blur won't close menu if focus moves into the menu (`e.relatedTarget` check)

---

### v3 — Horizontal Icon Menu, Plus Button First-Line Fix, Resize Handle Deduplication

**Bug fixes:**
1. **Plus button on first line** — `positionPlusBtn(null)` checks `document.activeElement === editor` before hiding; calls `showOnFirstLine()` if focused. `focus` and `click` listeners use `setTimeout(..., 0)` to let the browser commit cursor position first.
2. **Horizontal icon-only menu** — `#huz-block-menu` changed to `flex-direction: row`. Buttons show only glyphs. Tooltips are pure CSS via `::after`/`::before` reading `data-tip` attribute.
3. **Double resize handle** — Removed `.huz-resize-hint` SVG div entirely; adjusted overflow strategy.

---

### v4 — 5 Critical Bug Fixes (List Items, Menu Centering, Tooltips, Scrollbar, Padding)

**Bug fixes:**
1. **Plus button on list items** — `positionPlusBtn()` checks if block is `<ul>/<ol>`; walks selection anchor to nearest `<li>` and uses that element's `getBoundingClientRect()`.
2. **Menu vertical centering** — `setButtonTop(px)` sets button `top` to `px`, menu `top` to `px + 11` (button midpoint). Menu's `translateY(-50%)` aligns its center on that coordinate.
3. **Tooltips below icons** — `top: calc(100% + 7px)` + `border-bottom-color` arrow pointing up.
4. **Scrollbar** — `overflow-y: auto` strategy; padding standardized.
5. **Padding** — `16px` on top/right/bottom; left accounts for gutter column.

`showOnFirstLine()` updated to include `editor.scrollTop` offset.

---

### v5 — Static Frame / Internal Scroll Architecture

**Core architectural change:** `.huz-wrap` became a fixed-height frame (`height: 220px; overflow: hidden`) with `.huz-editor-body` using `position: absolute; inset: 0`. JS `activate()` toggled `.is-active` on first focus to switch to `height: auto`. One-way — no blur reset.

---

### v6 — Static Border Attempt, Auto-Grow + Blur Reset, Uniform Padding

**Changes:**
- Attempted removal of focus border glow from `.huz-wrap:focus-within`
- Smart auto-grow with blur reset via `.huz-is-active` JS class
- `#huz-editor` padding: `16px` all sides
- `.huz-plus-col` briefly set to `width: 0`

---

### v7 — Current Source (`huzlly_rich_editor_v7.html`)

**This is the version currently in the source file.**

**Core architectural change — wrapper as direct scroll host:**

The root problem in v5/v6 was that `overflow: hidden` on the wrapper clipped the scrollbar inside its boundary, and `overflow: visible` prevented the browser from showing the native resize grip. The v7 solution:

- `.huz-wrap` uses `overflow-x: hidden` + `overflow-y: auto` (not `hidden` on both axes → resize grip stays visible)
- `.huz-editor-body` is normal flow (`display: flex`, `min-height: inherit`)
- `#huz-editor` grows naturally to content height; wrapper clips/scrolls when content exceeds `min-height: 180px`
- Height state is managed entirely by CSS `:focus-within` (border/glow) — **removed in the UI update below**; no JS class toggling for height

**UI update applied to v7 (June 8, 2026):**

Two changes were applied as a UI-only update without a version increment:

1. **Borderless wrapper** — `.huz-wrap` `border` set to `none`. `.huz-wrap:focus-within` border/glow block removed. No border or box-shadow in any state.

2. **Compact left gutter geometry** — Plus button moved flush to left wall (`left: 0`), column width reduced to match button width (`22px`), menu and text start moved left with 10px gaps from button right edge:

| Property | Old value | New value | Reason |
|---|---|---|---|
| `.huz-plus-col` `width` | `44px` | `22px` | Column trimmed to exact button width; no dead space |
| `#huz-plus-btn` `left` | `11px` | `0` | Button flush to left wall of column |
| `#huz-block-menu` `left` | `49px` | `32px` | Button right edge (22px) + 10px gap |
| `#huz-editor` `padding-left` | `5px` | `10px` | Col width (22px) + 10px pad = 32px text start, 10px clear of button |
| `.huz-wrap` `border` | `1.5px solid var(--border)` | `none` | Borderless design requirement |
| `.huz-wrap` `transition` | `border-color var(--t), box-shadow var(--t)` | _(removed)_ | No border or shadow to animate |
| `.huz-wrap:focus-within` block | Present (accent border + glow) | _(removed entirely)_ | Borderless in all states |
| `#huz-plus-btn.open` `border-color` | `var(--plus-active)` (purple) | `#1a1a18` (black) | No purple in open/active state |
| `#huz-plus-btn.open` `color` | `var(--plus-active)` (purple) | `#1a1a18` (black) | No purple in open/active state |
| `#huz-plus-btn.open` `box-shadow` | `0 0 0 3px var(--accent-soft)` | _(removed)_ | No glow in any button state |
| `#huz-plus-btn:hover` `border-color` | `var(--plus-active)` (purple) | `#1a1a18` (black) | No purple on hover |
| `#huz-plus-btn:hover` `color` | `var(--plus-active)` (purple) | `#1a1a18` (black) | No purple on hover |
| `#huz-plus-btn:hover` `box-shadow` | `0 0 0 3px var(--accent-soft)` | _(removed)_ | No glow on hover |
| `#huz-plus-btn` `transition` | includes `box-shadow 0.15s ease` | _(removed from list)_ | No box-shadow to animate |

**`min-height` set to `180px`** (not 220px as in earlier experimental versions).

**What v7 did NOT implement from the v8 roadmap:**
- `@import` for Google Fonts is still present
- `document.addEventListener('click', ...)` is not scoped to `wrap.contains(e.target)`

---

## 5. Feature Reference

### 5.1 Floating Plus (+) Button

| Behavior | Implementation |
|---|---|
| Tracks active line | `positionPlusBtn(block)` reads `getBoundingClientRect()` of active block element |
| List item precision | If active block is `<ul>/<ol>`, walks selection anchor to nearest `<li>` for rect |
| First-line on empty editor | `showOnFirstLine()` reads caret rect from live selection; falls back to `padding-top + lineHeight/2` |
| Scroll compensation | Both `positionPlusBtn` and `showOnFirstLine` add `editor.scrollTop` to `top` offset |
| `+` → `×` morph | CSS `transform: rotate(45deg)` on `.open` class |
| Hover styling | `border-color: #1a1a18; color: #1a1a18` — black, no glow |
| Active/open styling | `border-color: #1a1a18; color: #1a1a18` — black, no glow |
| Menu open animation | `scaleX(0→1)`, `transform-origin: left center`, `cubic-bezier(0.22,1,0.36,1)` |
| Focus preservation | `mousedown` + `e.preventDefault()` on all menu interactions |
| Hidden by default | `opacity: 0; pointer-events: none` — `.visible` class enables it |
| Accessibility | `aria-label="Insert block"`, `aria-expanded` toggled on open/close |

### 5.2 Block Menu

| Property | Value |
|---|---|
| Layout | `display: flex; flex-direction: row` (horizontal pill) |
| Position | `position: absolute` within `.huz-plus-col`; `left: 32px` fixed in CSS |
| Vertical anchor | `top` set dynamically by `setButtonTop()`; `translateY(-50%)` centers it on button |
| Icons | Glyph-only (`H1`, `H2`, `H3`, `•`, `1.`, `❝`, `</>`，`—`, `B`, `I`, `U`, `S`, `⬅`, `⬌`, `➡`, `⌫`) |
| Tooltips | Pure CSS `::after`/`::before` reading `data-tip` attribute; appear **below** icon |
| Separators | Thin `1px` vertical `.huz-menu-sep` dividers between groups |
| Visibility trick | `visibility: hidden` (not `display: none`) so exit animation completes |
| Blur guard | `e.relatedTarget` check — editor blur won't close menu if focus moves into menu |
| Accessibility | `role="toolbar"`, `aria-label="Block & format options"` |

### 5.3 Supported Block Types

| Glyph | Block type | HTML produced |
|---|---|---|
| `H1` | Heading 1 | `<h1>` |
| `H2` | Heading 2 | `<h2>` |
| `H3` | Heading 3 | `<h3>` |
| `•` | Bullet list | `<ul><li>` |
| `1.` | Numbered list | `<ol><li>` |
| `❝` | Blockquote | `<blockquote>` |
| `</>` | Code block | `<pre>` |
| `—` | Divider | `<hr>` + new `<p>` after |

> **Default block:** `<p>` is the browser's default for Enter key in `contenteditable`. It is not a menu option but is produced by `Enter` in headings (explicit) and naturally by the browser elsewhere.

### 5.4 Inline Formatting

Operated via `document.execCommand` on the restored `savedRange`. All commands keep the menu open so the user can chain multiple formats.

| Glyph | Command | `data-cmd` value |
|---|---|---|
| **B** | Bold | `bold` |
| *I* | Italic | `italic` |
| U | Underline | `underline` |
| ~~S~~ | Strikethrough | `strikeThrough` |
| ⬅ | Align left | `justifyLeft` |
| ⬌ | Center | `justifyCenter` |
| ➡ | Align right | `justifyRight` |
| ⌫ | Clear formatting | `removeFormat` (via `#huz-clear-fmt`) |

Active state: bold/italic/underline/strikeThrough buttons gain `.active` class via `updateInlineBtnStates()` on menu open and after each command.

### 5.5 Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl/Cmd + B` | Bold |
| `Ctrl/Cmd + I` | Italic |
| `Ctrl/Cmd + U` | Underline |
| `Tab` | Indent (list) |
| `Shift + Tab` | Outdent (list) |
| `Enter` in H1/H2/H3 | Inserts new `<p>` after heading, places cursor |
| `Escape` | Closes block menu |

---

## 6. CSS Custom Properties & Design Tokens

All defined in `:root`:

```css
:root {
  --bg:          #ffffff;          /* editor + wrapper background */
  --surface:     #f7f7f5;          /* inline code background */
  --border:      #e8e8e4;          /* menu border, separators (NOT wrapper — wrapper is borderless) */
  --text:        #1a1a18;          /* primary text color; also tooltip bg */
  --text-muted:  #9b9b97;          /* placeholder, blockquote, strikethrough */
  --accent:      #534AB7;          /* purple — active btn, blockquote bar */
  --accent-soft: #eeecfb;          /* light purple — hover/active bg */
  --plus-idle:   #c7c7c2;          /* + button default border + color */
  --plus-active: #534AB7;          /* + button active/hover border + color */
  --menu-shadow: 0 6px 28px rgba(0,0,0,0.10), 0 1.5px 5px rgba(0,0,0,0.06);
  --tooltip-bg:  #1a1a18;          /* tooltip background */
  --radius:      8px;              /* wrapper border-radius */
  --font-body:   'DM Sans', system-ui, sans-serif;
  --font-mono:   'JetBrains Mono', 'Fira Mono', monospace;  /* code/pre blocks */
  --t:           0.15s ease;       /* standard transition for interactive controls */
}
```

> **Note:** `--border` and `--accent` are still present in `:root` but `--border` is no longer used on `.huz-wrap` — the wrapper has `border: none`. `--accent` is no longer used for a focus ring — the `:focus-within` glow block was removed. Both tokens remain available for block-menu border and blockquote styling.

---

## 7. Layout Geometry Reference

### 7.1 Horizontal Left-Side Layout (current v7 values)

```
[wrapper left wall]
|←── 22px .huz-plus-col ──→|←── #huz-editor ──────────────────────────→|
|←── 22px btn ────────────→|    padding-left: 10px
                             ↑
                            32px from wall = text cursor start

Gap from button right edge to text start:
  Button right edge = 0 + 22 = 22px
  Text start        = 22 + 10 = 32px
  Gap               = 32 - 22 = 10px  ✓

Gap from button right edge to menu left edge:
  Button right edge = 22px
  Menu left         = 32px  (set in CSS: left: 32px)
  Gap               = 32 - 22 = 10px  ✓

Menu centered = translateY(-50%) aligns menu midpoint to button midpoint
```

### 7.2 Key Measurements

| Element | Property | Value | Rationale |
|---|---|---|---|
| `.huz-plus-col` | `width` | `22px` | Matches button width exactly; no dead space in gutter |
| `#huz-plus-btn` | `left` | `0` | Flush to left wall of column |
| `#huz-plus-btn` | `width/height` | `22px` | Touch-friendly minimum |
| `#huz-block-menu` | `left` | `32px` | Button right edge (22px) + 10px gap |
| `#huz-editor` | `padding` | `16px 16px 16px 10px` | Left 10px → text starts 32px from wall, 10px from button |
| `.huz-wrap` | `min-height` | `180px` | Compact initial frame height |
| `#huz-editor` | `min-height` | `180px` | Matches wrapper; ensures body always fills wrapper |
| `.huz-wrap` | `border` | `none` | Borderless design — no idle or focus border |

### 7.3 Safe Clearance Proof

| Measurement | Value |
|---|---|
| Button right edge | `0 + 22 = 22px` from container wall |
| Button open shadow right edge | `22 + 3 = 25px` (3px box-shadow ring) |
| Text cursor start | `22 + 10 = 32px` from container wall |
| **Safe clearance** | **`32 − 25 = 7px`** — no character or shadow ever touches text |

---

## 8. State Machine — Height & Scroll Behavior

In v7, height state is driven by CSS alone (no JS class toggling for height). The wrapper is borderless in **all** states — there is no visual state change on focus or blur.

```
Page Load / Blur state:
  .huz-wrap → border: none; min-height: 180px; overflow-x: hidden; overflow-y: auto; resize: vertical
  #huz-editor → min-height: 180px; height: auto (grows to content)
  → If content exceeds 180px: native vertical scrollbar appears on .huz-wrap
  → Native resize grip visible (overflow-y: auto satisfies browser requirement)
  → No border, no glow — wrapper is visually transparent

    │  user focuses editor
    ▼

Active / Focused state (CSS :focus-within — no JS involved):
  .huz-wrap:focus-within → NO additional styling (border/glow block removed)
  → Layout/overflow/resize unchanged — wrapper still scrolls if content overflows
  → Editor continues to grow naturally (no fixed height cap while focused)
  → Visually identical to blur state — borderless throughout

    │  user blurs editor
    ▼

Page Load / Blur state (CSS reverts immediately — no blur timer, no defer)
```

> **v7 behavior note:** There is no JS-driven blur defer or `blurTimer` in the current code. The `+` button hides immediately on blur (unless focus moves to the menu or plus button — `e.relatedTarget` guard). Height/overflow never change. There is no focus ring or border change of any kind — the wrapper appears identical in idle and active states.

---

## 9. Key JavaScript Functions & State Variables

### 9.1 State Variables

| Variable | Type | Purpose |
|---|---|---|
| `activeBlock` | `Element \| null` | Currently active top-level block under the cursor |
| `menuOpen` | `boolean` | Whether `#huz-block-menu` is open |
| `savedRange` | `Range \| null` | Selection range saved on menu open; restored before `execCommand` |
| `saveTimer` | `number \| null` | Debounce timer ID for `syncToBubble()` |

### 9.2 Functions

| Function | Purpose |
|---|---|
| `init()` | Reads `data-initial-content`; injects into `#huz-editor` if not placeholder token and not empty |
| `syncToBubble()` | Debounced 500ms call to `bubble_fn_saveEditorData(editor.innerHTML)` |
| `syncNow()` | Immediate (non-debounced) save — called on `blur` |
| `getActiveBlock()` | Walks selection anchor up to nearest direct child of `#huz-editor`; returns `null` if not inside editor |
| `positionPlusBtn(block)` | Positions `+` button aligned to active block; resolves to `<li>` rect if inside a list; calls `showOnFirstLine()` if editor is focused but `block` is null |
| `showOnFirstLine()` | Positions `+` button using live caret `getBoundingClientRect()`; falls back to `padding-top + lineHeight/2` |
| `setButtonTop(px)` | Sets `plusBtn.style.top = px + 'px'`; sets `blockMenu.style.top = (px + 11) + 'px'` (button's vertical midpoint, for `translateY(-50%)` centering) |
| `openMenu()` | Saves current selection to `savedRange`; adds `.open` to menu and button; calls `updateInlineBtnStates()` |
| `closeMenu()` | Removes `.open` from menu and button; sets `aria-expanded` false |
| `restoreRange()` | Restores `savedRange` into the live selection (called before every `execCommand`) |
| `insertOrConvertBlock(type)` | Focuses editor; restores range; inserts or replaces the active block with the chosen type; calls `syncToBubble()` |
| `insertAfter(node, ref)` | DOM helper — inserts `node` after `ref` inside `#huz-editor`; appends if no ref |
| `placeCursorIn(el)` | Places collapsed selection at first text node inside `el` |
| `updateInlineBtnStates()` | Reflects `bold`/`italic`/`underline`/`strikeThrough` command state onto `.active` class of inline buttons |
| `onCursorMove()` | Shared handler for `keyup` and `mouseup` — calls `getActiveBlock()` + `positionPlusBtn()` |

---

## 10. Bubble.io Integration

### 10.1 Data Sync

```javascript
// Debounced (500ms) — fires on every input event
syncToBubble() → bubble_fn_saveEditorData(editor.innerHTML)

// Immediate — fires on blur
syncNow() → bubble_fn_saveEditorData(editor.innerHTML)
```

Configure in Bubble Toolbox plugin: expose a function named `saveEditorData`.

### 10.2 Initial Content Hydration

```html
<!-- Replace #BUBBLE_DYNAMIC_DATA# with your Bubble dynamic expression -->
<div
  id="huz-editor"
  contenteditable="true"
  spellcheck="true"
  data-placeholder="Write something… or click + to insert a block"
  data-initial-content="[Current page Note's Body]"
  style="cursor:text;color-scheme:light;"
></div>
```

`init()` logic:
```javascript
const raw = editor.getAttribute('data-initial-content') || '';
if (raw && raw !== '#BUBBLE_DYNAMIC_DATA#' && raw.trim() !== '') {
  editor.innerHTML = raw; // safe — Bubble renders server-side before the HTML reaches the browser
}
```

### 10.3 Bubble Element Settings

| Setting | Required value |
|---|---|
| Element height | **Fixed or Fit content** — wrapper has its own scroll; fixed height works fine |
| Toolbox plugin | Must be installed; `saveEditorData` function must be exposed |
| HTML Element | Paste the entire `huzlly_rich_editor_v7.html` content |

---

## 11. Known Bug Fixes & Root Cause Log

| # | Bug | Root Cause | Fix applied (version) |
|---|---|---|---|
| 1 | `+` button invisible on first empty line | `getActiveBlock()` returned `null` for empty editor; code treated `null` as "hide button" | `showOnFirstLine()` + `setTimeout(..., 0)` on focus/click (v3) |
| 2 | `+` button centers on full `<ul>/<ol>` not active `<li>` | `getBoundingClientRect()` called on parent list | Walk selection anchor to nearest `<li>` (v4) |
| 3 | Menu opens at top edge of button, not centered | Menu `top` set to same px as button `top` | `setButtonTop()` feeds `px + 11` to menu; `translateY(-50%)` in CSS (v4) |
| 4 | Tooltips appearing above icons | CSS placement inverted | `top: calc(100% + 7px)` + upward-pointing arrow (`border-bottom-color`) (v4) |
| 5 | Scrollbar disappeared / resize handle missing | `overflow: hidden` on both axes removes native grip; clipped scrollbar invisible | `overflow-x: hidden` + `overflow-y: auto` on `.huz-wrap` (v7) |
| 6 | Block menu clipped by wrapper overflow | Absolutely-positioned menu subject to overflow clipping | `left: 32px` fixed CSS anchor; menu sized to not exceed wrapper width (v7 UI update) |
| 7 | Caret invisible (white) while typing | `caret-color` not explicitly set | `caret-color: #1a1a18` on `#huz-editor` (v7) |
| 8 | I-beam cursor white/invisible on hover | OS dark-mode `color-scheme` inverted system cursor | `cursor: text; color-scheme: light` on editor (v7) |
| 9 | Cursor FOUC — I-beam slow to appear on page load | Cursor CSS written after `@import` render-blocked by font fetch | Hoisted cursor rules above `@import`; inline `style` attributes on markup (v7) |
| 10 | Text overlaps `+` button / menu | Left gutter undersized; editor left padding too small | `.huz-plus-col: 22px`, button `left: 0`, `#huz-editor` `padding-left: 10px` → 10px gap (v7 UI update) |
| 11 | Double resize handle in bottom-right | SVG resize hint overlapping native grip | Removed `.huz-resize-hint` SVG entirely (v3); finalized with `overflow-x/y` split (v7) |
| 12 | Word/char count footer cluttering UI | Initial feature included by default | Removed entirely — `.huz-footer`, `updateCount()` deleted (v2) |
| 13 | Fixed top toolbar blocking content | Always-visible toolbar wasted space | Removed; merged into floating `+` menu (v2) |

---

## 12. Conventions & Code Style

- All CSS class names and IDs are prefixed with `huz-` to avoid conflicts with Bubble's global stylesheet.
- JavaScript is entirely vanilla — no external libraries, no `import` statements.
- `document.execCommand` is used for inline formatting (Bold, Italic, Underline, Strikethrough, Align, Clear). While deprecated in spec, it remains the most reliable cross-browser approach for `contenteditable` contexts.
- `mousedown` + `e.preventDefault()` is used (not `click`) for all menu button interactions. This preserves the editor's focus and live selection before `restoreRange()` is called.
- The save function is debounced at **500ms** to prevent excessive Bubble workflow triggers during fast typing. `syncNow()` fires immediately on `blur`.
- The entire JS is wrapped in an IIFE (`(function () { 'use strict'; ... })()`) to avoid polluting global scope inside Bubble's iframe.
- Version numbers are tracked in the comment header block at the top of the source file.
- `aria-label`, `aria-expanded`, and `role` attributes are maintained on interactive elements for basic accessibility.

---

## 13. Bubble Setup Checklist

- [ ] Paste the entire content of `huzlly_rich_editor_v7.html` into a Bubble **HTML Element**
- [ ] Install the **Toolbox plugin** in Bubble
- [ ] Create a Bubble workflow exposing a function named `saveEditorData`
- [ ] Replace `data-initial-content="#BUBBLE_DYNAMIC_DATA#"` with your Bubble dynamic expression (e.g. `[Current page Note's Body]`)
- [ ] **Optional performance fix:** If Bubble loads DM Sans globally, delete the `@import url('https://fonts.googleapis.com/...')` line from the `<style>` block — this eliminates the render-blocking font fetch that can delay cursor CSS
- [ ] Test content hydration on page load (pre-filled content should appear immediately)
- [ ] Test save trigger fires on both input (debounced) and blur (immediate)
- [ ] Test vertical resize handle is visible and draggable in the bottom-right corner
- [ ] Test scrollbar appears when typing content that exceeds the editor's current height
- [ ] Verify wrapper is fully borderless in both idle and focused states
- [ ] Verify plus button appears flush to the left edge of the editor with 10px gap to text and menu

---

*This file is the canonical knowledge base for the rich-text-editor project. All implementation decisions, architectural choices, and bug resolutions should be recorded here. The source of truth for actual code is `huzlly_rich_editor_v7.html`.*
