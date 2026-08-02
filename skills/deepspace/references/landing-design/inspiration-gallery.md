# Inspiration Gallery

Pick one row by emotional adjacency, not product category, then read exactly its one example. Trace Direction → Style Tile → code; adapt the mechanism, never clone or import the example.

## The five archetypes

| # | Archetype | Emotion | Visual metaphor | Signature element | Example file |
|---|---|---|---|---|---|
| 01 | Cooking / warmth | Sunday kitchen warmth, nostalgic, tactile | A handwritten recipe card on a butcher-block table, morning light through a window | Torn-paper SVG section dividers | `examples/01-cooking-warmth.tsx` |
| 02 | Developer tool / precision | Sharp, confident, technical, precise | A blinking cursor in a dark server room, code compiling in real time | A live terminal in the hero that types real commands with realistic variable timing | `examples/02-devtool-precision.tsx` |
| 03 | Meditation / calm | Spacious calm, weightless, present | The horizon line at dawn, the pause between breaths | A breath circle pulsing at 4.5s per cycle as the hero centerpiece | `examples/03-meditation-calm.tsx` |
| 04 | Children's storybook / playful | Playful, imaginative, tactile, safe | A paper cut-out diorama, crayon scribbles on construction paper | Hand-drawn SVG elements that wobble + paper grain texture overlay | `examples/04-kids-storybook.tsx` |
| 05 | SaaS / clarity | The Friday-afternoon recap that lets the laptop close | A one-page-per-week paper folio in a manager's bottom drawer | An animated bento dashboard mockup that runs once on entry: chart draws itself, status flips, recap paragraph types in | `examples/05-saas-clarity.tsx` |

Style Tile shorthand (the full tile is in each example):

- 01 — Warm cream + terracotta · Fraunces + Source Sans 3 · light · editorial · subtle drift · second-person, no em dashes, warm.
- 02 — Near-black + cyan accent · IBM Plex Mono + Inter · dark · modern minimalism · mechanical · verb-first, no adjectives, max 10 words.
- 03 — Cream + sage · Cormorant + Lato · light · modern minimalism · stillness · generous sentences, no urgency words.
- 04 — Warm yellow + coral · Nunito only · light · hand-drawn illustrated · playful bouncy · second-person, short questions, joyful.
- 05 — Warm off-white + ink-blue + moss-green status · Inter + IBM Plex Mono · light · bento-modular with editorial restraint · subtle drift · declarative, second-person, max 14 words, no exclamation points.

If no emotion matches, choose by metaphor, then by transferable signature mechanism. Example 05 focuses on the animated bento mockup; navigation and FAQ variants live in the pattern library.

## What to look for when reading the example file

Read, in order: Direction block, signature implementation, typography/color commitments, motion personality. Then close the example before composing.

## What to learn from each archetype

| Example | Design lesson |
|---|---|
| 01 | Repeated texture/imperfection can make structure feel handmade. |
| 02 | One accent, mono type, mechanical easing, and a single terminal make restraint the identity. |
| 03 | A deliberately slow 4.5-second breath cycle makes motion serve calm instead of reading as a spinner. |
| 04 | Coordinated wobble, imperfect SVG, grain, and rounded shapes make playfulness intentional. |
| 05 | Editorial hierarchy restrains a bento layout; its mockup animates once, then becomes still. |

## Why only five archetypes

Five limits menu-driven cloning. If none fits, use an external landing page, magazine spread, or product photograph that shares the emotion.

## What reference code not to rely on

Do not ship installed sections verbatim or import examples. Direction owns the result; patterns supply structure; examples demonstrate another product's translation.
