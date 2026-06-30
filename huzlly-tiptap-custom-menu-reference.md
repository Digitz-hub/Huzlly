# Huzlly — Custom Tiptap Menu System Reference

**Project**: Huzlly (Bubble.io SaaS)
**Component**: Custom floating toolbar for "Rich Text Editor (Tiptap.dev)" Bubble plugin
**File**: `huzlly-custom-bubble-menu.html` → paste into a page-level Bubble Custom HTML element
**Status**: Working. Iteratively bug-fixed. See section refs for each fix.

---

## 1. Why this exists

The Bubble Tiptap plugin's built-in Bubble/Floating Menus break inside a **Repeating Group** because every row gets the same static HTML id (`tip-tap-menu-proposal`). Browsers only resolve duplicate IDs to the first match → wrong position lookups, stale menus across instances. DOM-patching didn't help (plugin caches references at init). **Solution: bypass the plugin menus entirely, drive everything off the raw Tiptap `editor` instance.**

---

## 2. Accessing the editor + critical environment facts

```js
// One .ProseMirror per editor instance, no iframes
document.querySelectorAll('.ProseMirror')[0].editor          // raw Tiptap instance
document.querySelectorAll('.ProseMirror')[0].editor.instanceId  // unique per instance
```

**Extension manifest (confirmed loaded):**
`paragraph, link, textStyle, editable, commands, focusEvents, keymap, doc, text, ListItem, characterCount, hardBreak, undoRedo, bold, italic, strike, fontFamily, color, heading, bulletList, orderedList, taskList, highlight, underline, code, blockquote, horizontalRule, table, tableRow, tableHeader, tableCell, placeholder, textAlign, subscript, superscript, fontSize, ListKeymap, preserveAttributes, fileHandler`

**`gapcursor` is NOT loaded** despite the Bubble plugin UI toggle. Verify: `editor.extensionManager.extensions.some(e => e.name === 'gapcursor') // false`

### ⚠️ Bundle is minified — two critical consequences

1. `constructor.name` is mangled (returns single letter). **Never branch on `constructor.name`.** Use duck-typing instead (e.g. `selection.$anchorCell` to detect `CellSelection`).
2. `window.tiptap` exposes extension classes by name, but ProseMirror internals (e.g. `CellSelection`) are not exposed on `window`.

### `preserveAttributes` extension — how it actually works

Exposes a single Tiptap attribute named `"data-attributes"` whose value is an object of all `data-*` pairs. To persist custom attributes, always use the command system:
```js
editor.chain().focus().updateAttributes('table', {
  'data-attributes': { 'data-my-key': 'value' }
}).run();
```
Direct DOM writes (`setAttribute`, `classList`) are **invisible to `editor.getHTML()`** which reads from `editor.state.doc`, not the live DOM. DOM writes are fine for UI-only state (hover, dropdowns) — never for anything that needs to survive Bubble's save/reload.

### Discovering available commands
```js
Object.keys(editor.commands).filter(k => /select|cell|table/i.test(k))
```

---

## 3. Architecture

Single `<script>` IIFE, no build step (Bubble Custom HTML, plain ES5-ish JS). Three floating UI singletons appended once to `document.body`, repositioned per active editor:

| Element | Trigger | Purpose |
|---|---|---|
| `#huzlly-custom-bubble-menu` | Non-empty text selection (not CellSelection) | Bold, Italic, Underline, Link, H1–H4, lists, align, HR, Insert Table, **Add Image** |
| `#huzlly-table-toolbar` | Cursor inside `tableCell`/`tableHeader` OR active `CellSelection` | Row/col add/delete, merge/split, delete table |
| `#huzlly-floating-menu` | Collapsed cursor on empty paragraph (not inside table) | Insert Table, Bullet List, Numbered List, HR, **Add Image** |
| `#huzlly-link-popover` | Link button in Bubble Menu | Set/edit/remove `href` |
| `#huzlly-image-popover` | Image button in Bubble Menu or Floating Menu | URL input OR device file picker → `setImage({ src })` |

**Auto-discovery**: `querySelectorAll('.ProseMirror')` → `.editor`, polled every 300ms + `MutationObserver` on `document.body`. Bound editors tracked in a `WeakSet` (`boundEditors`) — listeners attached once per instance.

---

## 4. Bubble Menu — behavior & bugs fixed

### Show/hide
- **Show**: `selectionUpdate` with non-empty selection (20ms debounce). NOT on `focus` — that caused flash with collapsed cursor.
- **Hide**: 120ms debounce (`scheduleHide`) — prevents flicker when mouse moves from text into menu.

### Key bugs fixed

| # | Bug | Fix |
|---|---|---|
| 4.3 | Menu hid on `mouseleave` even with text still selected | `mouseleave` now checks `selection.empty` + `isFocused` before hiding |
| 4.4 | Menu flashed on tap into different editor | Removed show logic from `focus` handler; only `selectionUpdate` drives show |
| 4.5 | Didn't hide after toolbar button click → outside click | Global `click` capture-phase listener as fallback |
| 4.6 | Active-state highlight lost on hover-leave | Each button tracks `btn.dataset.active`; hover handlers read it |
| 4.7 | Menu appeared mid-drag (character by character) | `isMouseDown` flag; show deferred to `mouseup` when dragging |
| 4.8 | Drag ending outside editor didn't show menu | `mouseup` re-scans all `.ProseMirror`, shows menu for focused editor with non-empty selection |
| 4.9 | Cell-drag triggered Bubble Menu instead of Table Toolbar | `isCellSelection()` duck-type check (`selection.$anchorCell`); routes to `scheduleHide` instead of `showMenuFor` |

### Right-click → show Bubble Menu
`contextmenu` listener on `editor.view.dom` per instance (inside `bindEditor`). Calls `e.preventDefault()` (suppresses native browser menu). Uses `positionMenuAtCoords(clientX, clientY)` — raw mouse coords instead of selection coords.

| Situation | Result |
|---|---|
| Right-click with text selected | Menu appears above click point |
| Right-click with no selection | Menu appears above click point, full toolbar accessible |
| Right-click inside table (CellSelection) | Skipped — Table Toolbar handles |
| Right-click with Ctrl/Meta | Skipped — OS shortcuts unaffected |

---

## 5. Table Toolbar — behavior & bugs fixed

### Trigger (`checkTableContext`)
Checks `editor.isActive('tableCell' | 'tableHeader')` with fallback walking `$from.path`.
**Priority rule**: if `!selection.empty` (text selected) → hide Table Toolbar, show Bubble Menu. Exception: `CellSelection` shows Table Toolbar.

### Structure

| Group | Dropdown items |
|---|---|
| **Rows** | Add Row Above, Add Row Below, ─, Delete Row |
| **Columns** | Add Column Left, Add Column Right, ─, Delete Column |
| **Cells** | Merge Cells, Split Cell, ─, *VERTICAL ALIGN*, Align Top, Align Middle, Align Bottom |
| *(standalone)* trash icon | Delete Table |

Dropdown behavior: click group → opens below toolbar (flips up if near top). Click another group → swaps. Click item → runs action + closes (toggle items stay open). Outside click → all close, toolbar hides.

### Key bugs fixed

| # | Bug | Fix |
|---|---|---|
| 5.5 | Cell click while dropdown open hid entire toolbar | Removed `isFocused` guard from `checkTableContext` — event firing is sufficient proof |
| 5.5.1 | Outside-click safety net checked only previous active editor | Now checks click against every `.ProseMirror` on page; backs off if inside any editor |

**Rule**: never gate floating-UI visibility on `editor.isFocused` in this Bubble environment — it reports `false` unreliably at exactly the moments `selectionUpdate` fires. This caused bugs in both `checkTableContext` and `checkFloatingMenuContext`.

---

## 6. Border toggle — REMOVED

Built and fully working, then removed by product decision. Lesson learned (see §2 re: `preserveAttributes`) — don't repeat it.

---

## 7. Floating Menu (`+` button + slide-in menu)

### UX
- Empty paragraph → small `+` button appears 10px left of editor edge, vertically centered on cursor line
- `+` click → menu slides in from the right (0.18s CSS transition via `fm-open` class + `requestAnimationFrame`)
- Item click → action + hide both
- Cursor moves / outside click → hide

### `isOnEmptyParagraph` — all four must be true
1. `sel.empty === true`
2. Parent node type is `paragraph`
3. `node.content.size === 0`
4. NOT inside `tableCell`/`tableHeader`

### Key issues fixed
- `isFocused` guard removed (same Bubble unreliability — see §5 note)
- Added `focus` event hook (20ms delay) because `selectionUpdate` doesn't fire on re-click of already-empty line
- `mouseleave` hide removed — hide is click/selection-driven only

---

## 8. Add Image feature

**Image Popover** (`#huzlly-image-popover`) — same visual language as link popover:
- **URL tab**: paste/type URL → Enter or Insert button
- **Device tab**: native file picker → `FileReader` → base64 `data:` URL → insert immediately (no server upload)
- Cancel / Escape / outside click → close
- Command: `editor.chain().focus().setImage({ src }).run()` (uses `fileHandler` extension)

**Bubble Menu** — `imageBtn` after Table button, toggles popover.

**Floating Menu** — `fImageBtn` built manually (NOT via `makeFloatingButton`) so it deliberately does NOT call `hideFloatingSystem()` on click — the popover needs to stay open.

**Cleanup**: popover closes on `editor.destroy`, outside click (checks both `imageBtn` and `fImageBtn` as anchors), Cancel/Apply/Enter/Escape.

---

## 9. Multi-cell selection

ProseMirror Tables creates `CellSelection` natively on cell-drag. Bug was our handlers misread it as text selection. Fix: `isCellSelection(editor)` duck-typed via `selection.$anchorCell`.

**Visual highlight** (auto via ProseMirror's `selectedCell` class):
```css
.ProseMirror table td, .ProseMirror table th { position: relative; }
.ProseMirror .selectedCell:after {
  content: ""; position: absolute; inset: 0;
  background: rgba(127, 119, 221, 0.3); pointer-events: none;
}
```

---

## 10. CSS fixes applied

### Table layout (prevents cell width creep)
```css
.ProseMirror table { border-collapse: collapse; width: 100%; table-layout: fixed; }
.ProseMirror table td, .ProseMirror table th { border: 1px solid #ced4da !important; min-width: 40px; word-break: break-word; }
```

### Column resize scrollbar bugs
```css
.ProseMirror { overflow-x: auto; overflow-y: visible; position: relative; }
.ProseMirror .column-resize-handle {
  position: absolute; right: -2px; top: 0; bottom: 0;
  width: 4px; background: #7F77DD; pointer-events: none; z-index: 20;
}
```

---

## 11. Google-Docs-style column resize (fixed total width)

### Problem
Stock `prosemirror-tables` `columnResizing` only updates the dragged column's `colwidth`, so total table width grows/shrinks freely → overflow.

### Fix
`appendTransaction` plugin installed per editor via `editor.registerPlugin()` (can't `.extend()` the bundled Table extension — minified build). Registered as a duck-typed plain object (no `instanceof` check in ProseMirror's EditorState):
```js
{ key: pluginKey, props: {}, spec: { key: pluginKey, appendTransaction: fn } }
```

**Detection**: `tr.before.content.size === tr.doc.content.size` (attr-only transaction — resize ops never change doc size).

**Logic**: detects dragged column → reads existing widths → redistributes delta proportionally across all other columns, clamped at min floor → dispatches corrective transaction in same cycle (invisible to user).

**Diagnostic**: logs `[GDocsResize] installed for editor instance <id>` on success. Always verify in Preview console after pasting a new script version.

---

## 12. Percentage-based column widths (portable HTML)

### Problem
Pixel `colwidth` values are meaningless when saved HTML is rendered on a different-width page.

### Fix
- Column widths stored as `style="width:NN.NN%"` on each cell (via `tr.setNodeMarkup` → goes through `editor.state.doc` → persists via `preserveAttributes`)
- `colwidth` always set to `null` (prevents ProseMirror's `<col style="width:Npx">` from rendering)
- `normalizeToPct()` ensures sum always = exactly 100 → table can never exceed container width
- Dragged column detected as the one non-null `colwidth` in the incoming transaction
- `MIN_COL_PCT = 5` floor (replaces old pixel `MIN_COL_WIDTH = 40`)
- Colspan cells get aggregate `%` (sum of their columns); split evenly on read-back

**Data flow:**
```
Drag → stock plugin writes colwidth (px) for one column
     → appendTransaction: px → % → redistribute → write style% on all cells, colwidth: null
     → Bubble saves: <td style="width:33.33%"> (no <col>, no px)
     → Renders correctly at any container width
```

---

## 13. List exit + table escape route

### Problems
1. No keyboard way to exit a Bullet/Numbered List (Enter on empty item didn't lift out)
2. Tables trap the cursor — `gapcursor` is not loaded (§2)

### Fix (both as `editor.registerPlugin()` duck-typed plugins)
- **List exit**: `props.handleKeyDown` intercepts `Backspace` at offset 0 of empty `listItem` → calls `toggleBulletList()`/`toggleOrderedList()` → lifts item out
- **Table escape**: `appendTransaction` scans top-level doc children; inserts `paragraph` after any `table` not followed by one. Also runs once at install time (`ensureTrailingParagraphAfterTables`) to fix pre-existing content.
- Both installed via `installListExitAndTableEscapeRoute(editor)` from `bindEditor()`, guarded by `editor.__listExitTableEscapeInstalled`

**Status**: deploy unconfirmed in production (§13.4 below). Keyboard workaround: Up/Down + Enter navigates above/below table via stock ProseMirror behavior.

### 13.4 Deploy verification
If `ed.__listExitTableEscapeInstalled` is `undefined` in console, the patched script isn't running. Always test in Bubble **Preview mode** (not canvas). Check `ed.__gdocsResizeInstalled` too.

---

## 14. Code review cleanup (no user-visible changes)

1. **`getActive` duplication** — `makeGroup()` now stores `item.getActive` on `row.__getActive`; `syncDropdownToggleStates()` calls it. Deleted the hand-rolled duplicate check and `row.dataset.alignValue` patch-up.
2. **Dead functions removed** — `getActiveTableElement()` and `getTableCellRange()` were leftovers from removed "Select Entire Table" feature. Deleted. (To re-add: walk `$from` to enclosing `table`, find first/last `tableCell`/`tableHeader` via `.descendants()`, call `editor.commands.setCellSelection({ anchorCell, headCell })`.)
3. **`window.prompt` replaced** — Link button now opens `#huzlly-link-popover` (styled inline panel, same dark/purple language as rest of toolbar). Pre-filled with existing `href`. Apply/Remove/Enter/Escape handling. Force-closed in `editor.destroy`.
4. **`mouseup` safety net hardened** — `isFocused` alone is unreliable; now corroborated with `document.activeElement` check (either true = sufficient).

---

## 15. Quick command reference

```js
// Text formatting
ed.chain().focus().toggleBold().run()
ed.chain().focus().toggleItalic().run()
ed.chain().focus().toggleUnderline().run()
ed.chain().focus().extendMarkRange('link').setLink({ href }).run()
ed.chain().focus().extendMarkRange('link').unsetLink().run()
ed.chain().focus().toggleHeading({ level: 1 }).run()  // 1–4
ed.chain().focus().toggleBulletList().run()
ed.chain().focus().toggleOrderedList().run()
ed.chain().focus().setTextAlign('left' | 'center' | 'right' | 'justify').run()
ed.chain().focus().setHorizontalRule().run()
ed.chain().focus().setImage({ src }).run()  // src = URL or base64 data:

// Table
ed.chain().focus().insertTable({ rows: 3, cols: 3, withHeaderRow: true }).run()
ed.chain().focus().addRowBefore().run()
ed.chain().focus().addRowAfter().run()
ed.chain().focus().deleteRow().run()
ed.chain().focus().addColumnBefore().run()
ed.chain().focus().addColumnAfter().run()
ed.chain().focus().deleteColumn().run()
ed.chain().focus().mergeCells().run()
ed.chain().focus().splitCell().run()
ed.chain().focus().setCellAttribute('verticalAlign', 'top' | 'middle' | 'bottom').run()
ed.chain().focus().deleteTable().run()
ed.commands.setCellSelection({ anchorCell, headCell })  // doc positions

// State queries
ed.isActive('bold' | 'italic' | 'underline' | 'link')
ed.isActive('heading', { level: 1 })
ed.isActive('bulletList' | 'orderedList')
ed.isActive({ textAlign: 'left' | 'center' | 'right' | 'justify' })
ed.isActive('tableCell' | 'tableHeader')

// Editor internals
ed.state.selection            // { from, to, empty, $from, $anchorCell? }
ed.view.coordsAtPos(pos)      // { top, left, bottom, right } — viewport coords
ed.view.dom                   // .ProseMirror contenteditable root
ed.getAttributes('link').href
ed.getAttributes('tableCell') // { verticalAlign? }
ed.instanceId                 // unique per instance

// ResolvedPos
$pos.depth / $pos.node(depth) / $pos.before(depth)
node.descendants(fn)          // fn(node, posRelativeToNode)
```

---

## 16. Fast context for a new AI session

- **Bubble.io app**, Tiptap Rich Text Editor plugin, buggy in Repeating Groups (duplicate IDs — §1)
- **Bypassed plugin menus** entirely; raw `editor` instance via `.ProseMirror` DOM node → `.editor`
- **One `<script>` tag**, page-level Custom HTML element, no build step, plain JS
- **Table Toolbar**: 3 dropdown groups (Rows, Columns, Cells) + standalone Delete Table button
- **Column resize**: Google-Docs-style fixed-total-width via `appendTransaction` plugin (`editor.registerPlugin()` duck-typed object) — §11
- **Column widths**: stored as `style="width:%"` per cell, `colwidth` always nulled — saved HTML is portable — §12
- **Link button**: custom popover `#huzlly-link-popover`, not `window.prompt()` — §14 item 3
- **Image button**: in both Bubble Menu and Floating Menu, URL or device file picker — §8
- **Right-click** inside editor → Bubble Menu at mouse coordinates — §4 (contextmenu)
- **Never gate on `editor.isFocused`** in this env — unreliable at exactly the wrong moments
- **Never direct-DOM-write anything that needs to persist** — `getHTML()` reads `editor.state.doc` only
- **Always test in Bubble Preview mode** (not canvas). Console open. Verify `__gdocsResizeInstalled` per instance.
- Attach the actual `.html` script file alongside this `.md` — this is narrative/rationale only.
