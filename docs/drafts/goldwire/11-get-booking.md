# Call 11 — Get a booking

[← Call 10 — Book the seat](10-book-seat.md) · [Summary](README.md) · Next: [Call 12 — Release a seat →](12-release-seat.md)

> **Draft.** Nothing here is live.

## What it does

Answers **"Am I in?"** — returns the booking exactly as last recorded, with every state it has been
in, who changed it, and when. It reads the record; it does not ask the school again or recalculate
anything. Use it after a booking came back `SUBMITTED`, to see whether the school has answered.

It changes nothing.

## Where it sits

- **Before:** [Call 10](10-book-seat.md) made booking `bkg_01J8Z42M7R`.
- **This call:** "What is the state of my booking?"
- **After:** nothing, or [Call 12](12-release-seat.md) to drop.

---

## The request

```http
GET /goldwire/v1/bookings/bkg_01J8Z42M7R?learner_id=lrn_8f3a2c91d7 HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fOGYzYTJjOTFkNyJ9.sig
X-Request-Id: 4a5b6c7d-8e9f-4a0b-9c1d-2e3f4a5b6c7d
Accept: application/json
```

## Every field you send

| Field | This example | Required? | What it means | Rules |
|---|---|---|---|---|
| `booking_id` (in the path) | `bkg_01J8Z42M7R` | yes | the booking, from Call 10 | a `bkg_` id |
| `learner_id` | `lrn_8f3a2c91d7` | yes | who is asking | must own the booking |
| `Authorization` (header) | `Bearer eyJ…` | yes | sign-in token | must belong to `lrn_8f3a2c91d7` |

---

## The reply — `200 OK`

The reply also carries the header `ETag: "e4d29d0e2b7c41a5f6"` — the version of the record read.

Here the learner paid with a 529 plan instead of the card (the second example in [Call 9](09-goldcard-authorize.md#other-replies-you-can-get-not-errors)), so the school answered "pending" first and confirmed when the plan paid:

```json
{
  "contract": "goldwire_v1",
  "request_id": "4a5b6c7d-8e9f-4a0b-9c1d-2e3f4a5b6c7d",
  "status": "ok",
  "as_of": "2026-10-09T15:40:12Z",
  "source": { "kind": "goldwire_ledger" },
  "not_held": [],
  "booking_id": "bkg_01J8Z42M7R",
  "booking_state": "CONFIRMED",
  "confirmation_number": "GW-225070-7Q4K-2M",
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "hold_id": "hld_01J8Z3XQ2K",
  "goldcheck_ref": "gck_5TR20P",
  "course": { "code": "ENGL 1301", "title": "Composition I", "credits": 3 },
  "section": {
    "section_number": "002",
    "crn": "21457",
    "term_id": "2027-SP",
    "meetings": [ { "days": ["TUE", "THU"], "start": "09:30", "end": "10:50", "room": "LA 114" } ],
    "starts_on": "2027-01-19"
  },
  "school": {
    "unitid": "225070",
    "answer": "registered",
    "reason": null,
    "registration_ref": "202720.21457",
    "waitlist_position": null,
    "answered_at": "2026-10-09T15:40:10Z"
  },
  "payment": {
    "mode": "prepay",
    "goldcard_authorization_id": "auth_9MN3R7",
    "goldcard_guarantee_id": null,
    "amount": { "amount": "241.00", "currency": "USD" },
    "funds_status": "captured",
    "capture": "on_confirmation"
  },
  "refund_policy": { "full_refund_until": "2027-02-02", "source": "pol_225070_refunds_2026_27" },
  "price_changes": [],
  "history": [
    { "at": "2026-09-25T14:11:50Z", "state": "SUBMITTED", "by": "learner",  "version_id": "0pQ1rT7uVx2yZa3bCd4eFg5hIj6kLm7nO", "note": null },
    { "at": "2026-09-25T14:11:52Z", "state": "SUBMITTED", "by": "school",   "version_id": "5sT6uV7wX8yZ9aB0.cD1eF2gH3iJ4kL5", "note": "pending: Awaiting 529 plan payment" },
    { "at": "2026-10-09T15:40:10Z", "state": "CONFIRMED", "by": "school",   "version_id": "3HL4kqtJlcpXroDTDmJ.rmSpXd3dIbrHY", "note": "registered, registration_ref 202720.21457" }
  ],
  "s3": {
    "key": "goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/bookings/bkg_01J8Z42M7R.json",
    "etag": "\"e4d29d0e2b7c41a5f6\"",
    "version_id": "3HL4kqtJlcpXroDTDmJ.rmSpXd3dIbrHY"
  }
}
```

In words: **confirmed on 9 October** when the 529 plan paid. Tue/Thu 9:30 in LA 114 from 19 January.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` / `as_of` | | rule version, tracking number, when read |
| `source.kind` | `goldwire_ledger` | from GoldWire's record, not a fresh question to the school |
| `booking_id` | `bkg_01J8Z42M7R` | |
| `booking_state` | `CONFIRMED` | see the table below |
| `confirmation_number` | `GW-225070-7Q4K-2M` | only when `CONFIRMED` (kept after a drop, for the record) |
| `section_id` / `hold_id` / `goldcheck_ref` | | carried through |
| `course.code` / `.title` / `.credits` | `ENGL 1301` / `Composition I` / `3` | copied in at booking so this reply reads on its own |
| `section.section_number` / `.crn` / `.term` / `.meetings` / `.starts_on` | `002` / `21457` / `2027-SP` / Tue/Thu 9:30 / 19 Jan | the same |
| `school.answer` | `registered` | the school's latest answer |
| `school.reason` | `null` | the school's words, if it refused or is pending |
| `school.registration_ref` | `202720.21457` | the school's own reference |
| `school.answered_at` | `2026-10-09T15:40:10Z` | |
| `payment.funds_status` | `captured` | where the money stands now: `authorized`, `committed`, `pending`, `captured`, `released` |
| `payment.capture` | `on_confirmation` | |
| `refund_policy` | until 2027-02-02 | |
| `price_changes[]` | `[]` | if the school later changed the price (e.g. re-rated residency): each `{ from, to, reason, at, approved }` |
| `history[]` | three entries | oldest first: when, which state, who (`learner`, `school`, `goldwire`, `goldcard`), the record version, and a note |
| `s3.key` / `.etag` / `.version_id` | | the current version |

**What each `booking_state` means**

| `booking_state` | Means |
|---|---|
| `SUBMITTED` | sent to the school; its answer is pending, or it hasn't answered yet |
| `CONFIRMED` | the school says the learner is registered |
| `WAITLISTED` | the school put the learner on its waitlist |
| `REJECTED` | the school said no; `school.reason` says why |
| `DROP_REQUESTED` | the learner asked to drop; waiting for the school |
| `DROPPED` | the school dropped the registration |

---

## Errors

Example:

```json
{
  "contract": "goldwire_v1",
  "request_id": "4a5b6c7d-8e9f-4a0b-9c1d-2e3f4a5b6c7d",
  "status": "error",
  "error": {
    "code": "BOOKING_NOT_FOUND",
    "message": "No booking bkg_01J8Z42M7X is held for learner lrn_8f3a2c91d7.",
    "retry": "fix_request",
    "field": "booking_id",
    "details": { "booking_id": "bkg_01J8Z42M7X", "learner_id": "lrn_8f3a2c91d7" }
  }
}
```

The same `BOOKING_NOT_FOUND` is returned whether the booking doesn't exist or belongs to someone else.

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `booking_id must look like bkg_…; got 'GW-225070-7Q4K-2M'.` | confirmation number sent instead of the booking id | send `bkg_01J8Z42M7R` |
| 400 | `VALIDATION_FAILED` | `learner_id is required.` | left out | send it |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_8f3a2c91d7.` | | send the matching pair |
| 404 | `BOOKING_NOT_FOUND` | `No booking bkg_01J8Z42M7X is held for learner lrn_8f3a2c91d7.` | wrong id, or someone else's | check the id |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | | wait |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference 4a5b6c7d-8e9f-4a0b-9c1d-2e3f4a5b6c7d.` | | try again |

## What gets recorded in S3

Nothing. It reads `…/bookings/bkg_01J8Z42M7R.json` and its list of versions (for `history`).
