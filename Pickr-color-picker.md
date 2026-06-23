# Pickr Color Picker — Bubble.io Integration — Session Update

**Continuing from:** Converting Pickr color picker trigger to class-based element
**Last updated:** June 23, 2026

---

## Context

Bubble.io project using an HTML element with the Pickr (`@simonwep/pickr`) colour picker library. Multiple instances of this HTML element exist on the same page (one per editable field — e.g. `titleColor`, `textColor`, `bgColor`), each with its own `rawColor` (current Bubble value) and `fieldName`.

Two JS-to-Bubble elements handle the data flow:
- **`colorValue`** — text, Trigger event ON, Publish value ON
- **`colorField`** — text, Trigger event ON, Publish value ON

Bubble workflow: triggered on `colorField`'s event → conditions check `colorField's value` → sets the relevant custom state from `colorValue's value`.

---

## Fixes applied in earlier session

### 1. Old-value-lag bug (JS-to-Bubble call order)
**Problem:** Workflow was triggering on `colorField` (the field-identifier call), but reading `colorValue` — except `colorValue` was being sent *after* `colorField` in code, so Bubble's workflow ran with the **previous** colour value. Each save was one colour "behind."

**Fix:** Swapped the call order — send `colorValue` first, then `colorField` (the trigger) last, so the workflow always reads the freshly-updated value:
```javascript
bubble_fn_colorValue(hexValue);   // value first
bubble_fn_colorField(fieldName);  // trigger last
```

### 2. Dead weight / redundant code cleanup
- Removed `closeOnScroll: false` from Pickr config — this is already Pickr's default, the line did nothing.
- Removed the `currentColor` variable and the entire `pickr.on('change', ...)` listener used only to track it — replaced with a direct `pickr.getColor()` call inside the Save button click handler. One less listener, no redundant state tracking.
  - *(Note: a `change` listener was reintroduced in this session for the dirty-state save button — see Fix #3 below. It now serves a real purpose instead of just tracking state.)*

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

---

## Fixes applied in this session

### 5. Save button moved to bottom of popup
**Problem:** Save button was rendering above the HEXA/RGBA result row inside `.pcr-interaction` (default flex order), which felt visually out of place.

**Fix:** Set explicit flex `order` on the Save button so it always lands last, after the result row (`order:1`) and the HEXA/RGBA toggle (`order:2`):
```css
#pickr-save-btn{
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
#pickr-save-btn{
    background:#D1D5DB;
    cursor:not-allowed;
}
#pickr-save-btn.pcr-dirty{
    background:#2563EB;
    cursor:pointer;
}
```
```javascript
function updateSaveButtonState() {
  var saveBtn = document.getElementById('pickr-save-btn');
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

**Fix (iterated through several approaches this session):**
- First tried `flex-wrap: nowrap` + explicit small swatch sizing (`14–18px`) + `justify-content: space-between` on `.pcr-swatches` — worked, but per later request the explicit sizing was reverted.
- **Final state:** swatch sizing is back to Pickr's own default (no overridden `width`/`height` on `.pcr-swatch`), and only wrapping/alignment is controlled:
```css
.pcr-app .pcr-swatches{
    flex-wrap:nowrap !important;
    justify-content:space-between !important;
}
```
- Swatch count was adjusted a few times during testing (8 → 6 → 7 → 14) purely to test row behavior at different counts. **Current count: 14 swatches** (see code block below) — at this count, with default sizing, the row **will wrap to 2 lines** since 14 default-sized swatches don't fit in one row width. If a strict single-row layout is needed again at 14 swatches, the explicit small-size CSS from the earlier iteration will need to be reapplied.

### 8. Dynamic popup position (upper vs lower half of screen)
**Note:** The current file includes a position-detection block that picks `top-start` vs `bottom-start` based on where the trigger sits on screen (added/carried over outside of this session's explicit requests — flagging here so it's documented and not lost):
```javascript
var triggerEl = document.querySelector('#pickr-trigger');
var dynamicPosition = 'bottom-start';
if (triggerEl) {
  var rect = triggerEl.getBoundingClientRect();
  var viewportHeight = window.innerHeight;
  dynamicPosition = (rect.top > viewportHeight * 0.5) ? 'top-start' : 'bottom-start';
}
```
This is passed into `position: dynamicPosition` in the Pickr config instead of the previously hardcoded `'bottom-start'`. **Needs verification** that this is actually wanted/tested — flagging for confirmation next session.

---

## ⚠️ Needs live testing (not verified in a real browser by Claude)

- Confirm the `!important` CSS lock on `.pcr-button` actually overrides Pickr's own internal style updates in all browsers used. If Pickr ever sets inline styles with `'important'` priority via `style.setProperty(prop, val, 'important')`, it could still win over the CSS rule — if so, the trigger may still flicker during preview, and the fix would need to move to a `useAsButton: true` setup instead.
- Confirm `pickr.setColor(defaultColor)` on `hide` doesn't cause any visible flash/flicker before the popup fully closes.
- Confirm 14 swatches at default sizing renders as expected (2 rows) and that this is the desired look — if a single row is wanted, revisit Fix #7's explicit sizing approach.
- Confirm the dynamic `top-start`/`bottom-start` position logic (Fix #8) behaves correctly across all instances on the page, especially ones near the vertical midpoint of the viewport.

---

## Current complete HTML code (per instance — copy into each Bubble HTML element)

> Each HTML element only needs **two values changed**: `rawColor` and `fieldName`. Everything else stays identical across all instances.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@simonwep/pickr/dist/themes/nano.min.css"/>
<div id="pickr-trigger"></div>
<style>
#pickr-trigger{
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
#pickr-save-btn{
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
#pickr-save-btn.pcr-dirty{
    background:#2563EB;
    cursor:pointer;
}
#pickr-save-btn.pcr-dirty:hover{
    background:#1d4ed8;
}
</style>
<script src="https://cdn.jsdelivr.net/npm/@simonwep/pickr/dist/pickr.min.js"></script>
<script>
function initPickr() {
  if(typeof Pickr === 'undefined') return setTimeout(initPickr, 100);

  if(window.myPickr) {
    window.myPickr.destroyAndRemove();
    window.myPickr = null;
  }

  // ---- YAHAN PE BUBBLE KI DYNAMIC VALUE DAALO ----
  var rawColor = '#343882';     // is field ki current colour - dynamic karo Bubble se
  var fieldName = 'titleColor'; // is picker ka field name - har HTML element mein alag likho
  // ------------------------------------------------

  var defaultColor = /^#([0-9A-Fa-f]{3}|[0-9A-Fa-f]{6})$/.test(rawColor) ? rawColor : '#2563EB';

  // Trigger page ke upper half mein hai ya lower half mein, us hisaab se popup ka direction decide karo
  var triggerEl = document.querySelector('#pickr-trigger');
  var dynamicPosition = 'bottom-start';
  if (triggerEl) {
    var rect = triggerEl.getBoundingClientRect();
    var viewportHeight = window.innerHeight;
    dynamicPosition = (rect.top > viewportHeight * 0.5) ? 'top-start' : 'bottom-start';
  }

  const pickr = Pickr.create({
    el: '#pickr-trigger',
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

  window.myPickr = pickr;

  var savedThisSession = false; // track kiya save hua ya nahi
  var pickrDefaultHexStr = ''; // init event mein normalized hex se set hoga

  // Trigger circle ka colour LOCK karta hai (sirf init / save pe update hota hai)
  function lockTriggerColor(color) {
    const btn = document.querySelector('.pcr-button');
    if (!btn) return;
    const rgba = color.toRGBA();
    btn.style.setProperty('--trigger-color', `rgba(${Math.round(rgba[0])}, ${Math.round(rgba[1])}, ${Math.round(rgba[2])}, ${rgba[3]})`);
  }

  // Save button ka color update karta hai - match = gray, mismatch = blue
  function updateSaveButtonState() {
    var saveBtn = document.getElementById('pickr-save-btn');
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

    // Save button inject karo HEXA/RGBA ke neeche
    var interaction = document.querySelector('.pcr-app .pcr-interaction');
    if(interaction) {
      var saveBtn = document.createElement('button');
      saveBtn.id = 'pickr-save-btn';
      saveBtn.textContent = 'SAVE';
      saveBtn.onclick = function() {
        var selected = pickr.getColor(); // abhi popup mein jo selected hai
        if(selected) {
          var hexValue = selected.toHEXA().toString();
          if(hexValue && hexValue !== '' && hexValue !== '#') {
            savedThisSession = true;           // mark kiya ki save hua
            lockTriggerColor(selected);        // trigger ko naya confirmed colour dikhao
            pickrDefaultHexStr = hexValue.toUpperCase(); // naya baseline - ab yehi "saved" colour hai
            updateSaveButtonState();           // button wapas gray ho jaaye
            bubble_fn_colorValue(hexValue);    // PEHLE colour value bhejo
            bubble_fn_colorField(fieldName);   // PHIR field name (trigger)
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

initPickr();
</script>
```

---

## Quick checklist for next session

- [ ] Confirm in live Bubble preview: trigger circle does NOT change while dragging inside popup
- [ ] Confirm: closing popup without Save → reopening shows Bubble's saved colour, not leftover preview
- [ ] Confirm: Save still sends correct `colorValue` + `colorField` to the workflow, one-to-one (no lag)
- [ ] Confirm: Save button is gray when preview colour matches saved colour, blue when it differs
- [ ] Confirm: 14 swatches render acceptably (currently wraps to 2 rows at default sizing) — decide if single-row is still wanted
- [ ] Confirm/verify the dynamic top/bottom popup positioning logic (Fix #8) — flagged as unconfirmed, was present in code but not explicitly requested this session
- [ ] Roll out the same changes to all remaining HTML elements (10–15 total per earlier discussion)
