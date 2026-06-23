# Pickr Color Picker — Bubble.io Integration — Session Snapshot

**Last updated:** June 23, 2026

---

## Project Overview

Bubble.io project using `@simonwep/pickr` colour picker inside a Reusable Element. One HTML element per editable field (e.g. `titleColor`, `textColor`, `bgColor`), duplicated 10–15 times on the same page.

**Bubble data flow:**
- `bubble_fn_colorValue(hexValue)` — sends the selected colour
- `bubble_fn_colorField(fieldName)` — triggers the workflow
- **Order matters:** `colorValue` must fire first, `colorField` second — otherwise the workflow reads the previous value

---

## File Structure (Current)

| File | Where it goes | Purpose |
|---|---|---|
| `pickr-global-header.html` | Bubble Page/App Header | CDN links + all shared CSS — loads once |
| `pickr-instance-element.html` | Each Bubble HTML Element | Trigger div + per-instance JS only |

---

## Per-Instance Setup

Only two values change per instance — everything else is identical:

```js
var rawColor  = '#343882';     // current saved colour from Bubble (dynamic)
var fieldName = 'titleColor';  // unique field name for this picker
```

---

## Key Implementation Details

### Global header loads once
Pickr CDN `<link>` and `<script>` are in the header — not repeated per instance. This eliminates 10–15x redundant network requests.

### IIFE scope isolation
Each instance script is wrapped in `(function(){ ... })()` to prevent global scope conflicts across iframes.

### No `setTimeout` polling
Removed `if(typeof Pickr === 'undefined') return setTimeout(initPickr, 100)` — Pickr is guaranteed loaded from the header before any instance script runs.

### `window.parent.Pickr` reference
Since Bubble renders each HTML element inside its own iframe, `Pickr` loaded in the page header lives on the **parent** window, not the iframe's `window`. The instance script resolves it explicitly, with a safety fallback:
```js
var PickrLib = (window.parent && window.parent.Pickr) ? window.parent.Pickr : window.Pickr;
var pickr = PickrLib.create({ ... });
```

### Instance keying via `fieldName`
```js
var instanceKey = 'pickr_' + fieldName;  // e.g. 'pickr_titleColor'
```
Each instance stores its Pickr object at `window[instanceKey]`, preventing cross-instance collisions.

### Trigger circle colour lock
`.pcr-button` background is controlled via a CSS custom property, only updated at two points — **init** and **Save click**. Live drag/preview does not affect the trigger circle.
```js
btn.style.setProperty('--trigger-color', 'rgba(...)');
```

### Unsaved preview reset
If the popup is closed without saving, `pickr.setColor(defaultColor)` restores Bubble's saved value inside the popup on next open.
```js
pickr.on('hide', function () {
  if (!savedThisSession) pickr.setColor(defaultColor);
  savedThisSession = false;
});
```

### Save button dirty state
Save button is gray (`#D1D5DB`, `cursor:not-allowed`) when the preview matches the saved colour, blue (`#2563EB`) when it differs. Controlled by `.pcr-dirty` class toggled on every `change` event.

### Null-safe interaction reference
`pickr.getRoot().interaction.result` is null-checked before accessing `.parentElement` — prevents a throw if Pickr's DOM isn't fully ready.

---

## Swatches (Current: 14)

```js
['#2563EB','#3B82F6','#06B6D4','#10B981','#22C55E',
 '#F59E0B','#EF4444','#EC4899','#8B5CF6','#A855F7',
 '#14B8A6','#84CC16','#F97316','#6366F1']
```

At default Pickr swatch sizing, 14 swatches wrap to 2 rows. Reduce count or add explicit small sizing to force a single row.

---

## Pending Verification (Not yet tested in live Bubble)

- [ ] Trigger circle does not flicker during colour drag inside popup
- [ ] Closing without Save → reopening shows Bubble's saved colour, not leftover preview
- [ ] Save sends correct `colorValue` + `colorField` with no value lag
- [ ] Save button dirty state (gray/blue) toggles correctly
- [ ] 14 swatches at 2 rows is acceptable — revisit if single row is needed
