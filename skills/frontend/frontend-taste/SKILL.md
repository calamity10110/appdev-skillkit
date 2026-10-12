---
name: frontend-taste
description: Design judgment for building web UI with a distinctive point of view instead of generic AI-looking output. Use when designing a new page, choosing palette and type, reviewing a layout for clichéd defaults, or writing interface copy that reads human.
---

# Frontend Taste — Design Judgment

Act like a design studio lead whose client already rejected anything templated. Every choice should trace back to the product's subject matter, audience, and job — not to habit.

## Start from the brief, not the palette

- If the subject isn't clear, name it before drawing anything:
  what is this thing, who is it for, what is it trying to get them to do.
- The product's materials and vernacular are where a distinct look comes from —
  a playful kids' product and a finance dashboard should never rhyme.
- Where the brief pins a direction, follow it exactly.
  Where it leaves an axis open, don't spend that freedom on a default.

## Recognize the generated look

Watch for the traits AI output clusters around:

- warm cream backgrounds with a terracotta accent;
- near-black with one acid-green or vermilion accent;
- broadsheet hairline-rule layouts with dense newspaper columns;
- the SaaS card kit — everything chopped into identical rounded cards
  with the same soft shadow and gradient washes as decoration;
- template chrome that appears regardless of subject:
  tracked-out ALL-CAPS eyebrow labels over every heading,
  meta strings joined by middle dots, `WORD — fragment` labels,
  monospace used as decorative data labels, an arrow appended to every link.

All are legitimate somewhere — none should appear everywhere by reflex.

## Typography hierarchy

- Type carries the page's personality. One family, or two that are clearly
  distinct — never a pile of near-identical faces.
- Choose faces deliberately, not the default you'd reach for on any project.
- Set an intentional scale with deliberate weights, widths, and line spacing.
- Headline treatment is an active design element, not a neutral container:
  let the type itself do visual work.
- Keep body line lengths under ~80 characters;
  give serif body copy slightly more line height than sans-serif.
- Avoid the common tells:
  - accenting a single word in a headline (one italic/bold/colored word),
  - all-caps labels,
  - gratuitous eyebrow labels above content.

## Color discipline

- Define a tight named palette — four to six colors, each with a role:
  base, surface, text, accent, maybe one expressive tone.
- One accent carries the brand; everything else stays quiet.
- Tinted near-blacks and gradient washes are crutches, not choices.
- If each color can't be named and justified in a sentence, it isn't a system yet.

## Spacing rhythm

- Spacing is information, not air. Build a scale (e.g. 4 / 8 / 16 / 32 / 64)
  and stick to it — consistent rhythm reads as polish, arbitrary gaps read as sloppy.
- Structural devices (borders, dividers, rules, numbered markers) must encode
  real structure, not decorate.
- A number sequence belongs only on actual sequences — steps, timelines.
  Check the content really is a sequence before adding `01 / 02 / 03` markers.
- Vertical rhythm between sections matters more than clever components inside them.

## Component consistency

- One radius language for the whole page; one shadow treatment
  (or none — flat is a choice); one way of handling focus states; one motion idiom.
- Components that share a hierarchy level should share visual DNA.
- Design the states, not just the resting look:
  hover, focus, active, disabled, loading, empty, error.
  Unstyled states are where interfaces fall apart.

## Bold in one place, quiet everywhere else

- Spend your risk budget once. Pick the single memorable element —
  a hero treatment, one striking headline, one animated moment —
  and keep everything around it disciplined.
- Decorations that don't serve the brief get cut.
- Before finishing, remove one thing: restraint is the difference
  between designed and decorated.

## Motion with intent

- Motion should answer a user's action: opening, expanding, confirming,
  revealing what changed.
- A single orchestrated moment beats scattered effects —
  section-by-section fade-up entrances are the generic tell.
- Non-triggered motion stays rare and purposeful.
- Respect reduced-motion preferences; keep everything keyboard-navigable.
- Responsiveness and accessibility are the quality floor,
  not features to announce.

## Copy is design

- Words exist to make the interface easier to understand, not to decorate.
- Name things the way users think — their task language, not your system internals.
- Write CTAs as the exact action ("Save changes", "Publish") and keep that
  verb consistent through the whole flow, including confirmations.
- Speak to the user in active voice, sentence case, no filler;
  tone matched to brand and audience.
- Errors explain what happened and how to fix it — in the interface's voice,
  never apologetic, never vague about what went wrong.
- Empty screens are invitations to act, not dead ends.
- Specific and legible always beats clever.

## Process: plan, critique, build

1. Sketch a compact design plan first:
   palette, type roles, layout concept (a few sentences plus a quick
   ASCII wireframe), and the principles that make this page unique.
2. Interrogate it: if a part reads like what you'd produce for any similar
   prompt, revise it and note what changed and why.
3. Build to the revised plan, watching CSS specificity so rules don't
   silently cancel each other out.
4. Critique your own result with screenshots — a picture is worth more
   than another paragraph of self-review.

## Input → Process → Output

**INPUT:** Brief: "Landing page for a kids' science kit subscription. Audience:
parents of 8–12 year olds. Job: get them to start a free trial."

**PROCESS:**
1. Name the subject, audience, and job before any palette: playful but
   parent-trustworthy; trial signup is the single conversion.
2. Sketch a compact plan: palette/typography roles, one ASCII wireframe, and the
   principle that makes this page unique ("lab-notebook warmth, not SaaS").
3. Interrogate the plan for generated tells: cream + terracotta? all-caps
   eyebrows? arrow on every link? Cut or rework each.
4. Spend the risk budget once: one memorable hero treatment (hand-drawn diagram
   style). Everything else stays disciplined — tight 5-color palette, one radius
   language, spacing scale 4/8/16/32/64.
5. Build, then critique with screenshots: does the type do visual work? Do
   states (hover/focus/loading/error) hold up? Remove one decorative thing.

**OUTPUT:** A page with a point of view: lab-notebook aesthetic, trial CTA as the
exact verb "Start the free trial" carried through the flow, errors that say what
happened and how to fix it — and a short note of what was cut in critique.

## Proven pattern vs avoid pattern

- **Proven:** Derive the look from the product's materials and vernacular. A
  science kit can borrow the lab notebook — graph lines, annotation marks —
  giving it a distinctiveness no palette tweak can match.
- **Avoid:** Template chrome regardless of subject: tracked-out ALL-CAPS
  eyebrows, middle-dot meta strings, monospace decorative labels. These appear
  in AI output for finance dashboards and kids' products alike — if your design
  would work unchanged for a different industry, it has no point of view.

## Telemetry, testing & debugging

- **Measure:** conversion on the trial CTA (the design's job), Largest
  Contentful Paint for the hero, CLS during font load (use `font-display: swap`
  + size-matched fallbacks).
- **Test:** screenshot critique rounds — compare your page against two real
  competitors side by side; if it blends in, the direction failed. A/B the hero
  treatment, not the button color.
- **Debug:** when a page "feels off" but you can't say why, screenshot it at 50%
  zoom — hierarchy problems (competing focal points, weak rhythm) become obvious
  when detail disappears. Check CSS specificity when styles silently cancel.
