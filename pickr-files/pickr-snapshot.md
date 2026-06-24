# Pickr Color Picker — Bubble.io Integration — Session Update

**Continuing from:** Converting Pickr color picker trigger to class-based element
**Last updated:** June 24, 2026 (Live Preview Debounce + saved Flag + HEXA-only mode)

---

## Context

Bubble.io project using a self-contained HTML element with the Pickr (`@simonwep/pickr`) colour picker library. Multiple instances of this HTML element exist on the same page **inside a Reusable Element** (one per editable field — e.g. `titleColor`, `textColor`, `bgColor`), duplicated 10–15 times.

**One** JS-to-Bubble element handles the entire data flow:
- **`colorData`** — text, Trigger event ON, Publish value ON — receives a single JSON string, e.g. `{"field":"titleColor","value":"#343882","saved":true}`

**Bubble workflow architecture:**

Two separate workflows both triggered by `colorData`:

**Workflow 1 — Live preview** (*Only when* `saved` = `false`):
- Condition: `This JavascripttoBubble's value:extract with Regex:first item` = `false`
  - Regex for `saved`: `(?<="saved":)(true|false)`
- Action: update UI preview state only — do NOT update `rawColor` custom state

**Workflow 2 — Save** (*Only when* `saved` = `true`):
- Condition: `This JavascripttoBubble's value:extract with Regex:first item` = `true`
  - Regex for `saved`: `(?<="saved":)(true|false)`
- Action: update `rawColor` custom state (which feeds back into `rawColor` in the HTML element)

**Common regex patterns for both workflows:**
- Field extraction: `(?<="field":")[^"]+`
- Value extraction: `(?<="value":")[^"]+`
- ⚠️ Case-sensitive — must match the exact `fieldName` string (e.g. `titleColor`, not `titlecolor`)

### Architecture decision: fully self-contained, single-file per instance
Bubble renders each HTML element inside its own isolated `<iframe>` — scripts loaded in the parent Page Header aren't reachable from inside a local element's scope. **Decision: stay 100% self-contained.** Every instance carries its own CDN `<link>`/`<script>` tags, CSS, and init logic.

---

## All fixes (cumulative)

### 1. Old-value-lag bug (JS-to-Bubble call order)
Swapped call order — `colorValue` sent before `colorField`. Superseded by Fix #9.

### 2. Dead weight / redundant code cleanup
Removed `closeOnScroll: false` and redundant `currentColor` tracking. `change` listener reintroduced later for dirty-state (Fix #6).

### 3. Trigger circle locked to saved colour
`.pcr-button` background locked via CSS custom property `--trigger-color` with `!important`. Only updated on `init` and Save click — live drag does not affect trigger circle.

### 4. Popup resets to saved colour on close without save
`savedThisSession` flag — if `false` on `hide`, `pickr.setColor(defaultColor)` resets popup. Partially superseded by Fix #10.

### 5. Save button moved to bottom of popup
Flex `order:3` on `.pickr-save-btn-el` — renders after result row (`order:1`) and type toggle (`order:2`).

### 6. Save button dirty/clean state
`pickrDefaultHexStr` holds normalized hex of saved colour. On every `change`, current preview vs baseline compared:
- Match → `.pcr-dirty` removed → gray button, `cursor: not-allowed`
- Mismatch → `.pcr-dirty` added → blue button, clickable

### 7. Swatches single row
`flex-wrap: nowrap` + `justify-content: space-between` on `.pcr-swatches`. Current count: **14 swatches** — wraps to 2 rows at default sizing.

### 8. Popup position changed to `left-middle`
`dynamicPosition` hardcoded to `'left-middle'`. `autoReposition: true` retained.

### 9. Single consolidated payload (race condition fix)
Replaced two separate `bubble_fn_colorValue` + `bubble_fn_colorField` calls with one `bubble_fn_colorData` call carrying a JSON payload.

### 10. `defaultColor` updated after save
`defaultColor` reassigned to new `hexValue` inside Save click handler — cancel/reopen always falls back to most recently saved colour, not page-load colour.

### 11. Null-safety on interaction-result lookup
```javascript
var result = pickr.getRoot().interaction.result;
var interaction = result ? result.parentElement : null;
```

### 12. IIFE scoping
Entire block wrapped in `(function() { ... })()` to avoid global scope leakage across 10–15 instances.

### 13. Duplicate Save button guard
```javascript
var existingBtn = interaction.querySelector('.pickr-save-btn-el');
if (existingBtn) existingBtn.remove();
```

### 14. Stale cursor glitch fix after popup close
`pointerEvents: 'none'` on entire `.pcr-app` on `hide`, restored on `show` — browser has no hover state to cache after popup closes.

### 15. HEXA-only mode
`rgba: false` in `interaction` config. `.pcr-type` buttons hidden via CSS (`display: none !important`) since Pickr still renders them with a single mode.

### 16. Live debounce preview + `saved` flag
**Problem:** Sending live colour updates to Bubble caused `rawColor` custom state to update → HTML element re-render → Pickr destroyed → popup closed mid-use.

**Fix:** Added `saved` boolean to payload. Two Bubble workflows — one for live preview (`saved: false`), one for actual save (`saved: true`). Only the save workflow updates `rawColor`.

- `change` event: 500ms debounced send with `saved: false`
- Save button click: immediate send with `saved: true`
- `hide` event: `clearTimeout(debounceTimer)` — pending debounce cancelled on popup close

```javascript
// change event — live preview
clearTimeout(debounceTimer);
debounceTimer = setTimeout(() => {
  var payload = { field: fieldName, value: hexValue, saved: false };
  bubble_fn_colorData(JSON.stringify(payload));
}, 500);

// save button click
var payload = { field: fieldName, value: hexValue, saved: true };
bubble_fn_colorData(JSON.stringify(payload));

// hide event
clearTimeout(debounceTimer);
```

---

## ⚠️ Needs live testing

- Confirm trigger circle does NOT change while dragging inside popup
- Confirm closing without Save → reopen shows most recently saved colour (Fix #10)
- Confirm 14 swatches wrapping to 2 rows is acceptable — if single row needed, add explicit swatch sizing CSS
- Confirm `saved: false` workflow does NOT update `rawColor` — popup must stay open during live drag
- Confirm `saved: true` workflow correctly updates `rawColor` and closes popup cleanly
- Confirm cursor glitch gone after popup close (Fix #14)
- Confirm no duplicate Save button on parent Reusable Element re-render (Fix #13)
- Confirm Bubble regex `(?<="saved":)(true|false)` correctly extracts `true`/`false` from payload

---

## Quick checklist for next session

- [ ] Roll out to all remaining HTML elements (10–15 total) — only `rawColor` and `fieldName` change per instance
- [ ] Delete old `colorValue` / `colorField` JS-to-Bubble elements once migrated
- [ ] Confirm both Bubble workflows (preview + save) are set up and tested end-to-end
- [ ] Decide if 14-swatch 2-row layout is final or if single-row sizing needed

### Verified (carried over)
- [x] Atomic data flow via single `colorData` JSON payload
- [x] Case-sensitive `fieldName` matching in Bubble regex
- [x] DOM preservation on re-render (duplicate Save button guard)
- [x] Fully self-contained per instance — no Page Header dependency

---

## Current complete HTML code (per instance)

> Only two values change per instance: `rawColor` and `fieldName`.

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
/* HEXA/RGBA toggle buttons hidden - sirf HEXA mode use hota hai */
.pcr-app .pcr-interaction .pcr-type{
    display:none !important;
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
        interaction: { hex: true, rgba: false, input: true }
      }
    });

    window[instanceKey] = pickr;

    var savedThisSession = false;
    var pickrDefaultHexStr = '';
    var debounceTimer = null;

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

              // saved: true — Bubble rawColor update krega
              var payload = { field: fieldName, value: hexValue, saved: true };
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

      // 500ms debounce se live preview value Bubble ko bhejo
      // saved: false — sirf preview workflow chalega, rawColor update nahi hoga
      clearTimeout(debounceTimer);
      debounceTimer = setTimeout(() => {
        var current = pickr.getColor();
        if (!current) return;
        var hexValue = current.toHEXA().toString();
        if (hexValue && hexValue !== '' && hexValue !== '#') {
          var payload = { field: fieldName, value: hexValue, saved: false };
          bubble_fn_colorData(JSON.stringify(payload));
        }
      }, 500);
    });

    // Popup open hone par pointer events restore karo
    pickr.on('show', () => {
      var app = pickr.getRoot().app;
      if (app) app.style.pointerEvents = '';
    });

    pickr.on('hide', () => {
      // Pending debounce cancel karo - popup close hone ke baad stale value na jaaye Bubble ko
      clearTimeout(debounceTimer);

      // Poore popup ke pointer events band - cursor glitch kisi bhi element pe na aaye
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
