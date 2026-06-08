# Rich Text Editor — Project Documentation

> **Single source of truth** for the custom Notion-like block-based rich text editor built for Bubble.io.  
> Last updated: June 8, 2026 · Current source file: **huzlly_rich_editor_v8.html**

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

A **self-contained, single-file** HTML/CSS/JS snippet (`huzlly_rich_editor_v8.html`) intended to be pasted directly into a Bubble.io **HTML Element**. It functions as a Notion-inspired block-based rich text editor with:

- A floating **Plus (+) button** that tracks the active cursor line and opens a contextual block-formatting menu.
- A **horizontal block-type-only toolbar** (expandable from the `+` button) with CSS-only tooltips and a staggered left→right reveal animation. Inline formatting buttons have been fully removed from this menu.
- A **contextual floating Inline Formatting Popover** (`#huz-inline-toolbar`) that appears dynamically above any non-collapsed text selection within the editor and hosts all inline formatting controls (Bold, Italic, Underline, Strikethrough, Clear Formatting).
- Full **Bubble.io Toolbox plugin** integration for real-time data sync.
- **Secure initial content hydration** from Bubble dynamic data via a `data-initial-content` attribute.
- **Borderless appearance** in all states — no wrapper border, no focus glow.
- **Compact left gutter**: plus button sits at 3px from left edge; 6px gap separates it from the block menu (menu `left: 38px`); 10px separates the menu from the text canvas.

---

## 2. Technology Stack & Constraints

| Concern | Decision |
|---|---|
| Framework | Vanilla JavaScript only — no React, Vue, or heavy frameworks |
| Styling | Plain CSS with custom properties; no preprocessors |
| Font | `'DM Sans', system-ui, sans-serif` via `--font-body`; Google Fonts `@import` is present but can be removed if Bubble loads DM Sans globally |
| Monospace font | `'JetBrains Mono', 'Fira Mono', monospace` via `--font-mono` (used in `<code>` blocks) |
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
        │     ├── #huz-plus-btn     ← floating + / × toggle (position: absolute, left: 3px)
        │     └── #huz-block-menu   ← horizontal BLOCK-ONLY toolbar (position: absolute, left: 38px)
        └── #huz-editor             ← contenteditable div (the typing canvas)

#huz-inline-toolbar                 ← floating selection popover (position: fixed, outside .huz-wrap)
```

`#huz-inline-toolbar` is rendered **outside** `.huz-wrap` in the DOM so that `position: fixed` coordinates are resolved against the viewport (not a positioned ancestor), preventing any clipping from `.huz-wrap`'s `overflow` property.

### 3.2 Scroll / Resize Model

The wrapper (`.huz-wrap`) owns both `resize: vertical` and `overflow`. This is the only correct way to show a native browser resize grip alongside a scrollbar — both must live on the same element.

```
Default state (page load / blur):
  .huz-wrap  → border: none; overflow-x: hidden; overflow-y: auto; resize: vertical; min-height: 180px
  #huz-editor → height: auto; min-height: 180px (grows to content, wrapper clips/scrolls)

Active state (focus):
  .huz-wrap  → :focus-within applies NO additional border or glow (borderless in all states)
  Layout:     same overflow/resize — wrapper still scrolls/clips if needed
```

### 3.3 Block Menu Positioning

`#huz-block-menu` uses `position: absolute` within `.huz-plus-col`. It is anchored at `left: 38px` (fixed in CSS — button right edge ~25px + ~13px gap) and its `top` is set dynamically by `setButtonTop()`. The menu uses `translateY(-50%)` as a static transform so its vertical midpoint aligns with the `+` button's center. The open/close animation is a staggered opacity + translateX reveal (not scaleX).

### 3.4 Inline Toolbar Positioning Engine

`#huz-inline-toolbar` uses `position: fixed`. Its coordinates are derived from the live text selection's bounding rect, read via `window.getSelection().getRangeAt(0).getBoundingClientRect()`, which returns **viewport-relative** coordinates — correctly matching `position: fixed`. Placement is:

```
toolbar.left  = selectionRect.left + selectionRect.width / 2   (centered over selection)
toolbar.top   = selectionRect.top - toolbarHeight - GAP        (9px above selection)
toolbar.transform = translateX(-50%)                           (center anchor)
```

A left-clamp (`Math.max(left, 80)`) prevents the toolbar from overflowing the left edge of the viewport. No right-clamp is applied in v8; can be added if very short viewports are a concern.

### 3.5 Selection State Machine

```
No selection / collapsed caret:
  #huz-inline-toolbar  → opacity: 0; visibility: hidden; pointer-events: none

Valid selection (non-collapsed, within #huz-editor, length > 0):
  #huz-inline-toolbar  → positioned above selection rect
                       → opacity: 1; visibility: visible; pointer-events: all
                       → button .active states reflect current execCommand state

Selection cleared / caret collapsed:
  scheduleHideInlineToolbar() → 80ms debounce → hide if still collapsed
```

The 80ms debounce on hide prevents the toolbar from flickering during the `mousedown` → `mouseup` cycle when the user first clicks (briefly collapsing the selection before the new one forms).

### 3.6 Cursor Rendering Pipeline (FOUC prevention)

1. Critical cursor CSS is placed **above** the `@import` in `<style>` — parsed synchronously before any network fetch starts.
2. `style="cursor:text;color-scheme:light;"` is inlined on `.huz-wrap` and `#huz-editor` — applied by the HTML parser before the CSS engine runs.

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
- Removed word/char count footer
- Removed fixed top toolbar
- Merged all formatting options into the floating `+` menu
- `+` morphs to `×` via CSS `transform: rotate(45deg)` on `.open` class
- Menu animates left→right with `scaleX(0→1)` + `cubic-bezier(0.22,1,0.36,1)`
- All menu interactions use `mousedown` + `e.preventDefault()` to preserve editor focus

---

### v3 — Horizontal Icon Menu, Plus Button First-Line Fix, Resize Handle Deduplication

**Bug fixes:**
1. **Plus button on first line** — `showOnFirstLine()` fallback when `block` is null
2. **Horizontal icon-only menu** — `flex-direction: row`; tooltips via CSS `::after`/`::before`
3. **Double resize handle** — Removed `.huz-resize-hint` SVG

---

### v4 — 5 Critical Bug Fixes (List Items, Menu Centering, Tooltips, Scrollbar, Padding)

**Bug fixes:**
1. Plus button on list items — walks to `<li>` rect
2. Menu vertical centering — `setButtonTop()` sends `px + 11` to menu; `translateY(-50%)` in CSS
3. Tooltips below icons — `top: calc(100% + 7px)` + upward-pointing arrow
4. Scrollbar — `overflow-y: auto` strategy
5. Padding — `16px` top/right/bottom; left accounts for gutter column

---

### v5 — Static Frame / Internal Scroll Architecture

**Core architectural change:** `.huz-wrap` became a fixed-height frame with JS `activate()` toggling.

---

### v6 — Static Border Attempt, Auto-Grow + Blur Reset, Uniform Padding

**Changes:** Attempted removal of focus border, smart auto-grow with blur reset.

---

### v7 — Current Source (`huzlly_rich_editor_v7.html`)

**Core architectural change — wrapper as direct scroll host:**
- `.huz-wrap` uses `overflow-x: hidden` + `overflow-y: auto`
- `.huz-editor-body` is normal flow (`display: flex`, `min-height: inherit`)
- Height state managed entirely by CSS; no JS class toggling for height

**UI update (June 8, 2026) — Visual refactor (chip buttons, transparent menu):**

| Property | Old value | New value |
|---|---|---|
| `.huz-wrap` `border` | `1.5px solid var(--border)` | `none` |
| `.huz-plus-col` `width` | `44px` | `22px` |
| `#huz-plus-btn` `left` | `11px` | `3px` |
| `#huz-block-menu` `left` | `49px` | `32px` |
| `#huz-block-menu` `background` | `#fff` | `transparent` |
| `#huz-block-menu` `border` | `1.5px solid var(--border)` | `none` |
| `#huz-block-menu` `box-shadow` | `var(--menu-shadow)` | `none` |
| `.huz-menu-btn` `border` | `none` | `1px solid #E5E7EB` |
| `.huz-menu-btn` `border-radius` | `6px` | `999px` |
| `.huz-menu-btn` `background` | `transparent` | `#ffffff` |
| `.huz-menu-btn:hover` `color` | `var(--accent)` (purple) | `#1a1a18` (dark) |
| `.huz-menu-btn.active` `color` | `var(--accent)` (purple) | `#2563EB` (blue) |
| `#huz-plus-btn.open` `border/color` | `var(--plus-active)` (purple) | `var(--plus-idle)` (unchanged) |
| `#huz-plus-btn:hover` `border/color` | purple | `#1a1a18` icon / idle border |

**Update (June 8, 2026) — Menu simplification & animation overhaul:**

| Change | Detail |
|---|---|
| `#huz-plus-btn` `font-size` | `16px` → `19px`; `font-weight` `400` → `300` — larger symbol, same button |
| `#huz-plus-btn.open:hover` | Added — icon darkens to `#1a1a18` while × is hovered |
| `#huz-block-menu` `left` | `32px` → `38px` — 6px more gap between button and menu |
| Menu open animation | `scaleX` → staggered `opacity` + `translateX(-6px→0)` per child with 22ms step delays |
| Menu `transform` | Now static `translateY(-50%)` only; not part of open/close animation |
| Bullet list icon | `•` text → inline SVG three-line bullet list |
| Numbered list icon | `1.` text → inline SVG numbered list |
| Quote block (`blockquote`) | **Removed** from menu |
| Code block (`pre`) | **Removed** from menu |
| Alignment buttons (L/C/R) | **Removed** from menu |
| Trailing separator after alignment | **Removed** (was before Clear Formatting) |
| `.huz-menu-btn` `line-height` | Added `1` — prevents font descenders pushing icons low |
| `.huz-menu-btn` `padding-bottom` | Added `1px` — optical vertical centering for DM Sans |

---

### v8 — Current Source (`huzlly_rich_editor_v8.html`)

**Core change — decoupled inline formatting into selection-driven floating popover:**

#### Block menu cleanup

All inline formatting controls and their separators have been removed from `#huz-block-menu`. The block menu now contains **structural block types only**:

| Removed from block menu | Reason |
|---|---|
| Bold (`bold`) button | Moved to `#huz-inline-toolbar` |
| Italic (`italic`) button | Moved to `#huz-inline-toolbar` |
| Underline (`underline`) button | Moved to `#huz-inline-toolbar` |
| Strikethrough (`strikeThrough`) button | Moved to `#huz-inline-toolbar` |
| Clear Formatting (`#huz-clear-fmt`) button | Moved to `#huz-inline-toolbar` |
| Two `.huz-menu-sep` separators flanking inline buttons | No longer needed |

Block menu stagger CSS reduced from 12 children to **8 children** (H1, H2, H3, sep, UL, OL, —, (trailing sep removed)).

#### New: `#huz-inline-toolbar` — contextual selection popover

A brand-new floating element rendered **outside** `.huz-wrap` in the DOM (after the wrapper, before `</body>`) using `position: fixed`.

**Markup:**
```html
<div id="huz-inline-toolbar" role="toolbar" aria-label="Inline formatting options">
  <button class="huz-inline-btn" data-cmd="bold"          data-tip="Bold"><b>B</b></button>
  <button class="huz-inline-btn" data-cmd="italic"        data-tip="Italic"><em>I</em></button>
  <button class="huz-inline-btn" data-cmd="underline"     data-tip="Underline"><u>U</u></button>
  <button class="huz-inline-btn" data-cmd="strikeThrough" data-tip="Strikethrough"><s>S</s></button>
  <div class="huz-inline-sep"></div>
  <button class="huz-inline-btn" id="huz-inline-clear"    data-tip="Clear Formatting">⌫</button>
</div>
```

**CSS design:**

| Property | Value | Rationale |
|---|---|---|
| `position` | `fixed` | Viewport coords — matches `getBoundingClientRect()` output; avoids clipping by `.huz-wrap` overflow |
| `background` | `#ffffff` | Clean white popover surface |
| `border` | `1px solid #E5E7EB` | Consistent with chip button borders |
| `border-radius` | `999px` | Full pill — visually distinct from block menu's flat transparent strip |
| `box-shadow` | `0 4px 16px rgba(0,0,0,0.10), 0 1px 4px rgba(0,0,0,0.06)` | Soft floating popover depth |
| `padding` | `4px 6px` | Tight breathing room around buttons |
| `gap` | `3px` | Compact chip spacing |
| Default state | `opacity:0; visibility:hidden; pointer-events:none; transform:translateY(4px)` | Hidden; subtle enter from below |
| `.visible` state | `opacity:1; visibility:visible; pointer-events:all; transform:translateY(0)` | Smooth 140ms fade+rise |

**`.huz-inline-btn` styling:**
Identical design language to `.huz-menu-btn` — pill chips (border-radius: 999px), white background, `1px solid #E5E7EB` border, `font-size: 11px`, `font-weight: 700`, `line-height: 1`, `padding-bottom: 1px`. Color: `var(--text-muted)` idle → `#1a1a18` hover → `#2563EB` active.

**Tooltip direction:** Appears **above** the buttons (since the toolbar floats above text), using `bottom: calc(100% + 7px)` with a downward-pointing arrow (`border-top-color`). This is intentionally inverted from `.huz-menu-btn` tooltips which appear below.

#### New JS: selection state machine

Three new event listeners drive inline toolbar visibility:

1. `document.addEventListener('selectionchange', ...)` — primary driver; reacts to any selection change globally, filters to editor-contained selections.
2. `editor.addEventListener('keyup', ...)` — catches shift+arrow keyboard selections (selectionchange alone can miss these in some browsers).
3. `inlineToolbar.addEventListener('mousedown', ...)` — handles button clicks with `e.preventDefault()` to prevent selection loss.

**`isValidEditorSelection()`** — gating function that returns `true` only when:
- A `Selection` object exists with `rangeCount > 0`
- `sel.isCollapsed === false`
- `sel.toString().length > 0`
- Both `sel.anchorNode` and `sel.focusNode` are within `#huz-editor`

**`showInlineToolbar()`** — calls `isValidEditorSelection()`, reads `getBoundingClientRect()`, computes `left` and `top`, clamps left to `≥ 80px`, sets `position:fixed` coords, calls `updateInlineBtnStates()`, adds `.visible`.

**`hideInlineToolbar()`** — removes `.visible`.

**`scheduleHideInlineToolbar()`** — 80ms debounced hide, checking `isValidEditorSelection()` before acting. Prevents toolbar flicker during `mousedown → mouseup` cycles.

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
| Symbol size | `font-size: 19px`, `font-weight: 300` — larger symbol, button dimensions unchanged |
| Hover (closed) | Icon darkens to `#1a1a18`; border stays `--plus-idle` |
| Hover (open / showing ×) | Icon darkens to `#1a1a18`; border stays `--plus-idle` |
| Idle open state | Both border and icon stay at `--plus-idle`; only difference from idle is rotation |
| Focus preservation | `mousedown` + `e.preventDefault()` on all menu interactions |
| Hidden by default | `opacity: 0; pointer-events: none` — `.visible` class enables it |
| Accessibility | `aria-label="Insert block"`, `aria-expanded` toggled on open/close |

### 5.2 Block Menu (Structural Blocks Only)

| Property | Value |
|---|---|
| Layout | `display: flex; flex-direction: row` (horizontal row of chips) |
| Position | `position: absolute` within `.huz-plus-col`; `left: 38px` fixed in CSS |
| Vertical anchor | `top` set dynamically by `setButtonTop()`; static `translateY(-50%)` centers it on button |
| Container appearance | Transparent — no background, border, or shadow |
| Icons | Glyph-only or inline SVG (H1, H2, H3, bullet SVG, numbered SVG, —) |
| Tooltips | Pure CSS `::after`/`::before` reading `data-tip` attribute; appear **below** icon |
| Separators | Thin `1px` vertical `.huz-menu-sep` divider between H1-H3 group and list group |
| Open animation | Staggered left→right: each child fades in + slides from `translateX(-6px)` to `0` with 22ms step delays (8 children max) |
| Close animation | Container opacity fades out; `visibility: hidden` after 0.18s |
| Blur guard | `e.relatedTarget` check — editor blur won't close menu if focus moves into menu |
| Accessibility | `role="toolbar"`, `aria-label="Block type options"` |

### 5.3 Supported Block Types

| Icon | Block type | HTML produced |
|---|---|---|
| `H1` | Heading 1 | `<h1>` |
| `H2` | Heading 2 | `<h2>` |
| `H3` | Heading 3 | `<h3>` |
| SVG bullet list | Bullet list | `<ul><li>` |
| SVG numbered list | Numbered list | `<ol><li>` |
| `—` | Divider | `<hr>` + new `<p>` after |

> **Default block:** `<p>` is the browser's default for Enter key in `contenteditable`. It is not a menu option but is produced by `Enter` in headings (explicit) and naturally by the browser elsewhere.

> **Retained CSS:** The editor CSS still includes `blockquote` and `pre` block styles in case existing content contains them — they just can't be inserted via the menu.

### 5.4 Inline Formatting Popover (`#huz-inline-toolbar`)

The popover appears automatically when the user makes a non-collapsed text selection within `#huz-editor`. It floats centered above the selected text.

| Button | Command | `data-cmd` value |
|---|---|---|
| **B** | Bold | `bold` |
| *I* | Italic | `italic` |
| U | Underline | `underline` |
| ~~S~~ | Strikethrough | `strikeThrough` |
| ⌫ | Clear formatting | `removeFormat` (via `#huz-inline-clear`) |

**Show conditions:** `isValidEditorSelection()` returns true — selection is non-collapsed, has string length > 0, and both anchor/focus nodes are within `#huz-editor`.

**Hide conditions:**
- Selection collapses to a caret (navigation, typing)
- `Escape` key pressed in editor
- Selection anchor/focus moves outside editor
- Clear Formatting applied (selection collapses after removeFormat)

**Button active state:** `.active` class (blue `#2563EB`, `#BFDBFE` border) applied via `updateInlineBtnStates()`, which queries `document.queryCommandState()` on open and after each command.

**Focus preservation:** `mousedown` + `e.preventDefault()` on `#huz-inline-toolbar` ensures the editor retains focus and the selection range is not destroyed when the user clicks a format button.

### 5.5 Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl/Cmd + B` | Bold (browser native on selection) |
| `Ctrl/Cmd + I` | Italic (browser native on selection) |
| `Ctrl/Cmd + U` | Underline (browser native on selection) |
| `Tab` | Indent (list) |
| `Shift + Tab` | Outdent (list) |
| `Enter` in H1/H2/H3 | Inserts new `<p>` after heading, places cursor |
| `Escape` | Closes block menu AND hides inline toolbar |
| Arrow keys (no Shift) | Collapses selection → schedules inline toolbar hide |
| `Shift` + Arrow / Home / End | Extends selection → shows inline toolbar |

---

## 6. CSS Custom Properties & Design Tokens

All defined in `:root`:

```css
:root {
  --bg:          #ffffff;          /* editor + wrapper background */
  --surface:     #f7f7f5;          /* inline code background */
  --border:      #e8e8e4;          /* separators (NOT wrapper — wrapper is borderless) */
  --text:        #1a1a18;          /* primary text color */
  --text-muted:  #9b9b97;          /* placeholder, blockquote, strikethrough */
  --accent:      #534AB7;          /* purple — blockquote bar only */
  --accent-soft: #eeecfb;          /* light purple — retained in :root, unused in current UI */
  --plus-idle:   #c7c7c2;          /* + button default border + color */
  --plus-active: #534AB7;          /* retained in :root, currently unused */
  --menu-shadow: ...;              /* retained in :root, currently unused (block menu is shadowless) */
  --tooltip-bg:  #1a1a18;          /* tooltip background */
  --radius:      8px;              /* wrapper border-radius */
  --font-body:   'DM Sans', system-ui, sans-serif;
  --font-mono:   'JetBrains Mono', 'Fira Mono', monospace;
  --t:           0.15s ease;       /* standard transition for interactive controls */
}
```

---

## 7. Layout Geometry Reference

### 7.1 Horizontal Left-Side Layout

```
[wrapper left wall]
|←── 22px .huz-plus-col ──→|←── #huz-editor ──────────────────────────→|
|  left:3px #huz-plus-btn  |    padding-left: 10px
|  right edge ≈ 25px       |
|                           ↑
|                          32px from wall = text cursor start
|
Block menu left: 38px  (button right edge ~25px + ~13px visual gap)
```

### 7.2 Inline Toolbar Geometry

```
viewport
  ↑
  ├──  [#huz-inline-toolbar]   ← position:fixed
  │        centered over selection rect
  │        top = selectionRect.top - ~38px (toolbar height) - 9px (GAP)
  │        left = selectionRect.left + selectionRect.width/2 → clamped ≥ 80px
  │        transform: translateX(-50%)
  │
  └──  [selected text]         ← inside #huz-editor
```

### 7.3 Key Measurements

| Element | Property | Value | Rationale |
|---|---|---|---|
| `.huz-plus-col` | `width` | `22px` | Matches button width exactly; no dead space in gutter |
| `#huz-plus-btn` | `left` | `3px` | 3px breathing room so border never visually clips left edge |
| `#huz-plus-btn` | `width/height` | `22px` | Touch-friendly minimum |
| `#huz-plus-btn` | `font-size` | `19px` | Enlarged symbol; button dimensions unchanged |
| `#huz-plus-btn` | `font-weight` | `300` | Lighter weight balances the larger symbol size |
| `#huz-block-menu` | `left` | `38px` | Button right edge (~25px) + ~13px visual gap |
| `#huz-editor` | `padding` | `16px 16px 16px 10px` | Left 10px → text starts 32px from wall |
| `.huz-wrap` | `min-height` | `180px` | Compact initial frame height |
| `#huz-editor` | `min-height` | `180px` | Matches wrapper |
| `.huz-wrap` | `border` | `none` | Borderless design — no idle or focus border |
| `#huz-inline-toolbar` | `padding` | `4px 6px` | Tight breathing room inside pill |
| `#huz-inline-toolbar` | `gap` | `3px` | Compact chip spacing |
| `.huz-inline-btn` | `width/height` | `28px / 26px` | Slightly smaller than `.huz-menu-btn` for compact popover |
| Inline toolbar GAP | 9px | — | Comfortable vertical separation above selection |

---

## 8. State Machine — Height & Scroll Behavior

In v7/v8, height state is driven by CSS alone (no JS class toggling for height). The wrapper is borderless in **all** states.

```
Page Load / Blur state:
  .huz-wrap → border: none; min-height: 180px; overflow-x: hidden; overflow-y: auto; resize: vertical
  #huz-editor → min-height: 180px; height: auto (grows to content)
  → No border, no glow — wrapper is visually transparent

    │  user focuses editor
    ▼

Active / Focused state:
  .huz-wrap:focus-within → NO additional styling
  → Layout/overflow/resize unchanged
  → Visually identical to blur state — borderless throughout

    │  user selects text (non-collapsed, within editor)
    ▼

Selection state:
  #huz-inline-toolbar → positioned above selection; opacity:1; visible; pointer-events:all

    │  selection collapses or moves outside editor
    ▼

Back to Active / Focused state (inline toolbar hidden after 80ms debounce)
```

---

## 9. Key JavaScript Functions & State Variables

### 9.1 State Variables

| Variable | Type | Purpose |
|---|---|---|
| `activeBlock` | `Element \| null` | Currently active top-level block under the cursor |
| `menuOpen` | `boolean` | Whether `#huz-block-menu` is open |
| `savedRange` | `Range \| null` | Selection range saved on block menu open; restored before block insertions |
| `saveTimer` | `number \| null` | Debounce timer ID for `syncToBubble()` |
| `inlineHideTimer` | `number \| null` | Debounce timer ID for `scheduleHideInlineToolbar()` (80ms) |

### 9.2 Functions

| Function | Purpose |
|---|---|
| `init()` | Reads `data-initial-content`; injects into `#huz-editor` if not placeholder token and not empty |
| `syncToBubble()` | Debounced 500ms call to `bubble_fn_saveEditorData(editor.innerHTML)` |
| `syncNow()` | Immediate (non-debounced) save — called on `blur` |
| `getActiveBlock()` | Walks selection anchor up to nearest direct child of `#huz-editor`; returns `null` if not inside editor |
| `positionPlusBtn(block)` | Positions `+` button aligned to active block; resolves to `<li>` rect if inside a list; calls `showOnFirstLine()` if editor is focused but `block` is null |
| `showOnFirstLine()` | Positions `+` button using live caret `getBoundingClientRect()`; falls back to `padding-top + lineHeight/2` |
| `setButtonTop(px)` | Sets `plusBtn.style.top = px + 'px'`; sets `blockMenu.style.top = (px + 11) + 'px'` |
| `openMenu()` | Saves current selection to `savedRange`; adds `.open` to block menu and button |
| `closeMenu()` | Removes `.open` from block menu and button; sets `aria-expanded` false |
| `restoreRange()` | Restores `savedRange` into the live selection (called before block insertions) |
| `insertOrConvertBlock(type)` | Focuses editor; restores range; inserts or replaces the active block with the chosen type; calls `syncToBubble()`. Handles: `hr`, `ul`, `ol`, `p`, `h1`, `h2`, `h3`. |
| `insertAfter(node, ref)` | DOM helper — inserts `node` after `ref` inside `#huz-editor`; appends if no ref |
| `placeCursorIn(el)` | Places collapsed selection at first text node inside `el` |
| `isValidEditorSelection()` | Returns `true` if current selection is non-collapsed, has string length > 0, and both anchor/focus nodes are inside `#huz-editor` |
| `showInlineToolbar()` | Validates selection, reads bounding rect, computes fixed coords, adds `.visible` to `#huz-inline-toolbar`, calls `updateInlineBtnStates()` |
| `hideInlineToolbar()` | Removes `.visible` from `#huz-inline-toolbar` |
| `scheduleHideInlineToolbar()` | 80ms debounced call to `hideInlineToolbar()` — prevents flicker during mousedown/mouseup cycles |
| `updateInlineBtnStates()` | Reflects `bold`/`italic`/`underline`/`strikeThrough` command state onto `.active` class of `.huz-inline-btn` buttons |
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

### 10.2 Initial Content Hydration

```html
<div
  id="huz-editor"
  contenteditable="true"
  spellcheck="true"
  data-placeholder="Write something… or click + to insert a block"
  data-initial-content="[Current page Note's Body]"
  style="cursor:text;color-scheme:light;"
></div>
```

### 10.3 Bubble Element Settings

| Setting | Required value |
|---|---|
| Element height | Fixed or Fit content |
| Toolbox plugin | Must be installed; `saveEditorData` function must be exposed |
| HTML Element | Paste the entire `huzlly_rich_editor_v8.html` content |

### 10.4 Note on `position: fixed` inside Bubble iframes

`#huz-inline-toolbar` uses `position: fixed`. In Bubble, HTML Elements are rendered inside iframes. `position: fixed` is resolved against the **iframe's viewport**, not the outer page viewport. This is correct behavior: the toolbar will always appear within the HTML Element's iframe frame, anchored above the selected text. No special configuration is needed.

---

## 11. Known Bug Fixes & Root Cause Log

| # | Bug | Root Cause | Fix applied (version) |
|---|---|---|---|
| 1 | `+` button invisible on first empty line | `getActiveBlock()` returned `null`; code hid button | `showOnFirstLine()` + `setTimeout(..., 0)` on focus/click (v3) |
| 2 | `+` button centers on full `<ul>/<ol>` not active `<li>` | `getBoundingClientRect()` called on parent list | Walk selection anchor to nearest `<li>` (v4) |
| 3 | Menu opens at top edge of button, not centered | Menu `top` set to same px as button `top` | `setButtonTop()` feeds `px + 11` to menu; `translateY(-50%)` in CSS (v4) |
| 4 | Tooltips appearing above icons | CSS placement inverted | `top: calc(100% + 7px)` + upward arrow (v4) |
| 5 | Scrollbar disappeared / resize handle missing | `overflow: hidden` on both axes removes native grip | `overflow-x: hidden` + `overflow-y: auto` on `.huz-wrap` (v7) |
| 6 | Caret invisible while typing | `caret-color` not set | `caret-color: #1a1a18` on `#huz-editor` (v7) |
| 7 | I-beam cursor white/invisible on hover | OS dark-mode `color-scheme` inverted cursor | `cursor: text; color-scheme: light` on editor (v7) |
| 8 | Cursor FOUC on page load | Cursor CSS written after `@import` render-blocked | Hoisted cursor rules above `@import`; inline `style` on markup (v7) |
| 9 | Text overlaps `+` button | Left gutter undersized | `.huz-plus-col: 22px`, `#huz-editor` `padding-left: 10px` (v7 UI update) |
| 10 | `×` hover state did not darken icon | `.open:hover` selector not present | Added `#huz-plus-btn.open:hover { color: #1a1a18 }` (v7 June 8 update) |
| 11 | Menu icons visually low in chips | Font descenders + default line-height shifting rendered baseline down | `line-height: 1` + `padding-bottom: 1px` on `.huz-menu-btn` (v7 June 8 update) |
| 12 | Inline toolbar flickers on initial click (before drag-select) | `selectionchange` fires on `mousedown` collapse, then again on selection | 80ms debounce via `scheduleHideInlineToolbar()` — only hides if still collapsed after delay (v8) |
| 13 | Clicking a format button collapsed the selection | `mousedown` on toolbar triggered blur/selection-loss on editor | `e.preventDefault()` on `inlineToolbar.addEventListener('mousedown', ...)` (v8) |
| 14 | Inline toolbar clipped by `.huz-wrap` overflow | `position: fixed` inside a positioned overflow ancestor is clipped | Moved `#huz-inline-toolbar` outside `.huz-wrap` in DOM; `position: fixed` now resolves to iframe viewport (v8) |
| 15 | Inline toolbar tooltip arrows pointed wrong direction | Block menu tooltips point up (below icon); inline toolbar sits above text, so tooltip should appear above buttons | Inverted tooltip CSS on `.huz-inline-btn`: `bottom: calc(100% + 7px)` + `border-top-color` downward arrow (v8) |

---

## 12. Conventions & Code Style

- All CSS class names and IDs are prefixed with `huz-` to avoid conflicts with Bubble's global stylesheet.
- JavaScript is entirely vanilla — no external libraries, no `import` statements.
- `document.execCommand` is used for inline formatting. While deprecated in spec, it remains the most reliable cross-browser approach for `contenteditable` contexts.
- `mousedown` + `e.preventDefault()` is used (not `click`) for all toolbar button interactions — critical for both block menu (preserves caret) and inline toolbar (preserves selection).
- The save function is debounced at **500ms**. `syncNow()` fires immediately on `blur`.
- The entire JS is wrapped in an IIFE to avoid polluting global scope inside Bubble's iframe.
- Inline SVG is used for list icons (bullet + numbered) to ensure crisp rendering at all DPR values and consistent cross-browser appearance.
- Block menu open animation is CSS-only: staggered `transition-delay` on each `#huz-block-menu > *` child, stepping by 22ms from left to right.
- Inline toolbar show/hide is CSS transition-driven (opacity + visibility + translateY); JS only adds/removes the `.visible` class.
- `selectionchange` is the primary event for inline toolbar state. `keyup` with shift detection handles keyboard-extended selections as a supplement.

---

## 13. Bubble Setup Checklist

- [ ] Paste the entire content of `huzlly_rich_editor_v8.html` into a Bubble **HTML Element**
- [ ] Install the **Toolbox plugin** in Bubble
- [ ] Create a Bubble workflow exposing a function named `saveEditorData`
- [ ] Replace `data-initial-content="#BUBBLE_DYNAMIC_DATA#"` with your Bubble dynamic expression
- [ ] **Optional performance fix:** If Bubble loads DM Sans globally, delete the `@import` line
- [ ] Test content hydration on page load
- [ ] Test save trigger fires on both input (debounced) and blur (immediate)
- [ ] Test vertical resize handle is visible and draggable
- [ ] Test scrollbar appears when content exceeds editor height
- [ ] Verify wrapper is fully borderless in both idle and focused states
- [ ] Verify plus button `+` symbol is visually larger and centered in the circle
- [ ] Verify hovering the rotated `×` darkens the icon correctly
- [ ] Verify block menu chips reveal left→right with stagger on open
- [ ] Verify block menu contains **only** H1, H2, H3, Bullet, Numbered, Divider — no inline format buttons
- [ ] Select text and verify inline toolbar appears centered above the selection
- [ ] Verify inline toolbar Bold/Italic/Underline/Strikethrough apply formatting correctly
- [ ] Verify `.active` (blue) state on inline buttons reflects current format state accurately
- [ ] Verify Clear Formatting (⌫) removes all inline styles from selected text
- [ ] Verify clicking an inline toolbar button does NOT collapse the text selection
- [ ] Verify inline toolbar hides when caret collapses (click without drag, arrow keys)
- [ ] Verify `Escape` hides both the block menu and inline toolbar
- [ ] Verify Quote, Code Block, Alignment buttons, and inline format buttons are absent from the block menu

---

*This file is the canonical knowledge base for the rich-text-editor project. All implementation decisions, architectural choices, and bug resolutions should be recorded here. The source of truth for actual code is `huzlly_rich_editor_v8.html`.*
