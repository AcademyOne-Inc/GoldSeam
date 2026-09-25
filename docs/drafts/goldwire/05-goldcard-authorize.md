# Call 5 — GoldCard authorize

[← Call 4 — Get a hold](04-get-hold.md) · [Summary](README.md) · Next: [Call 6 — Book the seat →](06-book-seat.md)

> **Draft.** Nothing here is live. This call belongs to **GoldCard**, not GoldWire — GoldWire never
> touches money. It is specified here so the ids passed between the services line up.

## What it does

Arranges the money for a held seat.

- **Prepay:** GoldCard **authorizes** the amount due now from the learner's chosen source — a bank
  account, a credit card, a loan, or a 529 college savings plan. The money is **not taken yet**; it is
  taken (captured) only when the school confirms the registration.
- **Reserve:** nothing is charged. GoldCard records the learner's card or account as a **guarantee**,
  used only if the school publishes a no-show or late-drop fee — like a restaurant that takes a card
  to hold a table.

## Where it sits

- **Before:** [Call 3](03-hold-seat.md) held seat `hld_01J8Z3XQ2K` and said `amount_due_now` is $241.00.
- **This call:** "Authorize $241.00 on my Visa for that seat."
- **After:** [Call 6 — Book the seat](06-book-seat.md) with the `goldcard_authorization_id`.

The learner's card, bank account, loan or 529 plan was set up in GoldCard earlier and is referred to
only by a **token** (`fs_…`). Card and account numbers never pass through this call.

---

## The request

```http
PUT /goldcard/v1/authorizations/01J8Z3ZP4M6Q8S0U2W4Y6A8C0E HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fOGYzYTJjOTFkNyJ9.sig
X-Request-Id: e8f9a0b1-c2d3-4e4f-9a5b-6c7d8e9f0a1b
Content-Type: application/json

{
  "learner_id": "lrn_8f3a2c91d7",
  "hold_id": "hld_01J8Z3XQ2K",
  "quote_id": "qt_01J8Z3V6N4",
  "payment_mode": "prepay",
  "funding_source": { "type": "credit_card", "token": "fs_Q2w8Lk3mZ9" },
  "amount": { "amount": "241.00", "currency": "USD" }
}
```

## Every field you send

| Field | This example | Required? | What it means | Rules |
|---|---|---|---|---|
| idempotency key (in the path) | `01J8Z3ZP4M6Q8S0U2W4Y6A8C0E` | yes | makes retries safe — **a retry never charges twice** | 16–64 letters, digits, `_` or `-` |
| `Authorization` (header) | `Bearer eyJ…` | yes | sign-in token | must belong to `lrn_8f3a2c91d7` |
| `learner_id` | `lrn_8f3a2c91d7` | yes | who is paying | must own the hold and the funding source |
| `hold_id` | `hld_01J8Z3XQ2K` | yes | the seat being paid for | must be `HELD` or `OFFERED`, not expired |
| `quote_id` | `qt_01J8Z3V6N4` | yes | the price | must be the hold's quote |
| `payment_mode` | `prepay` | yes | | must match the hold |
| `funding_source.type` | `credit_card` | yes | the kind of money | `bank_ach`, `credit_card`, `loan`, `plan_529`; the school must accept it |
| `funding_source.token` | `fs_Q2w8Lk3mZ9` | yes | the learner's saved card/account/plan in GoldCard | an `fs_` token |
| `amount` | `241.00 USD` | yes | how much to authorize | must equal the quote's `due.now` exactly; `0.00` for reserve |

## How each kind of money behaves

| `funding_source.type` | What GoldCard does | `funds_status` returned | When the money actually moves |
|---|---|---|---|
| `credit_card` | places an authorization hold on the card | `authorized` | captured when the school confirms; released if it refuses |
| `bank_ach` | authorizes a bank debit | `authorized` | settles 1–3 business days after capture |
| `plan_529` | asks the 529 plan to pay the school | `committed` | when the plan pays — often days to weeks. The school may register the learner before the money arrives |
| `loan` | requests a private loan disbursement, or records that the learner relies on aid the school pays out | `committed` or `pending` | when the lender or the school's aid office pays. Federal aid is paid by the school's own aid office, not GoldCard |
| any, with `reserve` | records a guarantee; charges nothing | `pending` | only if a published no-show / late-drop fee applies |

---

## The reply — `201 Created`

```json
{
  "contract": "goldcard_v1",
  "request_id": "e8f9a0b1-c2d3-4e4f-9a5b-6c7d8e9f0a1b",
  "status": "ok",
  "payment_mode": "prepay",
  "goldcard_authorization_id": "auth_7HF2Q9",
  "goldcard_guarantee_id": null,
  "hold_id": "hld_01J8Z3XQ2K",
  "quote_id": "qt_01J8Z3V6N4",
  "authorized_amount": { "amount": "241.00", "currency": "USD" },
  "funding_source": { "type": "credit_card", "display": "Visa ending 4242" },
  "funds_status": "authorized",
  "authorization_expires_at": "2026-10-02T14:06:30Z",
  "expected_settlement_on": null
}
```

In words: **$241.00 is authorized on the Visa ending 4242**, not yet charged. The authorization lasts
7 days.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` | `goldcard_v1` | GoldCard's own rule version |
| `request_id` | `e8f9a0b1-…` | tracking number |
| `status` | `ok` | |
| `payment_mode` | `prepay` | echoed |
| `goldcard_authorization_id` | `auth_7HF2Q9` | **send this to Call 6** (prepay) |
| `goldcard_guarantee_id` | `null` | for reserve: **send this to Call 6** instead |
| `hold_id` / `quote_id` | | echoed |
| `authorized_amount` | `241.00 USD` | equals what was asked |
| `funding_source.type` | `credit_card` | echoed |
| `funding_source.display` | `Visa ending 4242` | safe to show the learner |
| `funds_status` | `authorized` | `authorized` · `committed` (529 or loan requested, not yet paid) · `pending` (reserve, or waiting on the lender) |
| `authorization_expires_at` | `2026-10-02T14:06:30Z` | 7 days for card and bank; for 529 and loan, when the plan or lender must answer by |
| `expected_settlement_on` | `null` | for bank, 529 and loan: the day the money is expected |

---

## Other replies you can get (not errors)

**Reserve — no charge, a guarantee** (`payment_mode: "reserve"`, `amount: 0.00`):

```json
{
  "contract": "goldcard_v1",
  "request_id": "e8f9a0b1-c2d3-4e4f-9a5b-6c7d8e9f0a1b",
  "status": "ok",
  "payment_mode": "reserve",
  "goldcard_authorization_id": null,
  "goldcard_guarantee_id": "gtd_4KX81M",
  "hold_id": "hld_01J8Z3XQ2K",
  "quote_id": "qt_01J8Z3V6N4",
  "authorized_amount": { "amount": "0.00", "currency": "USD" },
  "funding_source": { "type": "credit_card", "display": "Visa ending 4242" },
  "funds_status": "pending",
  "authorization_expires_at": "2027-01-12T23:59:59Z",
  "expected_settlement_on": null
}
```

**529 plan** (`funding_source.type: "plan_529"`):

```json
{
  "contract": "goldcard_v1",
  "request_id": "e8f9a0b1-c2d3-4e4f-9a5b-6c7d8e9f0a1b",
  "status": "ok",
  "payment_mode": "prepay",
  "goldcard_authorization_id": "auth_9MN3R7",
  "goldcard_guarantee_id": null,
  "hold_id": "hld_01J8Z3XQ2K",
  "quote_id": "qt_01J8Z3V6N4",
  "authorized_amount": { "amount": "241.00", "currency": "USD" },
  "funding_source": { "type": "plan_529", "display": "529 plan — account ending 7731" },
  "funds_status": "committed",
  "authorization_expires_at": "2026-10-23T00:00:00Z",
  "expected_settlement_on": "2026-10-09"
}
```

In words: the withdrawal has been requested; the plan is expected to pay the school by 9 October.
Booking can go ahead — the school decides whether to register before the money arrives.

---

## Errors

Example — the card was declined:

```json
{
  "contract": "goldcard_v1",
  "request_id": "e8f9a0b1-c2d3-4e4f-9a5b-6c7d8e9f0a1b",
  "status": "error",
  "error": {
    "code": "PAYMENT_DECLINED",
    "message": "Visa ending 4242 was declined: insufficient funds.",
    "retry": "reauthorize",
    "field": "funding_source",
    "details": { "display": "Visa ending 4242", "decline": "insufficient_funds" }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `funding_source is required.` | left out (same message for every required field) | send it |
| 400 | `VALIDATION_FAILED` | `amount must be {"amount":"0.00","currency":"USD"} form with two decimal places; got '241'.` | a bare number sent | send `{ "amount": "241.00", "currency": "USD" }` |
| 400 | `VALIDATION_FAILED` | `funding_source.type must be one of bank_ach, credit_card, loan, plan_529; got 'visa'.` | | `credit_card` |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_8f3a2c91d7.` | | send the matching pair |
| 404 | `HOLD_NOT_FOUND` | `No hold hld_01J8Z3ZZ99 is held for learner lrn_8f3a2c91d7.` | | check the id |
| 404 | `FUNDING_SOURCE_NOT_FOUND` | `No funding source fs_Q2w8Lk3mZ0 is on file for learner lrn_8f3a2c91d7. Add it in GoldCard first.` | card not saved, or removed | add it in GoldCard |
| 409 | `HOLD_NOT_ACTIVE` | `Hold hld_01J8Z3XQ2K is WAITLISTED; only a HELD or OFFERED hold can be paid for.` | no seat yet | wait for `OFFERED` |
| 409 | `HOLD_NOT_ACTIVE` | `Hold hld_01J8Z3XQ2K is RELEASED; only a HELD or OFFERED hold can be paid for.` | learner let it go | hold again |
| 410 | `HOLD_EXPIRED` | `Hold hld_01J8Z3XQ2K expired at 2026-09-25T14:24:05Z. Hold a seat again.` | more than 20 minutes | Call 3 again |
| 422 | `AMOUNT_MISMATCH` | `amount $214.00 does not equal the amount due now on quote qt_01J8Z3V6N4: $241.00.` | typed wrong | send the `amount_due_now` from Call 3 |
| 422 | `QUOTE_HOLD_MISMATCH` | `Quote qt_01J8Z3V6N9 is not the quote on hold hld_01J8Z3XQ2K (qt_01J8Z3V6N4).` | | send the hold's quote |
| 422 | `PAYMENT_MODE_MISMATCH` | `Hold hld_01J8Z3XQ2K is for prepay, not reserve.` | | match the hold |
| 422 | `FUNDING_SOURCE_NOT_ACCEPTED` | `School 225070 does not accept plan_529 through GoldCard. Accepted: credit_card, bank_ach.` | | another source |
| 402 | `PAYMENT_DECLINED` | `Visa ending 4242 was declined: insufficient funds.` | | another source |
| 402 | `PAYMENT_DECLINED` | `Visa ending 4242 was declined: the card has expired.` | | update the card |
| 402 | `PAYMENT_DECLINED` | `Visa ending 4242 was declined: the issuer declined without a reason; contact the card issuer.` | | call the bank |
| 402 | `PAYMENT_DECLINED` | `529 plan — account ending 7731 was declined: the 529 plan refused the withdrawal: annual withdrawal limit reached.` | the plan said no | another source |
| 402 | `PAYMENT_DECLINED` | `Loan — Example Lender was declined: the lender declined the disbursement: enrollment not yet verified.` | lender needs proof of enrollment | reserve instead, then pay the school when the loan comes |
| 402 | `PAYMENT_DECLINED` | `Checking account ending 0012 was declined: the bank account could not be verified.` | | verify the account in GoldCard |
| 422 | `IDEMPOTENCY_CONFLICT` | `Key 01J8Z3ZP4M6Q8S0U2W4Y6A8C0E was already used for a different authorization request on 2026-09-25T14:06:30Z. Use a new key.` | | new key |
| 503 | `PAYMENT_PROVIDER_OFFLINE` | `The credit_card provider did not answer. Nothing was authorized. Try again shortly.` | card network down | retry with the **same key** |
| 500 | `INTERNAL_ERROR` | `GoldCard could not complete the request. Nothing was authorized. Reference e8f9a0b1-c2d3-4e4f-9a5b-6c7d8e9f0a1b.` | | retry with the same key |

## What gets recorded

In **GoldCard's own store**, not the GoldWire bucket. GoldCard reads the quote and hold records from
GoldWire's S3 bucket to check the amount and the hold, and writes nothing there.
