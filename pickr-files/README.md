# Pickr Color Picker — Bubble.io Setup Guide

A lightweight, customizable color picker built with the [Pickr](https://github.com/Simonwep/pickr) library, designed to work seamlessly inside Bubble.io as a self-contained HTML element. Each instance is fully independent — meaning you can have multiple color pickers on the same page (e.g. Title Color, Text Color, Background Color) without any conflicts.

The full code (`pickr-instance-element.html`) and session notes (`pickr-bubble-snapshot.md`) are available in this repository: **[GitHub link — coming soon]**

---

## Prerequisites

- **Toolbox plugin** installed in your Bubble app. Search for "Toolbox" in the Bubble Plugin Marketplace and install it — it's free.

---

## Step 1 — Add the HTML Element and Create a Custom State

Add a standard **HTML** element to your Bubble page where you want the color picker trigger circle to appear.

Set the HTML element's size to exactly **24px wide and 24px tall**. The trigger circle is sized to fit these dimensions — changing them will break the layout.

Next, create a custom state on this HTML element:

| Field | Value |
|---|---|
| State name | `hexValue` |
| Type | `text` |

**Why `hexValue` on the HTML element itself?**
The code reads `rawColor` (the current saved color) as a dynamic value from this element's own `hexValue` state. This state is only updated when the user clicks the **Save** button inside the picker (when `saved: true` is sent from JavaScript). If it updated on every color drag or preview, Bubble would re-render the HTML element on every change — which would destroy the Pickr instance and close the popup mid-use. Keeping `hexValue` stable until Save is clicked prevents this entirely.

---

## Step 2 — Paste the Code

Copy the full code from `pickr-instance-element.html` in this repository and paste it into the HTML element's content field in Bubble.

There are **two values you must change** for every instance:

### `rawColor`
```javascript
var rawColor = '#343882'; // Replace with dynamic value
```
This is the current saved color that the picker loads with on open. Set this to the `hexValue` custom state of this HTML element dynamically using Bubble's expression editor:

> `This HTMLElement's hexValue`

If `hexValue` is empty (first load, no color saved yet), the code automatically falls back to `#2563EB` (blue) as the default. You can change this fallback color directly in the code if needed.

### `fieldName`
```javascript
var fieldName = 'titleColor'; // Replace with a unique identifier
```
This is a unique string that identifies which color picker is sending data to Bubble. Every instance on the page **must have a different `fieldName`** — otherwise Bubble workflows won't be able to tell which picker triggered the event.

Use a clear, descriptive name with no spaces. Examples:
- `titleColor`
- `textColor`
- `bgColor`
- `borderColor`

> ⚠️ `fieldName` is case-sensitive. Whatever string you set here must match exactly in your Bubble workflow conditions.

---

## Step 3 — Add the JavascripttoBubble Element

From the Toolbox plugin, add a **JavascripttoBubble** element anywhere on the page. You only need **one** — it handles all color picker instances on the page.

Configure it with the following settings:

| Setting | Value |
|---|---|
| Suffix | `colorData` |
| Trigger event | ✅ On |
| Publish value | ✅ On |
| Value type | `Text` |

**Why these settings?**
Every time the user drags a color (live preview), clicks Save, or dismisses the popup without saving, the picker sends a JSON string to Bubble through this element. The `colorData` suffix matches the `bubble_fn_colorData(...)` call in the JavaScript code. Trigger event and Publish value must both be ON so Bubble can detect the event and read the value in a workflow.

**What the payload looks like:**

On every color change (live drag, 500ms debounced):
```json
{ "field": "titleColor", "value": "#FF5733", "saved": false }
```

On Save button click:
```json
{ "field": "titleColor", "value": "#FF5733", "saved": true }
```

On clicking outside the popup without saving:
```json
{ "field": "titleColor", "value": "#343882", "saved": false }
```
The picker sends the `defaultColor` (last saved colour) back to Bubble with `saved: false`. This fires Workflow 1 (live preview), reverting the Bubble custom state to the correct saved colour. The pending debounce is also cancelled first, so any in-flight preview value from the last drag doesn't fire after close.

**Extracting values in Bubble using Regex:**

Use Bubble's `extract with Regex` operator (`:first item`) to pull individual values out of the JSON string:

| Value | Regex pattern |
|---|---|
| `field` | `(?<="field":")[^"]+` |
| `value` | `(?<="value":")[^"]+` |
| `saved` | `(?<="saved":)(true\|false)` |

---

## Step 4 — Set Up Bubble Workflows

All color picker instances share a **single workflow trigger**:

> **Trigger:** `When JavascripttoBubble colorData has an event`

Inside this one trigger, you add **2 actions per color picker instance**. So if you have 3 instances (`titleColor`, `textColor`, `bgColor`), you'll have 6 actions total — all inside the same trigger.

Here's how to set up the 2 actions for each instance:

---

### Action 1 — Update your target element's color state

This action fires on both live preview (drag), dismiss without save (revert), and confirmed save. It updates whichever element or state you want the selected color to apply to — for example, a custom state on a parent group, a reusable element, or a data type field.

| Field | Value |
|---|---|
| Element | The element where you want the color applied (e.g. `1 - Proposal`) |
| Custom state | The state that holds the color value (e.g. `Title Color`) |
| Value | `This JavascripttoBubble's value:extract with Regex:first item` — using the `value` regex pattern |
| Only when | `This JavascripttoBubble's value:extract with Regex:first item` **is** `titleColor` — using the `field` regex pattern |

**Why no `saved` check here?**
This action runs on all `saved: false` payloads (live drag preview and dismiss revert) and `saved: true` (confirmed save) — so the color updates in real time as the user drags, and reverts correctly if dismissed. The `field` condition ensures only the right picker updates the right state.

---

### Action 2 — Update the HTML element's `hexValue` state

This action only fires when the user clicks Save. It updates the `hexValue` custom state on the HTML element itself — which feeds back into `rawColor` in the code, locking the picker's starting color for the next open.

| Field | Value |
|---|---|
| Element | The HTML element for this picker (e.g. `Title Colour - Picker`) |
| Custom state | `hexValue` |
| Value | `This JavascripttoBubble's value:extract with Regex:first item` — using the `value` regex pattern |
| Only when | `This JavascripttoBubble's value:extract with Regex:first item` **is** `titleColor` (field regex) **and** `This JavascripttoBubble's value:extract with Regex:first item` **is** `true` (saved regex) |

**Why the `saved: true` condition?**
If `hexValue` updated on every drag or revert, Bubble would re-render the HTML element — destroying the Pickr instance and closing the popup mid-use. Restricting this to `saved: true` keeps the popup stable during live preview and ensures the dismiss-revert payload (`saved: false`) doesn't accidentally overwrite the confirmed saved colour.

---

### Adding more instances

For each additional color picker (e.g. `textColor`, `bgColor`), add another pair of Action 1 + Action 2 inside the **same trigger** — just change the `fieldName` in the `Only when` conditions to match that instance's `fieldName`.

---

## Using Multiple Instances

You can add as many color picker instances to the same page as you need — each one is a separate HTML element with its own `hexValue` custom state and a unique `fieldName` in the code.

**Important:** Keep the number of simultaneously **visible** color picker instances to **4–5 at most**. Beyond that, Bubble may struggle to handle multiple Pickr instances rendering at the same time, which can cause pickers to break or behave unexpectedly. If you have more than 5 color pickers on a page, consider showing them in separate groups or sections so only a few are visible at any given time.

A single JavascripttoBubble element (`colorData`) handles all instances — you do not need to add more than one.

**Summary for each new instance:**
- Add a new HTML element (24×24px)
- Create a `hexValue` custom state on it (type: text)
- Paste the full code
- Set `rawColor` to `This HTMLElement's hexValue`
- Set a unique `fieldName` string
- Everything else in the code stays identical
