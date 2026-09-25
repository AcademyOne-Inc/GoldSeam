# Call 8 — Release a seat

[← Call 7 — Get a booking](07-get-booking.md) · [Summary](README.md) · Next: [Call 9 — Record the school's answer →](09-record-school-answer.md)

> **Draft.** Nothing here is live.

## What it does

Gives a seat back. One call covers three situations:

- **Let go of a hold** before booking — the seat goes back at once, any payment authorization is released.
- **Leave a waitlist.**
- **Drop a booking** — GoldWire asks the school to drop the registration, and says what the school's
  published refund policy gives back **today**.

The school issues any refund and decides any drop after its deadline. GoldWire reports what the
policy says and never promises more.

## Where it sits

- **Before:** a hold from [Call 3](03-hold-seat.md), or a booking from [Call 6](06-book-seat.md).
- **This call:** "I don't want this seat any more."
- **After:** check the result with [Call 7](07-get-booking.md) if the drop is still waiting on the school.

---

## The request

Dropping the confirmed booking on 20 January 2027, after classes started but before the
full-refund date:

```http
PUT /goldwire/v1/releases/01J9A7K3M5P7R9T1V3X5Z7B9D1 HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fOGYzYTJjOTFkNyJ9.sig
X-Request-Id: 6b7c8d9e-0f1a-4b2c-8d3e-4f5a6b7c8d9e
Content-Type: application/json

{
  "learner_id": "lrn_8f3a2c91d7",
  "booking_id": "bkg_01J8Z42M7R",
  "reason": "schedule_conflict",
  "note": "Took a job with morning hours."
}
```

## Every field you send

| Field | This example | Required? | What it means | Rules |
|---|---|---|---|---|
| idempotency key (in the path) | `01J9A7K3M5P7R9T1V3X5Z7B9D1` | yes | makes retries safe | 16–64 letters, digits, `_` or `-` |
| `Authorization` (header) | `Bearer eyJ…` | yes | sign-in token | must belong to `lrn_8f3a2c91d7` |
| `learner_id` | `lrn_8f3a2c91d7` | yes | who is releasing | must own the hold or booking |
| `hold_id` | *(not sent)* | **one of these two** | to let go of a hold or leave a waitlist | an `hld_` id |
| `booking_id` | `bkg_01J8Z42M7R` | **one of these two** | to drop a booking | a `bkg_` id |
| `reason` | `schedule_conflict` | yes | why; recorded and sent to the school on a drop | `changed_mind`, `schedule_conflict`, `chose_other_section`, `financial`, `other` |
| `note` | `Took a job with morning hours.` | only when `reason` is `other` | free text | up to 500 characters |

## What happens, by what you release

| You release | Its state now | What GoldWire does | `released_state` | Money |
|---|---|---|---|---|
| a hold | `HELD` or `OFFERED` | seat back immediately | `RELEASED` | GoldCard authorization released |
| a hold | `WAITLISTED` | removed from the school's waitlist | `RELEASED` | nothing was held |
| a booking | `SUBMITTED` | withdrawal sent to the school | `DROP_REQUESTED` | authorization released when the school confirms |
| a booking | `WAITLISTED` | removed from the school's waitlist | `RELEASED` | nothing was held |
| a booking | `CONFIRMED` | drop sent to the school | `DROPPED` if the school answers at once, else `DROP_REQUESTED` | refund per the school's policy for today's date |

---

## The reply — `201 Created`

```json
{
  "contract": "goldwire_v1",
  "request_id": "6b7c8d9e-0f1a-4b2c-8d3e-4f5a6b7c8d9e",
  "status": "ok",
  "as_of": "2027-01-20T16:05:31Z",
  "not_held": [],
  "hold_id": null,
  "booking_id": "bkg_01J8Z42M7R",
  "released_state": "DROPPED",
  "note": null,
  "refund": {
    "amount": { "amount": "241.00", "currency": "USD" },
    "percent": 100,
    "source": "pol_225070_refunds_2026_27",
    "paid_by": "goldcard"
  },
  "fee_due": null,
  "seats_after_release": { "capacity": 25, "enrolled": 24, "held": 0, "available": 1 },
  "s3": {
    "key": "goldwire/225070/2027SP/sec_225070_2027SP_ENGL1301_002/bookings/bkg_01J8Z42M7R.json",
    "etag": "\"a1b29d0e2b7c41a5f6\"",
    "version_id": "9Lm0nOp1qRs2tUv3.wXy4zA5bC6dE7fG"
  }
}
```

In words: **dropped.** Before 2 February, so the school's policy gives a **100% refund — $241.00**,
returned to the card through GoldCard.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` / `as_of` | | rule version, tracking number, when recorded |
| `status` | `ok` | `offline` if the school didn't answer the drop (see below) |
| `hold_id` / `booking_id` | `null` / `bkg_01J8Z42M7R` | whichever was released |
| `released_state` | `DROPPED` | `RELEASED`, `DROP_REQUESTED` or `DROPPED` |
| `note` | `null` | set when the school didn't answer |
| `refund.amount` | `241.00 USD` | what the school's policy gives today; `null` for a hold (nothing was taken) |
| `refund.percent` | `100` | 0–100, from the policy |
| `refund.source` | `pol_225070_refunds_2026_27` | the policy the refund came from |
| `refund.paid_by` | `goldcard` | `goldcard` (reversed to the card/account/plan) or `school` (the school refunds on its own account) |
| `fee_due` | `null` | reserve only: a published late-drop fee, `{ amount, source }` |
| `seats_after_release` | 25 / 24 / 0 / 1 | the section now — one seat free, which may go to the next person on the waitlist |
| `s3` | | the updated record |

---

## Other replies you can get (not errors)

**Letting go of a hold before paying** (`hold_id: "hld_01J8Z3XQ2K"`, `reason: "chose_other_section"`):

```json
{
  "contract": "goldwire_v1",
  "request_id": "6b7c8d9e-0f1a-4b2c-8d3e-4f5a6b7c8d9e",
  "status": "ok",
  "as_of": "2026-09-25T14:08:12Z",
  "not_held": [],
  "hold_id": "hld_01J8Z3XQ2K",
  "booking_id": null,
  "released_state": "RELEASED",
  "note": null,
  "refund": { "amount": null, "percent": null, "source": null, "paid_by": null },
  "fee_due": null,
  "seats_after_release": { "capacity": 25, "enrolled": 21, "held": 2, "available": 2 },
  "s3": { "key": "goldwire/225070/2027SP/sec_225070_2027SP_ENGL1301_002/holds/hld_01J8Z3XQ2K.json", "etag": "\"b2c39d0e2b7c41a5f6\"", "version_id": "Rs3tUv4wXy5zA6bC.dE7fG8hI9jK0lM1" }
}
```

**Dropping after the full-refund date** (on 9 February): `refund.percent: 50`,
`refund.amount: "120.50"` — whatever the school's policy sets for that date.

**Reserve booking dropped late** — no money was taken, but the school publishes a $25 late-drop fee:
`refund.amount: null`, `fee_due: { "amount": { "amount": "25.00", "currency": "USD" }, "source": "pol_225070_fees_2026_27" }`.
GoldCard charges it against the guarantee.

**The school didn't answer the drop** — `status: "offline"`, `released_state: "DROP_REQUESTED"`, and:

```json
"note": "School 225070 did not answer. The drop is recorded and will be resent every 5 minutes; check get_booking."
```

---

## Errors

Example — past the school's last drop date:

```json
{
  "contract": "goldwire_v1",
  "request_id": "6b7c8d9e-0f1a-4b2c-8d3e-4f5a6b7c8d9e",
  "status": "error",
  "error": {
    "code": "DROP_DEADLINE_PASSED",
    "message": "The last day to drop ENGL 1301 section 002 at school 225070 was 2027-04-02. A withdrawal now is the school's decision; contact the registrar.",
    "retry": "contact_school",
    "field": "booking_id",
    "details": { "booking_id": "bkg_01J8Z42M7R", "last_drop_date": "2027-04-02" }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `Send exactly one of hold_id or booking_id.` | neither, or both | send one |
| 400 | `VALIDATION_FAILED` | `reason must be one of changed_mind, schedule_conflict, chose_other_section, financial, other; got 'job'.` | | pick one; put detail in `note` |
| 400 | `VALIDATION_FAILED` | `note is required when reason is other.` | | add a note |
| 400 | `VALIDATION_FAILED` | `note may be at most 500 characters.` | | shorten it |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_8f3a2c91d7.` | | send the matching pair |
| 404 | `HOLD_NOT_FOUND` | `No hold hld_01J8Z3ZZ99 is held for learner lrn_8f3a2c91d7.` | | check the id |
| 404 | `BOOKING_NOT_FOUND` | `No booking bkg_01J8Z42M7X is held for learner lrn_8f3a2c91d7.` | | check the id |
| 409 | `ALREADY_RELEASED` | `bkg_01J8Z42M7R is already DROPPED (since 2027-01-20T16:05:31Z). Nothing was changed.` | done before | nothing |
| 409 | `ALREADY_RELEASED` | `hld_01J8Z3XQ2K is already EXPIRED (since 2026-09-25T14:24:05Z). Nothing was changed.` | ran out | nothing |
| 409 | `HOLD_ALREADY_BOOKED` | `Hold hld_01J8Z3XQ2K was booked as bkg_01J8Z42M7R. To give up the seat, release the booking.` | releasing the hold of a booked seat | send `booking_id` instead |
| 409 | `DROP_DEADLINE_PASSED` | `The last day to drop ENGL 1301 section 002 at school 225070 was 2027-04-02. A withdrawal now is the school's decision; contact the registrar.` | | contact the school |
| 422 | `IDEMPOTENCY_CONFLICT` | `Key 01J9A7K3M5P7R9T1V3X5Z7B9D1 was already used for a different release request on 2027-01-20T16:05:31Z. Use a new key.` | | new key |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | | wait |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference 6b7c8d9e-0f1a-4b2c-8d3e-4f5a6b7c8d9e.` | | retry with the same key |

## What gets recorded in S3

1. `goldwire/idempotency/01J9A7K3M5P7R9T1V3X5Z7B9D1.json` — written once.
2. A **new version** of the hold or booking record (`If-Match` on the version read) with the new state, the reason and note, and the refund figure with its policy. The receipt from Call 6 is locked and stays as it was; the drop is added to the booking's history.
