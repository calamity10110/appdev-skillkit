---
name: tailwind-patterns
description: Write disciplined utility-first CSS with Tailwind — compose utilities instead of custom CSS, merge classes safely, keep class names statically analyzable, and drive theming through CSS-variable design tokens. Triggers when writing or reviewing Tailwind classes, styling components, or setting up Tailwind theming.
---

# Tailwind Patterns

Utility-first discipline: Tailwind works best when the stylesheet stays a thin
composition layer over a token system. The moment custom CSS grows, the discipline
has slipped.

## 1. Composition over custom CSS

- **Utilities first, always.** If a style can be expressed with utilities, use them.
  Reach for custom CSS only for what utilities can't do: complex keyframes, masks,
  or genuinely novel effects.
- **Keep custom CSS in one place** (`@layer components` or a utilities file), never
  scattered in style tags. If the same custom block appears twice, promote it to a
  component or a shared class.
- **Don't fight the cascade with `!important`.** Specificity wars mean the
  composition is wrong — restructure instead.

## 2. Merge classes safely with cn()

- **Always merge conditional classes through `cn()`** (clsx + tailwind-merge):
  `cn("px-4 py-2", isActive && "bg-blue-600")`. Plain template-string concatenation
  leaves conflicting classes (`p-2 p-4`) and the browser picks the winner by
  stylesheet order — a silent bug.
- **tailwind-merge resolves conflicts intentionally** — the later class wins. Keep
  base styles first, overrides last, and make the order obvious.
- **Extract repeated class strings** into small components or `cn()`-based helper
  functions (e.g., `buttonStyles({ variant })`), not into CSS files.

## 3. Full class names — never dynamic

- **Write complete class names.** The Tailwind compiler only generates classes it
  can see statically. `bg-${color}-500` compiles to nothing — always enumerate:
  `color === "red" ? "bg-red-500" : "bg-blue-500"`.
- **Keep a safelist or explicit map** for any class that must be chosen at runtime:
  `const tones = { danger: "bg-red-500", ok: "bg-green-500" }`.
- **Same rule for variants:** no dynamic `dark:${x}` construction.

## 4. Design tokens via CSS variables

- **Theme through CSS variables**, not hard-coded values. In Tailwind v4, define
  tokens with `@theme { --color-primary: ...; --spacing-...: ...; }`; in v3, extend
  the config to read from variables.
- **Semantic token names** (`--color-surface`, `--color-accent`) beat literal ones
  (`--blue-500`) — the theme changes, the names don't.
- **Dark mode = variable swap.** Use the `dark:` variant to flip tokens
  (`dark:bg-surface-dark`), keeping components theme-agnostic.
- **Spacing and radius scale once:** define `--spacing-*` and `--radius-*` tokens
  and use them everywhere; no arbitrary `px-[13px]` one-offs except true
  exceptions.

## 5. Variant and responsive discipline

- **Mobile-first ordering:** base styles for mobile, then `sm:`, `md:`, `lg:`
  overrides. Keep breakpoint usage consistent across the codebase.
- **Group state variants at the end** of the class list (`hover:`, `focus-visible:`,
  `disabled:`, `dark:`) so the base → state progression reads left to right.
- **Prefer `focus-visible:` over `focus:`** so mouse users don't get focus rings
  they didn't ask for.

## 6. Review checklist

No dynamic class construction. All conditionals merged via `cn()`. Colors, spacing,
and radii come from tokens. `dark:` coverage complete. No `!important`. No arbitrary
values that should be tokens. Responsive behavior verified at small screens.

## Input → Process → Output

**INPUT:** A codebase with 40+ components, colors hard-coded as hexes, three
different grays for "the same" surface, dark mode implemented by overriding
individual classes, and `bg-${tone}-500` dynamic classes that silently render
nothing.

**PROCESS:**
1. Extract tokens: define semantic CSS variables in `@theme`
   (`--color-surface`, `--color-accent`, `--spacing-*`, `--radius-*`); replace
   every literal hex with its token.
2. Fix the dynamic classes: replace `bg-${tone}-500` with an explicit map
   (`{ danger: "bg-red-500", ok: "bg-green-500" }`) so the compiler sees full
   class names.
3. Rebuild dark mode as a variable swap: components keep `bg-surface`;
   `.dark` redefines `--color-surface`. Delete the per-component overrides.
4. Normalize conditionals: route every conditional class string through `cn()`
   so conflicting classes resolve intentionally (later wins) instead of by
   stylesheet order.
5. Verify: grep for remaining hex literals and `!important`; screenshot both
   themes; check small screens.

**OUTPUT:** One token system, zero hard-coded colors, dark mode as a token
redefinition, no silent-missing classes — and a theme change that is now a
variable edit instead of a component hunt.

## Proven pattern vs avoid pattern

- **Proven:** Semantic tokens + `cn()` merging. Tokens make theming a data
  problem; `cn()` (clsx + tailwind-merge) makes conditional classes
  deterministic. Together they eliminate the two most common Tailwind bugs:
  theme drift and conflicting-class roulette.
- **Avoid:** Dynamic class construction (`bg-${x}-500`, `dark:${y}`). The
  compiler only emits classes it can see statically — interpolated names compile
  to nothing and fail silently, with no error and no styles. Enumerate every
  class explicitly; safelist nothing you can write out.

## Telemetry, testing & debugging

- **Measure:** CSS bundle size before/after token migration (should shrink);
  count of arbitrary values (`px-[13px]`) as a drift metric — trend toward zero.
- **Test:** visual regression across both themes on every PR touching styles;
  a lint rule or grep check in CI for dynamic template-literal classes and raw
  hex values in component files.
- **Debug:** missing styles with no error → check the Tailwind `content` config
  covers the file, and search for interpolated class names. Wrong style winning
  → inspect for conflicting classes the merge didn't resolve (order in `cn()`
  matters). Dark-mode flash → a token missing its `.dark` definition.
