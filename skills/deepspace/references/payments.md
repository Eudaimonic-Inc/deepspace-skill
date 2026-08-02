# Payments — subscriptions, one-time products, refunds, cancellation

Load this reference for any money flow or before touching the payment hooks/helpers. Do not install Stripe libraries: declare the catalog in app manifests and use the SDK. Developer Connect onboarding lives only in the DeepSpace `/earnings` dashboard; apps may sell before onboarding, but payout waits for it.

## 1 — Declare what you sell

Two manifest files in the app, both synced by `deepspace deploy`:

### `src/subscriptions.ts` — recurring plans

```ts
export const subscriptionPlans = [
  { slug: 'free', name: 'Free', priceCents: 0 },
  {
    slug: 'pro',
    name: 'Pro',
    priceCents: 900,       // $9/mo — minimum $3/mo ($300)
    yearlyCents: 9000,     // optional; minimum $12/yr ($1200)
    trialDays: 7,          // optional free trial; max 365
    taxCode: 'txcd_10000000', // optional, defaults to digital services
  },
] as const
```

- Keep `slug` stable; rename `name`, not `slug`, because subscribers and gates reference it.
- `priceCents: 0` = free tier, never hits Stripe.
- Minimums exist because Stripe's per-charge fee ($0.30 + 2.9%) consumes sub-dollar charges.

### `src/products.ts` — one-time products

```ts
export const oneTimeProducts = [
  { productId: 'pro_unlock', name: 'Pro Unlock', amountCents: 1999, description: '…' },
] as const
```

- `productId` is the entitlement key. Use the same string in `useCheckout({ productId })`.
- Min `amountCents: 100`. Dropping a row deactivates it; existing purchases stay valid.

After editing either file: `deepspace deploy`. The CLI creates/updates Stripe Products + Prices and warns about grandfathered subscribers on price/slug changes.

## 2 — Subscriptions (recurring)

### Client: `useSubscription()`

```tsx
import { useSubscription } from 'deepspace'

function Paywall() {
  const sub = useSubscription()
  if (sub.isLoading) return null
  if (sub.isAtLeast('pro')) return <ProUI />
  return <button onClick={() => sub.subscribe('pro')}>Upgrade</button>
}
```

| Member | What it returns |
|---|---|
| `tier` | Current plan slug. Free-tier users return `'free'`. |
| `status` | `'none' \| 'trialing' \| 'active' \| 'past_due' \| 'canceled' \| …` |
| `entitled` | `true` iff status ∈ `{active, trialing}` (or free tier). |
| `hasTier(slug)` | Strict slug match **AND** entitled. |
| `isAtLeast(slug)` | Rank ≥ target slug **AND** entitled. |
| `interval` | `'month' \| 'year' \| null`. |
| `currentPeriodEnd`, `cancelAtPeriodEnd`, `trialEndsAt` | Read-only state. |
| `plans` | Plan catalog for `<PricingTable>`; its required `onSelect(slug, interval)` should call `subscribe`. |
| `subscribe(slug, { interval?, returnUrl?, cancelUrl? })` | Starts Checkout. Auto-redirects. |
| `openPortal(returnUrl?)` | Stripe Billing Portal for self-service (change card, cancel). |
| `refresh()` | Re-pull `/me`. |

**Gate on `hasTier`/`isAtLeast`, never on `tier` alone** — a `past_due` Pro user keeps `tier === 'pro'` while losing entitlement.

### Server: `requireSubscription`

```ts
import { requireSubscription, SubscriptionAuthError, SubscriptionRequiredError } from 'deepspace/server'

app.get('/api/premium', async (c) => {
  try {
    await requireSubscription(c, { atLeast: 'pro' }) // or { tier: 'pro' }
  } catch (e) {
    if (e instanceof SubscriptionAuthError)     return c.json({ error: 'unauthenticated' }, 401)
    if (e instanceof SubscriptionRequiredError) return c.json({ error: 'upgrade_required', required: e.required }, 402)
    throw e
  }
  // ... protected logic
})
```

The helper forwards the inbound `Authorization` header. The browser must attach it:

```ts
import { getAuthToken } from 'deepspace'
fetch('/api/premium', { headers: { Authorization: `Bearer ${await getAuthToken()}` } })
```

`getSubscription(c)` is the read-only variant returning the same shape.

## 3 — One-time products & ad-hoc charges

One hook, two modes:

```tsx
import { useCheckout } from 'deepspace'

// Product mode — entitlement-safe. Amount/name resolved server-side from products.ts.
const co = useCheckout({ productId: 'pro_unlock' })
if (co.owned) return <ProUI />
return <button onClick={() => co.chargeOnce({ productId: 'pro_unlock' })}>Buy</button>

// Ad-hoc mode — tips, donations. Caller picks amount.
co.chargeOnce({ amount: 500, name: 'Tip', description: '…' })
```

| Member | What it returns |
|---|---|
| `chargeOnce(opts)` | Starts Checkout. Auto-redirects. |
| `purchases` | Full purchase history. |
| `owned` | `true` iff the hook's `productId` was supplied **and** a non-refunded matching purchase exists. |
| `ownsProduct(id)` | Pure check over `purchases` for any productId. |
| `refresh()` | Re-pull purchase list. |

**Ad-hoc charges have `productId: null` and cannot be used to gate features** — use them for revenue collection only. Only product-mode charges are trustworthy with `owned`/`ownsProduct`.

## 4 — Cancellation (server)

```ts
import { cancelSubscription, CancelSubscriptionError } from 'deepspace/server'

// One user, end of current period (default):
await cancelSubscription(c, { userId: 'user_abc' })

// Everyone on a retired plan:
let res = await cancelSubscription(c, { planSlug: 'retired_pro', atPeriodEnd: true })
while (res.hasMore) res = await cancelSubscription(c, { planSlug: 'retired_pro' })
```

- Requires the inbound `Authorization` header (caller JWT). Platform verifies the JWT subject owns the app.
- Pass `atPeriodEnd: false` for an immediate cancel (handle refund separately if needed).
- Batched at 50; loop on `hasMore`. The flag is idempotent.
- Local state reconciles via Stripe webhook — read-back may lag a beat.

## 5 — Refunds (server)

```ts
import { refundInvoice, RefundError } from 'deepspace/server'

app.post('/api/admin/refund', requireMyAdmin, async (c) => {
  const r = await refundInvoice(c, {
    invoiceId,            // local UUID, NOT stripe inv_xxx
    amount: 500,          // optional partial in cents; full refund if omitted
    reason: 'requested_by_customer', // or 'duplicate' | 'fraudulent'
  })
  return c.json(r)
})
```

- Forwards caller JWT; platform rejects with `not_app_owner` (403) for non-owners. **Still gate the route in your own admin check.**
- Constraints: 90-day window from `paidAt`, 50/24h per app, no overdraw.
- Dashboard-initiated refunds reconcile automatically.

## 6 — Before shipping

- Use `trialDays` for trials, `subscribe(slug, { interval: 'year' })` for annual checkout, and `{ returnUrl, cancelUrl }` to override redirect defaults. Wire pricing with `<PricingTable plans={sub.plans} currentTier={sub.tier} onSelect={(slug, interval) => sub.subscribe(slug, { interval })} />`.
- Client requests to gated/admin routes must attach `Authorization: Bearer <jwt>`; server helpers forward but never mint it.
- Gate with `hasTier` / `isAtLeast`, not `tier`; ad-hoc charges have no durable entitlement. These boundaries are described at their APIs above.
- Catalog minimums ($3/month, $12/year, $1 one-time) are enforced during deploy sync.
- Checkout return can precede webhook reconciliation by about 1–2 seconds. Refresh once, then retry on a short delay or user action; never tight-loop.
- A `taxCode` applies to the whole plan, not separately to monthly and annual prices. Use separate plans when tax treatment differs.
