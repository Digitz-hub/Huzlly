# Pickr Color Picker — Bubble.io Integration — Session Update

**Continuing from:** Converting Pickr color picker trigger to class-based element
**Last updated:** June 24, 2026 (Single-File Architecture & Iframe Clean-up Consolidation)

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

### Documentation sync note
The code block in this file had fallen out of sync with the actual file in use — it still showed the older `id="pickr-trigger"` / `document.getElementById` version. The actual element already uses the **class-based** approach (`.pickr-trigger-el`, scoped lookups via `pickr.getRoot()`, and an `instanceKey` of `'pickr_' + fieldName` for safe per-instance storage). The code block below has been brought back in sync with what's actually running. The dynamic `top-start`/`bottom-start` position detection mentioned in Fix #8 is **not** present in the current file — position is currently hardcoded to `'bottom-start'`. Flag this if that logic still needs to be reinstated.

---

## Fixes applied in earlier sessions

### 1. Old-value-lag bug (JS-to-Bubble call order)
**Problem:** Workflow was triggering on `colorField` (the field-identifier call), but reading `colorValue` — except `colorValue` was being sent *after* `colorField` in code, so Bubble's workflow ran with the **previous** colour value. Each save was one colour "behind."

**Fix:** Swapped the call order — send `colorValue` first, then `colorField` (the trigger) last, so the workflow always reads the freshly-updated value:
```javascript
bubble_fn_colorValue(hexValue);   // value first
bubble_fn_colorField(fieldName);  // trigger last
```

> ⚠️ **Superseded by Fix #9 (this session).** Reordering two calls only reduced the race window, it didn't remove it — Bubble could still process them as two separate workflow triggers. The two-call architecture itself has now been replaced with a single consolidated call, eliminating the race condition at the root instead of working around it.

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
  const btn = document.querySelector('.pcr-button');
  if (!btn) return;
  const rgba = color.toRGBA();
  btn.style.setProperty('--trigger-color', `rgba(${Math.round(rgba[0])}, ${Math.round(rgba[1])}, ${Math.round(rgba[2])}, ${rgba[3]})`);
}
```
*(Now scoped via `pickr.getRoot().button` after the class-based conversion — see Context note above.)*

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
> ⚠️ **Partially superseded by Fix #10 (this session).** This correctly reset to `defaultColor` — but `defaultColor` itself was never updated after the initial page load, so it could go stale after a successful save. See Fix #10.

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

**Fix:** Reintroduced a `pickr.on('change', ...)` listener (removed earlier in Fix #2, now serves a real purpose). A `pickrDefaultHexStr` variable holds the normalized hex of the currently-saved colour — set on `init` from `instance.getColor()`, and updated again on every successful Save. On every `change` event, the live preview colour is compared against this baseline:
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
  var saveBtn = pickr.getRoot().interaction.result ? pickr.getRoot().interaction.result.parentElement.querySelector('.pickr-save-btn-el') : null;
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
On Save click, `pickrDefaultHexStr` is updated to the newly saved hex *before* `updateSaveButtonState()` runs, so the button immediately flips back to gray (its current preview now matches the new baseline).

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
- Swatch count was adjusted a few times during testing purely to test row behavior at different counts. **Current count: 14 swatches** — at this count, with default sizing, the row **will wrap to 2 lines** since 14 default-sized swatches don't fit in one row width. If a strict single-row layout is needed again at 14 swatches, the explicit small-size CSS from the earlier iteration will need to be reapplied.

### 8. Dynamic popup position (upper vs lower half of screen) — *removed in current file*
**Note:** An earlier iteration included a position-detection block that picked `top-start` vs `bottom-start` based on where the trigger sits on screen:
```javascript
var rect = triggerEl.getBoundingClientRect();
var viewportHeight = window.innerHeight;
dynamicPosition = (rect.top > viewportHeight * 0.5) ? 'top-start' : 'bottom-start';
```
This is **no longer present** in the current file — `dynamicPosition` is now hardcoded to `'bottom-start'`. Flagging here in case the dynamic version was actually still wanted; if so it needs to be reinstated (and re-scoped to use `pickr.getRoot()`/the class-based trigger lookup rather than a global selector).

---

## Fixes applied in this session (June 24, 2026)

### 9. Two-call race condition replaced with a single consolidated payload
**Problem:** Even with the call order fixed (Fix #1), sending `colorValue` and `colorField` as two separate `bubble_fn_*` calls meant Bubble received two independent workflow triggers in quick succession. Depending on Bubble's event queue timing, this could still occasionally race — the second call's workflow could start running before the first call's value had fully landed in the relevant state, especially on pages with other workflows competing for the event loop.

**Fix:** Replaced both calls with a single `bubble_fn_colorData` call carrying a JSON-stringified object with both pieces of data together. One call, one event, one atomic payload — no ordering or timing dependency at all:
```javascript
var payload = { field: fieldName, value: hexValue };
bubble_fn_colorData(JSON.stringify(payload));
```
On the Bubble side, this replaces the two `colorValue` / `colorField` elements with a single `colorData` element; the workflow now parses the JSON string to read out `field` and `value`.

### 10. Unsaved-preview-reset bug used a stale page-load colour instead of the latest saved colour
**Problem:** Fix #4 correctly reset the popup to `defaultColor` when closed without saving — but `defaultColor` was only ever set once, from `rawColor` at page load. So: save a colour → reopen the picker → drag to preview a different colour → close without saving → the picker incorrectly fell back to the *original page-load* colour instead of the colour that had actually just been saved.

**Fix:** `defaultColor` is now reassigned to the new `hexValue` inside the Save button's click handler, right alongside `pickrDefaultHexStr`. The "what to reset to on cancel" baseline now always reflects the most recently saved colour, not just the one Bubble handed over on load:
```javascript
defaultColor = hexValue;
```

### 11. Null-safety added around the interaction-result lookup on init
**Problem:** `pickr.getRoot().interaction.result.parentElement` assumed `.result` would always exist by the time `init` fires. In edge cases (e.g. a config without the `result` component enabled, or Pickr's DOM not being fully constructed yet on a slow iframe load), this would throw and silently break Save button injection for that instance.

**Fix:** Split into a guarded two-step lookup:
```javascript
var result = pickr.getRoot().interaction.result;
var interaction = result ? result.parentElement : null;
```
`interaction` is `null` instead of throwing if `.result` isn't there yet — the existing `if(interaction) {...}` block already handles that gracefully by simply skipping Save button injection.

### 12. Global scope namespace collisions (IIFE scoping)
**Problem:** Running a raw top-level function declaration (`function initPickr() {}`) across 10–15 duplicated element instances on the same page risks variable leakage if the runtime environment ever collapses iframe boundaries (e.g. some Bubble rendering paths, or if a future refactor moves shared code into a parent scope).

**Fix:** Wrapped the entire runtime block in an Immediately Invoked Function Expression (IIFE) and renamed the inner function so it's not relying on bare global scope:
```javascript
(function() {
  function initPickrInstance() { ... }
  initPickrInstance();
})();
```
This was already a single self-contained `<script>` per instance, so the practical risk was low — but the IIFE removes the ambiguity entirely and costs nothing.

### 13. Duplicate Save button stacking on re-render
**Problem:** Bubble can re-render an HTML element's contents when a parent component's visibility or data changes, re-running the whole `<script>` block. The old Pickr instance is destroyed via `destroyAndRemove()`, but in some re-render paths a stale `.pickr-save-btn-el` button could remain cached in the DOM, and a second one would get appended on top of it — stacking duplicate Save buttons inside the popup.

**Fix:** Added an explicit removal step immediately before creating the new Save button, so each `init` only ever has one Save button in the DOM at a time:
```javascript
var existingBtn = interaction.querySelector('.pickr-save-btn-el');
if (existingBtn) existingBtn.remove();
```

---

## ⚠️ Needs live testing (not verified in a real browser by Claude)

- Confirm the `!important` CSS lock on `.pcr-button` actually overrides Pickr's own internal style updates in all browsers used. If Pickr ever sets inline styles with `'important'` priority via `style.setProperty(prop, val, 'important')`, it could still win over the CSS rule — if so, the trigger may still flicker during preview, and the fix would need to move to a `useAsButton: true` setup instead.
- Confirm `pickr.setColor(defaultColor)` on `hide` doesn't cause any visible flash/flicker before the popup fully closes.
- Confirm 14 swatches at default sizing renders as expected (2 rows) and that this is the desired look — if a single row is wanted, revisit Fix #7's explicit sizing approach.
- **(New)** Confirm the Bubble-side workflow correctly parses the new `colorData` JSON payload and reads `field`/`value` out of it without errors — this requires updating the Bubble workflow to match the new single-element data flow (it previously expected two separate elements).
- **(New)** Confirm that after Save → reopen → preview a different colour → close without saving, the trigger and popup both correctly fall back to the *just-saved* colour (Fix #10), not the original page-load colour.
- **(New)** Confirm no console errors are thrown if Pickr's `result` component is ever unavailable at `init` time (Fix #11) — this is a defensive fix for an edge case that may not be easily reproducible in normal testing.
- Confirm whether the dynamic `top-start`/`bottom-start` positioning (old Fix #8) is actually still wanted, since it's currently absent from the file — flagged as an open question, not a bug.
- **(New)** Confirm the Bubble-side regex conditions (`(?<="field":")[^"]+` and `(?<="value":")[^"]+`) correctly extract `field` and `value` from the `colorData` JSON string across all 10–15 duplicated instances, with exact case-sensitive matching on `fieldName`.
- **(New)** Confirm that triggering a re-render of the parent Reusable Element (e.g. toggling visibility) does not produce a stacked/duplicate Save button (Fix #13).
- **(New)** Confirm the IIFE wrapping (Fix #12) doesn't change any externally-visible behavior — it shouldn't, but worth a quick sanity check across a couple of instances on the same page at once.

---

## Current complete HTML code (per instance — copy into each Bubble HTML element)

> Each HTML element only needs **two values changed**: `rawColor` and `fieldName`. Everything else stays identical across all instances. Multi-instance safety comes from the per-field `instanceKey` and from scoping all internal DOM lookups through `pickr.getRoot()` rather than global selectors.

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

  // Har instance ka apna unique storage key (fieldName se bana) - global window pollution avoid karne ke liye
  var instanceKey = 'pickr_' + fieldName;

  // NOTE: Bubble har HTML element ko apne ALAG iframe mein render karta hai
  // (especially jab Reusable Element ke andar Group mein repeated ho).
  // Isliye is iframe ke document mein sirf EK hi '.pickr-trigger-el' hoga -
  // koi clash possible nahi hai, simple querySelector kaafi hai.
  var triggerEl = document.querySelector('.pickr-trigger-el');

  if(window[instanceKey]) {
    window[instanceKey].destroyAndRemove();
    window[instanceKey] = null;
  }

  var defaultColor = /^#([0-9A-Fa-f]{3}|[0-9A-Fa-f]{6})$/.test(rawColor) ? rawColor : '#2563EB';

  // Popup hamesha trigger ke NEECHE khulega
  var dynamicPosition = 'bottom-start';

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

  var savedThisSession = false; // track kiya save hua ya nahi
  var pickrDefaultHexStr = ''; // init event mein normalized hex se set hoga

  // Trigger circle ka colour LOCK karta hai (sirf init / save pe update hota hai)
  // pickr.getRoot().button se apna hi button milta hai - poore page mein search nahi
  function lockTriggerColor(color) {
    const btn = pickr.getRoot().button;
    if (!btn) return;
    const rgba = color.toRGBA();
    btn.style.setProperty('--trigger-color', `rgba(${Math.round(rgba[0])}, ${Math.round(rgba[1])}, ${Math.round(rgba[2])}, ${rgba[3]})`);
  }

  // Save button ka color update karta hai - match = gray, mismatch = blue
  function updateSaveButtonState() {
    var saveBtn = pickr.getRoot().interaction.result ? pickr.getRoot().interaction.result.parentElement.querySelector('.pickr-save-btn-el') : null;
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
    lockTriggerColor(instance.getColor()); // Bubble ka saved colour dikhao
    pickrDefaultHexStr = instance.getColor().toHEXA().toString().toUpperCase(); // normalized reference

    // Save button inject karo HEXA/RGBA ke neeche - apne hi popup ke interaction mein (pickr.getRoot() se)
    // Null-safe: Pickr DOM fully constructed na ho to bhi error na aaye
    var result = pickr.getRoot().interaction.result;
    var interaction = result ? result.parentElement : null;
    if(interaction) {
      // Re-render guard: agar purana save button DOM mein reh gaya ho (destroyAndRemove ke baad
      // bhi cached element), to duplicate stack hone se pehle hata do
      var existingBtn = interaction.querySelector('.pickr-save-btn-el');
      if (existingBtn) existingBtn.remove();

      var saveBtn = document.createElement('button');
      saveBtn.className = 'pickr-save-btn-el';
      saveBtn.textContent = 'SAVE';
      saveBtn.onclick = function() {
        var selected = pickr.getColor(); // abhi popup mein jo selected hai
        if(selected) {
          var hexValue = selected.toHEXA().toString();
          if(hexValue && hexValue !== '' && hexValue !== '#') {
            savedThisSession = true;           // mark kiya ki save hua
            lockTriggerColor(selected);        // trigger ko naya confirmed colour dikhao
            pickrDefaultHexStr = hexValue.toUpperCase(); // naya baseline - ab yehi "saved" colour hai
            defaultColor = hexValue;           // reopen/cancel ke liye naya baseline (purana page-load colour overwrite)
            updateSaveButtonState();           // button wapas gray ho jaaye

            // Consolidated payload - dono values EK saath bhejo (race condition avoid karne ke liye)
            var payload = { field: fieldName, value: hexValue };
            bubble_fn_colorData(JSON.stringify(payload));

            pickr.hide();
          }
        }
      };
      interaction.appendChild(saveBtn);
      saveBtn.style.display = 'block';
      updateSaveButtonState(); // initial state - gray (kyunki abhi koi change nahi hua)
    }
  });

  // Jab bhi color preview mein change ho (drag, swatch click, hex type), save button color check karo
  pickr.on('change', () => {
    updateSaveButtonState();
  });

  // Popup band hone par check karo - agar bina save band hua hai
  // to popup ke andar wapas Bubble ki original value set karo
  pickr.on('hide', () => {
    if(!savedThisSession) {
      pickr.setColor(defaultColor);
    }
    savedThisSession = false; // reset flag agle round ke liye
  });
}

initPickrInstance();
})();
</script>
```

---

## Quick checklist for next session

- [ ] Update the Bubble workflow to read from the new single `colorData` element instead of the old `colorValue` + `colorField` pair — parse the JSON payload to pull out `field` and `value` (regex patterns above)
- [ ] Delete (or leave unused) the old `colorValue` / `colorField` JS-to-Bubble elements once the workflow has been migrated, to avoid confusion
- [ ] Confirm in live Bubble preview: trigger circle does NOT change while dragging inside popup
- [ ] Confirm: closing popup without Save → reopening shows the **most recently saved** colour, not the original page-load colour and not a leftover preview (Fix #10)
- [ ] Confirm: Save sends one consolidated `colorData` payload and the workflow fires exactly once per save, with no lag or race (Fix #9)
- [ ] Confirm: Save button is gray when preview colour matches saved colour, blue when it differs
- [ ] Confirm: 14 swatches render acceptably (currently wraps to 2 rows at default sizing) — decide if single-row is still wanted
- [ ] Decide whether the dynamic top/bottom popup positioning (old Fix #8) should be reinstated — it's currently absent, hardcoded to `bottom-start`
- [ ] Confirm re-rendering the parent Reusable Element doesn't stack a duplicate Save button (Fix #13)
- [ ] Roll out the same changes to all remaining HTML elements (10–15 total per earlier discussion)

### Reported as verified (carried over from latest update — not independently re-tested by Claude)
- [x] **Atomic data flow** — legacy `colorValue` + `colorField` workflows combined into a single `colorData` JSON payload
- [x] **Spelling/case sync** — Bubble regex conditions confirmed checking exact-case `titleColor` (capital C) to match the JS `fieldName` payload
- [x] **DOM preservation on re-render** — guarded by wiping any pre-existing `.pickr-save-btn-el` before injecting a new one (Fix #13)
- [x] **Local scoping** — all CDN assets, CSS, and JS kept self-contained per instance; no parent Page Header dependency
