# Rich Text Editor — Project Documentation

> **Single source of truth** for the custom Notion-like block-based rich text editor built for Bubble.io.  
> Last updated: June 2026 · Current source file: **huzlly_rich_editor_v9.html**  
> Architecture pivot applied: June 2026 — v9: Removed all floating dynamic UI elements; introduced Fixed Top Toolbar layout.

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

A **self-contained, single-file** HTML/CSS/JS snippet (`huzlly_rich_editor_v9.html`) intended to be pasted directly into a Bubble.io **HTML Element**. It functions as a block-based rich text editor with a traditional, fixed top toolbar layout — a deliberate architectural departure from the floating/dynamic UI mechanisms used in v1–v8.

**v9 core design principle:** All formatting controls are permanently visible and accessible in a single horizontal toolbar panel anchored to the top of the editor container. There are no floating, position-tracked, or selection-driven UI elements of any kind.

### What was removed in v9

| Removed element | Reason |
|---|---|
| Floating `+` button (`#huz-plus-btn`) | Replaced by persistent toolbar; no line-tracking UI required |
| Left gutter column (`.huz-plus-col`) | No longer needed without the `+` button; canvas now occupies full width |
| Floating block menu (`#huz-block-menu`) | Block insertion now handled directly from the fixed toolbar |
| Contextual inline selection toolbar (`#huz-inline-toolbar`) | Inline formatting now handled directly from the fixed toolbar |
| `.huz-editor-body` flex wrapper | Removed; `.huz-wrap` is now the direct flex column parent |
| All `getActiveBlock()`, `isCaretInsideEditor()`, `positionPlusBtn()`, `showOnFirstLine()` JS logic | Eliminated along with the floating `+` button |
| All `showInlineToolbar()`, `hideInlineToolbar()`, `scheduleHideInlineToolbar()` JS logic | Eliminated along with the floating inline toolbar |
| `openMenu()` / `closeMenu()` | Not required without the block menu |
| `savedRange` state variable + `restoreRange()` | Not needed; toolbar buttons can read the live selection directly |
| `inlineHideTimer` state variable | Not required |
| Adaptive placement constants (`TOOLBAR_HEIGHT_APPROX`, `TOOLBAR_GAP`, `TOOLBAR_MIN_TOP_SPACE`) | Not applicable without a floating popover |
| `selectionchange` document listener | No floating toolbar to update; selection state is read only on `keyup`/`mouseup` for button active states |
| `.above` / `.below` CSS direction classes | Not required |
| `isValidEditorSelection()` | Not required |

### What the editor now provides

- A **unified input card frame** (`.huz-wrap`) housing a permanent top toolbar and a full-width text canvas in a clean vertical stack.
- A **Fixed Top Toolbar** (`.huz-toolbar`) spanning the full internal width of the container, grouping all formatting actions in a single always-visible horizontal strip with `position: sticky; top: 0` to remain pinned during scrolling.
- A **Full-Width Editor Canvas** (`#huz-editor`) filling the remaining vertical space below the toolbar with no left gutter offset.
- Full **Bubble.io Toolbox plugin** integration for real-time data sync, unchanged from v8.
- **Secure initial content hydration** from Bubble dynamic data via a `data-initial-content` attribute, unchanged from v8.
- `data-placeholder` text updated to `"Add a description..."`.

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
| Accessibility | `aria-label`, `role="toolbar"`, `aria-pressed` on toolbar buttons |

---

## 3. Architecture

### 3.1 HTML Structure

```
.huz-wrap  #huz-wrap                ← unified input card frame (owns border, overflow, resize)
  ├── .huz-toolbar                  ← fixed top toolbar panel (permanent, always visible)
  │     ├── [Group 1: Inline Styles]  Bold, Italic, Strikethrough, Underline
  │     ├── .huz-tb-sep              ← vertical separator
  │     ├── [Group 2: Lists]          Bullet List (SVG), Numbered List (SVG)
  │     ├── .huz-tb-sep              ← vertical separator
  │     └── [Group 3: Blocks]         Horizontal Divider (—)
  └── #huz-editor                   ← contenteditable div (full-width typing canvas)
```

There are no floating or absolutely-positioned UI elements inside or outside `.huz-wrap`. Every interactive element is in the normal document flow within `.huz-wrap`. `#huz-inline-toolbar` (v8) no longer exists anywhere in the DOM.

### 3.2 Container Layout Model

`.huz-wrap` uses `display: flex; flex-direction: column` to stack `.huz-toolbar` and `#huz-editor` vertically. The toolbar has a fixed intrinsic height determined by its padding and button dimensions. The editor canvas below it fills all remaining space via `flex: 1`.

```
.huz-wrap (display: flex; flex-direction: column; border: ...; overflow-y: auto; resize: vertical)
  ├── .huz-toolbar  (flex-shrink: 0; position: sticky; top: 0; border-bottom: 1px solid --border)
  └── #huz-editor   (flex: 1; full-width; no left gutter offset)
```

### 3.3 Toolbar Layout Model

`.huz-toolbar` is a `display: flex; flex-direction: row; align-items: center` strip. Its children are button elements and separator elements arranged left-to-right in three groups, each group delineated by a `.huz-tb-sep` vertical rule. Separators are pure CSS elements with no interactive role.

```
.huz-toolbar (flex row, full width, padded horizontally)
  ├── .huz-tb-btn [data-cmd="bold"]          ← Group 1: Bold
  ├── .huz-tb-btn [data-cmd="italic"]        ← Group 1: Italic
  ├── .huz-tb-btn [data-cmd="strikeThrough"] ← Group 1: Strikethrough
  ├── .huz-tb-btn [data-cmd="underline"]     ← Group 1: Underline
  ├── .huz-tb-sep                            ← Structural divider
  ├── .huz-tb-btn [data-block="ul"]          ← Group 2: Bullet List (inline SVG)
  ├── .huz-tb-btn [data-block="ol"]          ← Group 2: Numbered List (inline SVG)
  ├── .huz-tb-sep                            ← Structural divider
  └── .huz-tb-btn [data-block="hr"]          ← Group 3: Horizontal Divider
```

### 3.4 Toolbar Button State Model

Toolbar buttons carrying `data-cmd` attributes correspond to inline formatting commands (`bold`, `italic`, `underline`, `strikeThrough`). These buttons maintain a visual `.active` state that reflects whether the current caret position or selection is inside text formatted with that command.

**Active state update triggers:**

- `keyup` event on `#huz-editor`
- `mouseup` event on `#huz-editor`
- Immediately after any toolbar button command is executed

Active state is read via `document.queryCommandState(cmd)` for each command. The `.active` class is toggled on the corresponding `.huz-tb-btn` element.

Buttons carrying `data-block` attributes (`ul`, `ol`, `hr`) do not carry an `.active` state — they are action-only triggers.

### 3.5 Toolbar Command Routing

Toolbar buttons use `mousedown` + `e.preventDefault()` for all interactions. This is critical: it prevents the editor from losing focus (and thus the caret position or text selection) when the user clicks a toolbar button.

**Command routing logic:**

```
toolbar mousedown
  │
  ├── btn has [data-cmd]
  │     → document.execCommand(cmd, false, null)
  │     → updateToolbarStates()
  │     → syncToBubble()
  │
  └── btn has [data-block]
        ├── "ul" → insertListBlock('ul')
        ├── "ol" → insertListBlock('ol')
        └── "hr" → insertDividerBlock()
              → syncToBubble()
```

### 3.6 Selection Persistence Strategy

Because all toolbar buttons use `mousedown` + `e.preventDefault()`, the browser never commits a `blur` on `#huz-editor` during toolbar interaction. The live selection (or collapsed caret) is fully intact at the moment `document.execCommand()` or the block insertion logic runs. No `savedRange` / `restoreRange()` pattern is needed.

For block insertions (`ul`, `ol`, `hr`), the insertion function reads `window.getSelection()` directly to determine where in the document to insert the new element.

### 3.7 Scroll / Resize Model

`.huz-wrap` owns both `resize: vertical` and `overflow-y: auto`. This is the only correct layout for a native browser resize grip alongside a scrollbar — both must live on the same element.

The toolbar does not scroll because it uses `position: sticky; top: 0` with a `z-index` above the canvas. This ensures it remains pinned to the top of the scroll container at all times regardless of editor content length. The `background: var(--bg)` rule on the toolbar is required for sticky behavior to correctly mask any canvas text that scrolls beneath it.

### 3.8 Cursor Rendering Pipeline (FOUC prevention)

Carried forward from v8 unchanged:

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

### v7 — Wrapper as Direct Scroll Host

**Core architectural change:**
- `.huz-wrap` uses `overflow-x: hidden` + `overflow-y: auto`
- `.huz-editor-body` is normal flow (`display: flex`, `min-height: inherit`)
- Height state managed entirely by CSS; no JS class toggling for height

**UI updates (June 2026):** Visual refactor — chip buttons, transparent block menu, borderless wrapper, gutter narrowed to 22px, `+` button symbol enlarged to `font-size: 19px` / `font-weight: 300`, hover and open state color corrections, menu animation changed from `scaleX` to staggered `opacity + translateX`, inline SVGs for list icons, Quote/Code/Align removed from menu.

---

### v8 — Contextual Inline Toolbar (Selection-Driven Popover)

**Core change:** Decoupled all inline formatting from the block menu into a new `#huz-inline-toolbar` element rendered outside `.huz-wrap` using `position: fixed`.

**Patch (June 2026) — Bug Fix 1:** Inline toolbar adaptive placement engine — flips below selection when there is insufficient vertical space above; horizontal position clamped within editor client rect; `border-radius` corrected from `999px` (oval) to `8px` (soft card); `.above`/`.below` CSS classes drive tooltip arrow direction.

**Patch (June 2026) — Bug Fix 2:** Plus button first-line click-back regression — added `isCaretInsideEditor()` helper; `positionPlusBtn(null)` now routes to `showOnFirstLine()` instead of hiding when caret is inside editor; `focus` and `click` listeners wrapped in `setTimeout(..., 0)` to allow browser selection to settle.

---

### v9 — Fixed Top Toolbar Architecture (June 2026)

**Summary:** Complete removal of all floating, dynamic, and position-tracked UI elements. A traditional fixed top toolbar replaces every floating mechanism from v1–v8. The entire class of DOM geometry tracking, selection debouncing, and iframe viewport clamping bugs is eliminated at the architectural level.

#### Removed

| Element / System | Replaced by |
|---|---|
| Floating `+` button (`#huz-plus-btn`) | Fixed toolbar (`.huz-toolbar`) |
| Left gutter column (`.huz-plus-col`) | Removed; canvas is now full-width |
| Floating block menu (`#huz-block-menu`) | Block buttons directly in `.huz-toolbar` |
| Floating inline toolbar (`#huz-inline-toolbar`) | Inline format buttons directly in `.huz-toolbar` |
| `.huz-editor-body` flex wrapper | Removed; `.huz-wrap` is now the direct flex column parent |
| `getActiveBlock()` | Not required |
| `isCaretInsideEditor()` | Not required |
| `positionPlusBtn()` / `showOnFirstLine()` / `setButtonTop()` | Not required |
| `showInlineToolbar()` / `hideInlineToolbar()` / `scheduleHideInlineToolbar()` | Not required |
| `openMenu()` / `closeMenu()` | Not required |
| `savedRange` state variable | Not required — selection is live at button mousedown |
| `restoreRange()` | Not required |
| `inlineHideTimer` state variable | Not required |
| Adaptive placement constants (`TOOLBAR_HEIGHT_APPROX`, `TOOLBAR_GAP`, `TOOLBAR_MIN_TOP_SPACE`) | Not required |
| `selectionchange` document listener | Not required — state updates driven by `keyup`/`mouseup` |
| `.above` / `.below` CSS direction classes | Not required |
| `isValidEditorSelection()` | Not required |

#### Added

| Element / System | Purpose |
|---|---|
| `.huz-toolbar` | Permanent fixed top toolbar panel; always visible; houses all formatting controls |
| `.huz-tb-btn` | Unified button class for all toolbar buttons (inline and block) |
| `.huz-tb-sep` | Thin vertical separator between button groups |
| `updateToolbarStates()` | Reads `queryCommandState()` for each inline command; toggles `.active` on corresponding `.huz-tb-btn` elements |
| `insertListBlock(type)` | Inserts a `<ul>` or `<ol>` at the current caret/selection position |
| `insertDividerBlock()` | Inserts an `<hr>` followed by a new `<p>` at the current caret/selection position |

#### Layout changes

| Property | v8 value | v9 value |
|---|---|---|
| `.huz-wrap` `flex-direction` | Not a flex container (block layout) | `column` |
| `.huz-editor-body` | Present — flex row wrapper | **Removed** |
| `.huz-plus-col` | `22px` wide left gutter | **Removed** |
| `#huz-editor` `padding-left` | `10px` (gutter offset) | `16px` (full symmetric padding) |
| New `.huz-toolbar` | Not present | First flex child of `.huz-wrap` |
| `#huz-editor` position in DOM | Second child of `.huz-editor-body` | Second (and final) flex child of `.huz-wrap`; `flex: 1` |
| `data-placeholder` | `"Write something… or click + to insert a block"` | `"Add a description..."` |

---

## 5. Feature Reference

### 5.1 Fixed Top Toolbar (`.huz-toolbar`)

The toolbar is a permanent, always-visible horizontal strip anchored to the top of `.huz-wrap`. It is the single source of all formatting actions in v9.

| Property | Value |
|---|---|
| Layout | `display: flex; flex-direction: row; align-items: center` |
| Width | Full internal width of `.huz-wrap` |
| Height | Determined by button height + vertical padding |
| Position | First child in `.huz-wrap` flex column; `position: sticky; top: 0` to prevent scrolling away |
| Separation from canvas | `border-bottom: 1px solid var(--border)` (bottom edge of toolbar) |
| Background | `var(--bg)` — matches wrapper; required for sticky behavior to mask scrolled canvas content |
| `z-index` | Above `#huz-editor` to ensure it overlaps when sticky |
| Focus preservation | All buttons use `mousedown` + `e.preventDefault()` — editor never loses focus |
| Accessibility | `role="toolbar"`, `aria-label="Formatting options"` |

### 5.2 Toolbar Button Groups (Left-to-Right)

#### Group 1 — Inline Styles

| Button label | `data-cmd` | Command | `.active` state |
|---|---|---|---|
| **B** | `bold` | `document.execCommand('bold')` | Yes — when caret/selection is inside bold text |
| *I* | `italic` | `document.execCommand('italic')` | Yes — when caret/selection is inside italic text |
| ~~S~~ | `strikeThrough` | `document.execCommand('strikeThrough')` | Yes — when caret/selection is inside strikethrough text |
| U | `underline` | `document.execCommand('underline')` | Yes — when caret/selection is inside underlined text |

All four are driven by `document.execCommand`. Active state is read via `document.queryCommandState(cmd)` and reflected as the `.active` CSS class on the button element.

#### Structural Divider

`.huz-tb-sep` — a thin vertical rule separating Group 1 from Group 2.

#### Group 2 — Lists

| Button label | `data-block` | Action | `.active` state |
|---|---|---|---|
| Bullet list SVG | `ul` | `insertListBlock('ul')` | No |
| Numbered list SVG | `ol` | `insertListBlock('ol')` | No |

Both buttons use inline SVG icons (identical to v7/v8 block menu icons) for crisp rendering at all DPR values. These are action-only — no active state tracking.

#### Structural Divider

`.huz-tb-sep` — a thin vertical rule separating Group 2 from Group 3.

#### Group 3 — Blocks

| Button label | `data-block` | Action | `.active` state |
|---|---|---|---|
| `—` | `hr` | `insertDividerBlock()` | No |

Action-only button. Inserts an `<hr>` at the current caret position followed by a new empty `<p>` to keep the cursor in a typeable block.

### 5.3 Toolbar Button Active State

Active state is applied exclusively to Group 1 (inline style) buttons. The update function runs:

1. After every `keyup` on `#huz-editor`
2. After every `mouseup` on `#huz-editor`
3. Immediately after any inline format command is executed from the toolbar

```javascript
function updateToolbarStates() {
  ['bold', 'italic', 'strikeThrough', 'underline'].forEach(function(cmd) {
    const btn = toolbar.querySelector('[data-cmd="' + cmd + '"]');
    if (btn) btn.classList.toggle('active', document.queryCommandState(cmd));
  });
}
```

This gives the user continuous feedback about the format at their cursor position even when no text is selected (caret-in-formatted-run detection).

### 5.4 Supported Block Insertions

| Action | HTML produced |
|---|---|
| Bullet List | `<ul><li>` — wraps selected text or creates new empty list item at caret |
| Numbered List | `<ol><li>` — same behavior |
| Horizontal Divider | `<hr>` followed by `<p><br></p>` — caret placed in the new `<p>` |

> **Default block:** `<p>` is the browser's default for Enter in `contenteditable`. It is not a toolbar button but is the natural output of pressing Enter in normal text or after block conversions.

> **Retained CSS:** The editor CSS retains styles for `h1`, `h2`, `h3`, `blockquote`, and `pre` to correctly render any existing saved content that contains these blocks. They are not insertable from the v9 toolbar.

### 5.5 Editor Canvas (`#huz-editor`)

| Property | Value |
|---|---|
| `contenteditable` | `true` |
| `spellcheck` | `true` |
| `data-placeholder` | `"Add a description..."` |
| Width | Full internal width of `.huz-wrap` — no left gutter offset |
| `padding` | `16px` uniform on all sides (left padding now symmetric with right; no gutter compensation) |
| `flex` | `1` — fills all vertical space below toolbar |
| `min-height` | Carried forward from v8 (`180px`) |

### 5.6 Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl/Cmd + B` | Bold (browser native) |
| `Ctrl/Cmd + I` | Italic (browser native) |
| `Ctrl/Cmd + U` | Underline (browser native) |
| `Tab` | Indent (list) |
| `Shift + Tab` | Outdent (list) |
| `Enter` in H1/H2/H3 | Inserts new `<p>` after heading, places cursor |
| `Escape` | No floating elements to close — no action in v9 |

### 5.7 Bubble Sync

Unchanged from v8. `syncToBubble()` fires debounced at 500ms on `input`. `syncNow()` fires immediately on `blur`.

---

## 6. CSS Custom Properties & Design Tokens

All defined in `:root`. Carried forward from v8; unused tokens are retained for compatibility with any existing Bubble database content that may reference these styles.

```css
:root {
  --bg:          #ffffff;          /* editor + wrapper background */
  --surface:     #f7f7f5;          /* inline code background */
  --border:      #e8e8e4;          /* separators, toolbar bottom rule, wrapper border */
  --text:        #1a1a18;          /* primary text color */
  --text-muted:  #9b9b97;          /* placeholder, blockquote, strikethrough */
  --accent:      #534AB7;          /* purple — blockquote left bar */
  --accent-soft: #eeecfb;          /* light purple — available for active states if desired */
  --plus-idle:   #c7c7c2;          /* retained in :root; no longer used in v9 UI */
  --plus-active: #534AB7;          /* retained in :root; no longer used in v9 UI */
  --menu-shadow: 0 6px 28px rgba(0,0,0,0.10), 0 1.5px 5px rgba(0,0,0,0.06);
                                   /* retained in :root; no longer used in v9 UI */
  --tooltip-bg:  #1a1a18;          /* tooltip background — available for toolbar button tooltips */
  --radius:      8px;              /* wrapper and toolbar corner radius */
  --font-body:   'DM Sans', system-ui, sans-serif;
  --font-mono:   'JetBrains Mono', 'Fira Mono', monospace;
  --t:           0.15s ease;       /* standard transition for interactive controls */
}
```

---

## 7. Layout Geometry Reference

### 7.1 Vertical Container Stack

```
.huz-wrap
  ╔══════════════════════════════════════════════════════╗
  ║  .huz-toolbar  (position: sticky; top: 0)            ║
  ║  [B] [I] [S] [U]  │  [ul] [ol]  │  [—]             ║
  ║──────────────────────────────────────────────────────║  ← border-bottom: 1px solid var(--border)
  ║  #huz-editor  (flex: 1; contenteditable)             ║
  ║  (full width, uniform padding, grows to content)     ║
  ║                                                      ║
  ╚══════════════════════════════════════════════════════╝
  ↕ resize: vertical (drag handle at bottom-right corner)
```

### 7.2 Horizontal Canvas Layout

```
[wrapper left wall]
|←── #huz-editor (padding: 16px all sides) ─────────────→|
|    full internal width; no gutter column                 |
|    text cursor start: 16px from left wall                |
```

There is no `.huz-plus-col` offset. The editor canvas occupies the full internal width of `.huz-wrap`.

### 7.3 Toolbar Horizontal Layout

```
[toolbar left padding]
  [B]  [I]  [S]  [U]   |sep|   [ul-svg]  [ol-svg]   |sep|   [—]
  ←── Group 1 ─────────────── ←── Group 2 ─────────────── ←── Group 3 ──→
[toolbar right padding]
```

Separators (`.huz-tb-sep`) are thin vertical rules approximately `16–18px` tall, matching the visual weight convention from v8's inline toolbar separators.

### 7.4 Key Measurements

| Element | Property | Value | Rationale |
|---|---|---|---|
| `.huz-wrap` | `display` | `flex; flex-direction: column` | Vertical stack: toolbar above canvas |
| `.huz-wrap` | `min-height` | `180px` | Compact initial frame height; carried from v8 |
| `.huz-wrap` | `overflow-x` | `hidden` | Prevents horizontal overflow bleed |
| `.huz-wrap` | `overflow-y` | `auto` | Native scrollbar when content exceeds height |
| `.huz-wrap` | `resize` | `vertical` | Native browser drag handle; must be on the scroll host |
| `.huz-toolbar` | `position` | `sticky; top: 0` | Prevents toolbar from scrolling with content |
| `.huz-toolbar` | `flex-shrink` | `0` | Toolbar never compresses; canvas takes remaining space |
| `.huz-toolbar` | `border-bottom` | `1px solid var(--border)` | Clean visual separation from canvas |
| `.huz-toolbar` | `background` | `var(--bg)` | Required for sticky to mask underlying canvas content |
| `.huz-toolbar` | `z-index` | Above canvas (e.g. `10`) | Prevents canvas text appearing over toolbar during scroll |
| `#huz-editor` | `flex` | `1` | Fills remaining height below toolbar |
| `#huz-editor` | `padding` | `16px` all sides | Symmetric; no left offset needed without gutter |
| `#huz-editor` | `min-height` | `180px` (inherited intent) | Canvas always fills visible area at minimum |
| `.huz-tb-btn` | `border-radius` | `999px` | Pill chip; consistent with v8 button aesthetic |
| `.huz-tb-sep` | `width` | `1px` | Thin divider rule |
| `.huz-tb-sep` | `height` | `~16–18px` | Aligns with button cap height |
| Removed: `.huz-plus-col` | `width` | — | No longer present |
| Removed: `#huz-plus-btn` | all | — | No longer present |
| Removed: `#huz-block-menu` | all | — | No longer present |
| Removed: `#huz-inline-toolbar` | all | — | No longer present |

---

## 8. State Machine — Height & Scroll Behavior

Height and scroll behavior is driven entirely by CSS in v9, consistent with v7/v8. There are no JS height state changes.

```
Page Load / Blur state:
  .huz-wrap    → min-height: 180px; overflow-x: hidden; overflow-y: auto; resize: vertical
                 display: flex; flex-direction: column
  .huz-toolbar → sticky; full width; border-bottom visible; always rendered
  #huz-editor  → flex: 1; height: auto (grows with content)
  → Toolbar always visible at top; canvas sits directly below it

    │  user focuses editor
    ▼

Active / Focused state:
  .huz-wrap:focus-within → implementation-defined border or shadow to signal focus
  .huz-toolbar → unchanged; always visible
  #huz-editor  → caret active; toolbar Group 1 button .active states reflect caret format

    │  user selects text (non-collapsed selection)
    ▼

Selection state:
  .huz-toolbar → Group 1 button .active states update to reflect selection format
  → No floating popover; toolbar is always present and always reflects state

    │  user clicks toolbar button
    ▼

Format applied state:
  → editor retains focus (mousedown + preventDefault)
  → selection/caret intact
  → execCommand or block insertion runs immediately against live selection
  → updateToolbarStates() refreshes .active classes
  → syncToBubble() schedules debounced save

    │  user blurs editor (clicks outside)
    ▼

Blur state:
  .huz-wrap    → returns to idle border/shadow
  .huz-toolbar → unchanged; always visible
  syncNow()    → immediate save fires
```

---

## 9. Key JavaScript Functions & State Variables

### 9.1 State Variables

| Variable | Type | Purpose |
|---|---|---|
| `saveTimer` | `number \| null` | Debounce timer ID for `syncToBubble()` (500ms) |

> All floating-UI state variables from v8 have been removed in v9: `activeBlock`, `menuOpen`, `savedRange`, `inlineHideTimer`, `TOOLBAR_HEIGHT_APPROX`, `TOOLBAR_GAP`, `TOOLBAR_MIN_TOP_SPACE`.

### 9.2 Functions

| Function | Purpose |
|---|---|
| `init()` | Reads `data-initial-content`; injects into `#huz-editor` if not placeholder token and not empty |
| `syncToBubble()` | Debounced 500ms call to `bubble_fn_saveEditorData(editor.innerHTML)` |
| `syncNow()` | Immediate (non-debounced) save — called on `blur` |
| `updateToolbarStates()` | Queries `document.queryCommandState()` for `bold`, `italic`, `strikeThrough`, `underline`; toggles `.active` on corresponding `.huz-tb-btn` elements |
| `insertListBlock(type)` | Reads live selection; inserts `<ul>` or `<ol>` with a `<li>` child at the current caret/selection position; places cursor inside the new `<li>`; calls `syncToBubble()` |
| `insertDividerBlock()` | Reads live selection; inserts `<hr>` followed by `<p><br></p>` at current caret position; places cursor in the new `<p>`; calls `syncToBubble()` |
| `insertAfter(node, ref)` | DOM helper — inserts `node` after `ref` inside `#huz-editor`; appends if no ref |
| `placeCursorIn(el)` | Places collapsed selection at first text node inside `el` |

> The following functions from v8 are **removed** in v9 and should not be re-introduced: `getActiveBlock()`, `isCaretInsideEditor()`, `positionPlusBtn()`, `showOnFirstLine()`, `setButtonTop()`, `openMenu()`, `closeMenu()`, `restoreRange()`, `isValidEditorSelection()`, `showInlineToolbar()`, `hideInlineToolbar()`, `scheduleHideInlineToolbar()`, `updateInlineBtnStates()`, `onCursorMove()`.

### 9.3 Event Listeners

| Target | Event | Handler |
|---|---|---|
| `.huz-toolbar` | `mousedown` | Route to `execCommand` or block insertion based on `data-cmd` / `data-block`; `e.preventDefault()` to preserve editor focus and live selection |
| `#huz-editor` | `keyup` | `updateToolbarStates()` |
| `#huz-editor` | `mouseup` | `updateToolbarStates()` |
| `#huz-editor` | `input` | `syncToBubble()` |
| `#huz-editor` | `blur` | `syncNow()` |
| `#huz-editor` | `keydown` | Handle `Tab`/`Shift+Tab` for indent/outdent; `Enter` in H1–H3 for explicit paragraph insertion |

> The `document`-level `selectionchange` listener from v8 is **removed**. There is no floating toolbar to show or hide in response to selection changes.

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
  data-placeholder="Add a description..."
  data-initial-content="[Current page Note's Body]"
  style="cursor:text;color-scheme:light;"
></div>
```

Replace `[Current page Note's Body]` with your Bubble dynamic expression (e.g. `data-initial-content="[Current page Project's Description]"`).

### 10.3 Bubble Element Settings

| Setting | Required value |
|---|---|
| Element height | Fixed or Fit content |
| Toolbox plugin | Must be installed; `saveEditorData` function must be exposed |
| HTML Element | Paste the entire `huzlly_rich_editor_v9.html` content |

### 10.4 Note on `position: sticky` inside Bubble iframes

`#huz-toolbar` uses `position: sticky; top: 0`. In Bubble, HTML Elements are rendered inside iframes. `position: sticky` is resolved against the nearest scrolling ancestor, which is `.huz-wrap` (the scroll container). This is correct behavior: the toolbar will remain pinned to the visible top of the editor as the user types long content and the canvas scrolls vertically. No special Bubble configuration is needed.

The `TOOLBAR_MIN_TOP_SPACE` constant and all iframe viewport edge-detection logic from v8 are no longer present — these were only required for the `position: fixed` floating inline toolbar, which does not exist in v9.

---

## 11. Known Bug Fixes & Root Cause Log

Entries 1–18 document bugs from v1–v8 for historical reference. The v9 architecture pivot (entry 19) represents a systematic elimination of an entire class of floating UI bugs that could not be fully eradicated by incremental patching.

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
| 10 | `×` hover state did not darken icon | `.open:hover` selector not present | Added `#huz-plus-btn.open:hover { color: #1a1a18 }` (v7 June update) |
| 11 | Menu icons visually low in chips | Font descenders + default line-height shifting rendered baseline down | `line-height: 1` + `padding-bottom: 1px` on `.huz-menu-btn` (v7 June update) |
| 12 | Inline toolbar flickers on initial click (before drag-select) | `selectionchange` fires on `mousedown` collapse, then again on selection | 80ms debounce via `scheduleHideInlineToolbar()` — only hides if still collapsed after delay (v8) |
| 13 | Clicking a format button collapsed the selection | `mousedown` on toolbar triggered blur/selection-loss on editor | `e.preventDefault()` on `inlineToolbar.addEventListener('mousedown', ...)` (v8) |
| 14 | Inline toolbar clipped by `.huz-wrap` overflow | `position: fixed` inside a positioned overflow ancestor is clipped | Moved `#huz-inline-toolbar` outside `.huz-wrap` in DOM; `position: fixed` resolves to iframe viewport (v8) |
| 15 | Inline toolbar tooltip arrows pointed wrong direction | Block menu tooltips point down (below icon); inline toolbar sits above text so tooltip should appear above buttons | Inverted tooltip CSS on `.huz-inline-btn`: `bottom: calc(100% + 7px)` + `border-top-color` arrow (v8) |
| 16 | Inline toolbar clips or hides behind adjacent elements near editor top boundary | Rigid `top = selRect.top - toolbarHeight - GAP` had no floor check; toolbar overflowed iframe top when selection was on the first line | Adaptive two-pass placement: flip to `topIfBelow` when `topIfAbove < 10px`; horizontal clamped within editor client rect (v8 patch, June 2026) |
| 17 | Inline toolbar displayed as oval pill | `border-radius: 999px` on `#huz-inline-toolbar` produced an oval at natural width | Changed to `8px` (v8 patch, June 2026) |
| 18 | `+` button disappears after user types on first line, changes focus, and clicks back | `positionPlusBtn(null)` checked `document.activeElement !== editor` (always true at click time before focus settles); `getActiveBlock()` returns `null` for bare root-level text nodes | Added `isCaretInsideEditor()`; routed null path to `showOnFirstLine()`; `focus`/`click` listeners wrapped in `setTimeout(..., 0)` to allow browser selection to settle (v8 patch, June 2026) |
| 19 | **(Architecture-level)** Entire class of viewport-clipping, selection-tracking-loss, placement-drift, focus-timing, and first-line-regression bugs inherent to floating dynamic UI elements | All floating elements (`#huz-plus-btn`, `.huz-plus-col`, `#huz-block-menu`, `#huz-inline-toolbar`) required continuous tracking of DOM geometry, browser selection state, scroll offsets, and iframe viewport boundaries. Each new edge case (first line, list items, near-top selections, re-focus after blur) required a dedicated patch that often introduced a new regression. The cumulative complexity of `getActiveBlock()`, `isCaretInsideEditor()`, adaptive placement engines, `savedRange`/`restoreRange()`, `selectionchange` debouncing, and `setTimeout` deferral was proportional to the number of floating elements in the tree, not the number of user-facing features. | **Eliminated entirely by v9 architecture pivot.** Replacing all floating UI with a permanent fixed top toolbar removes the geometric tracking problem at its root. The toolbar has no position to compute, no state to debounce, and no viewport boundary to respect. Selection persistence is trivially solved by `mousedown` + `e.preventDefault()`. Active state is read synchronously on `keyup`/`mouseup`. The entire JS module contracts to: init, sync, toolbar command routing, block insertion helpers, and `updateToolbarStates()`. (v9, June 2026) |

---

## 12. Conventions & Code Style

- All CSS class names and IDs are prefixed with `huz-` to avoid conflicts with Bubble's global stylesheet.
- JavaScript is entirely vanilla — no external libraries, no `import` statements.
- `document.execCommand` is used for inline formatting. While deprecated in spec, it remains the most reliable cross-browser approach for `contenteditable` contexts.
- `mousedown` + `e.preventDefault()` is used (not `click`) for all toolbar button interactions — critical for preserving the editor's focus and the live selection when a formatting button is pressed.
- The save function is debounced at **500ms**. `syncNow()` fires immediately on `blur`.
- The entire JS is wrapped in an IIFE to avoid polluting global scope inside Bubble's iframe.
- Inline SVG is used for list icons (bullet + numbered) to ensure crisp rendering at all DPR values and consistent cross-browser appearance.
- Toolbar button active states are updated synchronously on every `keyup` and `mouseup` — no debounce required since the operation is a simple `queryCommandState` read with a class toggle.
- `setTimeout(..., 0)` deferred evaluation patterns from v8 are no longer needed and are not present in v9. All caret/selection state is reliable at the time toolbar interactions fire because `mousedown` + `e.preventDefault()` guarantees no focus transition occurs.
- Critical cursor CSS remains hoisted above the `@import` rule and inlined on relevant elements to prevent FOUC, per the v7 fix.
- CSS tokens (`--plus-idle`, `--plus-active`, `--menu-shadow`) that are no longer used in the v9 UI are retained in `:root` for backward compatibility with any Bubble workflows or custom CSS that may reference them.

---

## 13. Bubble Setup Checklist

- [ ] Paste the entire content of `huzlly_rich_editor_v9.html` into a Bubble **HTML Element**
- [ ] Install the **Toolbox plugin** in Bubble
- [ ] Create a Bubble workflow exposing a function named `saveEditorData`
- [ ] Replace `data-initial-content="#BUBBLE_DYNAMIC_DATA#"` with your Bubble dynamic expression
- [ ] Verify `data-placeholder` reads `"Add a description..."`
- [ ] **Optional performance fix:** If Bubble loads DM Sans globally, delete the `@import` line
- [ ] Test content hydration on page load
- [ ] Test save trigger fires on both input (debounced) and blur (immediate)
- [ ] Test vertical resize handle is visible and draggable
- [ ] Test scrollbar appears when content exceeds editor height
- [ ] Verify the toolbar is **always visible** at the top of the editor in both idle and focused states
- [ ] Verify the toolbar **sticks** to the top when content is long enough to scroll the editor canvas
- [ ] Verify the toolbar is separated from the canvas by a clean single-pixel bottom border rule
- [ ] Verify the editor canvas is **full-width** with no left gutter offset
- [ ] Verify toolbar Group 1 (B, I, S, U) buttons exist and apply formatting to selected text
- [ ] Verify Bold button shows `.active` state when caret is inside bold text
- [ ] Verify Italic button shows `.active` state when caret is inside italic text
- [ ] Verify Strikethrough button shows `.active` state when caret is inside strikethrough text
- [ ] Verify Underline button shows `.active` state when caret is inside underlined text
- [ ] Verify active states update correctly when caret moves between formatted and unformatted runs using arrow keys
- [ ] Verify toolbar Group 2 (Bullet List, Numbered List) inline SVG icons render correctly
- [ ] Verify Bullet List button inserts a `<ul><li>` at the caret position
- [ ] Verify Numbered List button inserts an `<ol><li>` at the caret position
- [ ] Verify toolbar Group 3 (—) inserts an `<hr>` followed by a new `<p>` at the caret position
- [ ] Verify caret is placed correctly inside the new list item or paragraph after each block insertion
- [ ] Verify clicking any toolbar button does **not** collapse the text selection or lose editor focus
- [ ] Verify two `.huz-tb-sep` vertical separators are visible between the three button groups
- [ ] Confirm **no floating `+` button** appears anywhere in the editor
- [ ] Confirm **no floating block menu** appears anywhere in the editor
- [ ] Confirm **no contextual inline toolbar** appears above or below text selections
- [ ] Verify `Escape` key has no regressions (no floating elements to close in v9)
- [ ] Verify existing saved content containing `<h1>`–`<h3>`, `<blockquote>`, and `<pre>` renders correctly (CSS retained for legacy content compatibility)

---

*This file is the canonical knowledge base for the rich-text-editor project. All implementation decisions, architectural choices, and bug resolutions should be recorded here. The source of truth for actual code is `huzlly_rich_editor_v9.html`.*
