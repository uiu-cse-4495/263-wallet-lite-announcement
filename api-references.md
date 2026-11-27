# Wallet-Lite — API Reference

See [Getting Started](../documentations/GettingStarted.md) for setup and [Key Business Rules](./business-rules.md) for the main wallet rules.

## Endpoints

### Internal and Test Endpoints

The database-backed endpoints operate on the team database selected by the gateway. They do not require an application Bearer token; requests still go through the gateway and its assigned access key. `GET /_internal/health` is always available. The database reset, state inspection, and fake-clock endpoints are test-only: they are disabled in production and return `404` unless `WALLET_ENABLE_TEST_ENDPOINTS=true` in a non-production environment.

#### `GET /_internal/health`

Check whether the service is running.

**Success — 200:**
```json
{ "status": "healthy", "version": "wallet-lite-v1" }
```

#### `POST /_internal/reset-database`

Reset the selected team database. This **deletes all test data** and recreates the seeded admin and system accounts. Use only when a clean test database is intended.

**Body:** None.

**Success — 200:**
```json
{ "status": "reset" }
```

#### `GET /_internal/db-state`

Return a snapshot of the selected team database for test verification.

**Success — 200:** An object containing `users`, `wallets`, `systemAccounts`, `transactions`, `ledgerEntries`, `paymentRequests`, `disputes`, and `kycSubmissions` arrays. User, transaction, ledger, request, dispute, and KYC entries use the corresponding response fields documented below; wallet entries contain `userId` and `balance`, and system-account entries contain `name` and `balance`.

#### `POST /_internal/clock`

Set or clear the fake clock used for time-dependent behavior such as payment-request expiry and date-based limits.

**Set the clock:**
```json
{ "now": "2026-01-01T12:00:00.000Z" }
```

`now` must be a valid ISO timestamp with a timezone (`Z` or an offset).

**Success — 200:**
```json
{ "status": "clock set", "now": "2026-01-01T12:00:00.000Z" }
```

To clear the fake clock, send `{ "now": null }` (or omit `now`).

**Success — 200:**
```json
{ "status": "clock cleared" }
```

**Invalid timestamp — 400:**
```json
{ "error": "now must be a valid ISO timestamp or null" }
```

---

### Shared List Query Parameters

The list endpoints below support a shared query contract for sorting and exact field-value filtering.

| Param | Type | Description |
|---|---|---|
| `sort_by` | `asc \| desc` | Sort direction for the endpoint's default sort field. |
| `filter_field` | string | Field name to filter on (must be in the endpoint's allowed field list). |
| `filter_value` | string | Exact value to match for `filter_field` (case-insensitive for text fields). |

Validation rules:

- `sort_by` must be `asc` or `desc`.
- `filter_field` and `filter_value` must be provided together.
- Invalid `filter_field` returns `400` with the allowed field list.

Type matching notes:

- Boolean filters accept `true` or `false`.
- Number filters require an exact numeric match.
- Text filters are case-insensitive exact matches.
- Timestamp filters (`createdAt`, `occurredAt`, …) match the exact ISO timestamp.

Invalid list parameters return `400` with an error describing the invalid parameter.

---

### Shared Money-Movement Rules

These checks apply to `POST /transactions/send`, `/cash-in`, `/cash-out`, `/merchant-payment`, `POST /transactions/:id/refund`, and `POST /payment-requests/:id/accept`. Validation proceeds through authentication, role, idempotency key, request body, account/counterparty, PIN, applicable limits, and balance; the first failure is returned. Each endpoint documents its specific errors.

- A missing or invalid token returns `401`; an incorrect role returns `403`.
- Idempotency keys must be 8–64 characters. Reusing a key with a different request or while the first request is processing returns `409`; a successful replay returns the original response with `Idempotent-Replayed: true`.
- Suspended accounts, invalid counterparties, locked or incorrect PINs, exceeded limits, and insufficient balances are rejected.
- Limits apply to sends, cash-outs, and merchant payments. Recipient balance limits apply to sends and cash-ins.
- Successful requests return `201` with the transaction, including `ledgerEntries`, and a `_lab` field.

---

### Auth

#### `POST /auth/register`

Register a new user. A wallet with balance `0` is created for `customer`, `agent` and `merchant`.

**Body:**
```json
{
  "name": "Alice Rahman",
  "email": "alice@example.com",
  "phone": "01711111111",
  "password": "pass123",
  "pin": "12345",
  "role": "customer"
}
```
> `role` must be one of `customer`, `agent`, `merchant`. `phone` must match `01` + one digit 3–9 + 8 digits (11 digits total). `pin` is a string of exactly 5 digits.

**Success — 201:**
```json
{
  "id": "a1b2c3d4-...",
  "name": "Alice Rahman",
  "email": "alice@example.com",
  "phone": "01711111111",
  "role": "customer",
  "status": "active",
  "kycStatus": "unverified",
  "tier": "BASIC"
}
```

**Failures:**

| Condition | Status | Response |
|---|---|---|
| Missing any field | 400 | `{ "error": "name, email, phone, password, pin and role are required" }` |
| Invalid role (including `admin`) | 400 | `{ "error": "role must be one of customer, agent, merchant" }` |
| Invalid phone | 400 | `{ "error": "phone must be a valid 11-digit Bangladeshi mobile number" }` |
| Password shorter than 6 characters | 400 | `{ "error": "Password must be at least 6 characters long" }` |
| PIN not exactly 5 digits | 400 | `{ "error": "PIN must be exactly 5 digits" }` |
| Email already used | 409 | `{ "error": "Email already in use" }` |
| Phone already used | 409 | `{ "error": "Phone already in use" }` |

---

#### `POST /auth/login`

Log in and receive a JWT. Suspended users can log in.

For the seeded admin in the provided test environment, use `admin@wallet.lab` / `admin123`. The database reset endpoint restores this seeded account. These are test-fixture credentials; use them only with the assigned environment.

**Body:**
```json
{ "email": "alice@example.com", "password": "pass123" }
```

**Success — 200:**
```json
{ "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." }
```

**Failures:**

| Condition | Status | Response |
|---|---|---|
| Missing email or password | 400 | `{ "error": "email and password are required" }` |
| Email not found | 401 | `{ "error": "Invalid credentials" }` |
| Wrong password | 401 | `{ "error": "Invalid credentials" }` |

---

#### `GET /me`

**Success — 200:** Returns the requester's User object.

**Failures:** `401 { "error": "Missing token" }`.

---

#### `POST /me/pin`

Change your PIN. Uses and updates the same lockout counter as every PIN check.

**Body:**
```json
{ "currentPin": "12345", "newPin": "54321" }
```

**Success — 200:** `{ "status": "pin changed" }`

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Admin account | 403 | `{ "error": "Admin accounts do not have a PIN" }` |
| Missing `currentPin` or `newPin` | 400 | `{ "error": "currentPin and newPin are required" }` |
| `newPin` not exactly 5 digits | 400 | `{ "error": "PIN must be exactly 5 digits" }` |
| `newPin` equals `currentPin` | 400 | `{ "error": "New PIN must be different from the current PIN" }` |
| PIN locked / wrong PIN | 403 / 401 | Same as check 8 above |

---

### Wallet

#### `GET /wallet`

Current balance, tier and limit usage. Customers, agents and merchants only.

**Success — 200:**
```json
{
  "userId": "uuid",
  "balance": 250000,
  "tier": "BASIC",
  "kycStatus": "unverified",
  "limits": {
    "perTransaction": 500000,
    "daily": 1000000,
    "monthly": 3000000,
    "maxBalance": 2500000
  },
  "usage": {
    "dailyUsed": 200000,
    "monthlyUsed": 800000
  }
}
```
> `maxBalance` is `null` for agents and merchants.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Requester is admin | 403 | `{ "error": "Admin accounts do not have a wallet" }` |

---

#### `GET /wallet/statement`

The requester's own ledger entries, newest first by default. Nobody can read another user's statement.

**Query params:**

| Param | Description |
|---|---|
| `sort_by` | `asc` or `desc` (default sort field: `createdAt`, ties broken by entry creation order) |
| `filter_field` | One of: `id`, `transactionId`, `kind`, `direction`, `amount`, `balanceAfter`, `createdAt` |
| `filter_value` | Exact value for `filter_field` |

**Examples:**

```http
GET /wallet/statement?sort_by=asc
GET /wallet/statement?filter_field=kind&filter_value=fee
GET /wallet/statement?filter_field=direction&filter_value=debit&sort_by=desc
```

**Success — 200:** Returns an array of LedgerEntry objects, each with an extra `transactionType` field. `balanceAfter` is the wallet balance immediately after that entry, so reading the statement oldest-first, each entry's `balanceAfter` must equal the previous `balanceAfter` ± `amount`.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Requester is admin | 403 | `{ "error": "Admin accounts do not have a wallet" }` |
| Invalid list query params | 400 | e.g. `{ "error": "filter_field must be one of: id, transactionId, kind, direction, amount, balanceAfter, createdAt" }` |

---

### Transactions

#### `POST /transactions/send`

Send money from the requester to another customer. **Customers only.** Recipient is identified by phone.

**Headers:** `Idempotency-Key` (required).

**Body:**
```json
{ "recipientPhone": "01722222222", "amount": 150050, "pin": "12345", "reference": "Rent share" }
```
> `reference` is optional, max 100 characters.

**Success — 201:**
```json
{
  "id": "t1t2t3-...",
  "type": "send",
  "status": "completed",
  "fromUserId": "a1b2c3-...",
  "toUserId": "b1b2b3-...",
  "amount": 150050,
  "fee": 1501,
  "agentCommission": 0,
  "merchantFee": 0,
  "feeReturned": 0,
  "refundedAmount": 0,
  "parentTransactionId": null,
  "reference": "Rent share",
  "ledgerEntries": [ /* 4 entries, see Ledger entries per transaction type */ ],
  "createdAt": "2026-01-01T00:00:00.000Z"
}
```

**Endpoint-specific failures** (all other failures per Shared Money-Movement Rules above):

| Check | Condition | Status | Response |
|---|---|---|---|
| 2 | Requester's role is not `customer` | 403 | `{ "error": "Only customer accounts can send money" }` |
| 5 | Missing `recipientPhone`, `amount` or `pin` | 400 | `{ "error": "recipientPhone, amount and pin are required" }` |
| 5 | `amount` not a positive integer | 400 | `{ "error": "amount must be a positive integer (poisha)" }` |
| 5 | `amount` below minimum | 400 | `{ "error": "amount must be at least 1000" }` |
| 5 | `reference` longer than 100 characters | 400 | `{ "error": "reference must be at most 100 characters" }` |
| 7 | No user with that phone | 404 | `{ "error": "Recipient not found" }` |
| 7 | Recipient is yourself | 400 | `{ "error": "Cannot send money to yourself" }` |
| 7 | Recipient is not a customer | 400 | `{ "error": "Recipient must be a customer account" }` |
| 7 | Recipient is suspended | 400 | `{ "error": "Recipient account is suspended" }` |

---

#### `POST /transactions/cash-in`

An agent deposits cash into a customer's wallet. The agent's own wallet (their **float**) is debited by `amount`; the agent earns a commission. **Agents only.** Uses the agent's PIN.

**Headers:** `Idempotency-Key` (required).

**Body:**
```json
{ "customerPhone": "01711111111", "amount": 100100, "pin": "99999" }
```

**Success — 201:** Transaction object with `type: "cash_in"`, `fromUserId` = agent, `toUserId` = customer, `fee: 0`, `agentCommission` = `round(amount × 0.5%)`.

**Endpoint-specific failures:**

| Check | Condition | Status | Response |
|---|---|---|---|
| 2 | Requester's role is not `agent` | 403 | `{ "error": "Only agent accounts can perform cash-in" }` |
| 5 | Missing `customerPhone`, `amount` or `pin` | 400 | `{ "error": "customerPhone, amount and pin are required" }` |
| 5 | `amount` invalid / below minimum | 400 | same messages as `send` |
| 7 | No user with that phone | 404 | `{ "error": "Customer not found" }` |
| 7 | Target is not a customer | 400 | `{ "error": "Cash-in target must be a customer account" }` |
| 7 | Customer is suspended | 400 | `{ "error": "Recipient account is suspended" }` |
| 11 | Agent float < `amount` | 400 | `{ "error": "Insufficient balance" }` |

> Cash-in is **not** subject to per-transaction/daily/monthly limits (check 9 is skipped). The customer's **maximum balance** applies (check 10).

---

#### `POST /transactions/cash-out`

A customer withdraws cash at an agent. The customer is debited `amount + fee`; the agent's wallet is credited `amount` plus their share of the fee. **Customers only.**

**Headers:** `Idempotency-Key` (required).

**Body:**
```json
{ "agentPhone": "01744444444", "amount": 100000, "pin": "12345" }
```

**Success — 201:** Transaction object with `type: "cash_out"`, `fee` = customer fee, `agentCommission` = agent's share (`floor(fee × 70 / 100)`).

**Endpoint-specific failures:**

| Check | Condition | Status | Response |
|---|---|---|---|
| 2 | Requester's role is not `customer` | 403 | `{ "error": "Only customer accounts can cash out" }` |
| 5 | Missing `agentPhone`, `amount` or `pin` | 400 | `{ "error": "agentPhone, amount and pin are required" }` |
| 5 | `amount` invalid / below minimum | 400 | same messages as `send` |
| 7 | No user with that phone | 404 | `{ "error": "Agent not found" }` |
| 7 | Target is not an agent | 400 | `{ "error": "Cash-out target must be an agent account" }` |
| 7 | Agent is suspended | 400 | `{ "error": "Recipient account is suspended" }` |

---

#### `POST /transactions/merchant-payment`

A customer pays a merchant. The merchant receives `amount` and is charged a fee out of it. **Customers only.**

**Headers:** `Idempotency-Key` (required).

**Body:**
```json
{ "merchantPhone": "01755555555", "amount": 10050, "pin": "12345", "reference": "Order #881" }
```

**Success — 201:** Transaction object with `type: "merchant_payment"`, `fee: 0`, `merchantFee` = `min(round(amount × 1%), 10000)`. The customer's balance drops by exactly `amount`; the merchant's rises by `amount − merchantFee`.

**Endpoint-specific failures:**

| Check | Condition | Status | Response |
|---|---|---|---|
| 2 | Requester's role is not `customer` | 403 | `{ "error": "Only customer accounts can pay merchants" }` |
| 5 | Missing `merchantPhone`, `amount` or `pin` | 400 | `{ "error": "merchantPhone, amount and pin are required" }` |
| 5 | `amount` invalid / below minimum | 400 | same messages as `send` |
| 7 | No user with that phone | 404 | `{ "error": "Merchant not found" }` |
| 7 | Target is not a merchant | 400 | `{ "error": "Payment target must be a merchant account" }` |
| 7 | Merchant is suspended | 400 | `{ "error": "Recipient account is suspended" }` |

---

#### `GET /transactions`

List transactions the requester is part of (`fromUserId` or `toUserId`). Admin sees all.

**Query params:**

| Param | Description |
|---|---|
| `sort_by` | `asc` or `desc` (default sort field: `createdAt`) |
| `filter_field` | One of: `id`, `type`, `status`, `fromUserId`, `toUserId`, `amount`, `fee`, `parentTransactionId`, `createdAt` |
| `filter_value` | Exact value for `filter_field` |

**Examples:**

```http
GET /transactions?sort_by=asc
GET /transactions?filter_field=type&filter_value=send
GET /transactions?filter_field=status&filter_value=reversed&sort_by=desc
```

**Success — 200:** Returns an array of Transaction objects (without `ledgerEntries`).

**Failures:** `401 { "error": "Missing token" }`; `400` for invalid list query params.

---

#### `GET /transactions/:id`

**Success — 200:** The full Transaction object with nested `ledgerEntries` and nested `timeline` (chronological, oldest first). Ledger entries are shown **only for the requester's own account**, except for admin who sees all of them.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Transaction not found | 404 | `{ "error": "Transaction not found" }` |
| Requester is neither `fromUserId`, `toUserId`, nor admin | 403 | `{ "error": "You do not have permission to view this transaction" }` |

---

#### `GET /transactions/:id/timeline`

Just the timeline. Same visibility rule as `GET /transactions/:id`.

**Query params:**

| Param | Description |
|---|---|
| `sort_by` | `asc` or `desc` (default sort field: `occurredAt`) |
| `filter_field` | One of: `id`, `transactionId`, `event`, `actorId`, `actorRole`, `occurredAt` |
| `filter_value` | Exact value for `filter_field` |

**Success — 200:** Array of TransactionTimelineEvent objects, oldest first by default.

**Failures:** same as `GET /transactions/:id`, plus `400` for invalid list query params.

---

#### `POST /transactions/:id/refund`

The receiving **merchant** refunds all or part of a `merchant_payment` to the customer. Creates a new `refund` transaction.

**Headers:** `Idempotency-Key` (required).

**Body:**
```json
{ "amount": 3333, "pin": "55555" }
```

**Refund rules:**
- The original transaction must be a `merchant_payment` with status `completed`, and the requester must be its `toUserId`.
- Allowed within `REFUND_WINDOW_DAYS` (30) of the original `createdAt`. Exactly 30×24h later is still allowed; one millisecond more is not.
- `amount` ≥ `MIN_REFUND_AMOUNT` (100) and ≤ `original.amount − original.refundedAmount`. Several partial refunds are allowed until the total reaches `original.amount`.
- The customer receives exactly `amount`.
- The merchant gets part of the original merchant fee back, proportionally:
  - Normal refund: `feeReturned = floor(merchantFee × amount / original.amount)`.
  - **Final refund** (the one that makes `refundedAmount == original.amount`): `feeReturned = merchantFee − (sum of feeReturned of all earlier refunds)`. So over the life of a fully refunded payment the merchant gets back **exactly** the whole fee — no poisha lost or created.
- The merchant's wallet is debited `amount` and credited `feeReturned` (net `amount − feeReturned`), and `FEE_POOL` is debited `feeReturned`.
- A refund does **not** reduce the customer's limit usage (only reversal does).
- Updates `refundedAmount` on the original and records a `refunded` timeline event on it.

**Worked example** — payment of `10050`, `merchantFee = 101`:

| Refund | amount | feeReturned | Merchant net debit | Original `refundedAmount` afterwards |
|---|---|---|---|---|
| 1 | 3333 | `floor(101 × 3333 / 10050)` = 33 | 3300 | 3333 |
| 2 (final) | 6717 | `101 − 33` = 68 | 6649 | 10050 |

**Success — 201:** Transaction object with `type: "refund"`, `parentTransactionId` = original id, `amount`, `feeReturned`, `fromUserId` = merchant, `toUserId` = customer.

**Endpoint-specific failures:**

| Check | Condition | Status | Response |
|---|---|---|---|
| 2 | Requester's role is not `merchant` | 403 | `{ "error": "Only merchant accounts can issue refunds" }` |
| 5 | Missing `amount` or `pin` | 400 | `{ "error": "amount and pin are required" }` |
| 5 | `amount` not a positive integer | 400 | `{ "error": "amount must be a positive integer (poisha)" }` |
| 5 | `amount` below minimum | 400 | `{ "error": "amount must be at least 100" }` |
| 7 | Transaction not found | 404 | `{ "error": "Transaction not found" }` |
| 7 | Requester is not the receiving merchant | 403 | `{ "error": "Only the merchant who received this payment can refund it" }` |
| 7 | Not a `merchant_payment` | 400 | `{ "error": "Only merchant payments can be refunded" }` |
| 7 | Transaction is `reversed` | 409 | `{ "error": "Transaction has been reversed" }` |
| 7 | Older than 30 days | 400 | `{ "error": "Refund window has passed" }` |
| 7 | `amount` > remaining refundable amount | 400 | `{ "error": "Refund amount exceeds the refundable balance" }` |
| 11 | Merchant balance < `amount − feeReturned` | 400 | `{ "error": "Insufficient balance" }` |

> The transaction-ownership checks (404, 403, 400, 409, 400, 400) happen in place of check 7 and run **before** the PIN check. Customer limit checks (9) and recipient cap (10) are skipped for refunds.

---

### Payment Requests

#### `POST /payment-requests`

Ask someone to pay you. **Customers and merchants** can request; the payer must be a customer.

**Body:**
```json
{ "payerPhone": "01711111111", "amount": 50000, "note": "Lunch split", "expiresInMinutes": 60 }
```
> `note` optional (max 100 characters). `expiresInMinutes` optional, integer 5–1440, default `REQUEST_DEFAULT_EXPIRY_MINUTES` (60). `expiresAt = now + expiresInMinutes`.

**Success — 201:** PaymentRequest object with `status: "pending"`.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Requester's role is not `customer` or `merchant` | 403 | `{ "error": "Only customer and merchant accounts can request payments" }` |
| Requester suspended | 403 | `{ "error": "Your account is suspended" }` |
| Missing `payerPhone` or `amount` | 400 | `{ "error": "payerPhone and amount are required" }` |
| `amount` not a positive integer | 400 | `{ "error": "amount must be a positive integer (poisha)" }` |
| `amount` below minimum | 400 | `{ "error": "amount must be at least 1000" }` |
| `expiresInMinutes` outside 5–1440 or not an integer | 400 | `{ "error": "expiresInMinutes must be an integer between 5 and 1440" }` |
| `note` longer than 100 characters | 400 | `{ "error": "note must be at most 100 characters" }` |
| No user with that phone | 404 | `{ "error": "Payer not found" }` |
| Payer is not a customer | 400 | `{ "error": "Payer must be a customer account" }` |
| Payer is yourself | 400 | `{ "error": "Cannot request payment from yourself" }` |
| Payer is suspended | 400 | `{ "error": "Payer account is suspended" }` |
| Requester already has 10 pending requests | 409 | `{ "error": "Too many pending payment requests" }` |

---

#### `GET /payment-requests`

Requests where the requester is the `requesterId` **or** the `payerId`. Admin sees all.

**Query params:**

| Param | Description |
|---|---|
| `sort_by` | `asc` or `desc` (default sort field: `createdAt`) |
| `filter_field` | One of: `id`, `requesterId`, `payerId`, `amount`, `status`, `expiresAt`, `createdAt` |
| `filter_value` | Exact value for `filter_field` |

**Success — 200:** Array of PaymentRequest objects. `status` reflects expiry at the moment of the request.

**Failures:** `401 { "error": "Missing token" }`; `400` for invalid list query params.

---

#### `GET /payment-requests/:id`

**Success — 200:** The PaymentRequest object.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Not found | 404 | `{ "error": "Payment request not found" }` |
| Requester is neither requester, payer nor admin | 403 | `{ "error": "You do not have permission to view this payment request" }` |

---

#### `POST /payment-requests/:id/accept`

The payer pays the request. **Payer only.** Runs a `send` (requester is a customer) or `merchant_payment` (requester is a merchant) with `amount` = the request's amount, following every rule of that transaction.

**Headers:** `Idempotency-Key` (required).

**Body:**
```json
{ "pin": "12345" }
```

**Success — 201:** The resulting Transaction object, plus `"paymentRequestId"`. The request becomes `accepted` and gets `transactionId`.

**Endpoint-specific failures** (then the transaction's own checks 6–11 in the usual order):

| Check | Condition | Status | Response |
|---|---|---|---|
| 2 | Requester's role is not `customer` | 403 | `{ "error": "Only customer accounts can accept payment requests" }` |
| 5 | Missing `pin` | 400 | `{ "error": "pin is required" }` |
| 7 | Request not found | 404 | `{ "error": "Payment request not found" }` |
| 7 | Requester is not the request's `payerId` | 403 | `{ "error": "Only the payer can accept this payment request" }` |
| 7 | Status is `expired` (including: `now >= expiresAt`) | 409 | `{ "error": "Payment request has expired" }` |
| 7 | Status is `accepted` or `declined` | 409 | `{ "error": "Payment request is no longer pending" }` |

> Two simultaneous accepts of the same request (with different idempotency keys) must produce **one** transaction; the other gets the "no longer pending" `409`.

---

#### `POST /payment-requests/:id/decline`

The payer declines. **Payer only.** No PIN, no idempotency key.

**Success — 200:** The PaymentRequest object with `status: "declined"`.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Requester's role is not `customer` | 403 | `{ "error": "Only customer accounts can decline payment requests" }` |
| Not found | 404 | `{ "error": "Payment request not found" }` |
| Requester is not the `payerId` | 403 | `{ "error": "Only the payer can decline this payment request" }` |
| Status is `expired` | 409 | `{ "error": "Payment request has expired" }` |
| Status is `accepted` or `declined` | 409 | `{ "error": "Payment request is no longer pending" }` |

---

### Disputes

#### `POST /disputes`

A customer disputes a merchant payment they made.

**Body:**
```json
{ "transactionId": "t1t2t3-...", "reason": "not_received", "description": "Item never arrived" }
```
> `reason` must be one of `not_received`, `wrong_amount`, `unauthorized`, `duplicate`. `description` is optional, max 500 characters.

**Success — 201:** Dispute object with `status: "opened"`. Adds a `disputed` timeline event to the transaction.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Requester's role is not `customer` | 403 | `{ "error": "Only customer accounts can open disputes" }` |
| Requester suspended | 403 | `{ "error": "Your account is suspended" }` |
| Missing `transactionId` or `reason` | 400 | `{ "error": "transactionId and reason are required" }` |
| Invalid `reason` | 400 | `{ "error": "reason must be one of not_received, wrong_amount, unauthorized, duplicate" }` |
| `description` longer than 500 characters | 400 | `{ "error": "description must be at most 500 characters" }` |
| Transaction not found **or** not paid by the requester | 404 | `{ "error": "Transaction not found" }` |
| Not a `merchant_payment` | 400 | `{ "error": "Only merchant payments can be disputed" }` |
| Transaction is `reversed` | 409 | `{ "error": "Transaction has been reversed" }` |
| Older than 7 days (exactly 7×24h is still allowed) | 400 | `{ "error": "Dispute window has passed" }` |
| Payment already fully refunded | 409 | `{ "error": "Transaction has already been fully refunded" }` |
| A dispute already exists for this transaction (any status) | 409 | `{ "error": "A dispute already exists for this transaction" }` |

---

#### `GET /disputes`

Disputes where the requester is the `customerId` or the `merchantId`. Admin sees all.

**Query params:**

| Param | Description |
|---|---|
| `sort_by` | `asc` or `desc` (default sort field: `createdAt`) |
| `filter_field` | One of: `id`, `transactionId`, `customerId`, `merchantId`, `reason`, `status`, `refundedAmount`, `createdAt`, `updatedAt` |
| `filter_value` | Exact value for `filter_field` |

**Success — 200:** Array of Dispute objects.

**Failures:** `401 { "error": "Missing token" }`; `400` for invalid list query params.

---

#### `GET /disputes/:id`

**Success — 200:** The Dispute object.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Not found | 404 | `{ "error": "Dispute not found" }` |
| Requester is neither customer, merchant nor admin | 403 | `{ "error": "You do not have permission to view this dispute" }` |

---

### KYC

#### `POST /kyc`

Submit identity details for verification. **Customers only.**

**Body:**
```json
{ "fullName": "Alice Rahman", "nidNumber": "1234567890", "dateOfBirth": "1999-05-20" }
```
> `nidNumber`: digits only, length 10, 13 or 17. `dateOfBirth`: `YYYY-MM-DD`, must be a real date, and the customer must be **at least 18 years old** on the current (Dhaka) date — someone whose 18th birthday is today is old enough. `fullName`: 2–100 characters.

**Success — 201:** KycSubmission object with `status: "submitted"`. The user's `kycStatus` becomes `submitted`.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| No token | 401 | `{ "error": "Missing token" }` |
| Requester's role is not `customer` | 403 | `{ "error": "Only customer accounts can submit KYC" }` |
| Requester suspended | 403 | `{ "error": "Your account is suspended" }` |
| Missing any field | 400 | `{ "error": "fullName, nidNumber and dateOfBirth are required" }` |
| Invalid `nidNumber` | 400 | `{ "error": "nidNumber must be 10, 13 or 17 digits" }` |
| Invalid `dateOfBirth` format or date | 400 | `{ "error": "dateOfBirth must be a valid date in YYYY-MM-DD format" }` |
| Under 18 | 400 | `{ "error": "You must be at least 18 years old" }` |
| `kycStatus` is `submitted` | 409 | `{ "error": "KYC is already submitted and awaiting review" }` |
| `kycStatus` is `verified` | 409 | `{ "error": "KYC is already verified" }` |
| NID used by another user whose KYC is `submitted` or `verified` | 409 | `{ "error": "NID is already registered to another account" }` |

---

#### `GET /kyc`

The requester's latest KYC submission. **Customers only.**

**Success — 200:** KycSubmission object.

**Failures:** `401 { "error": "Missing token" }`; `403 { "error": "Only customer accounts have KYC" }`; `404 { "error": "No KYC submission found" }`.

---

### Admin

All admin endpoints return `401 { "error": "Missing token" }` without a token and `403 { "error": "Only admin accounts can perform this action" }` for any non-admin.

#### `GET /admin/kyc`

List KYC submissions.

**Query params:**

| Param | Description |
|---|---|
| `sort_by` | `asc` or `desc` (default sort field: `createdAt`) |
| `filter_field` | One of: `id`, `userId`, `fullName`, `nidNumber`, `status`, `createdAt` |
| `filter_value` | Exact value for `filter_field` |

**Success — 200:** Array of KycSubmission objects. **Failures:** `400` for invalid list query params.

---

#### `POST /admin/kyc/:userId/approve`

Approve the user's `submitted` KYC. Sets the user's `kycStatus` to `verified` and `tier` to `FULL`.

**Success — 200:** The KycSubmission object (`status: "verified"`).

**Failures:**

| Condition | Status | Response |
|---|---|---|
| User or submission not found | 404 | `{ "error": "KYC submission not found" }` |
| Submission is not `submitted` | 409 | `{ "error": "KYC is not awaiting review" }` |

---

#### `POST /admin/kyc/:userId/reject`

**Body:** `{ "reason": "NID photo unreadable" }` (required, 1–200 characters).

**Success — 200:** The KycSubmission object (`status: "rejected"`, `rejectionReason` set). The user's `kycStatus` becomes `rejected`; `tier` stays `BASIC`.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| Missing `reason` | 400 | `{ "error": "reason is required" }` |
| User or submission not found | 404 | `{ "error": "KYC submission not found" }` |
| Submission is not `submitted` | 409 | `{ "error": "KYC is not awaiting review" }` |

---

#### `POST /admin/agents/:id/float`

Load e-float into an agent's wallet from `FLOAT_POOL`. Creates a `float_load` transaction.

**Body:** `{ "amount": 5000000 }` (positive integer poisha).

**Success — 201:** Transaction object (`type: "float_load"`, `fromUserId: null`, `toUserId` = agent).

**Failures:**

| Condition | Status | Response |
|---|---|---|
| `amount` not a positive integer | 400 | `{ "error": "amount must be a positive integer (poisha)" }` |
| User not found | 404 | `{ "error": "Agent not found" }` |
| User is not an agent | 400 | `{ "error": "Float can only be loaded to agent accounts" }` |
| `amount` > `FLOAT_POOL` balance | 400 | `{ "error": "Float pool has insufficient balance" }` |

---

#### `POST /admin/users/:id/status`

Suspend or re-activate a user.

**Body:** `{ "status": "suspended" }` (`suspended` or `active`).

**Success — 200:** The User object.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| Invalid `status` | 400 | `{ "error": "status must be one of active, suspended" }` |
| User not found | 404 | `{ "error": "User not found" }` |
| Target is an admin | 400 | `{ "error": "Admin accounts cannot be suspended" }` |
| User already has that status | 409 | `{ "error": "User is already <status>" }` |

---

#### `POST /admin/transactions/:id/reverse`

Undo a transaction completely, as if it never happened (fees and commissions included). Creates a `reversal` transaction and sets the original's `status` to `reversed`.

**Rules:**
- Only `send`, `cash_in`, `cash_out` and `merchant_payment` can be reversed.
- Within `REVERSAL_WINDOW_HOURS` (24) of the original `createdAt` (exactly 24h is still allowed).
- Every account that was credited must still hold enough to be debited back (system accounts excluded). Otherwise nothing happens.
- A transaction that has any refund cannot be reversed.
- Reversed transactions stop counting toward the sender's daily/monthly usage.

**Success — 201:** The `reversal` Transaction object (`parentTransactionId` = original id) with its mirrored `ledgerEntries`. A `reversed` event is added to the original's timeline.

**Failures:**

| Condition | Status | Response |
|---|---|---|
| Transaction not found | 404 | `{ "error": "Transaction not found" }` |
| Type is not reversible | 400 | `{ "error": "Only send, cash_in, cash_out and merchant_payment transactions can be reversed" }` |
| Already reversed | 409 | `{ "error": "Transaction has already been reversed" }` |
| `refundedAmount > 0` | 409 | `{ "error": "Transaction has refunds and cannot be reversed" }` |
| Older than 24h | 400 | `{ "error": "Reversal window has passed" }` |
| A credited account cannot cover the reversal | 400 | `{ "error": "Recipient has insufficient balance to reverse this transaction" }` |

---

#### `POST /admin/disputes/:id/review`

Move a dispute from `opened` to `under_review`.

**Success — 200:** The Dispute object.

**Failures:** `404 { "error": "Dispute not found" }`; `409 { "error": "Dispute is not in opened state" }`.

---

#### `POST /admin/disputes/:id/resolve`

Resolve a dispute that is `under_review`.

**Body (refund):**
```json
{ "decision": "refund", "amount": 5000, "note": "Partial refund agreed" }
```
**Body (reject):**
```json
{ "decision": "reject", "note": "Delivery confirmed" }
```
> `decision` is `refund` or `reject`. `refund` requires `amount`; `reject` requires `note`. `note` is otherwise optional (max 200 characters).

**Behaviour:**
- `refund` performs a refund exactly as in `POST /transactions/:id/refund` (same fee-return formula, same remaining-amount limit, same `refund` transaction), with the **admin** as actor and no PIN or idempotency key. The dispute's `refundedAmount` = `amount` and its status becomes `refunded`. A *partial* amount is allowed; the dispute is still closed.
- `reject` sets `rejected`, stores the `note` in `resolutionNote`.
- Both add a `dispute_resolved` event to the transaction's timeline.
- If the merchant cannot afford the refund, nothing changes and the dispute stays `under_review`.

**Success — 200:** The Dispute object (a refund also returns `"refundTransactionId"`).

**Failures:**

| Condition | Status | Response |
|---|---|---|
| Dispute not found | 404 | `{ "error": "Dispute not found" }` |
| Invalid `decision` | 400 | `{ "error": "decision must be refund or reject" }` |
| `refund` without a positive integer `amount` | 400 | `{ "error": "amount must be a positive integer (poisha)" }` |
| `amount` below 100 | 400 | `{ "error": "amount must be at least 100" }` |
| `reject` without `note` | 400 | `{ "error": "note is required when rejecting" }` |
| Dispute is not `under_review` | 409 | `{ "error": "Dispute is not under review" }` |
| `amount` > remaining refundable amount | 400 | `{ "error": "Refund amount exceeds the refundable balance" }` |
| Merchant balance < `amount − feeReturned` | 400 | `{ "error": "Merchant has insufficient balance to cover the refund" }` |
