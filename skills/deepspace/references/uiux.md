# UI/UX Polish Guide

Load for the dynamic home, theme, primitives, interaction feedback, or generic-design feedback. Use `landing-design.md` instead for a marketing page.

The scaffold UI and themes are placeholders, not a house style. Design from the product's layout, typography, density, and tone. The copilot template's sidebar/main/chat-dock structure stays, but its content and theme still need product-specific design.

---

## 1. Home Page / First-Run State

The scaffold has two front-of-house pages:

- `src/pages/index.tsx` — the **static landing** at `/`. It lives at the top level of `src/pages/`, so no DeepSpace providers mount: no auth fetch, no WebSocket, and no data hooks. It's the marketing front door; design it with `references/landing-design.md`, not this procedure.
- `src/pages/(app)/home.tsx` — the **dynamic home** at `/home`, inside auth/record providers (including signed-out `allowAnonymous`). To put this surface at `/`, use `(app)/index.tsx` after removing the static landing; top-level `index.tsx` cannot use data hooks.

Replace the `home.tsx` stub rather than extending it. Any `placeholder page` or `Your app goes here` hit means the home is unfinished.

**Build the home page with this decision procedure — in order, no skipping:**

1. **Name the primary surface** (board, list, feed, document). That surface—not a poster describing it—is home.

2. **Pick the home skeleton by what the product is, and declare it.** The skeletons:
   - `product-preview-first` — the real primary surface, rendered with sample/preview data for visitors
   - `data-forward` — a dashboard/grid of live numbers, statuses, streaks above the fold
   - `search-first` — a search bar + the list, single column
   - `split-hero` — one-line pitch on one side, the live product surface on the other
   - `single-column-narrative` — for content/reading apps

   Write the declaration as the **first line of `src/pages/(app)/home.tsx`, before any code**, then make the JSX agree with it:

   ```tsx
   /* home pattern: data-forward — today's habit grid above the fold */
   ```

   The §5 gate requires this comment. Pick for the product, not implementation convenience; use the landing pattern library only for section-level structure.

3. **Signed-in home = the primary surface**, above the fold, with the user's real data.

4. **Signed-out home = the same surface in preview form** with sample/read-only data and an inline sign-in CTA; never an empty auth gate or icon/H1/button poster.

5. **Content bar:** app-specific H1, one-sentence purpose, primary action above the fold, and an actionable `EmptyState` for signed-in users with no data.

Known AI tells to avoid: "centered hero + three icon-title-description cards" (see `landing-design/pattern-library/features.md`) and its minimal cousin "centered icon badge + H1 + tagline + single CTA." Both read as template output regardless of theme.

---

## 2. Theme — Create One for the App

`slate` (`src/styles.css`) and `paper` (`src/themes.css`) are rendering examples, not product themes. Replace them before first deploy.

Themes are `[data-theme="<id>"]` CSS blocks overriding the shadcn tokens, activated via `<html data-theme="...">` in `index.html`. Switching is one attribute change; no JS, no FOUC. **This is the retheming surface for 95% of cases — not `DeepSpaceThemeProvider`.**

### The standard path

1. **Design the product palette** — background, foreground, card, primary (+foreground), secondary, muted, accent, border, ring. If unspecified, choose and state a one-line rationale.
2. **Add a theme block** — copy the `paper` block in `src/themes.css`, rename the selector, set your colors. Light themes must keep `color-scheme: light;` so native form controls match.
3. **Register it** — add an entry to the `THEMES` array in `src/themes.ts` (type safety + catalog), then set `data-theme="<your-id>"` in `index.html`.
4. **Shape** — set `--radius` smaller for sharp/technical or larger for soft/friendly.
5. **Update `<title>`** in `index.html` and replace the favicon. The defaults say "DeepSpace App".
6. **Wordmark & nav** — rebuild starter `Navigation.tsx` freely; restyle but retain the copilot `AppSidebar` shell and fixed-icon collapse. Preserve sign-in/out, `src/nav.ts` links, and test ids `app-navigation`, `nav-sign-in-button`, `nav-user-name`.

Set at least background, foreground, card, primary, secondary, accent, and ring. Edit the baseline `@theme` block in `styles.css` only when intentionally replacing the default rather than adding a theme.

### Shadows caveat

Tailwind v4's `@theme` bakes baseline shadow values into compiled utilities, so runtime `[data-theme]` overrides of `--shadow-*` tokens can't fully cancel them. For per-theme shadows on your own components, use literal arbitrary classes (`shadow-[0_2px_8px_0_rgba(0,0,0,0.08)]`) or scope a small utility under your `[data-theme]` block, and verify in the browser that the shadow changes when you switch themes.

### When to use `DeepSpaceThemeProvider` / `applyDeepSpaceTheme` instead

These are exported from `deepspace` (the root package — there is no `deepspace/theme` subpath) and drive `--theme-*` CSS variables consumed by cross-app / deployed DeepSpace components (pills, directory panels, mini-apps). They read from `--color-*` by default (`readThemeFromDOM`), so the token setup is usually enough and they just follow. Reach for them explicitly only when embedding DeepSpace surfaces on a deployed site or mini-app that needs a different theme from the main app.

### UI dark/light mode

Light themes set `color-scheme: light` inside the theme block, so native form controls (calendar icons, scrollbars) match. The SDK also reads `data-ui-theme="dark" | "light"` on `<html>` to switch between `UI_TOKENS_DARK` and `UI_TOKENS_LIGHT` (see `applyUIThemeTokens`) — set this if the app supports a light/dark toggle distinct from the theme picker.

---

## 2a. No Emojis in UI Chrome

Do not use emojis as app chrome (titles, nav, buttons, empty states, headers).

Allowed emoji contexts:
- **User-authored content** — messages, comments, posts. Users type what they type.
- **Message reactions** — the reaction picker itself (👍 🎉 ❤️ etc. as the selectable set).
- **The user explicitly asks for emojis** ("add a grocery emoji to the header").

Otherwise use `lucide-react`, inline SVG, or text. Build wordmarks through font, weight, tracking, and case.

## 3. UI Primitives — Use the Scaffolded Base UI Kit, Never Browser Defaults

The scaffold ships a copy-paste primitives kit in `src/components/ui/` (index at `src/components/ui/index.ts`), built on **Base UI** (`@base-ui/react` — headless, from the Radix/Floating-UI/MUI team) and styled entirely with the app's semantic theme tokens. The components are the app's own files — restyle or extend them freely; their *look* follows the theme tokens automatically. Overlay positioning, focus trapping, select label rendering, and nested-dialog stacking are already correct — do not hand-roll replacements, and never use browser-default controls (they ignore theme tokens and render as native widgets).

| Use case | Use this | Don't use |
|---|---|---|
| Select one of N options | `Select` + `SelectTrigger`/`SelectValue`/`SelectContent`/`SelectItem` | `<select>` / `<option>` |
| Menu / overflow / "…" actions | `DropdownMenu` + `Trigger`/`Content`/`Item` (+ `CheckboxItem`, `RadioItem`, `Separator`, `Sub`) | hacked `<select>`, raw `<ul>` dropdown |
| Confirm ("Are you sure?") | **`ConfirmModal`** (dedicated confirmation primitive) | `window.confirm()` |
| Modal dialog | `Modal` (simple controlled: `open`/`onClose`, `Modal.Header/Body/Footer`) or the `Dialog` family (`DialogTrigger`/`DialogContent`/… for triggers, nesting, custom composition) | positioned `<div>` hacks |
| Prompt for a string | `Modal` (or `Dialog`) with an `Input` inside | `window.prompt()` |
| Alerts / info banners | `useToast` for transient; inline token-styled banner (`border border-border bg-card` + lucide icon) for persistent | `window.alert()` |
| Success/error toast feedback | app-local `useToast` — `success()` / `error()` / `warning()` / `info()` | `alert()`, inline console text, silent mutations |
| Empty lists / no data | `EmptyState` (icon + title + description + action) | raw "No items" text |
| Loading placeholders | `animate-pulse` divs on `bg-muted` sized like the content; `Button loading` for pending actions | blank screens, hand-rolled CSS spinners |
| Form fields | `Input`, `Textarea`, `Label`, `Checkbox`, `Switch` | raw HTML equivalents |
| Search box | `SearchInput` (wraps `Input` with search icon + clear) | raw `<input type="search">` |
| Tabs | `Tabs`, `TabsList`, `TabsTrigger`, `TabsContent` | hand-rolled tab buttons |
| Anchored popups | `Popover`, `PopoverTrigger`, `PopoverContent` | absolutely-positioned divs |
| Tooltips | `Tooltip`, `TooltipTrigger`, `TooltipContent` — one app-level `TooltipProvider` (already mounted in `_app.tsx`, 200ms) owns the delay and groups nearby triggers so they switch instantly. Don't wrap individual tooltips in their own provider — a nested provider shadows the app-level one and breaks the grouping. Pass `delay` on a single `Tooltip` for one-off timing | `title=""` attribute |
| Avatars | `Avatar`, `AvatarImage`, `AvatarFallback` | raw `<img>` |
| Status pills | `Badge` | hand-rolled rounded divs |
| Cards / tables / separators | No primitive — token-styled elements (`rounded-lg border border-border bg-card p-4`; styled `<table>` with `border-border` rows; `border-t border-border`) | hardcoded colors |

**Critical import rule:** use the scaffold's local `src/components/ui`, not `deepspace`; the SDK does not export this app-local kit. Keep hooks such as `useToast` paired with the local providers mounted by `_app.tsx`.

**`useToast` is the default feedback channel** for any mutation. `const { success, error, warning, info } = useToast()` then:
- `success('Saved', 'Your changes have been saved.')` after the mutation resolves. Plain `create` / `put` / `remove` resolve optimistically (before the server accepts), so use the `*Confirmed` variant before toasting success on anything the user must trust.
- `error('Failed to save', err.message)` in the `catch` — **only `*Confirmed` variants throw on a server denial**. Plain mutations never hit the catch; their rejections surface through `RecordProvider`'s `onWriteError` toasts, already wired in the scaffold's `(app)/_layout.tsx`.
- No silent mutations — the user should always see confirmation.

### Base UI gotchas (already handled in the kit — don't undo them)

- **Custom trigger elements use the `render` prop, not `asChild`:** `<DialogTrigger render={<Button>Open</Button>} />`. The kit's `Button` is a native `<button>` and works as a `render` target.
- **`SelectValue` label rendering** — the kit derives an items map from its `SelectItem` children so the trigger shows the *label* (not the raw value) even before the popup ever opens. Render `SelectItem`s inline (direct children / `.map(...)` / fragments) — items inside your own wrapper component are invisible to the walk; pass `items={{ value: 'Label' }}` explicitly instead. Option values must be non-empty strings (`''` is the cleared state).
- **Nested dialogs** — the kit passes `forceRender` on backdrops so a modal-in-modal deepens the scrim. Opening a `Dialog` from inside a `Modal` just works.
- **Tabs active state** styles via `data-active` (not `data-selected`).
- **Open/close animations** depend on the custom `animate-in`/`animate-out` utilities in `src/styles.css` (with `animation-fill-mode: both`). Don't remove that block; components animate via `data-[open]`/`data-[closed]`.

---

## 3a. Prop Shapes You'll Otherwise Forget

**`Button`** — has a built-in `loading` prop. Do not hand-roll `{pending && <Spinner />}` + `disabled={pending}`:
```tsx
<Button loading={creating} onClick={handleCreate}>Create</Button>
// variants: 'default' | 'destructive' | 'outline' | 'secondary' | 'ghost' | 'link'
// sizes:    'default' | 'sm' | 'lg' | 'icon'
```

**`ConfirmModal`** — dedicated confirmation; use it instead of composing a dialog + footer + two buttons:
```tsx
<ConfirmModal
  open={confirmOpen}
  onClose={() => setConfirmOpen(false)}
  onConfirm={handleDelete}
  title={`Delete task '${task.title}'?`}
  description="This cannot be undone."
  confirmText="Delete"        // default 'Confirm'
  cancelText="Cancel"         // default 'Cancel'
  variant="destructive"       // default 'destructive' — pass 'default' for non-destructive confirms
  loading={deleting}
/>
```

**`EmptyState`**:
```tsx
<EmptyState
  icon={<Inbox />}
  title="No tasks yet"
  description="Create your first task to get started."
  action={{ label: 'New task', onClick: openCreate }}
  secondaryAction={{ label: 'Import', onClick: openImport }}  // optional
/>
```

**`AuthOverlay`** — render without `onClose` and gate with `!isSignedIn`. Returns `null` automatically when signed in or still loading:
```tsx
<AuthOverlay providers={['google', 'github']} />  // providers optional — defaults to both
```

**`useToast`** — four-level API, plus a generic `toast({ type, title, description, duration })`:
```tsx
const { success, error, warning, info, toast, dismiss, dismissAll } = useToast()
success('Saved', 'Changes saved successfully.')
error('Upload failed', err.message)
```

---

## 4. Interaction Polish (free wins)

- **Every async action** (mutate, upload, send): `Button loading={pending}` — it disables and shows the spinner. Use optimistic UI where the collection supports it.
- **Every destructive action** (delete, remove, leave): `ConfirmModal` that names the item in the body ("Delete task 'Buy milk'?"), not a generic "Are you sure?".
- **Every mutation**: follow the `useToast` and confirmed-write rules above.
- **Every form**: inline validation next to the field, not a global banner. Use `Label` + `Input` + a small `<p>` with the error.
- **Every list during initial load**: `animate-pulse` placeholder blocks (`bg-muted rounded-md`) shaped like the content. Never a blank screen with just "Loading…".
- **Hover/focus states** on every clickable element — the primitives handle this; raw `<div onClick>` does not. The scaffold also ships `*:focus-visible` outlines in `styles.css` — keep them.
- **Keyboard accessibility**: `Dialog`/`Modal`, `DropdownMenu`, `Select` all handle Esc/arrow keys + focus trapping; roll-your-own usually doesn't.

---

## 5. Verify With a Smoke Test

After customizing home, theme, and primitives, extend `smoke.spec.ts`:
- Home page renders the **real** H1 (assert the app-specific title, not the app id or "Welcome").
- The placeholder copy is **not** in the DOM (assert absence of "placeholder page").
- Primary CTA is visible and clickable.
- At least one real primitive opens on interaction (e.g., a `DropdownMenu` opens; clicking Delete opens the `ConfirmModal`).
- Page `<title>` is app-specific, not "DeepSpace".
- Spot-check a mutation and assert a toast appears.
- Keep the nav test hooks intact: `app-navigation`, `nav-sign-in-button`, `nav-user-name`.

Before declaring done, run both halves:

```bash
# Half 1 — ABSENCE: any hit below means the app is NOT ready
grep -rn "<select\|window\.confirm\|window\.alert\|window\.prompt" src/
grep -rn "placeholder page\|Your app goes here" src/
grep -rn 'data-theme="slate"' index.html

# Half 2 — PRESENCE: any MISS below means the home page is NOT done
grep "home pattern:" 'src/pages/(app)/home.tsx'   # the §1 skeleton declaration, first line (quote the path — parentheses)
grep 'data-theme="' src/themes.css                # your own theme block exists
```
