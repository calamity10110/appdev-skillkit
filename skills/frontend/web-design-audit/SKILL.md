---
name: web-design-audit
description: Audit shipped UI against accessibility, form, motion, typography, dark-mode, and internationalization standards. Triggers on requests to review, check, or audit a UI, interface, or site for accessibility and design quality. Acts as a reviewer, not a builder.
---

# Web Design Audit Checklist

A reviewer's checklist for shipped interfaces. Run through each section, note failures
as `file:line — issue — severity`, and group findings at the end by severity. A page
is not done until there are no blocking findings.

## How to audit

1. Render or read the target files/pages.
2. Walk the checklist top to bottom — accessibility first.
3. Report findings tersely: `file:line`, what fails, why it matters, the fix.
4. End with a pass/fail verdict and the top three fixes by impact.

## A. Accessibility (blocking)

- **Keyboard operability.** Everything clickable is reachable and operable by keyboard
  alone: tab order follows visual order, focus never gets trapped, modals trap focus
  and return it on close.
- **Visible focus states.** Every interactive element shows a clear focus indicator
  (outline or ring) that survives custom styling. No `outline: none` without a
  replacement.
- **Contrast.** Text meets WCAG AA: 4.5:1 for body text, 3:1 for large text and
  essential icons. Test both themes if dark mode exists.
- **Labels and roles.** Inputs have associated labels (not placeholder-only); icon
  buttons have accessible names; custom controls use correct ARIA roles and states.
- **No information by color alone.** Errors, statuses, and required fields pair
  color with text, icon, or pattern.
- **Reduced motion respected.** Honor `prefers-reduced-motion`: disable or simplify
  non-essential animation.
- **Touch targets.** Interactive elements are at least 44×44 CSS px; adjacent
  targets don't overlap.
- **Landmarks and headings.** One `h1`, logical heading order, landmarks
  (header/nav/main/footer) used so screen readers can navigate.

## B. Forms (blocking)

- **Every field has a visible label** positioned consistently (above or floating).
- **Errors are specific and proximal.** Messages name the field and the rule
  violated, appear next to the field, and announce to screen readers
  (`aria-describedby` / `aria-invalid`).
- **Inline validation timing.** Validate on blur or submit, not aggressively on
  every keystroke; confirm success states too, not just errors.
- **Submit is guarded.** Disable or debounce double submission; show pending state;
  preserve entered data on failure.
- **Autocomplete attributes** (`autocomplete`, `inputmode`, `type`) set so browsers
  and mobile keyboards assist entry.

## C. Motion and animation

- **Purposeful, not decorative.** Animation communicates state change (entering,
  loading, confirming). If removing it loses nothing, remove it.
- **Fast and interruptible.** Durations under ~300ms for UI feedback; never block
  interaction waiting on animation.
- **Layout thrash.** Animate `transform`/`opacity` only; animating width, height,
  or top/left forces reflow.
- **Loading states.** Skeleton or spinner appears before the user perceives a hang;
  no blank white flashes during navigation.

## D. Typography and layout

- **Type scale discipline.** A small, deliberate scale (e.g., 4–6 sizes); body text
  16px minimum; line length 45–75 characters; line height 1.4–1.6 for body.
- **Hierarchy.** One visual priority per viewport — a heading style should never
  compete with another element for attention.
- **Spacing system.** Consistent spacing scale (4/8px grid); related items grouped,
  unrelated items separated; no magic numbers.
- **Responsive.** No horizontal scroll at any breakpoint; tap targets reachable on
  small screens; content reflows, doesn't just shrink.

## E. Dark mode

- **Complete coverage.** Every surface, border, shadow, and text color has a dark
  variant — no unstyled flashes or unreadable combos.
- **Contrast re-checked** in dark mode (light text on dark surfaces often needs
  dimming, not pure white).
- **Images and illustrations** handled: transparent assets or theme-aware variants;
  elevation shown with lighter surfaces, not shadows.

## F. Internationalization

- **No hard-coded user-facing strings** without an i18n path; layout survives
  30–40% text expansion (German, French).
- **Locale-aware formatting** for dates, numbers, and currency — never hand-rolled.
- **RTL readiness:** logical CSS properties (`margin-inline-start`) instead of
  physical (`margin-left`) where the product may go RTL.
- **Fonts cover target scripts;** emoji and icons aren't used as the only carrier
  of meaning across cultures.

## Verdict format

```
PASS / FAIL — <page/area>
Blockers (n): file:line — issue
Warnings (n): file:line — issue
Top fixes: 1) … 2) … 3) …
```

## Input → Process → Output

**INPUT:** "Audit our checkout page before launch. It's a 3-step flow with card
entry, and we ship in a week."

**PROCESS:**
1. Keyboard pass first: tab through the entire flow with the keyboard alone —
   can you reach every field, complete the purchase, and never get trapped?
   (A modal that traps focus without returning it fails here.)
2. Forms: every field has a visible label (not placeholder-only); errors name
   the field and rule, sit next to the field, and announce via
   `aria-describedby`; submit is disabled while pending with data preserved.
3. Contrast and color: card-error states must not rely on red alone — pair with
   text and icon; verify 4.5:1 in both themes.
4. Motion and loading: skeleton appears before perceived hang; no layout shift
   when step 2 loads; `prefers-reduced-motion` honored.
5. i18n spot-check: 30% text expansion doesn't break the payment summary;
   currency formatted by locale, not hand-rolled.
6. Report findings as `file:line — issue — severity`, end with PASS/FAIL and
   the top three fixes by impact.

**OUTPUT:**
```
FAIL — checkout flow
Blockers (3): components/PaymentForm.tsx:42 — card errors shown by color only;
  app/checkout/page.tsx:118 — focus not returned after address modal closes;
  components/CardInput.tsx:17 — input has placeholder but no label
Warnings (2): …
Top fixes: 1) label + aria wiring on CardInput 2) focus return on modal
  3) text+icon on error states
```

## Proven pattern vs avoid pattern

- **Proven:** Audit keyboard-first, checklist-second. A single keyboard-only pass
  surfaces the highest-severity a11y failures (traps, unreachable controls,
  invisible focus) faster than any automated scan — automation then catches what
  hands can't (contrast ratios, missing attributes at scale).
- **Avoid:** Shipping error and status information by color alone. Red borders
  mean nothing to color-blind users and screen readers — and "we'll add text
  later" never ships. Every status pairs color with text, icon, or pattern from
  the first commit.

## Telemetry, testing & debugging

- **Measure:** automated a11y violation counts per page in CI (axe-core, fail on
  new violations); form abandonment at each checkout step; CLS during step
  transitions.
- **Test:** Lighthouse CI with accessibility assertions on every PR; Playwright
  keyboard-only E2E (tab-order assertions, focus-trap checks); screen-reader
  spot checks (NVDA/VoiceOver) on the critical flow before launch; visual
  regression on both themes.
- **Debug:** run axe DevTools on the failing page — it reports the exact
  element, rule, and fix. For focus issues, use devtools' accessibility tree
  view to see what a screen reader actually receives; for contrast, the
  devtools color picker shows the ratio inline. Reproduce keyboard traps by
  scripting the tab sequence in Playwright.
