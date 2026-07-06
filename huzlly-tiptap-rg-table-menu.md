# Huzlly — Custom Tiptap Table Menu inside Repeating Group

## Problem this solves
Same root problem as the bubble-menu and floating-menu scripts: any native
plugin mechanism that relies on `document.getElementById(id)` breaks inside a
**Repeating Group**, since every row's element can share the same static ID
and `getElementById()` only ever returns the first match.

This script bypasses that entirely for a **table-context menu** — the one
that should appear with merge cell / split cell / add row / add column /
delete row / delete column / delete table actions. It talks directly to each
row's raw Tiptap `Editor` instance and shows/hides/positions a **custom menu
element** based on real editor state, so it works correctly for every row.

**Important:** This script does NOT run any table commands itself (no
`mergeCells()`, `addRowAfter()`, etc. from this code). Those are applied
entirely by your existing Bubble workflows attached to each menu button.
This script's only job is: show the menu, hide the menu, position the menu
correctly.

---

## How it finds the editor instance for each row
Identical mechanism to the other two scripts:

```
wrapper (your ID Attribute, e.g. tiptap-proposal-0)
  → find child whose id starts with "tiptapEditor-"
    → find its child with class "ProseMirror"
      → .editor  ← the real Tiptap Editor instance
```

This logic lives in `getEditorFromWrapper()`.

---

## When the menu shows — `getActiveCellElement()`

The menu should appear for **any** of these:
- the cursor is just sitting inside a cell (no selection)
- a row, column, or multiple cells are selected (drag-select)
- the whole table is selected

It should **not** appear when actual text is highlighted inside a cell —
that's the bubble menu's job instead.

**Key fact this relies on:** in `prosemirror-tables`, a whole-table
selection, a row selection, and a column selection are **all** implemented as
the same `CellSelection` class — there is no separate "table selected"
selection type. Its `toJSON().type` is `'cell'`, the same trick the
bubble-menu script already uses to detect `'text'` selections, so no import
of the `CellSelection` class is needed.

So the logic in `getActiveCellElement()` is:

1. If `selection.toJSON().type === 'text'` and the selection is **not**
   empty (real highlighted text) → return `null` (hide; bubble menu's turn).
2. If `selection.toJSON().type === 'cell'` (a `CellSelection` — row, column,
   multi-cell, or whole-table) → anchor to `selection.$anchorCell` (the cell
   where the drag started), resolved via `view.nodeDOM(selection.$anchorCell.pos)`
   directly to the `<td>`/`<th>` element (see changelog #12 for why this
   isn't `view.domAtPos()` like the other case below).
3. Otherwise (plain cursor, `TextSelection`/`GapCursor`) → anchor to
   `selection.$from`, resolved via `view.domAtPos(pos)`, walking up to an
   Element, then `.closest('td, th')`. If there's no `td`/`th` ancestor, the
   cursor isn't in a table at all, and this correctly returns `null`.

This single function doubles as both the "are we in a table" check **and**
the anchor-element lookup — no separate boolean needed.

---

## Show / hide logic — direct focus/blur gating

Earlier versions of this script (see changelog) tried to derive visibility
purely from selection state, refreshed either reactively (`selectionUpdate`/
`transaction`) or on every `tryAttachAll()` rescan. Both approaches had the
same underlying flaw: **ProseMirror's selection does not clear itself when
the editor loses focus.** So even after clicking outside, the "cursor is in a
table cell" condition kept reading `true`, and anything that re-ran the check
(most commonly a `MutationObserver`-triggered rescan, which fires on *any*
DOM mutation anywhere on the page — extremely frequent on a live Bubble page)
would re-show a menu the user had just dismissed.

The fix: gate visibility directly on real editor focus, via `editor.on('blur', ...)`
to hide, and `editor.on('selectionUpdate', ...)`/`editor.on('transaction', ...)`
to show/reposition — rather than deriving it indirectly from selection state
plus incidental rescans.

- **`blur`** → hides the menu directly and immediately, **unless** the
  mousedown that caused this blur landed on the menu itself.
- **`selectionUpdate` / `transaction`** → recompute and show/hide/reposition
  based on the current selection. Deliberately **not** `focus` — see below.

**Why there's no `editor.on('focus', ...)` listener:** an earlier version
called `updateMenuForIndex()` on `focus` too, so the menu would show right
away if the cursor happened to already be sitting in a table cell. But
`focus` fires *before* ProseMirror has resolved a click to its new selection
— so an immediate check on `focus` reads the OLD (pre-click) selection,
which could flash the menu in the previous cell for an instant before the
real `selectionUpdate`/`transaction` fires a moment later with the correct
position. Since `selectionUpdate`/`transaction` already fire immediately
after `focus` with the settled position, nothing is lost by not reacting to
`focus` itself.

That last exception is necessary for the same reason already documented in
the bubble-menu script (changelog #3) and the floating-menu script (changelog
#8): clicking a menu button causes `mousedown` to blur the editor a moment
*before* that button's own Bubble workflow runs. Without the exception, the
`blur` handler would hide the menu the instant a button is clicked, before
the click's action ever visibly completes.

**How the exception is implemented:** a shared `mousedownWasInsideMenu` flag
is set inside the existing document-level `mousedown` capture-phase
listener — which fires *before* the browser's internal focus-change/blur
dispatch for that same click. The `blur` handler checks this flag: if it's
`true`, it does nothing (leaves the menu visible so the click can complete);
if `false`, it hides the menu right away.

**The outside-click listener still exists too** (same as the other two
menus) — it force-hides a menu if a click lands outside both the menu and
its matching editor wrapper. This remains useful as a backstop (e.g. for
clicks that land somewhere focus-tracking wouldn't otherwise catch), and it's
also where `mousedownWasInsideMenu` gets computed.

**`selectionUpdate`/`transaction` are debounced via a resettable `setTimeout`,
not called directly (and not via a single `requestAnimationFrame` — see
changelog #11 for why that wasn't enough).** A blurred `contenteditable`
regaining focus can cause the browser to momentarily restore the *previous*
selection before a click resolves to its actual new position — firing an
old-selection event, followed shortly after by a correct one. These two
events are not guaranteed to land within the same animation frame (a normal
mousedown-to-mouseup click gesture can span more than one), so
`scheduleUpdateMenuForIndex()` instead resets a short timer (`SETTLE_DELAY_MS`,
50ms by default) on every call; the actual update only runs once the
selection has stopped changing for that long, so the stale intermediate
state never gets painted no matter how many frames the burst spans. The
`blur` handler itself is NOT debounced — it hides the menu directly, since
an immediate hide there is always correct — but it does cancel any pending
scheduled update for that row first (see changelog #10/#11), so a queued
update from just before the blur can't override the hide a moment later.

**Showing also requires `editor.isFocused`, not just a matching selection**
(see changelog #10). `updateMenuForIndex()` checks `editor.isFocused` before
even calling `getActiveCellElement()`. This matters because the `scroll` and
`resize` listeners call `updateMenuForIndex()` directly, with no focus check
of their own — and ProseMirror's selection does not clear itself on blur. Without
this gate, scrolling the page after clicking away from a table cell would
recompute from that row's stale (but still "in table") selection and
resurrect a menu that had already been correctly hidden by `blur` or the
outside-click listener.

**Initial state is set once per row, at attach time — not on every
rescan.** `tryAttachAll()` still re-scans on `MutationObserver` + two
`setTimeout` safety nets, same as the other scripts, to catch rows that
render asynchronously. But `updateMenuForIndex(index)` is now only called
once, immediately after a row's listeners are attached for the first time —
not on every subsequent scan. Calling it on every scan was the original
source of the "have to click outside more than once" bug: a rescan
triggered by some unrelated page mutation would recompute from the stale
(still "in table") selection and re-show a menu that had just been hidden.
After the initial call, display is driven by real editor events (`blur`,
`selectionUpdate`, `transaction`), the `scroll`/`resize` listeners (gated by
`editor.isFocused`), and the outside-click listener.

---

## Positioning logic — `positionMenu()`

1. **Anchor cell**: `getActiveCellElement()` (see above) returns the actual
   `<td>`/`<th>` DOM element to anchor to — the cell where the selection
   started, or the cell the cursor is in.
2. **Horizontal**: anchored to the **start (left edge)** of the cell by
   default. Clamped so it can't run past the viewport edges (`8px` padding).
   A one-line swap in the code switches this to centered on the cell instead,
   if needed later.
3. **Vertical**: sits `8px` **below** the cell by default, flipping to
   `8px` above if there isn't enough room at the bottom of the viewport —
   same flip pattern as the other two menus, just with the default direction
   reversed (this menu prefers below; the other two prefer above).
4. **Transformed-ancestor correction**: identical technique to the
   floating-menu script's `getFixedContainingBlockOffset()`. If any ancestor
   of the menu element has a CSS `transform` other than `none` (common with
   Bubble's animated/conditional groups), that ancestor becomes the
   containing block for `position: fixed` descendants per the CSS spec — so
   fixed coordinates would otherwise be calculated relative to that
   ancestor's box instead of the real viewport. The script walks up from the
   menu element to find that ancestor and subtracts its own
   `getBoundingClientRect()` left/top from the target coordinates before
   applying them. The DOM structure is never touched — only the coordinate
   math adapts.

---

## Row re-scanning (MutationObserver)
Same strategy as the other two scripts — `tryAttachAll()` scans for all
wrapper elements matching `WRAPPER_ID_PREFIX`, attaches listeners (and sets
initial display state) for any row not yet attached, and a `MutationObserver`
on `document.body` re-runs this scan on every DOM change so newly-rendered
Repeating Group rows get picked up automatically. Two safety-net `setTimeout`
scans (1s, 3s) cover edge-case timing on initial page load.

---

## Configuration constants (top of the script)

| Constant | What it should be set to |
|---|---|
| `WRAPPER_ID_PREFIX` | The prefix of the ID Attribute on the Tiptap input element (before the row index number) — same value as the bubble-menu / floating-menu scripts |
| `MENU_ID_PREFIX` | The prefix of the ID Attribute on your custom table-menu element (before the row index number) |

Both assume the same per-row index suffix (`0`, `1`, `2`...) as generated by
"Current cell's index" in the Repeating Group.

---

## Setup checklist (Bubble side)

1. Tiptap input element → ID Attribute = `tiptap-proposal-[Current cell's index]`
   (already set from the other menu setups, reused here)
2. Your custom table-menu element (Group/element with merge cells, split
   cell, add/delete row, add/delete column, delete table buttons, etc.) →
   ID Attribute = `table-menu-tiptap-proposal-[Current cell's index]`
3. Each button → keep its existing Bubble workflow that runs the actual
   table command — this script never touches those workflows
4. Paste the script from `huzlly-tiptap-rg-table-menu.html` into an HTML
   element on the page (once per page, not per row) — can be the same HTML
   element already used for the other two scripts, or a separate one

**Worth adding on the Bubble side (not handled by this script):** "visible"
and "available" aren't the same thing for merge/split specifically. Merge
only makes sense with 2+ cells actually selected; split only makes sense on
a cell that's already merged (`colspan`/`rowspan` > 1). Consider gating
those two buttons' own conditionals on `editor.can().mergeCells()` /
`editor.can().splitCell()` so clicking them when unavailable doesn't silently
no-op.

---

## Known issues fixed during development (changelog)

1. **Same-ID-across-rows problem** — fixed the same way as the other two
   menus: bypass any native mechanism, key everything off the row index
   instead of a shared ID.
2. **Wrong horizontal anchor (menu appeared centered on the cell's right
   border instead of flush against it)** — root cause was anchoring math
   using `cellRect.right - menuWidth`, combined with `menuWidth` sometimes
   being read before the menu was actually visible. Fixed by anchoring to
   `cellRect.left` (start of cell) instead, with a commented one-line swap
   available for centered positioning if needed.
3. **Menu positioned above the cell when it should sit below** — swapped the
   default vertical placement from "above, flip below if no room" to "below,
   flip above if no room at the bottom of the viewport."
4. **Text selection inside a cell also triggered the table menu** — added an
   explicit check: if `selection.toJSON().type === 'text'` and the selection
   isn't empty (a real highlighted range), return `null` so the table menu
   hides and the bubble menu can take over instead.
5. **Menu visible on page load, before any interaction** — same root cause
   as the floating-menu script's changelog #5: the show/hide logic only ran
   reactively in response to editor events, so until the first event fired,
   the menu kept whatever default `display` Bubble gave it (visible). Fixed
   by calling `updateMenuForIndex()` once immediately after a row's listeners
   are attached.
6. **Fixed-position menu offset due to a transformed ancestor** — identical
   issue and fix as the floating-menu script's changelog #7: an ancestor
   with a CSS `transform` becomes the containing block for
   `position: fixed` descendants, throwing off the coordinate math even
   though the target `getBoundingClientRect()` values were correct. Fixed by
   adding `getFixedContainingBlockOffset()` and subtracting that ancestor's
   own rect from the target coordinates, without moving the menu in the DOM.
7. **Menu wouldn't hide on blur — took multiple outside clicks to dismiss** —
   root cause: visibility was being derived indirectly from selection state,
   refreshed on every `tryAttachAll()` rescan. Since ProseMirror's selection
   doesn't clear itself when the editor loses focus, "cursor is in a table
   cell" kept reading `true` after blur. Any subsequent rescan — most
   commonly one triggered by the `MutationObserver` firing on some unrelated
   DOM mutation elsewhere on the page — would recompute from that stale
   selection and re-show a menu the user had just dismissed by clicking
   outside, requiring another click (or several) to "win the race" against
   further incidental rescans. Fixed in two parts:
   - `tryAttachAll()` now calls `updateMenuForIndex()` only **once**, at the
     moment a row's listeners are first attached — not on every later scan.
   - Visibility is now gated directly on real focus state via
     `editor.on('focus', ...)` and `editor.on('blur', ...)`, so losing focus
     hides the menu immediately and directly, rather than waiting for an
     indirect signal.
8. **Direct blur-based hiding would have hidden the menu mid-click** — adding
   a `blur` handler reintroduces the exact class of bug already fixed in the
   bubble-menu script (changelog #3) and floating-menu script (changelog #8):
   clicking a menu button blurs the editor a moment before that button's own
   Bubble workflow runs. A naive `blur` → hide would close the menu before
   the click's action could visibly complete. Fixed by tracking a
   `mousedownWasInsideMenu` flag, set inside the existing document-level
   `mousedown` capture-phase listener (which fires before the browser's
   internal blur dispatch for that same click) — the `blur` handler checks
   this flag and skips hiding when the mousedown landed on the menu itself.
9. **Menu flashed visible in the previous cell for an instant when clicking
   back into the editor after having clicked away** — two related causes,
   fixed in sequence:
   - First cause: an `editor.on('focus', ...)` listener was calling
     `updateMenuForIndex()` immediately. But `focus` fires *before*
     ProseMirror has resolved the click to its new selection — so the
     immediate check read the OLD (pre-click) selection, which could still
     say "in table," showing the menu for an instant before the real
     `selectionUpdate`/`transaction` fired a moment later with the correct
     position and hid it again. Fixed by removing the focus-triggered check
     entirely — `selectionUpdate`/`transaction` already fire right after
     `focus` with the settled position, so nothing is lost by not reacting
     to `focus` itself.
   - Second, subtler cause: even without the `focus` listener, the same
     flash still happened when clicking into a *different table cell*
     specifically. A blurred `contenteditable` regaining focus can cause the
     browser to momentarily restore the PREVIOUS selection before the click
     resolves to its actual new cell — firing a `transaction`/
     `selectionUpdate` with the old (previous-cell) selection for an
     instant, immediately followed by another one with the real new
     position. Both fire synchronously, before any repaint, so removing the
     `focus` listener alone wasn't enough — the stale intermediate call
     could still paint before the next one arrived. Fixed by deferring the
     actual show/hide/position work (`updateMenuForIndex`) to
     `requestAnimationFrame` via a new `scheduleUpdateMenuForIndex()`
     wrapper: multiple back-to-back `selectionUpdate`/`transaction` calls
     within the same synchronous burst now collapse into a single scheduled
     frame, which reads whichever selection is final by the time it actually
     runs — so the stale intermediate state never gets painted to the
     screen. The `blur` handler itself remains a direct, unbatched call,
     since an immediate hide there is always correct and doesn't need to
     wait for anything to settle.
10. **Menu reappeared on scroll/resize after being correctly hidden** — root
    cause: the `scroll` and `resize` listeners called `updateMenuForIndex()`
    directly, with no check on whether the editor was actually focused.
    Since ProseMirror's selection does not clear itself on blur, any row
    whose editor had ever had a cursor placed in a table cell would still
    resolve to a valid `td`/`th` on every subsequent scroll or resize
    anywhere on the page — silently undoing both the `blur` hide and the
    outside-click hide. Fixed by requiring `editor.isFocused` inside
    `updateMenuForIndex()` before it's allowed to show the menu, so a
    genuinely blurred editor can no longer be resurrected by an unrelated
    scroll or resize event.
11. **Menu still flashed at its previous position on refocus, even after
    fix #9's `requestAnimationFrame` debounce** — root cause: the old-selection
    and corrected-selection events from a refocus are not guaranteed to land
    within the same animation frame. A normal mousedown-to-mouseup click
    gesture can take longer than one frame (~16ms), so the first
    (stale-selection) `selectionUpdate` could get its own scheduled frame and
    paint the menu at the old position *before* the corrected event arrived
    and scheduled a second, separate frame. Fixed by replacing the single-rAF
    debounce with a resettable `setTimeout` (`SETTLE_DELAY_MS`, 50ms): every
    call to `scheduleUpdateMenuForIndex()` now clears any previously pending
    timer and starts a new one, so the actual update only runs once the
    selection has stopped changing for the full delay — regardless of how
    many frames the burst spans. The `blur` handler was also updated to
    cancel this timer (previously it only cancelled a pending
    `requestAnimationFrame` handle), so a queued update from just before a
    blur can't fire afterward and undo the hide.
12. **Clicking a menu button hid the menu instead of running the button's
    Bubble workflow** — the existing `mousedownWasInsideMenu` flag correctly
    told the `blur` handler not to hide the menu, but that's a *reactive*
    fix: it tolerates the blur after the fact, it doesn't stop it from
    happening. The blur itself was still occurring, and since this
    project's table commands run through a plain Bubble workflow (not
    custom JS) whose plugin action targets "the currently focused editor,"
    losing focus for even a moment could make the command silently no-op —
    same underlying cause, worse symptom than a visual flash. Fixed by
    calling `e.preventDefault()` on the document-level `mousedown` listener
    whenever the click lands inside a menu, which stops the browser from
    ever shifting focus away from the editor in the first place — the same
    technique ProseMirror's own built-in menu bar uses. `preventDefault()`
    on `mousedown` does not block the subsequent `click`/`mouseup`, so the
    button's workflow still runs normally; there's just no `blur` event to
    race against anymore. The `mousedownWasInsideMenu`/`blur`-check logic
    was left in place as a backstop for edge cases (e.g. touch devices,
    where `preventDefault` on `mousedown` doesn't block focus the same way).
13. **Menu never appeared for row / column / whole-table selections — only
    for a plain cursor sitting in a single cell** — root cause was in
    `getActiveCellElement()`: for a `CellSelection`, `selection.$anchorCell.pos`
    points to the position immediately *before* the cell node, not to
    content inside it. `view.domAtPos()` at that kind of node-boundary
    position commonly resolves to the *parent* `<tr>` (with a child offset)
    rather than the `<td>`/`<th>` itself. Since `.closest('td, th')` only
    ever searches upward from a node, calling it on a `<tr>` correctly finds
    nothing and returns `null` — so every row/column/whole-table selection
    silently failed this check, even though the plain-cursor case (where
    `$from.pos` sits inside the cell's actual content, so `domAtPos`
    naturally resolves inside it) worked fine. Fixed by using
    `view.nodeDOM(selection.$anchorCell.pos)` for the `CellSelection` case
    instead — `nodeDOM()` is the correct tool for "this position points
    right before a node," returning that node's own DOM element directly
    with no boundary ambiguity. The plain-cursor case still uses
    `view.domAtPos()` as before, since that case doesn't have this problem.
