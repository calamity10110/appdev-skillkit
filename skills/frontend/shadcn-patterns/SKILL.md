---
name: shadcn-patterns
description: Build UIs with shadcn/ui components the intended way — install via the CLI, compose variants with cn() and cva, theme through CSS variables, and wire forms with react-hook-form + zod. Triggers when adding, customizing, or reviewing shadcn/ui components.
---

# shadcn/ui Patterns

shadcn/ui is a component *distribution*, not a dependency: components live in your
repo and you own them. The patterns below keep that ownership clean and upgradeable.

## 1. Install via CLI, never copy-paste

- **Add components with the CLI** (`npx shadcn@latest add button dialog ...`).
  This wires imports, the `cn()` util, and theme variables correctly. Copy-pasting
  from the docs skips the wiring and drifts over time.
- **Keep the generated file structure** (`components/ui/*`). Own the code — you
  can edit it — but keep edits deliberate and minimal so upstream updates stay
  mergeable.
- **One source of truth for the `cn()` util** (`lib/utils.ts`); every component
  and every custom wrapper imports from there.

## 2. Variants with cn() + cva

- **Define component variants with `cva`** (class-variance-authority):
  `buttonVariants({ variants: { variant: { primary: ..., ghost: ... }, size: {...} } })`.
  Variants are data, not if-else chains in JSX.
- **Compose with `cn()`:** `cn(buttonVariants({ variant, size }), className)` —
  the consumer's `className` comes last so overrides win predictably (tailwind-merge
  resolves conflicts).
- **Type variants from cva** (`VariantProps<typeof buttonVariants>`) so props stay
  in sync with the variant map.

## 3. Theme through CSS variables

- **All colors are CSS variables** (`--background`, `--foreground`, `--primary`,
  `--muted`, `--destructive`, ...). Components reference `bg-primary`,
  `text-muted-foreground` — never literal colors.
- **Dark mode is a variable redefinition** under `.dark` (or `[data-theme]`); no
  component changes needed. Verify every token has both definitions.
- **Customize the theme by editing the variables**, not by overriding component
  classes. Brand changes become a token edit, not a component hunt.

## 4. Forms: react-hook-form + zod

- **Wire forms with react-hook-form** for uncontrolled inputs and cheap renders;
  pair with **zod** schemas via the zod resolver for validation.
- **One schema, two jobs:** the same zod schema validates client-side and can be
  reused server-side — define it once, import it in both places.
- **Use shadcn's Form primitives** (`Form`, `FormField`, `FormItem`, `FormLabel`,
  `FormControl`, `FormMessage`) so labels, errors, and `aria` wiring come free.
- **Validate on blur/submit**, show `FormMessage` errors inline, and disable submit
  while pending.

## 5. Radix primitives: never import directly

- **Import from your `components/ui` wrappers, not from `@radix-ui/*`.** The
  wrappers carry your styling, tokens, and behavior defaults; direct Radix imports
  bypass them and fragment the design system.
- If a Radix primitive has no shadcn wrapper yet, **create the wrapper first** in
  `components/ui`, then use it everywhere.

## 6. Review checklist

Components installed via CLI. Variants defined in cva, composed with cn(),
consumer className last. No literal colors — everything through theme variables,
both themes defined. Forms use rhf + zod with the shared schema. No direct
`@radix-ui` imports. Custom edits to generated components are minimal and
documented.

## Input → Process → Output

**INPUT:** "Add a themed 'Invite member' dialog with an email form. It must work
in both themes, validate the email, and match our button/destructive variants."

**PROCESS:**
1. Install via CLI: `npx shadcn@latest add dialog button input form` — wires
   imports, `cn()`, and theme variables; never copy-paste from docs.
2. Define the button usage with `cva` variants already in the codebase
   (`variant: "destructive"` for the remove action); compose with
   `cn(buttonVariants({ variant, size }), className)` — consumer className last.
3. Build the form with shadcn Form primitives + react-hook-form + a zod schema
   (`z.string().email()`); the same schema is importable server-side for the
   invite API.
4. Theme check: dialog uses `bg-background`, `text-foreground`,
   `bg-destructive` — no literal colors; verify every token exists under
   `.dark` in the variables file.
5. Import the dialog from `components/ui/dialog`, not `@radix-ui/react-dialog`
   — the wrapper carries the project's styling and behavior defaults.

**OUTPUT:** A working invite dialog: CLI-installed, variant-consistent, validated
by a shared zod schema, theme-clean in both modes — and the next dialog the team
builds starts from the same wrappers, not from scratch.

## Proven pattern vs avoid pattern

- **Proven:** Own the components, respect the wrappers. shadcn is a
  distribution, not a dependency — installing via CLI and always importing from
  `components/ui` keeps one styled, themed, behavior-consistent source of truth
  that upstream updates can still merge into.
- **Avoid:** Importing `@radix-ui/*` directly. It bypasses your wrappers —
  their styling, tokens, and defaults — and silently fragments the design
  system: two dialogs that look and behave slightly differently, with no single
  place to fix them.

## Telemetry, testing & debugging

- **Measure:** form validation error rate and time-to-complete on key forms
  (analytics events on submit/error) — validation UX is measurable.
- **Test:** Playwright flows for dialog open/close (including Escape and overlay
  click), focus trap and focus return on close, keyboard-only form completion,
  and both themes via screenshot. Test the zod schema's edge cases as unit
  tests — it's shared with the server.
- **Debug:** theme looks wrong → open devtools and check which CSS variable is
  unresolved (a missing `.dark` definition shows as an empty value, not a
  fallback). Focus not trapped → the Radix primitive was imported directly
  instead of through the wrapper. Styling overridden unexpectedly → check
  `className` ordering in the `cn()` call (consumer's must be last).
