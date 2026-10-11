# Lukuman Learning Design System: Light & Dark Usage Rules (v1.1)

**Goal:** predominantly **light, clean and educational**, with carefully chosen dark sections for hierarchy.

| Colour | Role | Feeling |
|---|---|---|
| Light (`#F8FAFC` page, `#FFFFFF` cards) | default for most of the app | Clean |
| Dark Blue `#1E3A8A` | hierarchy and navigation | Trust & authority |
| Primary Blue `#2563EB` | actions | Action |
| Emerald `#10B981` | progress and success | Progress & success |
| Amber `#F59E0B` | premium and attention | Premium & attention |

All values are CSS variables in `public/design-system.css`. Every page links that one file; nothing else should contain a colour.

## Light usage (default)
- Page background `--ll-page` (#F8FAFC); cards and containers `--ll-surface` (#FFFFFF), separated by a subtle `--ll-border` and `--ll-shadow`.
- Always light: homepage sections, course listings, lesson content, quiz questions, ebook listings, student dashboard, profile, subscription comparison, forms, admin tables.
- Class: `.ll-card`, `.ll-table`, `.ll-input`, `.ll-pre`.

## Dark usage (hierarchy only)
- **Dark Blue** `--ll-navy`: hero (when needed), selected navigation, footer, premium promo sections, important banners, and important headings.
- Classes: `.ll-header`, `.ll-section-dark`, `.ll-footer`, `.ll-banner`, `.ll-nav-item[aria-current="page"]`.
- Inside dark sections: white headings (`--ll-on-dark`), light-grey secondary text (`--ll-on-dark-2`), white/light buttons (`.ll-btn--on-dark`), white links. **Never** blue text, dark text, or primary-blue buttons on Dark Blue.
- **Never make a whole page dark.** Use one or two dark blocks per page at most.
- **Main Text** `--ll-text` (#0F172A): headings, labels, lesson and course titles. **Secondary Text** `--ll-text-2` (#64748B): descriptions, metadata, helper text.

## Colour roles
- **Primary Blue** (`.ll-btn`, links, active states, progress bars, interactive elements). Never the background of a section.
- **Emerald**: completed lessons, successful quiz results, progress completion, success messages. Use as a *fill* (`.ll-progress--complete`). For green *text* use `--ll-success-text` on `--ll-success-bg` (`.ll-badge--success`, `.ll-alert--success`).
- **Amber**: Premium badges, "Best Value", premium highlights, small promo accents (`.ll-badge--premium`, `.ll-badge--best-value`, the left edge of `.ll-lock`). Sparingly. Never a page or section background. For amber *text* use `--ll-premium-text` on `--ll-premium-bg`.

## Contrast (WCAG AA, 4.5:1 minimum for text)
| Pairing | Ratio | OK |
|---|---|---|
| Main text on page / card | 17.1 / 17.9 | ✅ |
| Secondary text on card / page | 4.76 / 4.55 | ✅ (do not go lighter) |
| Primary Blue on card; white on Primary Blue | 5.17 | ✅ |
| White on Dark Blue; light grey on Dark Blue | 10.4; 7.0 | ✅ |
| Success text on success bg; Premium text on premium bg | 4.84; 6.37 | ✅ |
| Dark text on Amber ("Best Value") | 8.31 | ✅ |
| **Primary Blue on Dark Blue** | 2.0 | ❌ never |
| **Emerald as text on white** | 2.54 | ❌ use `--ll-success-text` |
| **Amber/yellow as text on white** | 2.15 | ❌ use `--ll-premium-text` |
| **White text on Amber or Emerald** | 2.15 / 2.54 | ❌ use dark text |
| **Dark text on Dark Blue** | <3 | ❌ never |

## Mobile
Readability first. The header is a slim bar; dark sections are compact (small padding); cards, lessons and forms stay light; buttons are full-width tap targets. Avoid large dark blocks.

## Building new pages
1. `<link rel="stylesheet" href="/design-system.css">`: the page is light automatically.
2. Wrap content in `.ll-card`; use `.ll-btn` for the main action, `.ll-btn--secondary` for others.
3. Wrap page content in `<main class="ll-container">` (dark header/footer bands sit outside it and span the full width). Need hierarchy? Add **one** `.ll-section-dark` (hero/promo) or `.ll-footer`.
4. Use `.ll-badge--success|premium|neutral|best-value`, `.ll-progress`, `.ll-alert--*`. Never write a hex colour in a page.

## Automatic enforcement
`test/design.test.js` fails the build if: the palette changes; the body is not light; amber or Primary Blue is used as a section background; a dark-blue block lacks white text; a component or page uses a raw hex colour or inline colour; or any approved text/background pair drops below 4.5:1. New pages that follow step 1-4 pass without extra work.
