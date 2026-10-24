# PaymentGateway contract

Language-agnostic types. Translate to Dart / TypeScript / Python / etc. using
the **project’s** naming and null-safety. Do not add PSP-specific fields to
these types — keep extras in adapter-private structs.

---

## Money

| Field | Type | Notes |
|-------|------|--------|
| `amountMinor` | integer | Smallest currency unit (XAF francs, cents, …) |
| `currency` | string | ISO 4217 (`XAF`, `XOF`, `EUR`, …) |

---

## Enums

**PaymentMethodKind:** `hosted` | `mobile_money` | `card` | `store_iap`

**Operator:** `mtn` | `orange` | `unknown`  
(Extend only if several adapters need it; otherwise keep operator in metadata.)

**PaymentStatus:** `pending` | `awaiting_user` | `succeeded` | `failed` | `cancelled` | `expired`

**NextAction:**

| `type` | Extra fields | Client does |
|--------|----------------|-------------|
| `none` | — | Poll `GET /payments/{id}` or wait for push |
| `redirect` | `url` | Open browser / WebView / `url_launcher` |
| `await_user_on_phone` | — | Show “approve on phone” |
| `open_store_sheet` | `productId` | Native IAP sheet |

---

## CreateChargeRequest

| Field | Required | Notes |
|-------|----------|--------|
| `orderId` | yes | Your order / entitlement id |
| `money` | yes | Amount + currency |
| `country` | yes | ISO alpha-2 (`CM`) |
| `method` | yes | Kind requested by UI |
| `idempotencyKey` | yes | Unique per **attempt** |
| `customer.phone` | if `mobile_money` | E.164 |
| `operator` | if `mobile_money` | `mtn` / `orange` |
| `customer.email` / `name` / `id` | no | |
| `returnUrl` / `cancelUrl` | if hosted | HTTPS |
| `metadata` | no | String map; round-trip only |

Reject in the **use-case** if `method` is not in `gateway.capabilities()`.

---

## Charge

| Field | Notes |
|-------|--------|
| `paymentId` | **Your** id (DB uuid / ulid) |
| `provider` | `payunit` \| `stripe` \| `iap` \| … |
| `providerPaymentId` | PSP id (`transaction_id`, intent id, store txn) |
| `orderId` | |
| `status` | Canonical `PaymentStatus` |
| `money` | Echo of request |
| `nextAction` | What the client must do now |

---

## PaymentRef

Identify a charge for `getStatus`:

- `paymentId` (preferred) and/or
- `provider` + `providerPaymentId`

---

## PaymentEvent (webhook / poll result)

| Field | Notes |
|-------|--------|
| `provider` | Adapter id |
| `providerPaymentId` | |
| `paymentId` | If known |
| `orderId` | If known |
| `status` | Canonical |
| `occurredAt` | ISO-8601 |

---

## Capabilities

```
hostedRedirect: bool
directMobileMoney: bool
card: bool
nativeStoreSheet: bool
refund: bool
payout: bool
listMethods: bool
```

UI reads this object. Never `if (provider == 'payunit')`.

---

## Persistence (minimum columns)

```
id                  # paymentId
order_id
provider            # adapter id, immutable for this row
provider_payment_id
amount_minor
currency
status
next_action_json    # optional cache
idempotency_key     # unique
created_at
updated_at
```

Do **not** replace `provider` when the default config changes.

---

## Dart sketch (adapt to the repo)

```dart
abstract class PaymentGateway {
  String get providerId;
  PaymentCapabilities capabilities();

  Future<Charge> createCharge(CreateChargeRequest request);
  Future<Charge> getStatus(PaymentRef ref);
  Future<PaymentEvent> parseWebhook({
    required Map<String, String> headers,
    required String rawBody,
  });
}
```

## TypeScript sketch

```ts
interface PaymentGateway {
  readonly providerId: string;
  capabilities(): Capabilities;
  createCharge(req: CreateChargeRequest): Promise<Charge>;
  getStatus(ref: PaymentRef): Promise<Charge>;
  parseWebhook(input: {
    headers: Record<string, string | undefined>;
    rawBody: string;
  }): Promise<PaymentEvent>;
}
```

Keep refund/payout off the required interface until an adapter needs them.
