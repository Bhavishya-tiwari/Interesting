# Browser Rendering Pipeline — Color Journey

How a color travels from the backend API to the light on your screen, explained with two examples from the pricing page.

---

## The Pipeline

```
Backend JSON
   ↓
React state update
   ↓
DOM update (CSS variable set)          ← just data, no pixels
   ↓
Style calculation (resolve final CSS)  ← just data
   ↓
Layout (positions & sizes)             ← just data
   ↓
Paint (draw instructions)              ← just a recipe
   ↓
GPU Rasterize (recipe → pixels)        ← first actual pixels created here
   ↓
GPU Effects (filters, blending)        ← backdrop-filter and opacity applied here
   ↓
GPU Composite (stack all layers)       ← final image in display color space
   ↓
Framebuffer → display cable → monitor  ← voltage to LEDs → light
   ↓
Your eyes
```

---

## Stage-by-Stage Breakdown

### 1. Backend JSON

The backend sends the theme configuration:

```json
{
  "theme": {
    "background": { "color": "#77FAFA" },
    "brand": { "color": "#FF00FF" }
  }
}
```

Nothing visual. Just data over the network.

### 2. React State Update

React receives the JSON and updates component state. The virtual DOM is rebuilt internally. Still nothing visual — just JavaScript objects in memory.

### 3. DOM Update

React diffs the virtual DOM against the real DOM and applies changes. For the theme, it sets CSS custom properties:

```js
document.documentElement.style.setProperty('--color-pricify-background', '119 250 250');
document.documentElement.style.setProperty('--color-pricify-brand', '255 0 255');
```

The DOM is a data structure in memory — like a spreadsheet describing what should appear on screen. No pixels yet.

### 4. Style Calculation

The browser's style engine resolves the final CSS for every element:

- `<main>` has class `bg-pricify-background` → resolves to `background-color: rgb(119, 250, 250)`
- Card inner div has `backdrop-filter: brightness(1.5)` → noted for GPU processing later
- Another card might have `background-color: rgb(255, 0, 255)` with `opacity: 0.25`

Output: a styled tree in memory. Still no pixels.

### 5. Layout

The browser calculates positions and sizes of every element:

- `<main>` → 1200×800px at position (0, 0)
- Pricing card → 350×500px at position (200, 300)

Output: a layout tree with exact coordinates. Still no pixels.

### 6. Paint (Draw Instructions)

The browser creates a list of drawing commands — a recipe, not the actual meal:

```
1. Fill rectangle (0, 0, 1200, 800) with rgb(119, 250, 250)
2. Draw text "my heading" at (500, 60) in orange
3. Draw card border at (200, 300, 350, 500)
4. ...
```

Elements with `backdrop-filter` or `opacity` get placed on their own **layers** — separate sheets that the GPU will handle independently.

Output: drawing instructions + layer assignments. Still no pixels.

### 7. GPU Rasterize

The CPU hands the drawing instructions to the GPU. The GPU converts them into actual grids of pixels — this is where pixels are born:

```
Page layer:  every pixel in the background area → (119, 250, 250)
Card layer:  transparent (backdrop-filter card) or solid color (opacity card)
```

### 8. GPU Effects

The GPU applies visual effects like backdrop-filter and opacity. This is where the two examples diverge — see below.

### 9. GPU Composite

The GPU stacks all layers together like transparency sheets — bottom to top — producing one final grid of pixels. This grid is stored in the display's color space (Display P3 on Mac, sRGB on most other monitors).

### 10. Framebuffer → Display

The final pixel grid (framebuffer) is sent from the GPU to the monitor through the display cable. The monitor's controller chip converts each pixel value into a voltage for the red, green, and blue sub-pixel LEDs. Higher voltage → brighter light. Your eye combines the three sub-pixel lights into one perceived color.

The relationship between pixel value and light output is not linear — it follows a curve (the "gamma curve"). Pixel value 128 (50% of 255) produces only about 22% of maximum light, not 50%.

---

## Example 1: Pricing Card with `backdrop-filter: brightness(1.5)`

Page background: `#77FAFA` → RGB(119, 250, 250)

The card has no background color of its own. It applies `backdrop-filter: brightness(1.5)` — meaning "take whatever is behind me and make it 1.5× brighter."

### What happens at the GPU Effects stage

The GPU captures the page background pixels behind the card and multiplies each channel by 1.5:

```
R: 119 × 1.5 = 178.5
G: 250 × 1.5 = 375     ← exceeds 255
B: 250 × 1.5 = 375     ← exceeds 255
```

G and B exceed 255 (the maximum for sRGB). What happens next depends on the browser:

**Firefox (follows the CSS spec):**

Clamps each channel to 255:

```
Result: (179, 255, 255) → #B3FFFF
```

**Chrome / Safari (deviates from spec):**

Does NOT clamp. The GPU works with floating-point numbers internally (values like 1.47 instead of 0–1 range), so 375 is stored as 375/255 = 1.47 — a valid number. Chrome composites in the display's color space (Display P3 on Mac), which can represent values beyond sRGB's range.

The unclamped values pass through to the P3 display, which renders them as "brighter than sRGB max." ColorZilla reads this brighter pixel and converts it to sRGB hex:

```
Result on P3 display: ColorZilla reads #E1FFFF
```

This is why the same page looks different in Chrome vs Firefox on a Mac with a P3 display — confirmed by the Chromium team in [W3C issue #7100](https://github.com/w3c/csswg-drafts/issues/7100):

> "Chrome does compositing in the colorspace of the display." — Christopher Cameron, Chromium

### When does this discrepancy appear?

Only when a channel exceeds 255 after multiplication. If no channel overflows, both browsers produce the same result:

```
(100, 50, 80) × 1.5 = (150, 75, 120)    ← no overflow, all browsers agree
```

### The formula

For predicting the card background color:

```
card_channel = min(bg_channel × 1.5, 255)
```

This matches the CSS spec and Firefox. Chrome on P3 displays will show a slightly different (brighter) result for colors where channels overflow — this is a browser behavior difference, not a formula error.

---

## Example 2: Pricing Card with Hardcoded Color + 25% Opacity

Page background: `#77FAFA` → RGB(119, 250, 250)
Card background: magenta `#FF00FF` → RGB(255, 0, 255) at 25% opacity

### What happens at the GPU Effects stage

The GPU blends the card color over the page background using alpha compositing:

```
result = foreground × alpha + background × (1 - alpha)
```

Channel by channel:

```
R = 255 × 0.25 + 119 × 0.75 = 63.75 + 89.25  = 153
G =   0 × 0.25 + 250 × 0.75 =  0    + 187.5   = 188
B = 255 × 0.25 + 250 × 0.75 = 63.75 + 187.5   = 251
```

Result: **(153, 188, 251)** → `#99BCFB`

### Why this always matches the formula

Alpha compositing is a weighted average of two values. Both inputs are in the 0–255 range. A weighted average of two numbers always falls between them:

- R = 153 is between 119 and 255 ✓
- G = 188 is between 0 and 250 ✓
- B = 251 is between 250 and 255 ✓

**No channel can ever exceed 255.** There is no overflow, so there is nothing for browsers to disagree about. Chrome, Firefox, Safari — all produce the same result. The formula matches perfectly on every browser and every display.

---

## Summary: Why Brightness Is Unpredictable but Opacity Is Not

|                          | Brightness (×1.5)                     | Opacity (alpha blend)                  |
| ------------------------ | ------------------------------------- | -------------------------------------- |
| Operation                | Multiply each channel by 1.5         | Weighted average of two colors         |
| Can exceed 255?          | Yes (e.g., 250 × 1.5 = 375)         | No (average is always between inputs)  |
| Browser disagreement?    | Yes — clamp vs don't clamp           | No — nothing to disagree about         |
| Formula always accurate? | Yes when no overflow, approximate when overflow | Yes, always                  |
| Cross-browser consistent?| No (Chrome ≠ Firefox on P3 displays) | Yes                                    |
