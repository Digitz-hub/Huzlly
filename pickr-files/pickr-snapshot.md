# Pickr Color Picker — Bubble.io Integration — Session Update

**Continuing from:** Converting Pickr color picker trigger to class-based element
**Last updated:** June 24, 2026 (Cursor Glitch Fix & Position Update)

---

## Context

Bubble.io project using a self-contained HTML element with the Pickr (`@simonwep/pickr`) colour picker library. Multiple instances of this HTML element exist on the same page **inside a Reusable Element** (one per editable field — e.g. `titleColor`, `textColor`, `bgColor`), duplicated 10–15 times.

**One** JS-to-Bubble element handles the entire data flow atomically:
- **`colorData`** — text, Trigger event ON, Publish value ON — receives a single JSON string, e.g. `{"field":"titleColor","value":"#343882FF"}`

**Bubble workflow architecture (confirmed working):**
- Trigger: *When JavascripttoBubble `colorData` has an event*
- Condition (*Only when*): `This JavascripttoBubble's value:extract with Regex:first item` is `titleColor`
  - Regex pattern for field extraction: `(?<="field":")[^"]+`
  - ⚠️ Case-sensitive — must match the exact `fieldName` string from the JS payload (e.g. `titleColor`, not `titlecolor` or `titleColour`)
- Action (*Value*): `This JavascripttoBubble's value:extract with Regex:first item`
  - Regex pattern for colour extraction: `(?<="value":")[^"]+`

### Architecture decision: fully self-contained, single-file per instance
An alternative was tried where shared assets (Pickr CDN files, shared CSS/JS) lived in the Page Header and only the per-instance config lived in the local HTML element. **This broke in production** — Bubble renders each HTML element inside its own isolated `<iframe>`, so scripts loaded in the parent header aren't reachable from inside a local element's scope without complex cross-iframe bridging, and the implementation failed silently. **Decision: stay 100% self-contained.** Every instance carries its own CDN `<link>`/`<script>` tags, CSS, and init logic — confirmed in Context note below and reflected in the code block.

---

## Fixes applied in earlier sessions

### 1. Old-value-lag bug (JS-to-Bubble call order)
**Problem:** Workflow was triggering on `colorField` (the field-identifier call), but reading `colorValue` — except `colorValue` was being sent *after* `colorField` in code, so Bubble's workflow ran with the **previous** colour value. Each save was one colour "behind."

**Fix:** Swapped the call order — send `colorValue` first, then `colorField` (the trigger) last, so the workflow always reads the freshly-updated value:
```javascript
bubble_fn_colorValue(hexValue);   // value first
bubble_fn_colorField(fieldName);  // trigger last
```

> ⚠️ **Superseded by Fix #9.** Reordering two calls only reduced the race window, it didn't remove it — Bubble could still process them as two separate workflow triggers. The two-call architecture itself has now been replaced with a single consolidated call, eliminating the race condition at the root instead of working around it.

### 2. Dead weight / redundant code cleanup
- Removed `closeOnScroll: false` from Pickr config — this is already Pickr's default, the line did nothing.
- Removed the `currentColor` variable and the entire `pickr.on('change', ...)` listener used only to track it — replaced with a direct `pickr.getColor()` call inside the Save button click handler. One less listener, no redundant state tracking.
  - *(Note: a `change` listener was reintroduced in a later session for the dirty-state save button — see Fix #6 below. It now serves a real purpose instead of just tracking state.)*

### 3. Trigger circle showing live preview instead of saved colour
**Problem:** Pickr's own button (`.pcr-button`, which is the trigger circle) updates its background live as the user drags/changes colour inside the popup — even before Save is clicked. This caused confusion between "what's currently set" vs "what's being previewed."

**Fix:** Locked the trigger circle's background to a CSS custom property with `!important`, which is only updated at two controlled moments — on **init** (shows Bubble's saved value) and on **Save click** (shows the newly confirmed value). Live preview/drag inside the popup no longer affects the trigger circle.
```css
.pcr-button {
    background: var(--trigger-color, #2563EB) !important;
}
```
```javascript
function lockTriggerColor(color) {
  const btn = pickr.getRoot().button;
  if (!btn) return;
  const rgba = color.toRGBA();
  btn.style.setProperty('--trigger-color', `rgba(${Math.round(rgba[0])}, ${Math.round(rgba[1])}, ${Math.round(rgba[2])}, ${rgba[3]})`);
}
```

### 4. Popup retaining unsaved preview colour on reopen
**Problem:** If the user previewed a colour inside the popup and closed it *without* clicking Save, reopening the picker showed the leftover unsaved preview colour instead of Bubble's actual saved value.

**Fix:** Added a `savedThisSession` flag, set to `true` only inside the Save button's click handler. On the Pickr `hide` event, if the flag is still `false` (popup closed without saving), the picker's internal colour is reset back to `defaultColor` (Bubble's value):
```javascript
pickr.on('hide', () => {
  if(!savedThisSession) {
    pickr.setColor(defaultColor);
  }
  savedThisSession = false;
});
```
> ⚠️ **Partially superseded by Fix #10.** This correctly reset to `defaultColor` — but `defaultColor` itself was never updated after the initial page load, so it could go stale after a successful save. See Fix #10.

### 5. Save button moved to bottom of popup
**Problem:** Save button was rendering above the HEXA/RGBA result row inside `.pcr-interaction` (default flex order), which felt visually out of place.

**Fix:** Set explicit flex `order` on the Save button so it always lands last, after the result row (`order:1`) and the HEXA/RGBA toggle (`order:2`):
```css
.pickr-save-btn-el{
    order:3;
    flex:1 1 100%;
    ...
}
```

### 6. Save button colour reflects dirty/clean state
**Problem:** Save button was always blue regardless of whether the previewed colour actually differed from Bubble's saved value — no visual cue for "nothing to save" vs "unsaved change pending."

**Fix:** Reintroduced a `pickr.on('change', ...)` listener. A `pickrDefaultHexStr` variable holds the normalized hex of the currently-saved colour — set on `init` from `instance.getColor()`, and updated again on every successful Save. On every `change` event, the live preview colour is compared against this baseline:
- **Match** → `.pcr-dirty` class removed → button is gray (`#D1D5DB`), `cursor: not-allowed`
- **Mismatch** → `.pcr-dirty` class added → button is blue (`#2563EB`), clickable

```css
.pickr-save-btn-el{
    background:#D1D5DB;
    cursor:not-allowed;
}
.pickr-save-btn-el.pcr-dirty{
    background:#2563EB;
    cursor:pointer;
}
```
```javascript
function updateSaveButtonState() {
  var result = pickr.getRoot().interaction.result;
  var saveBtn = result ? result.parentElement.querySelector('.pickr-save-btn-el') : null;
  if (!saveBtn) return;
  var current = pickr.getColor();
  if (!current) return;
  var currentHex = current.toHEXA().toString().toUpperCase();
  if (currentHex === pickrDefaultHexStr) {
    saveBtn.classList.remove('pcr-dirty');
  } else {
    saveBtn.classList.add('pcr-dirty');
  }
}

pickr.on('change', () => {
  updateSaveButtonState();
});
```
On Save click, `pickrDefaultHexStr` is updated to the newly saved hex *before* `updateSaveButtonState()` runs, so the button immediately flips back to gray.

### 7. Swatches forced into a single row
**Problem:** With 8 default swatches, the row was wrapping — 7 fit on the first line, 1 dropped to a second line.

**Fix (iterated through several approaches):**
- First tried `flex-wrap: nowrap` + explicit small swatch sizing (`14–18px`) + `justify-content: space-between` on `.pcr-swatches` — worked, but per later request the explicit sizing was reverted.
- **Current state:** swatch sizing is back to Pickr's own default (no overridden `width`/`height` on `.pcr-swatch`), and only wrapping/alignment is controlled:
```css
.pcr-app .pcr-swatches{
    flex-wrap:nowrap !important;
    justify-content:space-between !important;
}
```
- Swatch count is currently **14** — at this count, with default sizing, the row **will wrap to 2 lines**. If a strict single-row layout is needed again at 14 swatches, the explicit small-size CSS from the earlier iteration will need to be reapplied.

### 8. Dynamic popup position — *replaced with fixed `left-middle`*
**Note:** An earlier iteration included a position-detection block that picked `top-start` vs `bottom-start` based on where the trigger sits on screen. This has been removed. Position is now **hardcoded to `left-middle`** — popup opens to the left of the trigger circle, vertically centered.

### 9. Two-call race condition replaced with a single consolidated payload
**Problem:** Even with the call order fixed (Fix #1), sending `colorValue` and `colorField` as two separate `bubble_fn_*` calls meant Bubble received two independent workflow triggers in quick succession — could still occasionally race.

**Fix:** Replaced both calls with a single `bubble_fn_colorData` call carrying a JSON-stringified object with both pieces of data together. One call, one event, one atomic payload:
```javascript
var payload = { field: fieldName, value: hexValue };
bubble_fn_colorData(JSON.stringify(payload));
```

### 10. Unsaved-preview-reset bug used a stale page-load colour instead of the latest saved colour
**Problem:** Fix #4 correctly reset the popup to `defaultColor` when closed without saving — but `defaultColor` was only ever set once, from `rawColor` at page load. So: save a colour → reopen → drag to preview → close without saving → picker incorrectly fell back to the *original page-load* colour.

**Fix:** `defaultColor` is now reassigned to the new `hexValue` inside the Save button's click handler:
```javascript
defaultColor = hexValue;
```

### 11. Null-safety added around the interaction-result lookup on init
**Fix:** Split into a guarded two-step lookup:
```javascript
var result = pickr.getRoot().interaction.result;
var interaction = result ? result.parentElement : null;
```

### 12. Global scope namespace collisions (IIFE scoping)
**Fix:** Wrapped the entire runtime block in an IIFE:
```javascript
(function() {
  function initPickrInstance() { ... }
  initPickrInstance();
})();
```

### 13. Duplicate Save button stacking on re-render
**Fix:** Added an explicit removal step immediately before creating the new Save button:
```javascript
var existingBtn = interaction.querySelector('.pickr-save-btn-el');
if (existingBtn) existingBtn.remove();
```

---

## Fixes applied in this session (June 24, 2026)

### 14. Stale cursor glitch after popup close
**Problem:** After the popup closed, browser cached the hover state of the popup's elements (save button, swatches, type toggle) for a few milliseconds. If the cursor was over that area, `cursor: not-allowed` (from the inactive save button) would briefly show — visible as a red `/` cursor appearing right after close.

**Fix:** On `hide`, disable pointer events on the entire `.pcr-app` root element immediately. On `show`, restore them. This way the browser has no hover state to cache — the whole popup becomes non-interactive the instant it closes:
```javascript
pickr.on('show', () => {
  var app = pickr.getRoot().app;
  if (app) app.style.pointerEvents = '';
});

pickr.on('hide', () => {
  var app = pickr.getRoot().app;
  if (app) app.style.pointerEvents = 'none';

  if(!savedThisSession) {
    pickr.setColor(defaultColor);
  }
  savedThisSession = false;
});
```
Covers all popup elements in one shot — no need to target save button individually.

### 15. Popup position changed to `left-middle`
**Change:** `dynamicPosition` hardcoded to `'left-middle'` — popup opens to the left of the trigger circle, vertically centered. `autoReposition: true` retained (Pickr default).

---

## ⚠️ Needs live testing

- Confirm the `!important` CSS lock on `.pcr-button` actually overrides Pickr's own internal style updates in all browsers used.
- Confirm `pickr.setColor(defaultColor)` on `hide` doesn't cause any visible flash/flicker before the popup fully closes.
- Confirm 14 swatches at default sizing renders as expected (2 rows) and that this is the desired look.
- Confirm the Bubble-side workflow correctly parses the `colorData` JSON payload and reads `field`/`value` without errors.
- Confirm that after Save → reopen → preview a different colour → close without saving, the trigger and popup both correctly fall back to the *just-saved* colour (Fix #10).
- Confirm no console errors are thrown if Pickr's `result` component is ever unavailable at `init` time (Fix #11).
- Confirm the Bubble-side regex conditions correctly extract `field` and `value` from the `colorData` JSON string with exact case-sensitive matching.
- Confirm that triggering a re-render of the parent Reusable Element does not produce a stacked/duplicate Save button (Fix #13).
- **(New)** Confirm cursor glitch is fully gone after popup close — no `not-allowed` / red `/` cursor visible at the position where popup was (Fix #14).
- **(New)** Confirm `left-middle` position renders correctly across all instances and doesn't get clipped by the iframe boundary (Fix #15).

---

## Quick checklist for next session

- [ ] Roll out the same changes to all remaining HTML elements (10–15 total)
- [ ] Confirm Bubble workflow reads from single `colorData` element — delete old `colorValue` / `colorField` elements once migrated
- [ ] Confirm: trigger circle does NOT change while dragging inside popup
- [ ] Confirm: Save button is gray when preview colour matches saved colour, blue when it differs
- [ ] Decide if 14 swatches wrapping to 2 rows is acceptable or if single-row explicit sizing needs to come back

### Reported as verified (carried over — not independently re-tested by Claude)
- [x] **Atomic data flow** — legacy `colorValue` + `colorField` workflows combined into a single `colorData` JSON payload
- [x] **Spelling/case sync** — Bubble regex conditions confirmed checking exact-case `titleColor` (capital C)
- [x] **DOM preservation on re-render** — guarded by wiping any pre-existing `.pickr-save-btn-el` before injecting a new one (Fix #13)
- [x] **Local scoping** — all CDN assets, CSS, and JS kept self-contained per instance; no parent Page Header dependency

---

## Current complete HTML code (per instance — copy into each Bubble HTML element)

> Each HTML element only needs **two values changed**: `rawColor` and `fieldName`. Everything else stays identical across all instances.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@simonwep/pickr/dist/themes/nano.min.css"/>
<div class="pickr-trigger-el"></div>
<style>
.pickr-trigger-el{
    width:24px;
    height:24px;
    border-radius:50%;
    background:#2563EB;
    cursor:pointer;
}
.pcr-button {
    width:24px !important;
    height:24px !important;
    border-radius:50% !important;
    box-shadow: none !important;
    outline: none !important;
    border: none !important;
    background: var(--trigger-color, #2563EB) !important;
}
.pcr-button::before,
.pcr-button::after {
    display: none !important;
}
.pcr-button:hover,
.pcr-button:focus,
.pcr-button:active,
.pcr-button.active {
    box-shadow: none !important;
    outline: none !important;
    border: none !important;
    filter: none !important;
}
.pcr-app .pcr-interaction{
    display:flex;
    flex-wrap:wrap;
    gap:8px;
}
.pcr-app .pcr-swatches{
    flex-wrap:nowrap !important;
    justify-content:space-between !important;
}
.pcr-app .pcr-interaction .pcr-result,
.pcr-app .pcr-interaction .pcr-type,
.pcr-app .pcr-swatches button,
.pcr-app .pcr-swatches .pcr-swatch{
    border:none;
    outline:none !important;
    box-shadow:none;
    transition:.15s;
}
.pcr-app .pcr-interaction .pcr-result:hover,
.pcr-app .pcr-interaction .pcr-result:focus,
.pcr-app .pcr-interaction .pcr-type:hover,
.pcr-app .pcr-interaction .pcr-type:focus,
.pcr-app .pcr-swatches button:hover,
.pcr-app .pcr-swatches .pcr-swatch:hover,
.pcr-app .pcr-swatches button:focus,
.pcr-app .pcr-swatches .pcr-swatch:focus,
.pcr-app .pcr-swatches button.pcr-active,
.pcr-app .pcr-swatches .pcr-swatch.pcr-active{
    outline:none !important;
    box-shadow:0 0 4px rgba(0,0,0,.18) !important;
}
.pcr-app .pcr-interaction .pcr-result{
    flex:1 1 100%;
    order:1;
}
.pcr-app .pcr-interaction .pcr-type{
    order:2;
    flex:1;
    margin:0;
    padding:8px 0;
    border-radius:6px;
    background:#F9FAFB;
    color:#374151;
    font-size:12px;
    font-weight:600;
    letter-spacing:.03em;
    cursor:pointer;
}
.pcr-app .pcr-interaction .pcr-type:hover,
.pcr-app .pcr-interaction .pcr-type:focus{
    background:#F3F4F6;
}
.pcr-app .pcr-interaction .pcr-type.active{
    background:#2563EB;
    color:#fff;
}
.pickr-save-btn-el{
    display:none;
    order:3;
    flex:1 1 100%;
    width:100%;
    margin-top:8px;
    padding:8px 0;
    border-radius:6px;
    background:#D1D5DB;
    color:#fff;
    font-size:12px;
    font-weight:600;
    letter-spacing:.03em;
    cursor:not-allowed;
    border:none;
    outline:none;
    transition:.15s;
}
.pickr-save-btn-el.pcr-dirty{
    background:#2563EB;
    cursor:pointer;
}
.pickr-save-btn-el.pcr-dirty:hover{
    background:#1d4ed8;
}
</style>
<script src="https://cdn.jsdelivr.net/npm/@simonwep/pickr/dist/pickr.min.js"></script>
<script>
(function() {
  function initPickrInstance() {
    if(typeof Pickr === 'undefined') return setTimeout(initPickrInstance, 100);

    // ---- YAHAN PE BUBBLE KI DYNAMIC VALUE DAALO ----
    var rawColor = '#343882';     // is field ki current colour - dynamic karo Bubble se
    var fieldName = 'titleColor'; // is picker ka field name - har HTML element mein alag likho
    // ------------------------------------------------

    var instanceKey = 'pickr_' + fieldName;
    var triggerEl = document.querySelector('.pickr-trigger-el');

    if(window[instanceKey]) {
      window[instanceKey].destroyAndRemove();
      window[instanceKey] = null;
    }

    var defaultColor = /^#([0-9A-Fa-f]{3}|[0-9A-Fa-f]{6})$/.test(rawColor) ? rawColor : '#2563EB';
    var dynamicPosition = 'left-middle';

    const pickr = Pickr.create({
      el: triggerEl,
      theme: 'nano',
      default: defaultColor,
      position: dynamicPosition,
      hideOnSelect: false,
      autoReposition: true,
      swatches: ['#2563EB','#3B82F6','#06B6D4','#10B981','#22C55E','#F59E0B','#EF4444','#EC4899','#8B5CF6','#A855F7','#14B8A6','#84CC16','#F97316','#6366F1'],
      components: {
        preview: true,
        opacity: true,
        hue: true,
        interaction: { hex: true, rgba: true, input: true }
      }
    });

    window[instanceKey] = pickr;

    var savedThisSession = false;
    var pickrDefaultHexStr = '';

    function lockTriggerColor(color) {
      const btn = pickr.getRoot().button;
      if (!btn) return;
      const rgba = color.toRGBA();
      btn.style.setProperty('--trigger-color', `rgba(${Math.round(rgba[0])}, ${Math.round(rgba[1])}, ${Math.round(rgba[2])}, ${rgba[3]})`);
    }

    function updateSaveButtonState() {
      var result = pickr.getRoot().interaction.result;
      var saveBtn = result ? result.parentElement.querySelector('.pickr-save-btn-el') : null;
      if (!saveBtn) return;
      var current = pickr.getColor();
      if (!current) return;
      var currentHex = current.toHEXA().toString().toUpperCase();
      if (currentHex === pickrDefaultHexStr) {
        saveBtn.classList.remove('pcr-dirty');
      } else {
        saveBtn.classList.add('pcr-dirty');
      }
    }

    pickr.on('init', (instance) => {
      lockTriggerColor(instance.getColor());
      pickrDefaultHexStr = instance.getColor().toHEXA().toString().toUpperCase();

      var result = pickr.getRoot().interaction.result;
      var interaction = result ? result.parentElement : null;
      if(interaction) {
        var existingBtn = interaction.querySelector('.pickr-save-btn-el');
        if (existingBtn) existingBtn.remove();

        var saveBtn = document.createElement('button');
        saveBtn.className = 'pickr-save-btn-el';
        saveBtn.textContent = 'SAVE';
        saveBtn.onclick = function() {
          var selected = pickr.getColor();
          if(selected) {
            var hexValue = selected.toHEXA().toString();
            if(hexValue && hexValue !== '' && hexValue !== '#') {
              savedThisSession = true;
              lockTriggerColor(selected);
              pickrDefaultHexStr = hexValue.toUpperCase();
              defaultColor = hexValue;
              updateSaveButtonState();

              var payload = { field: fieldName, value: hexValue };
              bubble_fn_colorData(JSON.stringify(payload));

              pickr.hide();
            }
          }
        };
        interaction.appendChild(saveBtn);
        saveBtn.style.display = 'block';
        updateSaveButtonState();
      }
    });

    pickr.on('change', () => {
      updateSaveButtonState();
    });

    // Popup open hone par pointer events restore karo
    pickr.on('show', () => {
      var app = pickr.getRoot().app;
      if (app) app.style.pointerEvents = '';
    });

    pickr.on('hide', () => {
      // Poore popup ke pointer events band - cursor glitch kisi bhi element pe na aaye
      // (save button, swatches, type toggle - sab cover ho jaate hain)
      var app = pickr.getRoot().app;
      if (app) app.style.pointerEvents = 'none';

      if(!savedThisSession) {
        pickr.setColor(defaultColor);
      }
      savedThisSession = false;
    });
  }

  initPickrInstance();
})();
</script>
```
