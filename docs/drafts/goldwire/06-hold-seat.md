# Call 6 — Hold a seat

[← Call 5 — Get person](05-get-person.md) · [Summary](README.md) · Next: [Call 7 — Get a hold →](07-get-hold.md)

> **Draft.** Nothing here is live.

## What it does

Takes **one seat out of the section for 20 minutes** at the quoted price, while the learner pays. It
is the airline's "hold this fare": no one else can take that seat until the hold is booked, released,
or runs out.

If the last seat went between the quote and now, the learner can **join the waitlist** instead (if
they asked to), or is told the section is **full** — that is a normal answer, not an error.

## Where it sits

- **Before:** [Call 4](04-get-section-fees.md) gave quote `qt_01J8Z3V6N4`: $241.00, prepay, fixed until 14:33:40.
- **This call:** "Hold me a seat in section 002 at that price."
- **After:** pay for it → [Call 8 — GoldCard authorize](08-goldcard-authorize.md), then [Call 9 — Book the seat](09-book-seat.md). The learner can watch the clock with [Call 7](07-get-hold.md).

---

## The request

The last part of the path is an **idempotency key** — any unique string the client makes up. If the
network drops and the client sends the same request again with the same key, GoldWire returns the
first answer and **does not take a second seat**.

```http
PUT /goldwire/v1/holds/01J8Z3XH7TQ5W2A9R6C4M0PBNE HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fOGYzYTJjOTFkNyJ9.sig
X-Request-Id: b2e7f4a9-1c6d-4e38-8f52-0a7b3d9e6c15
Content-Type: application/json

{
  "learner_id": "lrn_8f3a2c91d7",
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "quote_id": "qt_01J8Z3V6N4",
  "payment_mode": "prepay",
  "goldcheck_ref": "gck_5TR20P",
  "accept_waitlist": true
}
```

## Every field you send

| Field | This example | Required? | What it means | Rules |
|---|---|---|---|---|
| idempotency key (in the path) | `01J8Z3XH7TQ5W2A9R6C4M0PBNE` | yes | makes retries safe | 16–64 letters, digits, `_` or `-`; new for every new hold; kept 24 hours |
| `Authorization` (header) | `Bearer eyJ…` | yes | the learner's sign-in token | must belong to `lrn_8f3a2c91d7` |
| `learner_id` | `lrn_8f3a2c91d7` | yes | who is holding | must own the quote |
| `section_id` | `sec_225070_2027SP_ENGL1301_002` | yes | which section | must be the quote's section |
| `quote_id` | `qt_01J8Z3V6N4` | yes | the fixed price from Call 4 | must not be expired; must have a price (residency set) |
| `payment_mode` | `prepay` | yes | how it will be paid | must match the quote |
| `goldcheck_ref` | `gck_5TR20P` | recommended | the GoldCheck answer the learner relied on | kept with the hold and the booking; not re-checked |
| `accept_waitlist` | `true` | no (default `false`) | if the section just filled, put me on the waitlist | `true` / `false` |

No other fields are accepted.

## What GoldWire checks, in order

1. The key: used before with the same body → return the first answer. Used before with a different body → `IDEMPOTENCY_CONFLICT`.
2. The token and every field.
3. The quote: it exists and is this learner's → it has not expired → it is for this section → it is for this payment mode → it has a price.
4. The price: re-reads the school's fees. If the school changed them since the quote → `PRICE_CHANGED`.
5. The learner: not already holding or booked in **any section of this course this term**, and fewer than 5 active holds overall.
6. The section: still bookable, not cancelled.
7. **Takes the seat** in one indivisible step ("take one if more than zero are left"), so two learners can never get the last seat.
8. Writes the hold to S3. If that write fails, the seat is given back.

---

## The reply — `201 Created`

```json
{
  "contract": "goldwire_v1",
  "request_id": "b2e7f4a9-1c6d-4e38-8f52-0a7b3d9e6c15",
  "status": "ok",
  "as_of": "2026-09-25T14:04:05Z",
  "not_held": [],
  "hold_id": "hld_01J8Z3XQ2K",
  "hold_state": "HELD",
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "quote_id": "qt_01J8Z3V6N4",
  "payment_mode": "prepay",
  "goldcheck_ref": "gck_5TR20P",
  "hold_expires_at": "2026-09-25T14:24:05Z",
  "waitlist_position": null,
  "waitlist_full": null,
  "seats_after_hold": { "capacity": 25, "enrolled": 21, "held": 3, "available": 1 },
  "next": {
    "service": "goldcard.authorize",
    "payment_mode": "prepay",
    "amount_due_now": { "amount": "241.00", "currency": "USD" }
  },
  "s3": {
    "key": "goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/holds/hld_01J8Z3XQ2K.json",
    "etag": "\"a90b9d0e2b7c41a5f6\""
  }
}
```

In words: **the seat is held until 14:24:05** (20 minutes). One seat is left for anyone else. Next,
GoldCard must authorize **$241.00**.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` / `as_of` | | rule version, tracking number, when the seat was taken |
| `status` | `ok` | |
| `hold_id` | `hld_01J8Z3XQ2K` | **the hold — send this to Call 8 and Call 9** |
| `hold_state` | `HELD` | `HELD` = a seat is yours for now · `WAITLISTED` = no seat, you are on the waitlist · `SECTION_FULL` = no seat and no waitlist place; nothing was held |
| `section_id` / `quote_id` / `payment_mode` / `goldcheck_ref` | | echoed |
| `hold_expires_at` | `2026-09-25T14:24:05Z` | book before this or the seat goes back |
| `waitlist_position` | `null` | your place, when `WAITLISTED` |
| `waitlist_full` | `null` | when `SECTION_FULL`: `true` if the waitlist was full too |
| `seats_after_hold` | 25 / 21 / 3 / 1 | the section after this hold: capacity, enrolled, held (now including yours), available |
| `next.service` | `goldcard.authorize` | what to call next |
| `next.amount_due_now` | `241.00` | exactly what to authorize (`0.00` for reserve) |
| `s3.key` / `s3.etag` | `…/holds/hld_01J8Z3XQ2K.json` | where the hold is recorded |

**Same request sent twice** (same key, same body): `200 OK`, header `Idempotent-Replayed: true`, and
exactly the reply above. No second seat.

---

## Other replies you can get (not errors)

**The last seat went; the learner said `accept_waitlist: true`** — `201 Created`:

```json
{
  "contract": "goldwire_v1",
  "request_id": "b2e7f4a9-1c6d-4e38-8f52-0a7b3d9e6c15",
  "status": "ok",
  "as_of": "2026-09-25T14:04:05Z",
  "not_held": [],
  "hold_id": "hld_01J8Z3XQ2K",
  "hold_state": "WAITLISTED",
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "quote_id": "qt_01J8Z3V6N4",
  "payment_mode": "prepay",
  "goldcheck_ref": "gck_5TR20P",
  "hold_expires_at": null,
  "waitlist_position": 1,
  "waitlist_full": null,
  "seats_after_hold": { "capacity": 25, "enrolled": 23, "held": 2, "available": 0 },
  "next": { "service": null, "payment_mode": "prepay", "amount_due_now": { "amount": "0.00", "currency": "USD" } },
  "s3": { "key": "goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/holds/hld_01J8Z3XQ2K.json", "etag": "\"a90b9d0e2b7c41a5f6\"" }
}
```

In words: first on the waitlist. Nothing to pay now. If a seat opens, this hold becomes `OFFERED` for
24 hours (see [Call 7](07-get-hold.md)).

**The last seat went; `accept_waitlist` was `false`** — `201 Created`, `hold_state: "SECTION_FULL"`,
no `hold_id`, no `s3`, `seats_after_hold.available: 0`. Nothing was held. Pick another section.

---

## Errors

Example — the school raised its fees after the quote:

```json
{
  "contract": "goldwire_v1",
  "request_id": "b2e7f4a9-1c6d-4e38-8f52-0a7b3d9e6c15",
  "status": "error",
  "error": {
    "code": "PRICE_CHANGED",
    "message": "The school's price for section sec_225070_2027SP_ENGL1301_002 changed after quote qt_01J8Z3V6N4 was issued: was $241.00, now $256.00. Request a new quote.",
    "retry": "new_quote",
    "field": null,
    "details": { "quote_id": "qt_01J8Z3V6N4", "old_total": "241.00", "new_total": "256.00", "currency": "USD" }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `idempotency_key must be 16–64 characters of letters, digits, _ or -; got 'abc'.` | key too short | make a longer key (a UUID works) |
| 400 | `VALIDATION_FAILED` | `quote_id is required.` | left out (same message for `learner_id`, `section_id`, `payment_mode`) | send it |
| 400 | `VALIDATION_FAILED` | `quote_id has the wrong format; got 'Q-01J8Z3V6N4'.` | | send the `qt_` id |
| 400 | `VALIDATION_FAILED` | `payment_mode must be reserve or prepay; got 'credit_card'.` | | `prepay` |
| 400 | `VALIDATION_FAILED` | `accept_waitlist must be true or false.` | sent `"yes"` | `true` |
| 400 | `VALIDATION_FAILED` | `Unknown field 'price'. hold_seat accepts learner_id, section_id, quote_id, payment_mode, goldcheck_ref, accept_waitlist.` | a price sent by the client | drop it — the price comes from the quote |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_8f3a2c91d7.` | | send the matching pair |
| 404 | `QUOTE_NOT_FOUND` | `No quote qt_01J8Z3V6N9 is held for learner lrn_8f3a2c91d7.` | wrong id, or another learner's quote | get a new quote |
| 404 | `SECTION_NOT_FOUND` | `No section sec_225070_2027SP_ENGL1301_009 is held. List sections with get_sections.` | | run Call 3 |
| 404 | `GOLDCHECK_REF_NOT_FOUND` | `No GoldCheck answer gck_9ZZ00Q is held for learner lrn_8f3a2c91d7.` | | drop it or fix it |
| 409 | `PRICE_CHANGED` | `The school's price for section sec_225070_2027SP_ENGL1301_002 changed after quote qt_01J8Z3V6N4 was issued: was $241.00, now $256.00. Request a new quote.` | school changed its fees | new quote; show the learner the difference |
| 409 | `ACTIVE_HOLD_EXISTS` | `Learner lrn_8f3a2c91d7 already holds a seat in ENGL 1301 for 2027-SP (hold hld_01J8Z3W1AB, section 004, expires 2026-09-25T14:15:00Z). Release it first or book it.` | holding another section of the same course | release that hold ([Call 11](11-release-seat.md)) or book it |
| 409 | `ALREADY_BOOKED` | `Learner lrn_8f3a2c91d7 already has booking bkg_01J8Y9P2QR (CONFIRMED) in ENGL 1301 for 2027-SP.` | already registered in this course | nothing to do, or drop that booking first |
| 409 | `HOLD_LIMIT_REACHED` | `Learner lrn_8f3a2c91d7 has 5 active holds, the most allowed. Book or release one first.` | | release one |
| 409 | `SECTION_CANCELLED` | `Section 002 of ENGL 1301 (sec_225070_2027SP_ENGL1301_002) was cancelled by the school on 2026-10-14.` | | another section |
| 409 | `SECTION_NOT_BOOKABLE` | `Section sec_225070_2027SP_ENGL1301_002 cannot be booked through GoldWire: registration closed on 2027-01-26.` | | contact the school |
| 410 | `QUOTE_EXPIRED` | `Quote qt_01J8Z3V6N4 expired at 2026-09-25T14:33:40Z. Request a new quote for section sec_225070_2027SP_ENGL1301_002.` | more than 30 minutes since Call 4 | new quote |
| 422 | `QUOTE_SECTION_MISMATCH` | `Quote qt_01J8Z3V6N4 is for section sec_225070_2027SP_ENGL1301_002, not sec_225070_2027SP_ENGL1301_005.` | quote from one section, hold on another | quote the section you want |
| 422 | `PAYMENT_MODE_MISMATCH` | `Quote qt_01J8Z3V6N4 was priced for prepay, not reserve. Request a quote for reserve.` | changed mind about how to pay | new quote |
| 422 | `QUOTE_HAS_NO_PRICE` | `Quote request for section sec_225070_2027SP_ENGL1301_002 had residency unknown, so no price was fixed. Request a quote with residency set.` | | new quote with residency |
| 422 | `IDEMPOTENCY_CONFLICT` | `Key 01J8Z3XH7TQ5W2A9R6C4M0PBNE was already used for a different hold request on 2026-09-25T14:04:05Z. Use a new key.` | key reused for something else | new key |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | | wait |
| 503 | `SEAT_STORE_UNAVAILABLE` | `Seats could not be counted just now, so no seat was held. Try again shortly.` | | retry with the **same key** |
| 503 | `SCHOOL_OFFLINE` | `School 225070 did not answer, so the waitlist could not be joined. No seat was held. Try again shortly.` | only when joining a waitlist | retry with the same key |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference b2e7f4a9-1c6d-4e38-8f52-0a7b3d9e6c15.` | | retry with the same key |

## What gets recorded in S3

Two files, each written once (`If-None-Match: *`):

1. `goldwire/idempotency/01J8Z3XH7TQ5W2A9R6C4M0PBNE.json` — the key, a fingerprint of the request body, and the reply. This is what makes a retry return the same answer.
2. `goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/holds/hld_01J8Z3XQ2K.json` — the hold: learner, section, quote, payment mode, GoldCheck ref, state `HELD`, expiry.

Later changes to the hold (`BOOKED`, `EXPIRED`, `RELEASED`, `OFFERED`) are written as **new versions**
of file 2, never edits. The seat count itself is kept in a separate fast store, not in S3.
