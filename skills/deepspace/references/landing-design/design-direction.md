# How to write a Design Direction

The Design Direction is the forcing function at the top of your landing page file. It's the single most important part of building a good landing page — more than any pattern choice, font pick, or animation. Most AI-generated landing pages look generic because the agent never committed to anything specific about the product before writing code. This exercise is how you commit.

Read this file the first time you fill the Direction block, or whenever the sentence test keeps rejecting what you write.

---

## The six prompts, explained

### 1. The product — one sentence, what it does for whom

Write it like you'd explain it to a friend in an elevator. Not the marketing pitch, not the tagline, not the feature list. The actual thing.

**Bad**: "An AI-powered productivity platform."
**Bad**: "A modern solution for team collaboration."
**Good**: "A daily journal app for runners that asks one question after every run."
**Good**: "A recipe book that teaches weeknight cooking to people who are scared of knives."
**Good**: "A digital whiteboard where screenwriters pin scene cards and rearrange acts."

Notice what the good ones do: they name a specific user, a specific activity, and a specific frame. That's how they become concrete enough to design for.

### 2. The one emotion this landing page should evoke

Be specific. "Trust," "excitement," "confidence" — these are words you'd find in any marketing deck. They're not emotions, they're categories. Push further. The emotion should be a specific feeling you could put a finger on in real life.

**Bad**: "Trust and professionalism."
**Bad**: "Excitement about what's possible."
**Good**: "The calm of finishing the last item on a to-do list."
**Good**: "The small pride of setting a personal record."
**Good**: "Sunday morning, second coffee, nowhere to be."
**Good**: "The focus that settles after you crack open a fresh notebook."

When the emotion is this specific, every design choice downstream gets easier. A muted beige background serves "Sunday morning coffee." A shimmering typewriter effect serves "the focus of a fresh notebook." Generic emotions produce generic pages because every design choice is a coin flip.

### 3. The visual metaphor — a concrete real-world image

This is the most underused prompt. Most AI-generated landing pages have no metaphor at all — they just apply "modern SaaS aesthetic" and hope. Don't do that.

The metaphor is a concrete real-world image you could photograph. Not other software. Not other landing pages. Not abstract vocabulary like "clean" or "premium." An *image*.

**Bad**: "Clean and modern."
**Bad**: "Premium feel with a focus on trust."
**Good**: "A handwritten recipe card on a butcher-block table, morning light through a window."
**Good**: "A blinking cursor in a dark server room, the hum of fans."
**Good**: "A polaroid slowly developing in someone's hand."
**Good**: "A topographic map pinned to the wall of a hunting cabin."

When the metaphor is an image, you know what color it is, what texture, what light, what fonts it would use, what motion would fit. The metaphor becomes the brief for every design decision.

### 4. Three references from OUTSIDE this product's own industry

Most designers making a cooking app will look at other cooking apps. That's why every cooking app looks the same. The way to make yours look different is to deliberately look *outside* the category for references.

**Bad** (for a cooking app): "Allrecipes, Food Network, NYT Cooking."
**Good** (for a cooking app): "Kinfolk magazine, Le Creuset product packaging, Wes Anderson color palettes."

**Bad** (for a dev tool): "Stripe, Linear, Vercel."
**Good** (for a dev tool): "Teenage Engineering TX-6, Swiss railway signage, analog modular synth patch cables."

**Bad** (for a kids storybook app): "Other kids apps, Nickelodeon websites."
**Good** (for a kids storybook app): "Eric Carle illustrations, the color script of Pixar's Up, Sanrio stationery."

The references don't have to be websites. They can be products, magazines, films, books, brands, objects. They just have to share your chosen emotion or metaphor — and they must come from outside your product's own category. Forcing yourself outside the category is how you avoid cloning your competitors.

### 5. The ONE signature visual element this page will have

One thing. Something memorable. Something a friend could describe to another friend in one sentence and they'd know which landing page you meant.

**Good**: "Torn-paper SVG section dividers that look like kraft paper."
**Good**: "A mini playable Tetris board running in the hero."
**Good**: "A breath circle that pulses at 4.5 seconds per cycle as the hero centerpiece."
**Good**: "An animated topographic line pattern that rolls across the background as you scroll."
**Good**: "A live terminal in the hero that types real commands with realistic variable timing."
**Good**: "A polaroid that slowly develops over three seconds when the hero scrolls into view."

Signature elements are the difference between "another AI landing page" and "oh, the one with the X." Commit to one before writing code.

### 6. The hero visual — concretely, what animates on screen

Not "product mockup." Not "atmospheric background." Describe, in one sentence, the thing the user sees moving in the first 5 seconds of loading the page.

**Bad**: "An animated hero background."
**Bad**: "A product mockup with some motion."
**Good**: "A three-column Kanban with cards that move between columns every two seconds."
**Good**: "A slow cinemagraph of morning light moving across a kitchen counter while steam rises from a mug."
**Good**: "A topographic line pattern that animates outward from the center, like ripples on a pond."
**Good**: "A terminal window that types `$ watch-run --live` and then starts streaming fake test results."

If you can't describe the motion concretely, the hero will come out as "a centered headline with a Tailwind gradient background" — the default failure mode.

---

## The Style Tile — concrete commitments

The 6 prompts above produce a *brief* — what the page should feel like. The brief is necessary but not sufficient. Before you start coding, you also have to translate the brief into concrete design tokens: a **Style Tile** of 6 specific decisions you can write down in one line each.

1. **Color** — dominant + accent + saturation level
2. **Type pair** — heading font + body font (specific names)
3. **Theme** — light or dark
4. **Art direction** — one archetype (modern minimalism, editorial, brutalism, bento, glassmorphism, etc.)
5. **Motion personality** — one archetype (stillness, subtle drift, cinematic, kinetic, playful, mechanical)
6. **Voice** — three concrete *behaviors* (not adjectives)

These six commits live alongside the Design Direction block as a preamble comment in your landing page file. The full options menu with concrete font pairings, archetype descriptions, and pick rules is in `style-tile.md`. Open that file when you're filling each commit, scan the relevant table, pick, move on.

**The behaviors-not-adjectives rule** is the only one that needs explanation. "Friendly," "professional," "modern" are categories — different writers do different things with each. Write *behaviors* the agent can mechanically check: "uses second-person, max 12-word sentences, never starts with 'we'." Three behaviors is enough to lock voice consistency across the whole page without becoming a stylebook.

A good Style Tile makes every section choice obvious — once you've committed to "Fraunces + Source Sans, dark mode, editorial archetype, cinematic motion, voice = lyrical and terse," you don't need to second-guess every section while composing. You just check what you wrote.

## The sentence test

After filling the six prompts AND the Style Tile, read everything back. Ask yourself: **could this direction describe any other product?**

If yes, rewrite until the answer is no. Generic directions produce generic landing pages. The whole point of this exercise is to commit to something specific about THIS one product.

Examples of directions that fail the sentence test:
- "A modern SaaS tool. Clean and trustworthy. Like Stripe and Linear. A floating product mockup." → Describes any SaaS startup. Rewrite.
- "An AI assistant. Intelligent and helpful. Like ChatGPT and Perplexity. A chat interface." → Describes every AI product. Rewrite.

Examples of directions that pass the sentence test:
- "A daily journal for runners that asks one question after every run. The calm of finishing a long run — the kind where you walk slowly back to the house afterward. A handwritten postcard that someone wrote to themselves, set on a scratched wooden table. References: A Field Guide to Getting Lost (book), Rivendell Bicycle catalog, the UI of a Garmin watch from 2012. Signature: a single handwritten question that fades in after the page loads, different every refresh. Hero: the question appears in cursive handwriting, then a soft glow grows behind it as if a lamp just turned on in the next room."

That direction cannot describe any other product. Every design decision downstream is now either obviously right or obviously wrong relative to the brief.

---

## Cohesion with the rest of the app

The landing page is not a separate project — it sits in the same codebase as the app, reads from the same `@theme` block in `src/styles.css`, and links to the same authenticated surfaces. Direction should extend the app's identity, not fight it.

Concrete rule: the **body font** and **color palette** in the Style Tile must match the app's `styles.css`. The **heading font**, **art direction**, **motion personality**, and **voice** are your free picks — they're how the landing page feels distinct from the app while still sharing visual DNA. A landing page that looks like a different product than its own app is broken — sign-in leads to an inconsistent identity.

If the app's theme hasn't been finalized yet, **build the app's theme first** (SKILL.md Step 5 / uiux.md §2), then reach for this skill. The app identity drives the landing page, not the other way around.

---

## If you're stuck

Stuck filling the Direction block usually means one of:

- **You don't understand the product well enough.** Go read the rest of the app's code. Look at the data model. Look at the main user flow. Come back.
- **You're trying to match a category instead of committing to the product.** If you keep writing "modern SaaS aesthetic," stop and ask: what's unusual about THIS app? What's the one thing that makes it different from the generic version? Start there.
- **You're afraid of committing to something specific.** The fear is: "what if my direction is wrong?" The answer is: a wrong direction you commit to produces a specific landing page that can be iterated. A right direction you don't commit to produces a generic landing page that can't be iterated because there's nothing to iterate toward. Commit.

Write the Direction. Apply the sentence test. Then compose.
