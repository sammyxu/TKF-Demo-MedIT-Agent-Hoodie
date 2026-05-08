# UI/UX Design Guide — U of T Brand System

## Design philosophy
Institutional but not bureaucratic; energetic but not playful. Carries U of T trust and gravitas while staying frictionless on a phone.

### Pillars
- **Trustworthy** — U of T Blue dominates
- **Transparent** — points/status/history always visible
- **Frictionless** — short forms, linear flows, specific errors
- **Accessible** — WCAG 2.1 AA; colour never the sole indicator
- **Mobile-first** — designed for thumbs first

## Colour system (U of T palette — no substitutions)
| Token | Hex | Usage |
|---|---|---|
| Primary (U of T Blue) | `#1E3765` | Nav, headers, primary buttons |
| Secondary (Teal) | `#007894` | Secondary CTAs, links, points-fill bar |
| Magenta | `#AB1368` | Destructive (Reject), error states |
| Yellow | `#F1C500` | Warnings, cap alerts (black text only) |
| Lime | `#8DBF2E` | Success, approved badges (black text only) |
| Sky Blue | `#6FC7EA` | Light section backgrounds |
| Purple | `#6D247A` | Decorative accent only |
| Gray-50 / 600 / 900 | `#F8FAFC / #4B5563 / #111827` | Canvas / body / dark text |

### Status colour mapping
- **Pending** — amber (`#FFF3CD` bg, `#856404` text)
- **Approved** — lime tint
- **Rejected** — magenta tint

## Typography
Open, energetic, upward-moving. Hierarchy via weight and scale, not colour alone.

## Component library highlights
Buttons · Cards · Forms · Status badges · **Points progress widget** (teal fill, yellow cap zone) · Cap warning banner · Dashboard empty state · Submission success banner · Tables · Modals · Navigation.

## Page-by-page specs covered
Login · Student dashboard · Activity submission · Reward catalog · Admin queue · Admin catalog management.

## Responsive strategy
Mobile-first breakpoints. Forms collapse to single column; tables become card lists on narrow viewports.

## Accessibility
- WCAG 2.1 AA throughout
- Keyboard navigation for all interactive elements
- Screen reader labels on icon-only controls
- Colour contrast verified for every token pairing
- State changes announced via ARIA live regions where relevant

## Tailwind implementation
Design tokens map 1:1 to Tailwind theme extensions; HTML mockups in [/html](../html/) are the visual source of truth — preserve their `<style>` blocks verbatim during conversion to React.

## Logo / brand asset rules
Use official U of T marks only; respect clear-space requirements; never recolour the logo outside the approved palette.
