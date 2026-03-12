# iOS Safari `backdrop-filter` Grey Flash Bug

## Problem Summary

On some iPhones, when loading the pricing page, **all white colors across the entire screen appear greyish for approximately 5 seconds**, then correct themselves to pure white.

### Key Observations

1. **Scope**: The grey tint affects the **entire display**, not just the iframe containing the pricing page. Parent page elements (navbar, footer, etc.) are also affected.

2. **Duration**: Lasts approximately 5 seconds after page load.

3. **Device-specific**: Only affects certain iPhone models/iOS versions.

4. **Screenshots fail to capture it**: Taking a screenshot shows correct white colors, suggesting this is a GPU compositor/display pipeline issue, not a final rendered output issue.

### Affected Theme Configuration

```json
{
  "theme": {
    "brand": { "color": "#ff8a00", "opacity": 1 },
    "background": { "color": "#ffffff", "opacity": 1 },
    "accent": { "color": "#ff8a00", "opacity": 1 },
    "textContrast": { "color": "#FFFFFF", "opacity": 1 },
    "textBase": { "color": "#262121", "opacity": 1 }
  }
}
```

---

## Root Cause Identification

### The Culprit: `backdrop-brightness-150`

The pricing cards use `backdrop-filter: brightness(1.5)` (via Tailwind's `backdrop-brightness-150` class):

```tsx
// src/client/components/pricing-page/PricingCard.tsx (line 267)
<div className={clsxm(
  'flex h-full w-full flex-col backdrop-brightness-150',
  // ...
)}>
```

This class is also used in:
- `PricingTable.tsx` (line 382) - "No Plans Available" container
- `PlanCustomisationSection.tsx` (line 170)
- `OrderSummary.tsx` (line 203)

### Why This Causes the Issue

#### 1. CSS Filter Color Space Specification

According to [W3C CSS Working Group Issue #7100](https://github.com/w3c/csswg-drafts/issues/7100):

> **CSS filter specification requires that "Functions must operate in the sRGB color space"**

This means:
- When `backdrop-filter: brightness(1.5)` is applied
- iOS Safari converts content from Display P3 (iPhone's native color space) to sRGB
- This conversion affects the **entire compositor output**, not just the filtered element

#### 2. Display P3 to sRGB Conversion

iPhones use **Display P3** color space natively, which has a wider gamut than sRGB:

- Display P3 white ≠ sRGB white (in terms of raw values)
- During the P3 → sRGB conversion, white (`#ffffff`) can appear slightly grey
- This affects **all** white elements on screen, including content outside the iframe

#### 3. GPU Compositor Behavior

- `backdrop-filter` forces creation of separate compositor layers
- The filter triggers color space conversion at the GPU level
- During compositor stabilization (~5 seconds on some devices), the conversion artifacts are visible
- After stabilization, proper layer isolation kicks in

### Why Screenshots Don't Capture It

Screenshots on iOS are captured from the **final composited Display P3 output**, not the intermediate framebuffer. The grey appearance exists only in the **intermediate rendering pipeline** during GPU composition, which is why it's visible on screen but not in screenshots.

### Why Only Some iPhones

- Different iPhone models have different GPU capabilities
- Older/lower-end devices may take longer to properly isolate compositor layers
- iOS version differences affect WebKit's compositor implementation
- The timing of layer stabilization varies by device

---

## Technical Evidence

### WebKit Bug Reports and PRs

1. **[WebKit PR #26211](https://github.com/WebKit/WebKit/pull/26211)** - "Backdrop-filter forces compositing on the root element and uses extra memory"
   - Confirms backdrop-filter creates separate compositing layers
   - Root element becomes a "backdrop root," separate from base background color
   - Merged March 2024

2. **[WebKit Bug #275919](https://bugs.webkit.org/show_bug.cgi?id=275919)** - "Backdrop-filter: blur with a transparent web view renders incorrectly"
   - Documents rendering issues with backdrop-filter on iOS

3. **[W3C CSSWG Issue #7100](https://github.com/w3c/csswg-drafts/issues/7100)** - "Should no-op filters produce different output from no filter?"
   - Confirms filters operate in sRGB color space
   - Documents that Display-P3 colors get converted/clamped when filters are applied
   - Notes that Chrome handles this differently than Safari

### StackOverflow Reports

- [Brightness filter update issues in Safari](https://stackoverflow.com/questions/74967586/why-updating-brightness-filter-has-not-effect-in-safari)
- [backdrop-filter rendering delays on iOS](https://stackoverflow.com/questions/65450735/backdrop-filter-doesnt-work-on-safari-most-of-the-times)

---

## The Irony

On a **white background** (`#ffffff`), `backdrop-brightness-150` has **no visible effect**:

```
white (255, 255, 255) × 1.5 = (382.5, 382.5, 382.5) → clamped to (255, 255, 255) = white
```

So the filter:
- Provides **zero visual benefit** on white backgrounds
- But causes **significant rendering issues** on iOS
- Triggers GPU compositor complexity for no reason

The filter is only visually useful on **non-white/colored backgrounds** where brightening creates a visible lightening effect on the cards.

---

## Recommended Solution

### Option 1: Conditional Backdrop Filter (Recommended)

Skip `backdrop-brightness-150` when the background is white or near-white:

```tsx
// Helper function
const isWhiteOrNearWhite = (hexColor?: string): boolean => {
  if (!hexColor) return true; // Default CSS is near-white (249, 250, 251)
  const hex = hexColor.replace('#', '');
  const r = parseInt(hex.substring(0, 2), 16);
  const g = parseInt(hex.substring(2, 4), 16);
  const b = parseInt(hex.substring(4, 6), 16);
  return r >= 250 && g >= 250 && b >= 250;
};

// In PricingCard.tsx
const bgColor = pricingPage.theme?.background?.color;
const useBackdropFilter = !isWhiteOrNearWhite(bgColor);

// In className
className={clsxm(
  'flex h-full w-full flex-col',
  useBackdropFilter && 'backdrop-brightness-150',
  // ...
)}
```

**Pros:**
- Zero visual change for white backgrounds
- Fixes iOS issue completely for white backgrounds
- Preserves functionality for colored backgrounds

**Cons:**
- Requires code changes in multiple files

### Option 2: Replace with Solid Background

Instead of using `backdrop-filter`, calculate a lighter version of the background color and apply it as a solid `background-color`:

```tsx
// Calculate 50% lighter version of background
const lighterBg = `color-mix(in srgb, ${bgColor}, white 50%)`;
```

**Pros:**
- Eliminates all backdrop-filter GPU issues
- Works consistently across all devices

**Cons:**
- Requires modern CSS support (color-mix)
- Changes visual appearance slightly

### Option 3: CSS Custom Property Approach

Define the backdrop filter value via CSS custom property, set based on background:

```css
:root {
  --card-backdrop-filter: brightness(1.5);
}

/* Override for white backgrounds */
:root[data-bg-white="true"] {
  --card-backdrop-filter: none;
}
```

---

## Files Requiring Changes

If implementing Option 1:

1. `src/client/components/pricing-page/PricingCard.tsx` (line 267)
2. `src/client/components/pricing-page/PricingTable.tsx` (line 382)
3. `src/client/components/customize-plans/PlanCustomisationSection.tsx` (line 170)
4. `src/client/components/customize-plans/OrderSummary.tsx` (line 203)

Consider creating a shared utility or hook for the color check logic.

---

## Testing Checklist

After implementing the fix:

- [ ] Test on iPhone with white background theme - grey flash should be gone
- [ ] Test on iPhone with colored background theme - backdrop brightness effect should still work
- [ ] Test on Android/Chrome - no visual changes expected
- [ ] Test on Desktop Safari - verify no regressions
- [ ] Verify parent page (navbar/footer) no longer affected

---

## References

- [W3C CSS Filter Effects Specification](https://drafts.fxtf.org/filter-effects/)
- [WebKit Backdrop Filter Documentation](https://webkit.org/blog/3632/introducing-backdrop-filters/)
- [Display P3 Color Space in WebKit](https://webkit.org/blog/10042/wide-gamut-color-in-css-with-display-p3/)
- [WebKit PR #26211 - Backdrop-filter compositing](https://github.com/WebKit/WebKit/pull/26211)
- [W3C CSSWG Issue #7100 - Filter color space behavior](https://github.com/w3c/csswg-drafts/issues/7100)
