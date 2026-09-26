# Call 7 — Get a hold

[← Call 6 — Hold a seat](06-hold-seat.md) · [Summary](README.md) · Next: [Call 8 — GoldCard authorize →](08-goldcard-authorize.md)

> **Draft.** Nothing here is live.

## What it does

Reads one hold: **what state it is in and how many seconds are left.** Use it to show the learner a
countdown while they pay, and to find out when a waitlist place has turned into a seat.

It changes nothing.

## Where it sits

- **Before:** [Call 6](06-hold-seat.md) made hold `hld_01J8Z3XQ2K`, expiring 14:24:05.
- **This call:** "Is my seat still held? How long do I have?"
- **After:** keep going with [Call 8](08-goldcard-authorize.md) and [Call 9](09-book-seat.md) — or, if the hold expired, start again at [Call 6](06-hold-seat.md).

---

## The request

```http
GET /goldwire/v1/holds/hld_01J8Z3XQ2K?learner_id=lrn_8f3a2c91d7 HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fOGYzYTJjOTFkNyJ9.sig
X-Request-Id: 3d8e1f02-6a4b-4c7d-9e10-2f3a4b5c6d7e
Accept: application/json
```

## Every field you send

| Field | This example | Required? | What it means | Rules |
|---|---|---|---|---|
| `hold_id` (in the path) | `hld_01J8Z3XQ2K` | yes | the hold, from Call 6 | an `hld_` id |
| `learner_id` | `lrn_8f3a2c91d7` | yes | who is asking | must own the hold |
| `Authorization` (header) | `Bearer eyJ…` | yes | sign-in token | must belong to `lrn_8f3a2c91d7` |

---

## The reply — `200 OK`

Read at 14:09:05, five minutes after the hold was taken:

```json
{
  "contract": "goldwire_v1",
  "request_id": "3d8e1f02-6a4b-4c7d-9e10-2f3a4b5c6d7e",
  "status": "ok",
  "as_of": "2026-09-25T14:09:05Z",
  "source": { "kind": "goldwire_ledger" },
  "not_held": [],
  "hold_id": "hld_01J8Z3XQ2K",
  "hold_state": "HELD",
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "quote_id": "qt_01J8Z3V6N4",
  "payment_mode": "prepay",
  "goldcheck_ref": "gck_5TR20P",
  "hold_expires_at": "2026-09-25T14:24:05Z",
  "seconds_left": 900,
  "waitlist_position": null,
  "booking_id": null,
  "history": [
    { "at": "2026-09-25T14:04:05Z", "state": "HELD", "by": "learner" }
  ],
  "s3": {
    "key": "goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/holds/hld_01J8Z3XQ2K.json",
    "etag": "\"a90b9d0e2b7c41a5f6\"",
    "version_id": "Yk2P0sQz8Rr1uT4vW7xA.bC3dE6fG9hJ"
  }
}
```

In words: still held, **15 minutes left**.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` / `as_of` | | rule version, tracking number, when read |
| `source.kind` | `goldwire_ledger` | read from GoldWire's own record, not from the school |
| `hold_id` | `hld_01J8Z3XQ2K` | |
| `hold_state` | `HELD` | see the table below |
| `section_id` / `quote_id` / `payment_mode` / `goldcheck_ref` | | as recorded in Call 6 |
| `hold_expires_at` | `2026-09-25T14:24:05Z` | for `HELD` and `OFFERED` |
| `seconds_left` | `900` | until expiry; never negative; `null` when not `HELD`/`OFFERED` |
| `waitlist_position` | `null` | for `WAITLISTED` |
| `booking_id` | `null` | for `BOOKED`: the booking made from this hold |
| `history[]` | one entry | every state the hold has been in: when, what, and who caused it (`learner`, `goldwire`, `school`) |
| `s3.key` / `.etag` / `.version_id` | | the current version of the hold record |

**What each `hold_state` means**

| `hold_state` | Means | What the learner does |
|---|---|---|
| `HELD` | the seat is theirs until `hold_expires_at` | pay (Call 8) and book (Call 9) |
| `WAITLISTED` | on the waitlist; no seat yet | wait; check back |
| `OFFERED` | a seat opened for them from the waitlist; held **24 hours** | pay (Call 8) and book (Call 9) |
| `BOOKED` | turned into a booking | read it with [Call 10](10-get-booking.md) using `booking_id` |
| `EXPIRED` | time ran out; the seat went back | hold again (Call 6) |
| `RELEASED` | the learner let it go | nothing |

---

## Other replies you can get (not errors)

**A waitlist place became a seat** — the learner was waitlisted at 14:04; someone dropped on 3 October:

```json
{
  "contract": "goldwire_v1",
  "request_id": "5f6a7b8c-9d0e-4f1a-8b2c-3d4e5f6a7b8c",
  "status": "ok",
  "as_of": "2026-10-03T16:20:00Z",
  "source": { "kind": "goldwire_ledger" },
  "not_held": [],
  "hold_id": "hld_01J8Z3XQ2K",
  "hold_state": "OFFERED",
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "quote_id": "qt_01J8Z3V6N4",
  "payment_mode": "prepay",
  "goldcheck_ref": "gck_5TR20P",
  "hold_expires_at": "2026-10-04T16:15:00Z",
  "seconds_left": 86100,
  "waitlist_position": null,
  "booking_id": null,
  "history": [
    { "at": "2026-09-25T14:04:05Z", "state": "WAITLISTED", "by": "learner" },
    { "at": "2026-10-03T16:15:00Z", "state": "OFFERED",    "by": "goldwire" }
  ],
  "s3": { "key": "goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/holds/hld_01J8Z3XQ2K.json", "etag": "\"c01d9d0e2b7c41a5f6\"", "version_id": "Qm7R2sT5uV8wX1yZ.aB4cD7eF0gH3iJ" }
}
```

The learner is also notified. If the price changed since the quote, Call 8 will say so and a new
quote is needed.

**The hold ran out** — `hold_state: "EXPIRED"`, `seconds_left: null`, and `history` gains
`{ "at": "2026-09-25T14:24:05Z", "state": "EXPIRED", "by": "goldwire" }`.

---

## Errors

Example — someone else's hold:

```json
{
  "contract": "goldwire_v1",
  "request_id": "3d8e1f02-6a4b-4c7d-9e10-2f3a4b5c6d7e",
  "status": "error",
  "error": {
    "code": "HOLD_NOT_FOUND",
    "message": "No hold hld_01J8Z3ZZ99 is held for learner lrn_8f3a2c91d7.",
    "retry": "fix_request",
    "field": "hold_id",
    "details": { "hold_id": "hld_01J8Z3ZZ99", "learner_id": "lrn_8f3a2c91d7" }
  }
}
```

The same `HOLD_NOT_FOUND` is returned whether the hold does not exist or belongs to another learner,
so the service never reveals someone else's hold.

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `hold_id must look like hld_…; got '01J8Z3XQ2K'.` | prefix missing | send `hld_01J8Z3XQ2K` |
| 400 | `VALIDATION_FAILED` | `learner_id is required.` | left out | send it |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_8f3a2c91d7.` | | send the matching pair |
| 404 | `HOLD_NOT_FOUND` | `No hold hld_01J8Z3ZZ99 is held for learner lrn_8f3a2c91d7.` | wrong id, or another learner's | check the id |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 2 seconds.` | polling more than once a second | poll every 5–10 seconds |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference 3d8e1f02-6a4b-4c7d-9e10-2f3a4b5c6d7e.` | | try again |

## What gets recorded in S3

Nothing. It reads `…/holds/hld_01J8Z3XQ2K.json` and its list of versions (for `history`).
