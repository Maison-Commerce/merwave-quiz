# Merwave Hair Analysis — Quiz Styleguide

This styleguide documents the design system used in `/quiz/index.html` — the 17-screen hair-type diagnostic funnel for the **Merwave Wavy Hair Starter Kit**.

**Avatar:** the "Curly Girl Refugee" — women who believe their hair is curly, have used curly products for years, and watch their "curls" fall flat, greasy and lifeless.
**The reframe the funnel runs on:** she doesn't have difficult curly hair — she has fine **wavy (Type 2)** hair crushed by curly products built for thick, coily textures. The quiz manufactures that realisation one click at a time, then hands her the one kit built for the hair type she actually has.

The visual system is lifted from the **Merwave UI Kit** (Figma) — a warm, soft, editorial wellness aesthetic. It is the Merwave counterpart to the indigo Restfulness build that previously lived here.

---

## 1. Brand Identity / Overall Feel

A **warm, soft, premium hair-care diagnostic**. Cream surfaces (`#FDFDFC`, `#F3F3ED`), a dusty-blue primary accent (`#86A6C9`), and a soft-yellow secondary (`#F9F2B3`) used sparingly. Typography pairs an editorial serif (**Quincy CF**, Bold 700) for headings with a humanist sans (**Proxima Nova**) for body. Unlike the borders-only Restfulness build, **Merwave uses soft drop shadows** (`--shadow-lg`, `--shadow-xl`) on cards and selected options. Single-column, 640px shell. No countdown timers, no aggressive urgency.

Vs the landings, the quiz adds: sticky **topbar** (logo + progress + counter 1/17…17/17), sticky bottom **step-nav**, **option buttons**, two **loaders**, a **diagnosis card + wave-meter**, a **projection chart**, and a full **sales page** on screen 17.

---

## 2. Color Palette (CSS custom properties in `:root`)

### Accent — dusty blue (Brand 1)
| Token | Value | Use |
|---|---|---|
| `--color-brand1-1800` | `#86A6C9` | primary accent: eyebrow, why-box icon, progress/selected gradient start |
| `--color-brand1-1200` | `#9FB8D4` | gradient end, option hover border |
| `--color-brand1-400`  | `#DFE7F1` | soft accent surface (feature icon bg, sale eyebrow) |
| `--color-primary-gradient` | `linear-gradient(90deg,#86A6C9,#9FB8D4)` | progress fill, selected-option border, loader bar, chart line start |

### Surfaces — warm cream (Brand 2)
| Token | Value | Use |
|---|---|---|
| `--color-brand2-200`  | `#FDFDFC` | page background |
| `--color-brand2-800`  | `#F3F3ED` | soft surface — option icon, why-box, proof-quote, card headers, media wells |
| `--color-brand2-1800` | `#EAE9DF` | stronger surface — progress track, checklist icon bg, skeleton base |
| `--color-brand2-2400` | `#BAB9B1` | disabled CTA bg |
| `--color-brand2-2800` | `#83827C` | muted text (tertiary) |

### Secondary — soft yellow / olive (Brand 3)
| Token | Value | Use |
|---|---|---|
| `--color-brand3-1800` | `#F9F2B3` | free-gift block bg (screen 17) |
| `--color-brand3-3200` | `#434130` | text/icon on the yellow block |

### Ink / Text
| Token | Value | Use |
|---|---|---|
| `--color-ink` | `#3D3D3D` | **CTA background**, primary text |
| `--color-text-primary`   | `#3D3D3D` | headings, primary body |
| `--color-text-secondary` | `#5C5358` | subtitles, option labels, copy paragraphs |
| `--color-text-tertiary`  | `#83827C` | helpers, counters, axis labels, back button |

### Status / Semantic
| Token | Value | Use |
|---|---|---|
| `--color-accent` | `#12B76A` | success green — ticks, "Wash 1" payoff, guarantee, qualify checks |
| `--color-accent-dark` | `#027A48` | strong success text (diagnosis "HIGH") |
| `--color-star` | `#FEC84B` | gold stars |
| `--color-warn` | `#F79009` | wave-meter "suppressed/low" label |
| `--color-bad`  | `#F04438` | meter danger zone |
| `--color-border-subtle` | `#E6E5DD` | warm hairline — option/card borders, dashed dividers |

---

## 3. Typography

```css
--font-heading: "Quincy CF", "Fraunces", "Playfair Display", Georgia, serif;  /* Bold 700 */
--font-primary: "Proxima Nova", -apple-system, "Segoe UI", Helvetica, Arial, sans-serif;
--font-accent:  "Quincy CF", "Fraunces", Georgia, serif;  /* italic 700 for <em> */
```

**Fonts are wired and self-hosted** from `quiz/fonts/quincy-cf/` (Quincy CF — **licensed** webfonts: Regular 400, Medium 500, Bold 700 + Bold Italic, woff2+woff) and `quiz/fonts/proxima-nova/` (Regular 400 + Bold 700, woff/ttf). One caveat:
- Proxima Nova **Medium (500)** and **Semibold (600)** were not supplied, so the CSS cascade maps 500→Regular and 600→Bold. Add those two faces if the heavier-than-regular option labels need to be exact.

### Type scale (Desktop → Mobile @ ≤640px)
| Role | Class | Desktop | Mobile | Weight | Font |
|---|---|---|---|---|---|
| Screen H1 | `.step__title` | 32px | 26px | 700 | heading |
| Screen question | `.step__question` | 24px | 20px | 700 | heading |
| Loader title | `.loader__title` | 28px | 28px | 700 | heading |
| Sales headline | `.sale-headline` | 34px | 28px | 700 | heading |
| Subtitle | `.step__subtitle` | 16px | 15px | 400 | body |
| Eyebrow | `.step__eyebrow` | 11px | 11px | 700 | body; uppercase, ls 1px, color brand1-1800 |
| Option label | `.option__label` | 15px | 15px | 600 | body |
| CTA | `.btn-cta` | 16px | 16px | 700 | body |

### Italic / em rule
Wrap an emphasised phrase in `<em>` inside `.step__title`, `.sale-headline` or `.diagnosis-card__diagnosis`. The `<em>` swaps to the serif italic **in `--color-brand1-1800` (dusty blue)** — the one branded color accent on headings. The hook phrase *"You're not curly. **You're wavy.**"* uses this on screen 14.

---

## 4. Spacing, Radii, Shadows, Containers

```css
--space scale: 4 8 12 16 20 24 32 40 48 64 80
--radius-sm: 8px;  --radius-md: 12px;  --radius-lg: 16px;  --radius-full: 9999px;
--shadow-lg: 0 4px 6px -2px rgba(16,24,40,.03), 0 12px 16px -4px rgba(16,24,40,.08);
--shadow-xl: 0 8px 8px -4px rgba(16,24,40,.03), 0 20px 24px -4px rgba(16,24,40,.08);
.quiz-shell { max-width: 640px; padding: 48px 24px 140px; }   /* 140px = fixed step-nav clearance */
.sale-shell { max-width: 680px; padding: 32px 24px 120px; }
```

CTA is a **999px pill** in `--color-ink` (charcoal `#3D3D3D`), white label. Cards/selected options carry `--shadow-lg`; the sales CTA card carries `--shadow-xl`. **Shadows are part of the Merwave language** — do not strip them to "flat" the way the Restfulness build required.

---

## 5. Step Conventions

17 numbered screens, all **active** (note: unlike the old Restfulness build, screen 17 is a real, shown sales page, not a hidden deprecated block). Engine lives in the Alpine `quiz()` factory.

| # | Screen | Type | Answer key / notes |
|---|---|---|---|
| 1 | Hair Type Self-ID | single + proof-quote | `hairType`; no Back |
| 2 | Primary Concern | single, 5, emoji icons | `concern` |
| 3 | Symptom Recognition | single, 3 | `symptom` |
| 4 | Severity | single, 4, **wave-flattening visual scale** | `severity` (SVG amplitude per option) |
| 5 | Duration | single, 4 | `duration` |
| 6 | Desired Outcome | **multi**, 5 | `desired[]`; Continue required |
| 7 | Validation Interstitial | static + before/after strip | Continue; reframe seeded ("you're not bad at curly hair") |
| 8 | Strand Type | single, 4 | `strandType`; fineness = crux of diagnosis |
| 9 | Product Load | single, 4 | `productLoad` |
| 10 | Loader #1 | timed bar (~4s) | auto-advance, no nav |
| 11 | What She's Tried | **multi** + why-box (villain named) | `tried[]`; Continue required |
| 12 | Wet vs Dry | single, 3 + why-box (mechanism) | `wetDry`; Continue required (no auto-advance — she reads the why-box) |
| 13 | Loader #2 | animated checklist (~7s) | auto-advance, no nav |
| 14 | Diagnosis (the aha) | wave-meter + diagnosis card + copy | Continue = "Show me what's possible" |
| 15 | Projection | growth chart + date + copy | Continue |
| 16 | Qualification Pre-Landing | product + 4 benefit circles + guarantee | Continue = "See my kit" |
| 17 | Sales Page (the close) | full offer → Shopify checkout | own in-page CTA "Bring my waves back" |

### Engine specifics
- `progress()` — front-loaded curve `1-(1-t)^1.6` over 16 (so screen ~8 reads ~70%+).
- `showBack()` — hidden on 1, 10, 13. `showContinue()` — `[6,7,11,12,14,15,16]`.
- `canContinue()` — screen 6 needs `desired`, 11 needs `tried`, 12 needs `wetDry`.
- Loaders respect `prefers-reduced-motion` (jump to final in ~500ms).
- `projectedDate` — computed `today + 56 days` (≈8 weeks, the PDF placeholder). Swap to a fixed `[confirm]` date if the client prefers.
- Step nav is hidden entirely on screen 17 (the sales page scrolls with its own CTA card).

---

## 6. Key Components

- **Option button** `.option` — white, 2px border, 12px radius; selected = transparent border with `--color-primary-gradient` border-box trick + `--shadow-lg`; right `.option__check` fills with the gradient.
- **Why-box** `.why-box` — cream box with a serif-italic `i` in a dusty-blue circle; used on screens 11 (villain) and 12 (mechanism).
- **Before/after strip** `.beforeafter` — 2-col grid, 3:4 cells, "Before"/"After" pills (charcoal / green). Currently empty wells flagged `[add image]` on screen 7.
- **Wave-meter** (screen 14) — semicircular gauge, red→green gradient arc, needle resting **low** ("suppressed"), label `Low — Suppressed` in `--color-warn`. Communicates "high potential, currently suppressed."
- **Diagnosis card** — header (wavy-line avatar icon) + dashed rows; "Wave potential: HIGH" and "Room for improvement: HIGH" use `.diagnosis-card__row-val--good` (green).
- **Projection chart** — SVG line dusty-blue→green, "YOU ARE HERE" (charcoal) → "WASH 1 — WAVES RETURN" (blue, early) → "WAVES THAT LAST" (green, top-right).
- **Sales page (17)** — eyebrow pill, serif headline w/ `<em>`, price block (£65 / ~~£85~~ / Save £20), "why this kit" bullets, 5 numbered steps, free-gift block on the yellow Brand-3 surface, CTA card (`--shadow-xl`), guarantee, endorsement names, payment row.

---

## 7. Outstanding / `[confirm]` items

Real content now wired (from client-supplied assets):
- **Screen 1 social proof** — real verified review (Dawn H.), initial avatar. ✓
- **Screen 17 "Stylist's approval"** — real quotes + handles for Emma Kenyon, Richard James, Naomi Jones, Rein & Co. ✓
- **Screen 17 "What customers say"** — 3 real verified reviews (Sharron G., Lois King, Paula Mooney). ✓

Still `[confirm]` (hardcoded from PDF, flagged inline):
- `47%` stat (screens 7, 16) · `[5,376]` review count + `★4.8` (screens 7, 17) — substantiated by real reviews but the exact count/rating still needs the client's figure
- Projection date (screen 15)
- `90-day` guarantee terms (screens 16, 17)
- Scarcity mechanism (screen 17 — intentionally **no fake countdown**)

Pending assets / config:
- **Fonts:** wired ✓ — licensed Quincy CF webfonts (woff2). Optionally add Proxima Nova Medium/Semibold.
- **Logo:** wired ✓ — `assets/logo.png` (Merwave slate wordmark) in the topbar.
- **Product image:** injected at runtime from the Shopify product image (CDN); local `product-kit.png` is only a fallback.
- **Before/after:** placeholder wells on screen 7 still need real flat-vs-defined images.
- **Shopify checkout:** wired ✓ and verified live — `merwavehair.myshopify.com`, handle `merwave-starter-kit`, Headless public Storefront token. `cartCreate` returns a valid `checkoutUrl` (£65 GBP).

Compliance: screens 14–15 are wellness/self-assessment positioning, flagged for legal review.

---

## 8. Things to NOT do

- **Don't** reintroduce the indigo/lavender Restfulness palette. Merwave is warm cream + dusty blue + soft yellow.
- **Don't** strip the soft shadows to go "flat" — shadows are part of this brand (the opposite of the Restfulness rule).
- **Don't** use Satoshi / ITC Clearface. Body = Proxima Nova, headings = Quincy CF.
- **Don't** add countdown timers or fake scarcity — the doc explicitly bans a fake countdown.
- **Don't** gradient-text-fill copy. The `<em>` rule is plain serif italic in dusty-blue `currentColor`. Gradients are reserved for bordering/filling chrome (progress, selected option, chart, loader).
- **Don't** turn screen 17 back into a hidden block — it is the active close.
