# Design Color Tokens (v1)

This document captures the cleaned palette, semantic tokens, and accessibility guidance based on the provided raw color list.

## 1. Raw Input
```
#b48f8f #737373 #e5e7eb#ffffff #b38d8d #e6e6e6 #adb2c6 #d6c5bf #c3a5a5 #ffffff#c3a5a5 #737373
```

## 2. Normalized Unique Palette
| Token ID | Hex | Notes |
|----------|-----|-------|
| white | #ffffff | Base background |
| gray-cool-200 | #e5e7eb | Cool light neutral (choose over #e6e6e6) |
| gray-warm-200 (alt) | #e6e6e6 | Nearly duplicate of #e5e7eb (optional) |
| warm-taupe-200 | #d6c5bf | Warm accent surface |
| rose-300 | #c3a5a5 | Brand tint / subtle surface |
| rose-390 | #b38d8d | Near-duplicate of primary (optional hover) |
| rose-400 | #b48f8f | Primary brand base |
| blue-gray-300 | #adb2c6 | Cool accent / info |
| gray-600 | #737373 | Secondary text / icons |

## 3. Recommended Consolidated Set
Keep: #b48f8f, #c3a5a5, #d6c5bf, #adb2c6, #e5e7eb, #ffffff, #737373
Drop / Archive: #b38d8d (too close to #b48f8f), #e6e6e6 (duplicate light neutral)
Add for contrast (not in original): #2f2f2f (primary text), #8c5f5f (active), #a07272 (hover), #c25757 (error), #5d865d (success)

## 4. Semantic Tokens
| Semantic Name | Hex | Purpose |
|---------------|-----|---------|
| background.base | #ffffff | Main app background |
| background.subtle | #f5f5f5 | Section background (generated) |
| surface.base | #ffffff | Panels/cards |
| surface.alt | #e5e7eb | Secondary panels / tab background |
| border.subtle | #e5e7eb | Light separators |
| border.accent | #d6c5bf | Highlight borders / focus ring inner |
| brand.primary | #b48f8f | Primary actions, highlights |
| brand.primary-hover | #a07272 | Hover state |
| brand.primary-active | #8c5f5f | Active/pressed |
| brand.primary-subtle | #c3a5a5 | Subtle backgrounds (badges) |
| accent.cool | #adb2c6 | Info / secondary CTA |
| text.primary | #2f2f2f | Body text (AA) |
| text.secondary | #737373 | Labels, helper text |
| text.inverse | #ffffff | On dark/brand surfaces |
| status.error | #c25757 | Errors/destructive |
| status.success | #5d865d | Success confirmations |
| status.warning | #d6c5bf | Mild warning / caution (reusing warm accent) |
| focus.ring | #adb2c6 | Outer focus outline |

## 5. Accessibility Notes
- #737373 on #ffffff is borderline for small text; use #2f2f2f for body copy.
- Ensure button text contrast: if using brand.primary (#b48f8f) background, prefer dark text (#2f2f2f) OR darken background to #8c5f5f for white text.
- Maintain consistent state deltas: hover darken ~12%, active ~20%.

## 6. Tailwind Extension Example
```js
// tailwind.config.js snippet
extend: {
  colors: {
    brand: {
      300: '#c3a5a5',
      400: '#b48f8f',
      500: '#a07272',
      600: '#8c5f5f',
      700: '#724646'
    },
    accent: { cool: '#adb2c6' },
    warm: { 200: '#d6c5bf' },
    gray: {
      50: '#ffffff',
      100: '#f5f5f5',
      200: '#e5e7eb',
      600: '#737373',
      800: '#2f2f2f'
    }
  }
}
```

## 7. CSS Variables
```css
:root {
  --color-bg: #ffffff;
  --color-bg-subtle: #f5f5f5;
  --color-surface: #ffffff;
  --color-surface-alt: #e5e7eb;
  --color-border-subtle: #e5e7eb;
  --color-border-accent: #d6c5bf;
  --color-brand: #b48f8f;
  --color-brand-hover: #a07272;
  --color-brand-active: #8c5f5f;
  --color-brand-subtle: #c3a5a5;
  --color-accent-cool: #adb2c6;
  --color-text-primary: #2f2f2f;
  --color-text-secondary: #737373;
  --color-text-inverse: #ffffff;
  --color-status-error: #c25757;
  --color-status-success: #5d865d;
  --color-status-warning: #d6c5bf;
  --focus-ring: 0 0 0 2px #ffffff, 0 0 0 4px #adb2c6;
}
```

## 8. JSON Tokens
See tokens/design-tokens.json.

## 9. Next Steps
- Decide button color strategy (light vs dark brand) and finalize contrast tests.
- Generate dark mode palette (optional).
- Integrate tokens.css in global stylesheet and Tailwind config.

---
End v1.