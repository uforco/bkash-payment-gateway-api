# `service/bkash.ts` — Documentation

Server-side helpers for **bKash Tokenized Checkout**: grant/cache an auth token, create a payment, and execute a payment after user approval.

Marked with `"use server"` so these functions run on the server only.

---

## Dependencies

| Import | Purpose |
|---|---|
| `axios` | HTTP calls to bKash APIs |
| `uuid` (`v4`) | Fallback merchant invoice number |
| `@/models/bkashToken` | MongoDB model to cache `auth_token` |

---

## Types

### `BkashConfig`

Credentials and base URL passed into every public function.

| Field | Type | Description |
|---|---|---|
| `base_url` | `string \| undefined` | bKash API base URL |
| `username` | `string \| undefined` | Merchant username (grant token header) |
| `password` | `string \| undefined` | Merchant password (grant token header) |
| `app_key` | `string \| undefined` | App key (`x-app-key` + grant body) |
| `app_secret` | `string \| undefined` | App secret (grant body) |

### `PaymentDetails`

Data used when creating a payment.

| Field | Type | Description |
|---|---|---|
| `amount` | `number` | Product price (must be ≥ 1) |
| `callbackURL` | `string` | URL bKash calls after user action |
| `orderID` | `string` | Merchant invoice number |
| `reference` | `string` | Payer reference (defaults to `"1"`) |
| `name` | `string` | Buyer name (accepted; not sent in create body) |
| `email` | `string` | Buyer email (accepted; not sent in create body) |
| `phone` | `string` | Buyer phone (accepted; not sent in create body) |

---

## Public Functions

### `createPayment(bkashConfig, paymentDetails)`

Creates a Tokenized Checkout payment session.

**Validations**

| Condition | Return |
|---|---|
| `amount` missing / falsy | `{ statusCode: 2065, statusMessage: 'amount required' }` |
| `amount < 1` | `{ statusCode: 2065, statusMessage: 'minimum amount 1' }` |
| `callbackURL` missing | `{ statusCode: 2065, statusMessage: 'callbackURL required' }` |

**API call**

- **Method / path:** `POST {base_url}/tokenized/checkout/create`
- **Headers:** from `authHeaders()` (token + `x-app-key`)
- **Body:**

| Field | Value |
|---|---|
| `mode` | `"0011"` |
| `currency` | `"BDT"` |
| `intent` | `"sale"` |
| `amount` | from `paymentDetails` |
| `callbackURL` | from `paymentDetails` |
| `payerReference` | `reference` or `"1"` |
| `merchantInvoiceNumber` | `orderID` or `"Inv_"` + first 6 chars of a UUID |

**Returns:** `response.data` from bKash on success; the caught error object on failure.

---

### `executePayment(bkashConfig, paymentID)`

Completes a payment after the user approves it on the bKash page.

**API call**

- **Method / path:** `POST {base_url}/tokenized/checkout/execute`
- **Headers:** from `authHeaders()`
- **Body:** `{ paymentID }`

**Returns:** `response.data` on success; `null` on error.

---

## Internal Helpers

### `authHeaders(bkashConfig)`

Builds headers for Create / Execute requests:

```ts
{
  "Content-Type": "application/json",
  Accept: "application/json",
  authorization: <id_token from grantToken>,
  "x-app-key": bkashConfig.app_key
}
```

---

### `grantToken(bkashConfig)`

Returns a usable `id_token`:

1. Loads the first document from the `Bkash` collection.
2. If **no document** exists, or `updatedAt` is older than **1 hour** (`3600000` ms), calls `setToken()`.
3. Otherwise returns the cached `auth_token`.
4. On error, logs and returns `null`.

---

### `setToken(bkashConfig)`

Requests a new token and stores it in MongoDB.

**API call**

- **Method / path:** `POST {base_url}/tokenized/checkout/token/grant`
- **Body:** from `tokenParameters()` — `{ app_key, app_secret }`
- **Headers:** from `tokenHeaders()` — username, password, JSON content type

**Persistence**

- If a `Bkash` document exists → update `auth_token` and `save()`
- Else → `Bkash.create({ auth_token })`

**Returns:** `response.data.id_token`

---

### `tokenParameters(bkashConfig)`

```ts
{ app_key, app_secret }
```

### `tokenHeaders(bkashConfig)`

```ts
{
  "Content-Type": "application/json",
  Accept: "application/json",
  username,
  password
}
```

---

## Token Flow (summary)

```text
createPayment / executePayment
        │
        ▼
   authHeaders()
        │
        ▼
   grantToken()
        │
        ├── cached token fresh (< 1 hour) → return auth_token
        │
        └── missing or expired → setToken()
                                    │
                                    ▼
                         POST .../token/grant
                                    │
                                    ▼
                         save id_token in MongoDB
                                    │
                                    ▼
                         return id_token
```

---

## Notes

- `name`, `email`, and `phone` on `PaymentDetails` are not included in the Create Payment request body.
- Token cache depends on MongoDB via the `Bkash` model; grant/create/execute will fail if the DB is unavailable when a token is needed.
- Create Payment errors return the error object; Execute Payment errors return `null`.
