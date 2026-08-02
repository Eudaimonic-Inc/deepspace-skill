# Pattern library — index

Copy-pasteable TSX by section. Load only the section files you need; snippets assume placement in `src/pages/index.tsx`, so adjust relative imports if extracting components.

## Pick your sections, then load only those files

| Section | How many | Section file |
|---|---|---|
| Navigation | 1 (or 0 — some pages need none) | `pattern-library/nav.md` |
| Hero | 1 | `pattern-library/hero.md` |
| Features | 1 (sometimes 2) | `pattern-library/features.md` |
| Social proof | 0–1 (only if you have real proof) | `pattern-library/social-proof.md` |
| CTA | 1 | `pattern-library/cta.md` |
| Footer | 1 | `pattern-library/footer.md` |
| Scroll & motion | 0–N (only if your Direction calls for it) | `pattern-library/scroll-motion.md` |

Labels such as `N1`, `H1`, and `F1` are stable identifiers. Choose after committing to a Direction.

## How patterns integrate with DeepSpace

- Clean primitives: `Typewriter`, `ScrollReveal`, `StaggerContainer`, `staggerChild`, `AnimatedStat`, `cn`, `motion`, `AnimatePresence`, `useInView`, `ChevronDown`.
- `GlassCard`, `PlaceholderImage`, `BrowserMockup`, and `SectionHeading` contain known gate violations. Prefer inline semantic surfaces or repair `primitives.tsx`.
- CTAs navigate to public `/home`; target a route under `(app)/(protected)/` when sign-in is required. No “landing seen” storage flag exists.

## Universal rules

- Use semantic tokens; replace every `TODO`; use icons/SVG rather than pictograph emoji.
- Wrap the tree in `<MotionConfig reducedMotion="user">`; manually gate scroll transforms, loops, and CSS keyframes.
- Run `anti-ai-checklist.md` after composition.

## After you compose

Eyeball-check that the page serves its Direction. If it does not, revise the Direction or composition rather than adding more patterns.
