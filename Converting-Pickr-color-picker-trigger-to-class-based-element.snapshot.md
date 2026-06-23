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

## Fixes applied in this session

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

## ⚠️ Needs live testing (not verified in a real browser by Claude)

- Confirm the `!important` CSS lock on `.pcr-button` actually overrides Pickr's own internal style updates in all browsers used. If Pickr ever sets inline styles with `'important'` priority via `style.setProperty(prop, val, 'important')`, it could still win over the CSS rule — if so, the trigger may still flicker during preview, and the fix would need to move to a `useAsButton: true` setup instead.
- Confirm `pickr.setColor(defaultColor)` on `hide` doesn't cause any visible flash/flicker before the popup fully closes.

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
    width:100%;
    margin-top:8px;
    padding:8px 0;
    border-radius:6px;
    background:#2563EB;
    color:#fff;
    font-size:12px;
    font-weight:600;
    letter-spacing:.03em;
    cursor:pointer;
    border:none;
    outline:none;
    transition:.15s;
}
#pickr-save-btn:hover{
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

  const pickr = Pickr.create({
    el: '#pickr-trigger',
    theme: 'nano',
    default: defaultColor,
    position: 'bottom-start',
    hideOnSelect: false,
    autoReposition: true,
    swatches: ['#2563EB','#3B82F6','#06B6D4','#10B981','#22C55E','#F59E0B','#EF4444','#EC4899'],
    components: {
      preview: true,
      opacity: true,
      hue: true,
      interaction: { hex: true, rgba: true, input: true }
    }
  });

  window.myPickr = pickr;

  var savedThisSession = false; // track kiya save hua ya nahi

  // Trigger circle ka colour LOCK karta hai (sirf init / save pe update hota hai)
  function lockTriggerColor(color) {
    const btn = document.querySelector('.pcr-button');
    if (!btn) return;
    const rgba = color.toRGBA();
    btn.style.setProperty('--trigger-color', `rgba(${Math.round(rgba[0])}, ${Math.round(rgba[1])}, ${Math.round(rgba[2])}, ${rgba[3]})`);
  }

  pickr.on('init', (instance) => {
    lockTriggerColor(instance.getColor()); // Bubble ka saved colour dikhao

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
            bubble_fn_colorValue(hexValue);    // PEHLE colour value bhejo
            bubble_fn_colorField(fieldName);   // PHIR field name (trigger)
            pickr.hide();
          }
        }
      };
      interaction.appendChild(saveBtn);
      saveBtn.style.display = 'block';
    }
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
- [ ] Roll out the same two-line change (`rawColor`, `fieldName`) to all remaining HTML elements (10–15 total per earlier discussion)
