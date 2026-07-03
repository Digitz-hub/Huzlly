# Pickr Color Picker — Bubble.io Integration — Snapshot

**Last updated:** July 3, 2026
**Fixes covered:** #1–32 + housekeeping pass + UI polish pass (see session log below)

---

## Project context

Bubble.io project using a self-contained HTML element with the Pickr (`@simonwep/pickr@1.9.1`) colour picker library. Multiple instances of this HTML element exist on the same page inside a Reusable Element — one per editable colour field (e.g. `titleColor`, `textColor`, `bgColor`), duplicated 10–15 times.

**Files:**
- `pickr-instance-element.html` — the full per-instance code (only `rawColor` and `fieldName` change between instances)
- `pickr-library-loader.html` — single dedup-guarded loader for `pickr.min.js`, placed once inside the Reusable Element (Fix #31)
- `README.md` — Bubble setup guide (HTML element, Toolbox plugin, workflows)
- `pickr-snapshot.md` — this file

**Key design decisions:**
- Auto-save on popup close (outside click) — no SAVE button
- CANCEL button and Escape key both revert to last confirmed save and send `saved: false`
- `colorHex` custom state on each HTML element holds the last confirmed saved colour — only updated on `saved: true` to prevent Bubble re-rendering the element mid-edit and destroying the picker
- One `JavascripttoBubble` element (`colorData` suffix) handles all instances on the page
- `bubble_fn_colorData` guard (`safeBubbleSend`) prevents a ReferenceError if the Toolbox element hasn't initialized yet on first page load — payload is dropped silently but this race window is too small to matter in practice (user can't reach the picker before the bridge is ready)
- `defaultColor` is intentionally mutated on each confirmed save as an in-memory cache — safe because Bubble re-renders resync `rawColor` on every save anyway
- `pickr.min.js` is loaded exactly once per page via a single dedup-guarded loader element (`pickr-library-loader.html`), instead of once per instance (Fix #31)

---

## Architecture overview

Each instance runs inside an IIFE. On script parse:
1. Stamps `data-field` from `fieldName` onto the first unclaimed `.pickr-trigger-el` div, then marks it claimed — both at parse time (Fixes #24, #29)
2. Calls `initPickrInstance()`, which retries every 100ms until `Pickr` is available on `window`
3. On re-init, runs `cleanup()` on the old instance before destroying it (Fixes #25, #26)

On `init`: injects CANCEL button into Pickr's interaction panel.
On `show`: attaches scoped `keydown` listener for Escape-as-cancel.
On `change`: debounced 500ms send to Bubble with `saved: false`.
On `hide`: runs `cleanup()`, then either skips (if cancelled) or auto-saves with `saved: true`.

Bubble workflow trigger: `When JavascripttoBubble colorData has an event`
- Action 1 (all payloads): updates the target element's colour state — condition on `field` value
- Action 2 (`saved: true` only): updates `colorHex` on the HTML element — condition on `field` + `saved`

---

## Fix log

### #21 ⭐ Multi-instance trigger pairing
`document.querySelector('.pickr-trigger-el')` matched the same first element on the page for every instance. `document.currentScript.parentElement` didn't work either — Bubble relocates `<script>` tags so `parentElement` doesn't contain the picker's own div.

**Fix:** claim-the-next-unclaimed pattern. Each instance grabs the first `.pickr-trigger-el:not([data-pickr-claimed])` and immediately marks it claimed. Relies only on Bubble preserving script execution order (confirmed true).

**Watch for:** if a parent Reusable Element re-renders all instance scripts simultaneously, claimed markers could reset and re-claim out of order. Not observed. *(Superseded by Fix #29 — ordering assumption fully eliminated.)*

---

### #22 Hex-with-opacity validation
`toHEXA()` returns 8-digit hex (`#RRGGBBAA`) when opacity < 100%. Old regex only matched 3/6-digit, so any saved colour with opacity silently fell back to `#2563EB` on next page load.

**Fix:** regex updated to also accept 8-digit hex.

---

### #23 Cancel revert re-entered the debounce pipeline
`runCancel()` called `pickr.setColor(defaultColor)` without the `silent` flag, firing Pickr's internal `change` event and scheduling a new debounce send mid-cancel.

**Fix:** `pickr.setColor(defaultColor, true)` — silent flag prevents the change event.

---

### #24 ⭐ Trigger claim moved to script-parse time
Fix #21's claim ran inside `initPickrInstance()`, which is delayed by the Pickr-availability retry loop. Multiple instances polling simultaneously could resolve in setTimeout-queue order, not document order — wrong trigger pairing.

**Fix:** claim moved to synchronous script-parse time, before the retry loop starts.

---

### #25 Stale cleanup reference on re-init
Old `window[instanceKey + '_cleanup']` reference was left on `window` until the new cleanup overwrote it — small window where a dead closure sat referenced for no reason.

**Fix:** `delete window[instanceKey + '_cleanup']` immediately after `destroyAndRemove()`.

---

### #26 ⭐ Listener + timer leak on destroy-while-open
If Bubble re-rendered while a popup was open, `hide` never fired. The Escape `keydown` listener and any pending debounce timer stayed alive, holding a closure over the now-dead instance. The listener would misfire on every future Escape press; the timer could send a stale payload.

**Fix:** `cleanup()` function centralizes both removals — called from `hide` (normal path) and explicitly before `destroyAndRemove()` on re-init (abnormal path).

---

### #27 Production resilience (4 sub-fixes)
- `safeBubbleSend()` guards every `bubble_fn_colorData` call against a ReferenceError if the Toolbox element isn't ready yet
- Null-guards in `lockTriggerColor()` around `getRoot()` and the `color` argument
- CDN pinned to `@simonwep/pickr@1.9.1` explicitly
- `console.warn` calls gated behind `window.PICKR_DEBUG` (defaults `false`, set once via typeof guard so 10–15 instances don't clobber each other)

---

### #28 Silent failure on missing trigger element
Previously, if an instance's script ran but found no unclaimed `.pickr-trigger-el` (e.g. instance count grows past available trigger divs, or a div is accidentally deleted/mistyped when adding a new field), the only signal was a `console.warn` gated behind `PICKR_DEBUG` — invisible in production. The picker simply never bound and the trigger circle sat dead, with zero way to detect this without manually opening the console.

**Fix:** when no unclaimed trigger is found, the instance now also sends a payload to Bubble via `bubble_fn_colorData`: `{ field: fieldName, value: null, saved: false, error: 'no_trigger_found' }`. This is additive only — `value: null` means it won't satisfy existing Workflow 1/2 conditions (which expect a real hex string), so current workflows are unaffected. Intended to be picked up by a separate, optional dev-facing workflow (e.g. an error log table or notification) — not surfaced to end users, since this is a setup-level issue (missing/mismatched trigger div) that only a developer can fix, not something an end user causes or can act on.

**Root causes to avoid (so this should rarely fire):**
- Copying only the `<script>` block when adding a new field and forgetting the accompanying `<div class="pickr-trigger-el">` markup
- Manually typing the `pickr-trigger-el` class name instead of copy-pasting (typo risk)
- Adding a new script instance without a 1:1 matching trigger div (mismatched instance count vs. div count)
- Placing the trigger div inside conditional visibility logic that can leave it absent from the DOM at script-parse time

---

### #29 ⭐ Trigger pairing made fully order-independent
Fix #21's claim-the-next-unclaimed pattern still had one theoretical failure mode: if a parent Reusable Element triggered a simultaneous re-render of multiple instances, scripts could resolve in microtask/setTimeout order rather than document order — pairing script A with trigger B. Additionally, adding a new instance required updating `data-field` in the markup *and* `fieldName` in the script — two values that had to stay in sync manually.

**Fix:** `fieldName` is now the single source of truth. At parse time, the script grabs the first unclaimed `.pickr-trigger-el`, stamps `data-field` from `fieldName` onto it, marks it claimed, then re-queries by identity (`[data-field="..."]`). The div in markup stays generic — no `data-field` attribute to maintain. All subsequent lookups, including `initPickrInstance()` retries, are position-independent.

**Per-instance editing is now exactly 2 values** — `rawColor` and `fieldName` — same as before, with no third value to keep in sync.

**Why the first unclaimed grab is still safe here:** the initial stamp still uses positional order, but it's synchronous at parse time (same as Fix #24) and only runs once per instance. After stamping, identity takes over entirely — re-renders query by `data-field`, not position, so the ordering assumption doesn't persist.

---

### #31 ⭐ Single Pickr library load — deduplicated

**Problem:** every one of the 10–15 duplicated Pickr instance elements shipped its own `<script src="pickr.min.js">` tag, creating 10–15 concurrent loads per page. This was the root cause behind the detachment race that Fix #30 patches around (orphaned retry polling for `Pickr` while a re-render removes the trigger node).

**Attempted approaches, in order:**

1. App-wide Head Code (Settings → SEO/metatags → Script/meta tags in header) — added the single script tag here. Verified via view-source that it was not rendering in the page's `<head>` at all. Root cause unconfirmed — possibly a propagation delay, possibly a Free-plan restriction, possibly requires an actual deploy (not just test-version save). Not resolved, left as an open question.
2. Page-level SEO/metatags field (per-page HTML header, set on the Page itself rather than app-wide) — also did not take effect. Likely because the Pickr instances live inside a Reusable Element, which doesn't inherit the host page's header the same way.
3. Single HTML element inside the Reusable Element (final, working approach) — placed one dedicated HTML element inside the Reusable Element, separate from the 10–15 per-field trigger elements (`pickr-library-loader.html`), containing:

```html
<script>
if (!document.querySelector('script[src*="pickr.min.js"]')) {
  var s = document.createElement('script');
  s.src = 'https://cdn.jsdelivr.net/npm/@simonwep/pickr@1.9.1/dist/pickr.min.js';
  document.head.appendChild(s);
}
</script>
```

**Why the guard matters:** unlike a true page-load-once Head Code injection, this element lives inside a Reusable Element that can itself re-render. Without the `querySelector` dedup check, every re-render would insert another copy of the library — recreating the exact multi-load problem this fix exists to solve. The guard makes it safe regardless of how many times the Reusable Element remounts.

**Verified:** `document.querySelectorAll('script[src*="pickr.min.js"]').length` returns `1` on load, in the Bubble editor preview (`version-test`).

**Unchanged / still required:** Fix #30 (generation counter + `isConnected` guard) stays in place permanently in every per-instance script. It still covers the residual race window during the one remaining load, and covers any future re-render-while-loading edge case — this fix makes that window rare, not eliminated.

---

### #32 ⭐ Trigger sized/shaped to match the Bubble HTML element (not a fixed 24px circle)

**Problem:** trigger was hardcoded to `24px × 24px` with `border-radius:50%` — a fixed circle regardless of how the containing Bubble HTML element was sized or shaped. Requirement changed: the trigger should fill and match whatever size/shape the Bubble element itself is given (square, rounded-rect, any dimensions), fully controlled from the Bubble editor, not from this code.

**Fix, step 1 — percentage sizing:** `.pickr-trigger-el` and `.pcr-button` changed from `24px` to `width:100%; height:100%`, and `border-radius:50%` removed from both. Added `html, body { width:100%; height:100%; margin:0; padding:0; }` since Bubble renders HTML elements inside an iframe, which doesn't fill its parent by size automatically.

**Fix, step 2 — the actual bug this surfaced:** after step 1, the trigger rendered as a small rectangle in the corner instead of filling the element. Root cause: `Pickr.create()` replaces the original `.pickr-trigger-el` div in the DOM with its own wrapper (`<div class="pickr"><button class="pcr-button">`). That `.pickr` wrapper has no explicit size of its own (shrink-wraps to content), so `.pcr-button`'s `width:100%; height:100%` was resolving against the wrapper's auto size, not the full element. Fixed by forcing the wrapper itself: `.pickr { width:100% !important; height:100% !important; display:block !important; }`.

**Why `24px` didn't have this problem:** an absolute pixel value doesn't depend on an ancestor having an explicit size, so it rendered fine regardless of the `.pickr` wrapper's auto-sizing. Percentage sizing exposed the wrapper as the missing link.

**Also required in Bubble editor (not code-side):** the HTML element's **"Fit height to content"** property must stay **unchecked** — otherwise Bubble calculates the element's height from its content instead of the reverse, and the fill-parent CSS above has nothing stable to size against.

**Net effect:** trigger's size and shape are now fully driven by the Bubble HTML element's own dimensions and corner-radius — no size/shape values left in this code to keep in sync per instance.

---

### Housekeeping pass (no logic changes)
- `dynamicPosition` variable removed — `'left-middle'` inlined directly into `Pickr.create()`
- All Hindi inline comments replaced with English
- All block comments tightened and consistent
- Fix #27's comment in HTML corrected (was incorrectly labelled #2)
- Local variable `hexValue` in `change` and `hide` handlers renamed to `colorHex` to match the Bubble custom state name
- Bubble custom state renamed from `hexValue` to `colorHex` — README updated throughout
- Tip note text updated: `Changes save automatically when you close this picker.`

---

### UI polish pass (no logic changes)
- **CANCEL button restyled:** white background, `#e0e0e0` border, `#9e9e9e` text, `10px 24px` padding, `8px` border-radius, `transition: all .3s ease`. Hover state turns border `#ffcdd2` and text `#ef5350` (red-tinted) — signals destructive intent without being aggressive at rest.
- **Popup rounded corners:** `.pcr-app` given `border-radius: 12px`, `border: 1px solid #e0e0e0`, `box-shadow: 0 4px 16px rgba(0,0,0,.10)`, and `overflow: hidden` to clip inner canvas to the rounded corners.
- **Tip note removed:** `.pickr-tip-note-el` CSS, its stale-guard on re-init, and its JS creation/injection all deleted. Auto-save behaviour is self-evident from the UI — CANCEL is the only explicit action, so the note was answering a question users no longer ask.

---

## Known accepted risks

| Risk | Why accepted |
|---|---|
| `defaultColor` mutated in-memory on save | Safe — Bubble re-render resyncs `rawColor` on every confirmed save. Only a problem if re-render doesn't fire AND user cancels in the same session. Not observed. Also self-correcting even without re-render: `defaultColor` is updated in-memory on every confirmed save regardless, so saving the same colour twice in one session still leaves the correct revert point. |
| `safeBubbleSend` drops payload silently if bridge not ready | Race window is too small to hit in practice — user can't open the picker before `bubble_fn_colorData` is ready. Guard prevents a crash; no retry needed. |
| `position: 'left-middle'` hardcoded | `autoReposition: true` handles viewport edge cases. At worst a cosmetic flash. |
| Explicit `pickr.hide()` after `runCancel()` in CANCEL handler | No double-fire currently. CDN pinning mitigates forward-compat risk if Pickr's event semantics change. |
| `.pickr` wrapper forced to `100% !important` globally (Fix #32) | Applies to all 10-15 instances via one shared class, but each instance's `.pickr` wrapper sits inside its own distinct Bubble HTML element, so `100%` resolves per-instance correctly. `!important` needed to beat Pickr's own default styling on that wrapper. |

---

## ⚠️ Needs live testing

- [ ] Trigger pairing across all 10–15 instances — re-verify after Fix #29 (identity-based stamp)
- [ ] Destroy-while-open scenario (force a Reusable Element re-render mid-edit) — confirm no stray Escape behaviour or stale save afterward (Fix #26)
- [ ] **Combined case:** destroy-while-open + simultaneous re-render across multiple instances at once — checks whether claimed markers reset and trigger pairing breaks (overlap of Fix #21's watch-for and Fix #26)
- [ ] Save the same colour twice in one session (no Bubble re-render in between), then cancel — confirm revert lands on the second save's colour, not a stale one
- [ ] Colour with opacity < 100% survives a full page reload (Fix #22)
- [ ] CANCEL/Escape revert sends no stale debounce payload after the revert (Fix #23)
- [ ] `window.PICKR_DEBUG = true` in console re-enables both warning types
- [x] Multi-instance trigger pairing — verified working on a real Bubble page (Fix #21)
- [ ] Trigger pairing under simultaneous re-render of multiple instances — confirm Fix #29 eliminates the ordering failure mode (Fix #29)
- [ ] Normal outside-click close → `saved: true` fires, `colorHex`/`rawColor` update correctly
- [ ] CANCEL button → revert + close, no auto-save double-fire
- [ ] Escape → identical behaviour to CANCEL
- [ ] Escape listener removed on `hide` — only one active listener at a time across all instances
- [ ] Trigger circle colour does not change mid-drag, only on auto-save or cancel-revert
- [ ] No duplicate CANCEL button on parent Reusable Element re-render
- [ ] Force a missing-trigger scenario (delete/rename a `.pickr-trigger-el` div on one instance) — confirm `bubble_fn_colorData` receives `error: 'no_trigger_found'` and existing workflows ignore it without side effects (Fix #28)
- [ ] Force multiple Reusable Element re-renders in a row — confirm script count stays at `1` each time (Fix #31)
- [ ] Confirm popup opens correctly across all 10–15 instances after this change (Fix #31)
- [ ] Retest the original detach-while-open scenario (Fix #26/#30) now that load timing has changed (Fix #31)
- [x] Trigger fills its Bubble HTML element (size + shape) instead of showing as a small rectangle — verified fix after correcting `.pickr` wrapper sizing (Fix #32)
- [ ] Confirm trigger fill works across all 10–15 instances at different Bubble element sizes/shapes (square, rounded-rect, etc.) (Fix #32)
- [ ] Confirm "Fit height to content" stays unchecked on every instance's Bubble HTML element — re-check after any editor changes (Fix #32)
- [ ] Resize the Bubble HTML element after page load (if dynamically resizable) — confirm trigger resizes with it, not just on initial render (Fix #32)
