---
name: ui-ux-decisions
description: Make data-driven visual design decisions — choose a style direction, enforce palette discipline, pair fonts, and run a pre-delivery quality pass. Triggers when starting a UI's visual design, restyling an app, or checking design quality before shipping. For full style/palette/font databases, see the upstream project noted below.
---

# Data-Driven UI/UX Decisions

A decision process, not a style. Follow it in order: direction → palette → type →
components → pre-delivery check. Decisions are written down before pixels are pushed.

> **Full databases:** the upstream project `nextlevelbuilder/ui-ux-pro-max-skill`
> (GitHub; companion CLI `npx uiprocli init`) ships the complete searchable
> databases — 70+ style directions, ~190 palettes, 70+ font pairings — plus a
> generator and reasoning rules. This skill teaches the decision process; the
> upstream holds the raw data.

## 1. Pick a style direction first

- **Name the direction explicitly** before styling anything: e.g., minimal SaaS,
  brutalist editorial, glassy fintech, warm craft, clinical medical. Vague ("modern,
  clean") is not a direction.
- **Derive from the product's job:** trust products want restraint and hierarchy;
  creative tools want expressive surfaces; consumer apps can be playful. Match the
  visual energy to the user's intent, not to trends.
- **Write a 3-word style contract** (e.g., "calm, dense, precise") and reject any
  styling choice that breaks it. Pin one reference: a screenshot or the closest
  database entry.
- **One direction per product.** Mixed metaphors (glassmorphism cards on a
  brutalist page) read as unfinished, not eclectic.

## 2. Palette discipline: 60-30-10

- **60% dominant** (backgrounds, large surfaces), **30% secondary** (cards,
  sidebars, sections), **10% accent** (CTAs, active states, key highlights).
- **Accent is a budget, not a color.** Spend it where the user's eye must go:
  primary actions, selected states, critical alerts. If everything is accented,
  nothing is.
- **Build from tokens, not hexes.** Define semantic tokens
  (`--color-surface`, `--color-primary`, `--color-danger`) and reference them
  everywhere; never hard-code hex values in components.
- **Check contrast at decision time**, not after: body text and accent-on-surface
  must clear AA (4.5:1) before the palette is approved.
- **Neutrals carry the design.** Spend real effort on the gray scale — warm vs
  cool, and enough steps between surface and border — before choosing the accent.
- **Dark mode is a second palette**, not an inverted one: re-derive surfaces for
  elevation (lighter = higher), dim text instead of pure white, re-verify contrast.

## 3. Font pairing rules

- **Two typefaces maximum:** one for display/headings, one for body. A third is
  allowed only for code or data.
- **Pair by contrast of role, not decoration:** a distinctive display face with a
  neutral, highly-legible body face. Two decorative faces fight; two neutral faces
  bore.
- **Check the pairing at real sizes:** display at hero size, body at 14–16px, and
  the smallest caption you'll ship. Pairings that only work at one size fail.
- **Limit weights:** 2–3 weights total. Every extra weight is a font file the user
  downloads.
- **Line length and rhythm:** 45–75 characters per line, 1.4–1.6 line height for
  body; tighter for display. Set the scale from a ratio (1.125 / 1.25), not ad hoc.

## 4. Component consistency

- **One component, one look.** Buttons, inputs, and cards share radius, border
  treatment, and shadow language across the whole product.
- **States are designed, not defaulted:** hover, focus, active, disabled, loading,
  and error states for every interactive component — no browser defaults in
  production.
- **Density is a decision:** data-dense tools can be compact; consumer flows need
  air. Pick one and hold it.

## 5. Pre-delivery checklist

Run before any design is called done:

1. Style contract intact — nothing violates the 3-word direction.
2. 60-30-10 respected; accent spent only on intent points.
3. All colors via semantic tokens; no stray hexes.
4. Contrast passes AA in both themes.
5. Two typefaces max; weights limited; scale consistent.
6. Every interactive component has all six states.
7. Focus indicators visible everywhere.
8. Spacing follows the grid; no magic numbers.
9. Responsive at 360px, 768px, 1440px — no horizontal scroll.
10. Reduced-motion alternative for every animation.

## 6. Anti-patterns to reject

- Gradient text/buttons as the default "premium" signal.
- Purple-blue gradients, glassmorphism, and generic hero layouts — the markers of
  unconsidered AI-generated UI.
- More than one accent color competing on a single screen.
- Placeholder-lorem design approved as final; real content changes every layout.

## Input → Process → Output

**INPUT:** A product brief: "Restyle our analytics dashboard. Users say it looks
generic, the dark mode is unreadable, and nobody can tell which chart is the
call to action."

**PROCESS:**
1. Name the direction from the job: it's a trust product — direction:
   "calm, dense, precise". Write the 3-word contract; reject anything that breaks it.
2. Apply 60-30-10: 60% quiet dark surfaces, 30% panel backgrounds, 10% accent
   reserved for the primary CTA chart and selected states.
3. Check contrast at decision time: body text and the accent on dark surfaces must
   clear 4.5:1 before approval — the unreadable dark mode dies here, not in QA.
4. Pick two typefaces: neutral body face, distinctive display face for metric
   numerals; verify at 12px captions and hero metric sizes.
5. Hold component consistency: one radius, one border language, all six states
   (hover/focus/active/disabled/loading/error) for every control.
6. Run the 10-point pre-delivery checklist end to end.

**OUTPUT:** A decision record: style contract ("calm, dense, precise"), token list
(`--color-surface`, `--color-panel`, `--color-accent` + dark re-derivations),
type pair + scale, and a checklist sign-off — plus the rejected options
(purple-blue gradient hero, glassmorphism cards) noted as contract violations.

## Proven pattern vs avoid pattern

- **Proven:** Decide the style contract *before* touching color. One sentence of
  direction rejects a hundred bad choices for free — teams with a contract ship
  coherent UI without a designer in every review.
- **Avoid:** Picking the accent color first. Color without direction produces
  decoration, not design: every screen gets the "premium" gradient and the
  product looks like every other AI-generated dashboard. Direction → tokens →
  components, in that order.

## Telemetry, testing & debugging

- **Measure:** contrast ratios per token (AA 4.5:1) in both themes — bake into CI
  with a token-contrast check; track cumulative layout shift during restyles.
- **Test:** visual regression (Chromatic or Percy) on the pre-delivery checklist
  states — especially dark mode and focus/hover/error states, where regressions
  hide. Screenshot at 360/768/1440px.
- **Debug:** when a palette feels wrong, don't tweak hexes — re-check the 60-30-10
  split (accent creep is the usual culprit). Use browser devtools' contrast
  overlay and the axe DevTools extension to find AA failures by element.
