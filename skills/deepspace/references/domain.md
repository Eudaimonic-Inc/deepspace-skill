_Load when buying, attaching, or managing a custom domain with `deepspace app domain`._

# Custom domains

```bash
npx deepspace app domain search <query> [--limit N]
npx deepspace app domain buy <domain> [--app <app-id-or-live-name>]
npx deepspace app domain list
npx deepspace app domain status <domain>
npx deepspace app domain attach <domain> --app <app-id-or-live-name>
npx deepspace app domain detach <domain> --yes
npx deepspace app domain renew <domain> --auto on|off
```

Use `--json` for structured results and command help for current flags. For `buy` and `attach`, `--app` accepts the immutable app id or current live name; without it, the CLI resolves `DEEPSPACE_APP_ID` from the surrounding app. Names are URL labels, not durable identity, so scripts should prefer the id.

## Purchase flow

`buy` rechecks availability and price, creates a Stripe Checkout session, then polls provisioning. This spends money and enables auto-renew:

1. Show the domain, annual charge, renewal posture, and target app to the user.
2. Obtain explicit approval before passing `--yes`.
3. Keep the command in the foreground while the user completes Checkout. Do not impose a timeout.

Use `--no-open` to print the Checkout URL or `--no-wait` for an intentional handoff; continue later with `domain status`. JSON mode returns the checkout session without polling and supplies that status action.

Cloudflare-registrar domains commonly provision in about 60–90 seconds; the non-Cloudflare registrar path can take 15–60 minutes for nameserver propagation. Interrupting local polling does not cancel server-side provisioning.

## Other mutations

- `attach` re-points an owned registration to the resolved immutable app id. Renames do not require reattachment.
- `detach` stops routing but keeps the registration and auto-renew setting; `attach` can restore routing.
- `renew --auto off` changes billing posture but does not detach or release the domain.

Treat purchase and renewal changes as user decisions. Do not automate `buy` in CI or tests, and do not use a real owned domain for routine tests. Non-interactive purchase/detach refuses without `--yes`; never add that flag without the corresponding approval.
