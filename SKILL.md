---
name: payment-gateway
description: >-
  Provider-agnostic PaymentGateway port so checkout, orders, and UI never couple
  to a PSP. Use when implementing, swapping, or adding any payment provider
  (Payunit, Stripe, IAP, Mobile Money), or when the user mentions
  payment adapter, passerelle de paiement, or changer de moyen de paiement.
---

# PaymentGateway port (swap providers without a rewrite)

Checkout, orders, entitlements, and UI talk **only** to a `PaymentGateway`
**port**. Payunit, Stripe, Apple/Google IAP, etc. are **adapters**.

**Switching a provider = new adapter + config flip.** Never rewrite the paywall,
order fulfillment, or client payment API.

Provider skills (`payunit-integration`, …) **must**
apply this skill first. They must not wire their SDK/types into UI or domain.

Canonical types: [contract.md](contract.md)  
Adapter mappings + how to add a PSP: [adapters.md](adapters.md)

---

## Hard rules (non-negotiable)

1. **Domain and UI never import a PSP SDK, DTO, or status enum.**
2. Persist `provider` + `providerPaymentId` on every payment row.
3. Fulfill orders from a **canonical** `PaymentEvent`, not from raw PSP payloads.
4. Public app API is provider-neutral:
   - `POST /payments/charges`
   - `GET /payments/{id}`
   - `POST /webhooks/payments/{provider}` ← only place a provider name appears
5. Secrets stay on the **server**. Clients call **your** API (IAP store sheet is the exception).
6. Confirm money movement via **webhook and/or getStatus** — never UI/return-URL alone.
7. Fulfillment is **idempotent** on `(provider, providerPaymentId)`.

If a project already has this port: **add/replace an adapter only**.

---

## Step 0 — Scan, then reuse or create the port

1. Run the provider skill’s project-scan subagent (or equivalent).
2. Look for an existing `PaymentGateway` / `PaymentPort` / `PspAdapter`.
3. Branch:

| Scan result | Do this |
|-------------|---------|
| Port exists | Implement `{Provider}Adapter`. Register it. Do not touch checkout UI or `applyPaymentEvent`. |
| No port | Create the port + **one** adapter (the requested PSP). Wire DI/config. |
| Tight coupling to a PSP | Extract a port, wrap the old code as an adapter, then add the new one. |

Recommended layout (rename to match the repo):

```
payments/
  domain/          # port + canonical types + applyPaymentEvent
  application/     # createCharge / getStatus use-cases
  adapters/
    payunit/
    stripe/        # optional
    iap/           # optional
  http/            # provider-neutral routes + /webhooks/payments/:provider
```

---

## Port (implement in the project’s language)

```
PaymentGateway
  providerId: string
  capabilities() -> Capabilities
  createCharge(CreateChargeRequest) -> Charge
  getStatus(PaymentRef) -> Charge
  parseWebhook({ headers, rawBody }) -> PaymentEvent   // throw if invalid
  refund?(RefundRequest) -> Charge                     // if capabilities.refund
  payout?(PayoutRequest) -> Charge                     // if capabilities.payout
```

Full field list: [contract.md](contract.md).

`Capabilities` tell the UI what to show (phone field vs redirect vs store sheet)
**without** naming a PSP.

---

## Shared HTTP contract (client ↔ your server)

### `POST /payments/charges`

```json
{
  "orderId": "ord_123",
  "amountMinor": 10000,
  "currency": "XAF",
  "country": "CM",
  "method": "hosted",
  "customer": { "phone": "+2376…" },
  "operator": "mtn",
  "returnUrl": "https://app.example/pay/return",
  "idempotencyKey": "ord_123_attempt1"
}
```

`method`: `hosted` | `mobile_money` | `card` | `store_iap`

### Response

```json
{
  "paymentId": "pay_abc",
  "status": "awaiting_user",
  "nextAction": { "type": "redirect", "url": "https://…" }
}
```

`nextAction.type`: `none` | `redirect` | `await_user_on_phone` | `open_store_sheet`

Client: follow `nextAction`. Never branch on `provider`.

### `GET /payments/{id}` → same `Charge` shape

### `POST /webhooks/payments/{provider}`

1. Resolve adapter by `{provider}` (or 404).
2. `event = adapter.parseWebhook(headers, rawBody)` — verify signature here.
3. `applyPaymentEvent(event)` — **one** function for every PSP.
4. Return 200 quickly.

Keep old provider webhook URLs during a migration; both call `applyPaymentEvent`.

---

## Wiring / switching

```
PAYMENT_PROVIDER=payunit|stripe|iap
```

Factory (language-agnostic):

```
resolveGateway(config):
  switch config.PAYMENT_PROVIDER
    payunit -> PayunitAdapter(...)
    stripe  -> StripeAdapter(...)
    iap     -> StoreIapAdapter(...)
    else    -> error
```

Register **one** `PaymentGateway` in DI. Use-cases depend on the port, not a class.

### Checklist: switch A → B (hours, not a rewrite)

- [ ] `BAdapter` implements `PaymentGateway`
- [ ] Status + webhook mapped to canonical types ([adapters.md](adapters.md))
- [ ] Webhook route `/webhooks/payments/b` registered in the PSP dashboard
- [ ] `PAYMENT_PROVIDER=b`
- [ ] **Zero** checkout / order / entitlement file changes
- [ ] Sandbox charge + webhook + idempotent double-delivery tested

In-flight payments stay on their original `provider` row. `getStatus` / webhooks
use that row’s adapter, not the new default — keep adapters constructible by id.

```
gatewayFor(payment.provider)  // not only the default
```

---

## Fulfillment (one function)

```
applyPaymentEvent(event):
  payment = load by (event.provider, event.providerPaymentId)
            or event.orderId
  if payment.status already terminal: return  // idempotent
  map event.status -> payment.status
  if succeeded: fulfill order / grant entitlement once
  if failed|cancelled|expired: mark unpaid, allow retry with new idempotencyKey
```

Never fulfill from a redirect query string.

---

## State machine (UI + domain)

```
pending → awaiting_user → succeeded | failed | cancelled | expired
```

Adapters map PSP statuses into this set only. Unknown PSP status → keep
`pending` / `awaiting_user` and poll `getStatus`.

---

## UI rules

- Screens call `createCharge` / read `Charge.status` + `nextAction`.
- Show phone + operator **only if** `capabilities.directMobileMoney`.
- Show “Pay” redirect **only if** `capabilities.hostedRedirect`.
- Show store purchase **only if** `capabilities.nativeStoreSheet`.
- No “Pay with Payunit” copy unless the user asked for PSP branding.

---

## When a provider skill runs

1. Apply **this** skill (port).
2. Then apply the PSP skill for payloads, auth, sandbox numbers.
3. Implement `XxxAdapter` + webhook verification inside that adapter.
4. Map statuses using [adapters.md](adapters.md).
5. Do not add `/payments/payunit/...` or `/payments/stripe/...` as the **app**
   API. Those may exist as **private** adapter internals.

---

## Agent checklist

- [ ] Port exists (reused or created); UI/domain have **no** PSP types
- [ ] Adapter implements `createCharge`, `getStatus`, `parseWebhook`
- [ ] Payments table stores `provider` + `providerPaymentId`
- [ ] `applyPaymentEvent` is the only fulfillment path
- [ ] `PAYMENT_PROVIDER` (or DI) selects the default adapter
- [ ] Historical payments still resolve `gatewayFor(provider)`
- [ ] Webhook signature verified **in the adapter**
- [ ] Switch procedure documented in the PR/commit if a PSP was added

## Additional resources

- Canonical types: [contract.md](contract.md)
- Adapter cookbook (Payunit, IAP): [adapters.md](adapters.md)
- Payunit skill: `~/.cursor/skills/payunit-integration/SKILL.md`
