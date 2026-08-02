# Landing Page Design

Load for a marketing/landing/splash page or generic-design feedback on one. For the authenticated product home, use `uiux.md`. The workflow is Direction → Style Tile → one inspiration archetype → composition → grep gate.

---

## The workflow — 5 steps, in order

### 1. Decide where the landing page lives

Two paths:

- **Static (default):** rewrite `src/pages/index.tsx` at `/`. Top-level pages mount no providers or app chrome, which suits marketing and crawlers.
- **Dynamic feature:** `npx deepspace add landing` installs `(app)/landing.tsx`, primitives, and optional sections. Providers and global navigation mount. Treat every installed section as a skeleton; choose one real front door and remove or repoint the other.

Either path, the rest of this workflow is the same.

#### Hide the global Navigation on the landing route — required if the landing lives under `(app)/`

Static landings inherit no app chrome. For a landing under `(app)/`, hide the global nav on that route so it does not stack with landing chrome:

```tsx
// src/pages/(app)/_layout.tsx
import { useLocation } from 'react-router-dom'

export default function AppLayout() {
  const { pathname } = useLocation()
  const isLanding = pathname === '/landing'

  return (
    <div className="flex h-screen flex-col">
      {!isLanding && <Navigation />}
      <main className="min-h-0 flex-1 overflow-y-auto"><Outlet /></main>
    </div>
  )
}
```

Keep the layout's existing providers and suspense boundary around this conditional.

### 2. Fill the Design Direction block BEFORE any JSX

At the top of the landing page file (`src/pages/index.tsx`, or `src/pages/(app)/landing.tsx` on the feature-install path), write a prose block with two halves: a 6-prompt **brief** (product, emotion, visual metaphor, three references, signature element, hero visual) and a 6-token **Style Tile** (color, type pair, theme, art direction, motion personality, voice). The brief is prose. The Style Tile is six one-line commitments.

Put it in a multi-line comment block at the top of the file. It stays in source as documentation:

```tsx
/**
 * Design Direction
 *
 * Product: <one sentence — who it's for, what it does>
 * Emotion: <one specific feeling — not "trust" or "excitement">
 * Metaphor: <a concrete real-world image>
 * References: <three from OUTSIDE this product's category>
 * Signature: <the ONE memorable visual this page has>
 * Hero: <what animates on screen in the first 5 seconds>
 *
 * Style Tile
 * - Color: <dominant + accent + saturation>
 * - Type: <heading font + body font + why>
 * - Theme: <light | dark + why>
 * - Art direction: <one archetype from style-tile.md>
 * - Motion: <one personality from style-tile.md>
 * - Voice: <three behaviors, semicolon-separated>
 */
```

If you can't fill a prompt, you don't understand the product well enough to design for it yet — go read the rest of the app first. Load `landing-design/design-direction.md` for the full guidance on writing a good brief, and `landing-design/style-tile.md` for the Style Tile menus.

Apply the **sentence test** at the end: if everything you wrote could describe any other product, rewrite.

### 3. Read ONE inspiration archetype + its example file

Use `inspiration-gallery.md` to pick the closest *emotion*, then read only that row and example file. Learn how its Direction becomes code; never import or clone the example.

### 4. Compose the page

Build section by section. Load `landing-design/pattern-library.md` first — it's a tiny index (~50 lines) that lists which sub-files to load for each page section. Then load **only** the sub-file(s) you need:

- 1 nav pattern → `landing-design/pattern-library/nav.md` (or skip — some landing pages don't need a nav)
- 1 hero pattern → `landing-design/pattern-library/hero.md`
- 1–2 feature patterns → `landing-design/pattern-library/features.md` (never 3 identical cards — see rule #3)
- 0–1 social proof pattern → `landing-design/pattern-library/social-proof.md` (only if it earns its place)
- 1 CTA pattern → `landing-design/pattern-library/cta.md`
- 1 footer pattern → `landing-design/pattern-library/footer.md`
- 0–N scroll/motion patterns → `landing-design/pattern-library/scroll-motion.md` (only if your Direction calls for scroll choreography; a calm/quiet direction skips these entirely)

A typical landing page reaches for 4–5 of these files. Don't load all 7. Adapt each pattern's content and visual tokens to serve your Direction. **The pattern is the structure; your Direction is the soul.**

The installed landing sections are an alternate skeleton, but they contain known semantic-token violations. The gate will find them; fix rather than suppress every hit.

### 5. Fill images, run the grep gate

- Generate atmospheric images through a cataloged image integration; inspect its YAML payload and include `no text, no words, no letters, no writing, no logos` in every prompt.
- Persist generated images via `useR2Files` if you need a stable URL (otherwise the image regenerates on every render).
- Build product mockups as animated React components (inline SVG, styled divs, Framer Motion). **Never** use AI-generated images for UI screenshots.
- Fill every image slot before finishing.
- Run the complete gate in `landing-design/anti-ai-checklist.md` from the app root. Any hit is a bug to fix.

---

## Hard rules (non-negotiable)

These are grep-checkable. Ship violations = broken landing page.

1. **Hero headline**: 3–8 words. No exceptions.
2. **Body copy**: under ~150 words total across the whole page. If you're writing a paragraph, build a visual instead.
3. **No 3-identical-cards pattern** for features. If the features section renders three of the same thing with the same structure, redesign it (tabs, alternating rows, bento grid, single showcase, etc.).
4. **No purple→indigo, violet, or blue→purple gradients.** This is the most-common AI color tell. Use your accent color in `primary` shades only.
5. **No hardcoded colors in JSX.** No hex, no `violet-400`, no `indigo-500`, no `rgb()` or `rgba()`. Semantic tokens only: `bg-background`, `text-foreground`, `bg-primary`, `bg-muted`, `border-border`, `text-muted-foreground`, etc.
6. **No fractional-opacity on foreground.** Patterns like `bg-foreground/[0.06]` or `border-foreground/[0.08]` are old-template tells. Use `bg-muted`, `border-border`, `bg-card`.
7. **Pick a font that's clear, elegant, and fits the product — never gimmicky.** Reason about the product's tone first, then pick a font that serves it. Inter, Montserrat, DM Sans, Manrope, Fraunces, Playfair Display, Cormorant, JetBrains Mono are all valid headline picks when they fit. **Avoid display fonts that read as costume:** Syne, Bebas Neue, Anton, Fjalla One, Oswald, Impact, Josefin Sans, Pacifico, Lobster. The test: would a thoughtful designer ship this font for *this* product? See `landing-design/style-tile.md`.
8. **Product mockups must be React components**, not AI-generated images.
9. **Every AI-generated image prompt must include** `no text, no words, no letters, no writing, no logos`.
10. **Dramatic type scale.** Headlines should be at least 3× the size of body text. If they look similar, the page looks flat.
11. **One commanding visual in the hero.** An animated React mockup, a full-bleed atmospheric image, or a bold environment — something. A hero that is just text on a flat background is a failure.
12. **Never ship the scaffolded `landing` feature sections verbatim.** The scaffold is a skeleton. Shipping `HeroSection` + `FeaturesGridSection` + `TestimonialsSection` + `FAQSection` with placeholder copy swapped in reproduces the exact AI-generic look this skill is designed to break.
13. **Animations must respect `prefers-reduced-motion`.** Wrap the landing tree in `<MotionConfig reducedMotion="user">` from framer-motion — that auto-disables transform/layout animations for users who request reduced motion. Manual gates are needed only for `useTransform` from `useScroll` (parallax, pinned scroll), `setInterval` / `setTimeout` / `requestAnimationFrame` loops, and CSS keyframes. Call `useReducedMotion()` and short-circuit those.
14. **No pictograph emojis (🚀 ✨ 💡 🎉 ⭐ 🔥 ❤️ 👋 etc.).** They render inconsistently across platforms and read as AI-generated. Use `lucide-react` icons or inline SVG. Plain typographic marks (`✓ ✗ → ← ↑ ↓ ★`) ARE allowed as text glyphs.

See `landing-design/anti-ai-checklist.md` for expanded rationale on every rule + the complete grep gate commands.

---

## Pre-commit gate

Run the single complete command block and visual checklist in `landing-design/anti-ai-checklist.md`; do not maintain a second copy here.

---

## Reference files (load on demand)

- `design-direction.md` — fill the brief or repair a generic one.
- `style-tile.md` — scan only the decision table you need.
- `inspiration-gallery.md` + `examples/0N-*.tsx` — select and read exactly one archetype; examples are read-only.
- `pattern-library.md` + its section files — choose only the 4–5 sections you need.
- `anti-ai-checklist.md` — run before finishing.
