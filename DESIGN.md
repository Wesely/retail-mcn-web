# Design

## Theme

Light, near-white cool surfaces (chroma ~0, faint cool tint — NOT cream/sand). One brand identity, art-directed per audience. Internal pages restrained; external pages committed.

## Color (OKLCH)

Shared brand tokens (defined at top of every page CSS):
- `--ink`      oklch(0.24 0.02 255)   — primary text / structural dark
- `--ink-2`    oklch(0.40 0.02 255)   — secondary text
- `--muted`    oklch(0.55 0.015 255)  — tertiary / captions
- `--bg`       oklch(0.985 0.004 235) — page background (cool near-white)
- `--surface`  oklch(1 0 0)           — cards / panels
- `--line`     oklch(0.91 0.006 240)  — borders
- `--coral`    oklch(0.66 0.19 35)    — ENERGY accent (page2 primary, page1 small accent)
- `--coral-d`  oklch(0.55 0.17 33)    — coral hover/text-on-light
- `--teal`     oklch(0.55 0.10 195)   — TRUST accent (page1 primary, support elsewhere)
- `--teal-d`   oklch(0.42 0.08 200)   — deep teal
Status (page4): red oklch(0.57 0.18 25) · amber oklch(0.72 0.13 75) · green oklch(0.60 0.13 155)

Strategy per page: page1 Restrained→Committed (ink + teal lead, coral spark) · page2 Committed (coral leads) · page3/4 Restrained (ink + neutrals + teal action) · index ink-forward with both accents.

## Typography

- Display (Latin + numerals only): **Bricolage Grotesque** 600–800. Characterful grotesque, energetic+professional. NOT on reflex-reject list.
- Body + CJK headings: **Noto Sans TC** 400/500/700/900. Carries Chinese at all sizes.
- Mono (internal data/labels): **JetBrains Mono** 400/700.
- No gradient text. No all-caps body. Hierarchy via weight (900 vs 400) + size. h1 clamp max ≤ 6rem. Display letter-spacing ≥ -0.04em. `text-wrap: balance` on headings.

## Motion

Brand pages: one orchestrated entrance + intentional hover. Product pages: 150–250ms state transitions only. Ease-out-expo `cubic-bezier(0.16,1,0.3,1)`. Every animation has prefers-reduced-motion fallback. Reveals enhance already-visible content (no visibility gating).

## Bans enforced

No eyebrows-per-section, no 01/02/03 numbered markers as scaffolding, no side-stripe accent borders, no gradient text, no hero-metric template, no glassmorphism-by-default.
