# How to write a Design Direction

Use this when first filling the source-file Direction block or when it fails the sentence test. Commit before choosing patterns, type, or motion.

---

## The six prompts, explained

### 1. The product — one sentence, what it does for whom

Name the user, activity, and frame—not a tagline or feature list.

- Bad: “An AI-powered productivity platform.”
- Good: “A daily journal for runners that asks one question after every run.”

### 2. The one emotion this landing page should evoke

Name a felt moment, not a marketing category.

- Bad: “Trust and professionalism.”
- Good: “Sunday morning, second coffee, nowhere to be.”

### 3. The visual metaphor — a concrete real-world image

Choose a photographable image, not software or adjectives. It should imply color, texture, light, type, and motion.

- Bad: “Clean and premium.”
- Good: “A handwritten recipe card on butcher block in morning light.”

### 4. Three references from OUTSIDE this product's own industry

Pick products, magazines, films, books, brands, or objects that share the emotion/metaphor but not the category.

- Bad for a dev tool: “Stripe, Linear, Vercel.”
- Good: “Teenage Engineering TX-6, Swiss railway signage, modular-synth patch cables.”

### 5. The ONE signature visual element this page will have

Choose one memorable element a person could identify in a sentence: e.g. torn-paper SVG dividers, a 4.5-second breath circle, or a realistic live terminal.

### 6. The hero visual — concretely, what animates on screen

Describe what moves in the first five seconds, not “a mockup with motion.” Example: “A three-column Kanban moves one card every two seconds.”

---

## The Style Tile — concrete commitments

Translate the brief into six one-line decisions:

1. **Color** — dominant + accent + saturation level
2. **Type pair** — heading font + body font (specific names)
3. **Theme** — light or dark
4. **Art direction** — one archetype (modern minimalism, editorial, brutalism, bento, glassmorphism, etc.)
5. **Motion personality** — one archetype (stillness, subtle drift, cinematic, kinetic, playful, mechanical)
6. **Voice** — three concrete *behaviors* (not adjectives)

Keep them beside the Direction in the source preamble. Scan only the matching tables in `style-tile.md`. Voice must be mechanically checkable behavior (for example, “second-person; max 12 words; never starts with ‘we’”), not adjectives.

## The sentence test

Ask: **could this describe another product?** If yes, rewrite.

- Fails: “A modern SaaS tool. Clean and trustworthy. Like Stripe and Linear. A floating mockup.”
- Passes: “A post-run journal asks runners one question. It feels like a handwritten postcard on a scratched table; a new cursive question fades in on each visit.”

---

## Cohesion with the rest of the app

Match the app's body font and palette. Heading type, art direction, motion, and voice may differentiate the landing without making sign-in feel like a different product. Finalize the app theme first.

---

## If you're stuck

- If the product is unclear, inspect its data model and main flow.
- If the brief describes a category, identify what makes this app unusual.
- Prefer a specific, revisable direction over a generic safe one.
