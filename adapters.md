# Writing and mapping adapters

An adapter is the **only** module allowed to know a PSP. It implements
`PaymentGateway` from [contract.md](contract.md).

Provider skills (`payunit-integration`, …) supply
payloads, auth, and sandbox data. This file supplies the **port mapping**.

---

## Adapter skeleton

```
class XxxAdapter implements PaymentGateway
  providerId = "xxx"

  capabilities(): map what this PSP can do in *this* project

  createCharge(req):
    assert capabilities allow req.method
    translate req → PSP API
    persist payment row (provider, providerPaymentId)
    return Charge with canonical status + nextAction

  getStatus(ref):
    load row → call PSP status API → map status → update row → return Charge

  parseWebhook(headers, rawBody):
    verify signature (fail closed)
    parse body
    optionally re-fetch PSP object (always, if the PSP does not sign webhooks)
    return PaymentEvent  // never mutate orders here
```

`applyPaymentEvent` lives **outside** the adapter (application/domain).

---

## Status maps

### Payunit → canonical

| Payunit `transaction_status` | Canonical |
|------------------------------|-----------|
| `PENDING` | `awaiting_user` (hosted/MoMo in progress) or `pending` |
| `SUCCESS` | `succeeded` |
| `FAILED` | `failed` |
| `CANCELLED` | `cancelled` |

`createCharge` (initialize): status `awaiting_user`, `nextAction.redirect` with
`transaction_url`. Custom MoMo (`makepayment`): `await_user_on_phone`.

Webhook: map `data.transaction_status`; `providerPaymentId` = `transaction_id`.  
Payunit does **not** document HMAC webhook signing — match `transaction_id` to a stored pending row; optionally re-fetch via `GET /paymentstatus/{id}` before fulfillment.

### Store IAP → canonical

| Store | Canonical |
|-------|-----------|
| purchase started | `awaiting_user`, `nextAction.open_store_sheet` |
| purchased / restored | `succeeded` |
| error / canceled by user | `failed` / `cancelled` |

`providerPaymentId` = store transaction / purchase id.  
Server webhook optional (Play RTDN / App Store Server Notifications) — still
emit `PaymentEvent` into `applyPaymentEvent`. Client store callbacks are UX
only until verified.

---

## Capabilities (typical)

| Adapter | hostedRedirect | directMobileMoney | nativeStoreSheet | payout |
|---------|----------------|-------------------|------------------|--------|
| Payunit | yes | yes (custom UI) | no | no |
| Store IAP | no | no | yes | no |

Return the subset **this app** actually wired, not the PSP’s full catalog.

---

## Method → PSP API

| `method` | Payunit |
|----------|---------|
| `hosted` | `POST /initialize` → `transaction_url` |
| `mobile_money` | `GET /gateways` + `POST /makepayment` |
| `card` | hosted / checkout (not in sandbox) |
| `store_iap` | n/a |

If the adapter cannot perform `method`, throw a **domain** error
(`UnsupportedPaymentMethod`) — the UI should have hidden it via capabilities.

---

## Webhook URLs

Dashboard / PSP config:

```
https://<api>/webhooks/payments/payunit
https://<api>/webhooks/payments/iap
```

One Express/FastAPI/Nest route param `{provider}` is enough:

```
adapter = adapters[provider] ?? 404
event = adapter.parseWebhook(...)
await applyPaymentEvent(event)
return 200
```

---

## Adding a new PSP (copy this)

1. Read this skill + the PSP’s own skill/docs.
2. Create `adapters/{id}/` with HTTP client + secret config.
3. Implement `PaymentGateway` (`providerId` = `{id}`).
4. Fill status + method maps (table like above).
5. Register in `resolveGateway` and `gatewayFor(provider)`.
6. Point PSP dashboard at `/webhooks/payments/{id}`.
7. Sandbox: createCharge → user action → webhook → idempotent replay.
8. Flip `PAYMENT_PROVIDER={id}` when cutover is ready.

Do **not** add `{id}` to checkout widgets, l10n (except optional brand line),
or order tables beyond the `provider` column.

---

## Anti-patterns

| Don’t | Do |
|-------|-----|
| `PayunitTransaction` in a Flutter widget | `Charge` / `NextAction` |
| `POST /payments/payunit/initialize` as the app API | `POST /payments/charges` |
| `if (status == 'SUCCESSFUL')` in orders | `if (event.status == succeeded)` |
| One global client that talks to today’s PSP | `gatewayFor(row.provider)` |
| New checkout screen per PSP | Capabilities + one checkout |
| Fulfill inside `parseWebhook` | Return `PaymentEvent` only |
