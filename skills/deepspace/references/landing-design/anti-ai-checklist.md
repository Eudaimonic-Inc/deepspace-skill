# Anti-AI Checklist + Pre-commit Grep Gate

Read before finishing. `landing-design.md` defines the rules; this file gives compact remediation and the canonical gate.

## Rule remediation

| # | Violation | Fix |
|---|---|---|
| 1 | Hero headline exceeds 3–8 words | Shorten; put explanation below. |
| 2 | Page body exceeds ~150 words | Replace explanation with a visual. |
| 3 | Three identical feature cards | Use tabs, alternating rows, bento, a showcase, or a typographic list. |
| 4 | Purple/indigo/violet gradient | Use semantic `primary` shades from the app theme. |
| 5 | Hex, RGB, or named Tailwind colors in JSX | Use semantic tokens (`background`, `foreground`, `primary`, `muted`, `card`, `border`). |
| 6 | `*-foreground/[opacity]` | Use `bg-muted`, `text-muted-foreground`, `border-border`, or `bg-card`. |
| 7 | Costume-like display font | Pick for product tone and readability; use `style-tile.md` pairings. |
| 8 | AI-generated product mockup | Build the UI in React/inline SVG; generated images are atmospheric only. |
| 9 | Generated-image prompt lacks a negative-text clause | Add `no text, no words, no letters, no writing, no logos`. |
| 10 | Flat type scale | Make the headline at least 3× body size. |
| 11 | Text-only hero | Add one commanding React mockup, atmospheric image, or signature element. |
| 12 | Scaffold/example cloned | Rewrite installed sections; examples are read-only teaching artifacts. |
| 13 | Motion ignores reduced-motion | Wrap with `<MotionConfig reducedMotion="user">`; manually gate `useTransform`, timers/RAF, and CSS keyframes. |
| 14 | Pictograph emoji in JSX | Use Lucide, inline SVG, or typographic marks (`✓`, `→`, `★`). Eyeball BMP emoji the gate misses. |

The installed `LandingPage.tsx` and `primitives.tsx` contain known rule 5/6 violations. Treat their gate hits as bugs in the app copy, not false positives.

## The grep gate — run before finishing

From the app root (e.g., `cd ~/Desktop/Work/<app-name>`), scope to landing files:

```bash
# ── Hardcoded colors — should never appear ───────────────────────────────────
grep -rnE "#[0-9a-fA-F]{6}|#[0-9a-fA-F]{3}\b" --include="*.tsx" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/ 2>/dev/null
grep -rnE "rgba?\([0-9]" --include="*.tsx" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/ 2>/dev/null
grep -rnE "\b(violet|indigo|purple|fuchsia|rose|amber|emerald|teal|cyan|sky|blue|green|red|orange|yellow|lime|pink)-[0-9]{3}" --include="*.tsx" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/ 2>/dev/null

# ── Fractional-opacity foreground patterns ───────────────────────────────────
grep -rnE "(bg|text|border)-foreground/(\[|[0-9])" --include="*.tsx" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/ 2>/dev/null

# ── Continuous animations — advisory; review each for a useReducedMotion gate
grep -rnE "repeat:\s*Infinity|setInterval\(|requestAnimationFrame\(" --include="*.tsx" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/ 2>/dev/null

# ── Pictograph emojis (Unicode 1F000-1FFFF). Plain marks like ✓ ✗ → ★ are NOT
# caught (they're in the BMP < 1F000) and are allowed as text glyphs.
# Needs PCRE: use `rg "[\u{1F000}-\u{1FFFF}]" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/` if `grep -P` is unavailable.
grep -rnP "[\x{1F000}-\x{1FFFF}]" --include="*.tsx" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/ 2>/dev/null

# ── Template placeholder copy + generic marketing phrases ────────────────────
grep -rniE "My App|Welcome to [Mm]y|[Ll]orem [Ii]psum|Your DeepSpace app is running" --include="*.tsx" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/ 2>/dev/null
grep -rniE "streamline your|transform your|cutting.edge|state.of.the.art|next.generation|revolutionary|world.class|best.in.class|game.chang" --include="*.tsx" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/ 2>/dev/null

# ── Unfilled TODOs ───────────────────────────────────────────────────────────
grep -rnE "TODO[: ]" --include="*.tsx" src/pages/index.tsx 'src/pages/(app)/landing.tsx' src/components/landing/ 2>/dev/null

# ── Illegal import from landing-design/examples/ (read-only reference) ──────
grep -rn "from.*landing-design/examples" --include="*.tsx" src/ 2>/dev/null
```

**Any hit is a bug to fix before shipping.** If the grep gate is clean but you're still unsure, run the eyeball checks:

- [ ] Design Direction block (prose, not placeholders) is present at the top of the landing page file
- [ ] Hero headline is 3–8 words
- [ ] Hero has a commanding visual (mockup, atmospheric bg, signature element), not just centered text
- [ ] Features section is not 3 identical cards
- [ ] Every image slot is filled (real `integration.post('freepik|openai|gemini/generate-image...')` URL, uploaded R2 asset, or a code-based React visual)
- [ ] The page doesn't look like a generic purple-gradient SaaS landing page
- [ ] The scaffolded landing sections have been replaced or substantively rewritten — not shipped as-is

If any eyeball check fails, the bug isn't in the grep gate — it's in the design. Go back to the Design Direction block and check: does the code actually serve the direction you wrote? Usually the answer is "the direction is too vague" or "I drifted from my own direction."
