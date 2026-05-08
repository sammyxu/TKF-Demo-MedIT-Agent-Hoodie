---
name: uoft-web-styling
description: Apply University of Toronto brand and web styling guidelines to websites, web components, HTML artifacts, and UI designs for the TFOM Student Ambassador System or any U of T–branded web project. Use this skill whenever the user asks to build, style, or design any web page, component, artifact, or UI that should follow U of T branding — including dashboards, forms, portals, landing pages, or any frontend work associated with the TFOM system. Trigger even if the user just says "make it look good" or "style this" in the context of a U of T project. This skill covers colours, typography, layout, cards, buttons, animations, photography patterns, and the caret design element.
---

# University of Toronto Web Styling Skill

This skill ensures all web output for U of T–branded projects conforms to the official [U of T Brand Web Styling Guidelines](https://brand.utoronto.ca/document/13#/websites/web-styling).

Read this skill before writing any HTML, CSS, or React for a U of T web project. The guidelines below are authoritative — do not substitute personal design preferences.

---

## 1. Colour

### Core Palette

| Name | Hex | Use |
|---|---|---|
| **U of T Blue** (primary) | `#1E3765` | Headers, footers, primary CTAs, dark backgrounds |
| **Teal** (secondary) | `#007894` | Accents, icons, CTAs |
| **Purple** | `#6D247A` | Accent colour |
| **Magenta** | `#AB1368` | Accent colour |
| **Sky Blue** | `#6FC7EA` | Light accent (black text only) |
| **Yellow** | `#F1C500` | Light accent (black text only) |
| **Lime Green** | `#8DBF2E` | Light accent (black text only) |

### Rules
- **Light backgrounds** (recommended): use lighter tints of secondary/accent colours to separate sections.
- **Dark backgrounds**: use U of T Blue (`#1E3765`) sparingly, only to emphasize important areas.
- **White text** is accessible on: `#1E3765`, `#007894`, `#6D247A`, `#AB1368`.
- **Black text** is accessible on: `#6FC7EA`, `#F1C500`, `#8DBF2E`.
- Never use accent colours for large backgrounds, headers, or footers.
- Never let accent colours become the dominant colour on a page.
- Use colour gradients to add depth. U of T Blue → Teal is a reliable on-brand gradient.

### TFOM System Colour Application
- **Student Portal:** Light teal/sky blue tints for section backgrounds; U of T Blue for nav/header.
- **Admin Portal:** Same palette; use amber/yellow (`#F1C500` tint) for warning banners (cap warnings), red-tinted backgrounds for error states.
- **Status badges:** Pending → amber; Approved → lime green tint; Rejected → magenta tint.

---

## 2. Typography

### Fonts
| Role | Font | Fallback |
|---|---|---|
| **Primary** | Trade Gothic Next Heavy / Regular | Arial, sans-serif |
| **Secondary (web only)** | Kepler (Adobe Fonts) | Times New Roman, serif |

### Rules
- Body copy: minimum **16px**, Regular weight (400).
- Headings: Bold (600) or Heavy (800).
- Use Trade Gothic Next for all UI text; Kepler only as a decorative secondary.
- Max two to three fonts per page.
- Write short, scannable copy with headers between sections.

### CSS Font Stack (use in all stylesheets)
```css
body {
  font-family: 'Trade Gothic Next', Arial, sans-serif;
  font-size: 16px;
  font-weight: 400;
}
h1, h2, h3 {
  font-family: 'Trade Gothic Next', Arial, sans-serif;
  font-weight: 800;
}
```

> **Note:** Trade Gothic Next is a licensed font. For PoC/prototype work, **Arial Bold** is the approved fallback and is acceptable for development artifacts.

---

## 3. Buttons & CTAs

| Type | Style |
|---|---|
| **Primary CTA** | White text on dark colour (U of T Blue `#1E3765` or Teal `#007894`) |
| **Secondary CTA** | Dark text on light background colour |
| **Destructive / Reject** | White text on Magenta `#AB1368` |

### Rules
- Round corners: **minimum 10px** border-radius on all buttons and cards.
- Consistent shape, type treatment, and spacing across all pages.
- Add subtle box-shadow for depth (not large or overly dark).

### Example CSS
```css
.btn-primary {
  background: #1E3765;
  color: #ffffff;
  border-radius: 10px;
  padding: 12px 24px;
  font-family: 'Trade Gothic Next', Arial, sans-serif;
  font-weight: 700;
  box-shadow: 0 2px 8px rgba(30, 55, 101, 0.18);
}
.btn-secondary {
  background: #E8F4F8;
  color: #1E3765;
  border-radius: 10px;
  padding: 12px 24px;
  font-weight: 600;
}
```

---

## 4. Cards & Layout

- Cards: **white background**, rounded corners (≥ 10px), subtle shadow.
- The section containing cards may have a tinted (not white) background.
- Don't overuse cards — only when there's enough content to warrant a grid.
- Max-width for content areas: **~640px** centered for forms; wider for dashboards.
- Use **shadows** and **overlapping elements** to create depth.

```css
.card {
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 2px 12px rgba(30, 55, 101, 0.10);
  padding: 24px;
}
```

---

## 5. Mobile / Responsive

- Single-column stacked layout on screens < 768px.
- Minimum touch target: **44×44px** for all buttons and interactive elements.
- Textareas and inputs should resize to full-width on mobile.
- Date pickers: use native mobile input with custom styled fallback.
- Always test card collages and image arrangements on mobile before finalising.

---

## 6. Animation & Effects

- Use **subtle, quick** fade-in or scroll-triggered animations.
- Evoke: rising, expansion, floating — never aggressive "fly-in" or "whooshing".
- Typical patterns: elements fade/translate up on load; banner images scale gently.
- Never use animation that interferes with accessibility or causes disorientation.
- Prefer `transition: opacity 0.2s ease, transform 0.25s ease` for most micro-interactions.

---

## 7. Photography & Images

- **Headboard images:** appear "behind" content, partially covered — abstract/textural preferred.
- **Gradients on images:** subtle overlay to improve contrast and legibility.
- Colour tinting repeat elements (banners, headboards) makes the site feel cohesive.
- Ensure photos have **square corners** (no border-radius on photos).
- Don't use heavy overlays that obscure image content.

---

## 8. The Caret (⌃)

Only use the caret if the **Defy Gravity logo** is present on the page.

- Use sparingly — max **two instances per page**.
- Use as a large background decorative element only; allow it to be cropped.
- Light, subtle colours — never overpower other elements.
- **Never** use as: an arrow, navigational element, inline bullet, clickable object, or pattern.

---

## 9. Accessibility Checklist

Before delivering any styled component:
- [ ] All text/background combinations pass WCAG AA contrast (use a contrast checker).
- [ ] Color is never the **sole** indicator of state (always pair with icon + text).
- [ ] All form fields have explicit `<label>` elements (not placeholder-only).
- [ ] Error states use `aria-invalid="true"` and `aria-describedby`.
- [ ] Error banners use `role="alert"`.
- [ ] Page is keyboard-navigable in logical tab order.
- [ ] Touch targets ≥ 44×44px on mobile.
- [ ] WCAG 2.1 AA compliance target throughout.

---

## 10. Quick Reference: TFOM Component Palette

| Component | Background | Text | Border/Accent |
|---|---|---|---|
| Page header / nav | `#1E3765` | `#ffffff` | — |
| Section (light) | `#EAF4F8` (sky blue tint) | `#1E3765` | — |
| Card | `#ffffff` | `#1E3765` | shadow |
| Primary button | `#1E3765` | `#ffffff` | — |
| Secondary button | `#EAF4F8` | `#1E3765` | — |
| Warning banner (cap) | `#FFF8DC` (yellow tint) | `#7A6000` | `#F1C500` left border |
| Error banner | `#FDE8F0` (magenta tint) | `#6B0032` | `#AB1368` left border |
| Success / approved | `#EEF7E0` (lime tint) | `#3A5200` | `#8DBF2E` left border |
| Pending badge | `#FFF3CD` | `#856404` | — |

---

## 11. Logo & Brand Assets

**Primary logo (hot link):**
`https://medit.med.utoronto.ca/themes/webpac/images/icons/temerty-medicine-wordmark-coloured.svg`

This is the authoritative Temerty Faculty of Medicine wordmark. Hot-link directly from the MedIT origin in page headers — do not bundle a local copy.

```html
<img
  src="https://medit.med.utoronto.ca/themes/webpac/images/icons/temerty-medicine-wordmark-coloured.svg"
  alt="Temerty Faculty of Medicine"
  height="36"
/>
```

Other brand assets (use when the Temerty wordmark is not appropriate):

- `logo-primary.svg` — Primary U of T logo
- `uoft-logo.svg` — U of T logo
- `defy-gravity-logo-white.webp` — Defy Gravity logo (white; required for caret usage)

Always include the appropriate logo in the page header. Prefer SVG formats for crispness at all sizes.

---

## Full Guidelines Reference

For the complete source document, read: `references/full-guidelines.md`

Load it when you need detail on photography patterns, animation examples, or edge cases not covered above.
