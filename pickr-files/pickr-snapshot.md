# Pickr Color Picker — Bubble.io Integration — Snapshot

**Last updated:** June 26, 2026
**Fixes covered:** #1–27 + housekeeping pass (see session log below)

---

## Project context

Bubble.io project using a self-contained HTML element with the Pickr (`@simonwep/pickr@1.9.1`) colour picker library. Multiple instances of this HTML element exist on the same page inside a Reusable Element — one per editable colour field (e.g. `titleColor`, `textColor`, `bgColor`), duplicated 10–15 times.

**Files:**
- `pickr-instance-element.html` — the full per-instance code (only `rawColor` and `fieldName` change between instances)
- `README.md` — Bubble setup guide (HTML element, Toolbox plugin, workflows)
- `pickr-snapshot.md` — this file

**Key design decisions:**
- Auto-save on popup close (outside click) — no SAVE button
- CANCEL button and Escape key both revert to last confirmed save and send `saved: false`
- `colorHex` custom state on each HTML element holds the last confirmed saved colour — only updated on `saved: true` to prevent Bubble re-rendering the element mid-edit and destroying the picker
- One `JavascripttoBubble` element (`colorData` suffix) handles all instances on the page
- `bubble_fn_colorData` guard (`safeBubbleSend`) prevents a ReferenceError if the Toolbox element hasn't initialized yet on first page load — payload is dropped silently but this race window is too small to matter in practice (user can't reach the picker before the bridge is ready)
- `defaultColor` is intentionally mutated on each confirmed save as an in-memory cache — safe because Bubble re-renders resync `rawColor` on every save anyway

---

## Architecture overview

Each instance runs inside an IIFE. On script parse:
1. Immediately claims its trigger `.pickr-trigger-el` div via `data-pickr-claimed` attribute (Fix #24)
2. Calls `initPickrInstance()`, which retries every 100ms until `Pickr` is available on `window`
3. On re-init, runs `cleanup()` on the old instance before destroying it (Fixes #25, #26)

On `init`: injects CANCEL button and tip note into Pickr's interaction panel.
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

**Watch for:** if a parent Reusable Element re-renders all instance scripts simultaneously, claimed markers could reset and re-claim out of order. Not observed, but retest if instance count grows.

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

### Housekeeping pass (no logic changes)
- `dynamicPosition` variable removed — `'left-middle'` inlined directly into `Pickr.create()`
- All Hindi inline comments replaced with English
- All block comments tightened and consistent
- Fix #27's comment in HTML corrected (was incorrectly labelled #2)
- Local variable `hexValue` in `change` and `hide` handlers renamed to `colorHex` to match the Bubble custom state name
- Bubble custom state renamed from `hexValue` to `colorHex` — README updated throughout
- Tip note text updated: `Changes save automatically when you close this picker.`

---

## Known accepted risks

| Risk | Why accepted |
|---|---|
| `defaultColor` mutated in-memory on save | Safe — Bubble re-render resyncs `rawColor` on every confirmed save. Only a problem if re-render doesn't fire AND user cancels in the same session. Not observed. |
| `safeBubbleSend` drops payload silently if bridge not ready | Race window is too small to hit in practice — user can't open the picker before `bubble_fn_colorData` is ready. Guard prevents a crash; no retry needed. |
| `position: 'left-middle'` hardcoded | `autoReposition: true` handles viewport edge cases. At worst a cosmetic flash. |
| Explicit `pickr.hide()` after `runCancel()` in CANCEL handler | No double-fire currently. CDN pinning mitigates forward-compat risk if Pickr's event semantics change. |

---

## ⚠️ Needs live testing

- [ ] Trigger pairing across all 10–15 instances — re-verify after Fix #24 (claim moved to parse time)
- [ ] Destroy-while-open scenario (force a Reusable Element re-render mid-edit) — confirm no stray Escape behaviour or stale save afterward (Fix #26)
- [ ] Colour with opacity < 100% survives a full page reload (Fix #22)
- [ ] CANCEL/Escape revert sends no stale debounce payload after the revert (Fix #23)
- [ ] `window.PICKR_DEBUG = true` in console re-enables both warning types
- [x] Multi-instance trigger pairing — verified working on a real Bubble page (Fix #21)
- [ ] Normal outside-click close → `saved: true` fires, `colorHex`/`rawColor` update correctly
- [ ] CANCEL button → revert + close, no auto-save double-fire
- [ ] Escape → identical behaviour to CANCEL
- [ ] Escape listener removed on `hide` — only one active listener at a time across all instances
- [ ] Trigger circle colour does not change mid-drag, only on auto-save or cancel-revert
- [ ] No duplicate CANCEL button / tip note on parent Reusable Element re-render
