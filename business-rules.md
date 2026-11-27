# Wallet-Lite — Key Business Rules

See [API References](./api-references.md) for endpoint details and responses.

## Money and fees

Amounts are positive integer poisha (`100` poisha = `1` BDT). Transactions use a double-entry ledger: entries for each transaction balance to zero, and total wallet and system-account money is conserved.

| Transaction | Fee or commission |
|---|---|
| Send | Sender pays 1%, minimum 500 and maximum 2,500 poisha |
| Cash-in | Customer pays no fee; agent receives 0.5% commission |
| Cash-out | Customer pays 1.5%, minimum 1,000 and maximum 5,000 poisha; agent receives 70% of the fee (rounded down) |
| Merchant payment | Fee of 1% (capped at 10,000 poisha) is deducted from the merchant's amount |

Percentage fees are rounded half up unless the rule specifies rounding down. Values are defaults; the service's environment variables can override them.

## Customer limits

| Limit (poisha) | BASIC | FULL |
|---|---:|---:|
| Per outgoing transaction | 500,000 | 2,500,000 |
| Daily outgoing total | 1,000,000 | 5,000,000 |
| Monthly outgoing total | 3,000,000 | 20,000,000 |
| Maximum wallet balance | 2,500,000 | 50,000,000 |

Customers start at BASIC. Approved KYC upgrades them to FULL. Agents and merchants use FULL transaction limits and have no wallet maximum. Limits apply to completed sends, cash-outs, and merchant payments; usage is based on Dhaka calendar days and months. Cash-in, refunds, and reversals do not count as outgoing usage.

## PIN and repeat requests

- Money movement, refunds, payment acceptance, and PIN changes require the acting user's 5-digit PIN. Three consecutive wrong attempts lock it for 30 minutes; a correct PIN clears the counter.
- An `Idempotency-Key` is required on money-moving endpoints marked in the API reference. Keys are per user and retained for 24 hours. Repeating the same successful request returns its original result; reusing a key with a different request returns `409`.
- Balance, limit, refund, and idempotency checks must remain correct when requests run concurrently.

## Lifecycles

```text
Payment request: pending → accepted | declined | expired
Dispute:         opened → under_review → refunded | rejected
KYC:             unverified → submitted → verified | rejected
                 rejected → submitted (resubmission)
```

Payment requests expire at `expiresAt`. Accepting one runs the corresponding send or merchant-payment rules. Disputes apply only to eligible merchant payments; KYC approval gives the customer FULL tier.

## Permissions

| Action | Who |
|---|---|
| Send, cash-out, merchant payment | Customer |
| Cash-in | Agent |
| Refund | Merchant that received the payment |
| Create a payment request | Customer or merchant |
| Accept or decline a payment request | The customer asked to pay |
| Open a dispute or submit KYC | Customer |
| Review KYC/disputes, load agent float, suspend users, reverse transactions | Admin |

Users can access only their own wallet, statement, and permitted transactions or requests. Suspended users cannot initiate money movement; money cannot be sent to a suspended user.
