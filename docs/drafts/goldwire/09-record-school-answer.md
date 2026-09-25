# Call 9 — Record the school's answer (internal)

[← Call 8 — Release a seat](08-release-seat.md) · [Summary](README.md)

> **Draft.** Nothing here is live. **Learners and their apps never call this.** It is called only by
> the school adapter — the piece that connects GoldWire to one school's registration system.

## What it does

Carries **the school's later answer** onto a booking: the learner was registered, refused,
waitlisted, dropped — or the school corrected the price (for example, it re-rated the learner from
in-district to out-of-state). GoldWire moves the booking and the money to match and tells the learner.

This is how a booking that came back `SUBMITTED` in [Call 6](06-book-seat.md) becomes `CONFIRMED`
days later.

## Where it sits

- **Before:** a booking is `SUBMITTED` (the school is waiting for the 529 payment) or `DROP_REQUESTED`.
- **This call:** the school's system says "registered" on 9 October.
- **After:** the learner sees it in [Call 7 — Get a booking](07-get-booking.md) and gets a notification.

---

## The request

```http
PUT /goldwire/v1/bookings/bkg_01J8Z42M7R/school-answer HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJhZGFwdGVyXzIyNTA3MCJ9.sig
If-Match: "c3d49d0e2b7c41a5f6"
X-Request-Id: 8d9e0f1a-2b3c-4d4e-8f5a-6b7c8d9e0f1a
Content-Type: application/json

{
  "unitid": "225070",
  "answer": "registered",
  "reason": null,
  "registration_ref": "21457/2027SP",
  "waitlist_position": null,
  "new_total": null,
  "answered_at": "2026-10-09T15:40:10Z"
}
```

## Every field you send

| Field | This example | Required? | What it means | Rules |
|---|---|---|---|---|
| `Authorization` (header) | `Bearer eyJ…` | yes | the **school adapter's** token — not a learner's | scoped to one school: `225070` |
| `If-Match` (header) | `"c3d49d0e2b7c41a5f6"` | yes | the version of the booking this answer is about | the ETag from the last read; stops two answers overwriting each other |
| `booking_id` (in the path) | `bkg_01J8Z42M7R` | yes | the booking | a `bkg_` id at this school |
| `unitid` | `225070` | yes | the school | must match the token and the booking |
| `answer` | `registered` | yes | the school's decision | `registered`, `pending`, `waitlisted`, `refused`, `dropped`, `re_rated` |
| `reason` | `null` | for `refused`, `pending`, `re_rated` | the school's own words; shown to the learner exactly | up to 500 characters |
| `registration_ref` | `21457/2027SP` | for `registered` | the school's reference | |
| `waitlist_position` | `null` | for `waitlisted` | | whole number, 1 or more |
| `new_total` | `null` | for `re_rated` | the school's corrected price | money, e.g. `{ "amount": "367.00", "currency": "USD" }` |
| `answered_at` | `2026-10-09T15:40:10Z` | yes | when the school decided | |

## What happens, by answer

| The booking is | The school says | The booking becomes | Money | The learner is told |
|---|---|---|---|---|
| `SUBMITTED` | `registered` | `CONFIRMED`; a confirmation number is issued; the receipt is written | prepay captured | "You're registered in ENGL 1301, section 002. Confirmation GW-225070-7Q4K-2M." |
| `SUBMITTED` | `pending` | stays `SUBMITTED`; reason added | unchanged | "Your registration is pending: {reason}." |
| `SUBMITTED` | `waitlisted` | `WAITLISTED` | authorization released | "You're number {n} on the waitlist." |
| `SUBMITTED` | `refused` | `REJECTED`; seat returned | authorization or guarantee released | "Your registration was not accepted: {reason}." |
| `CONFIRMED` or `DROP_REQUESTED` | `dropped` | `DROPPED` | refund per the school's policy, as in [Call 8](08-release-seat.md) | "You've been dropped from ENGL 1301. Refund: {amount}." |
| `SUBMITTED` or `CONFIRMED` | `re_rated` | unchanged; a price change is added | the learner must **approve** the difference in GoldCard; nothing more is charged without approval | "The school changed your price from {old} to {new}: {reason}. Please review." |

---

## The reply — `200 OK`

The full booking, exactly as [Call 7](07-get-booking.md) returns it, now at `CONFIRMED`:

```json
{
  "contract": "goldwire_v1",
  "request_id": "8d9e0f1a-2b3c-4d4e-8f5a-6b7c8d9e0f1a",
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
  "section": { "section_number": "002", "crn": "21457", "term": "2027SP",
               "meetings": [ { "days": "TR", "start": "09:30", "end": "10:50", "room": "LA 114" } ], "starts_on": "2027-01-19" },
  "school": { "unitid": "225070", "answer": "registered", "reason": null, "registration_ref": "21457/2027SP",
              "waitlist_position": null, "answered_at": "2026-10-09T15:40:10Z" },
  "payment": { "mode": "prepay", "goldcard_authorization_id": "auth_9MN3R7", "goldcard_guarantee_id": null,
               "amount": { "amount": "241.00", "currency": "USD" }, "funds_status": "captured", "capture": "on_confirmation" },
  "refund_policy": { "full_refund_until": "2027-02-02", "source": "pol_225070_refunds_2026_27" },
  "price_changes": [],
  "history": [
    { "at": "2026-09-25T14:11:50Z", "state": "SUBMITTED", "by": "learner", "version_id": "0pQ1rT7uVx2yZa3bCd4eFg5hIj6kLm7nO", "note": null },
    { "at": "2026-09-25T14:11:52Z", "state": "SUBMITTED", "by": "school",  "version_id": "5sT6uV7wX8yZ9aB0.cD1eF2gH3iJ4kL5", "note": "pending: Awaiting 529 plan payment" },
    { "at": "2026-10-09T15:40:10Z", "state": "CONFIRMED", "by": "school",  "version_id": "3HL4kqtJlcpXroDTDmJ.rmSpXd3dIbrHY", "note": "registered, registration_ref 21457/2027SP" }
  ],
  "s3": { "key": "goldwire/225070/2027SP/sec_225070_2027SP_ENGL1301_002/bookings/bkg_01J8Z42M7R.json",
          "etag": "\"e4d29d0e2b7c41a5f6\"", "version_id": "3HL4kqtJlcpXroDTDmJ.rmSpXd3dIbrHY" }
}
```

Every field is explained on the [Call 7 page](07-get-booking.md#every-field-you-get-back).

**A re-rate** — the school found the learner is out-of-district (`answer: "re_rated"`,
`new_total: 367.00`, `reason: "Residency documents show an out-of-district address; in-state rate applies."`)
adds to the booking:

```json
"price_changes": [
  { "from": { "amount": "241.00", "currency": "USD" },
    "to":   { "amount": "367.00", "currency": "USD" },
    "reason": "Residency documents show an out-of-district address; in-state rate applies.",
    "at": "2026-10-12T10:02:00Z",
    "approved": false }
]
```

The booking stays `CONFIRMED`. GoldCard asks the learner to approve the extra $126.00; if they
don't, they can drop under the refund policy.

---

## Errors

Example — two answers raced; this one was about an older version:

```json
{
  "contract": "goldwire_v1",
  "request_id": "8d9e0f1a-2b3c-4d4e-8f5a-6b7c8d9e0f1a",
  "status": "error",
  "error": {
    "code": "VERSION_CONFLICT",
    "message": "Booking bkg_01J8Z42M7R changed since version \"c3d49d0e2b7c41a5f6\" (now \"e4d29d0e2b7c41a5f6\"). Read it again and resend the answer if it still applies.",
    "retry": "fix_request",
    "field": "If-Match",
    "details": { "booking_id": "bkg_01J8Z42M7R", "if_match": "\"c3d49d0e2b7c41a5f6\"", "current_etag": "\"e4d29d0e2b7c41a5f6\"" }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Adapter should |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `registration_ref is required when answer is registered.` | | send it |
| 400 | `VALIDATION_FAILED` | `reason is required when answer is refused.` | also for `pending` and `re_rated` | send the school's words |
| 400 | `VALIDATION_FAILED` | `waitlist_position is required when answer is waitlisted.` | | send it |
| 400 | `VALIDATION_FAILED` | `new_total is required when answer is re_rated.` | | send it |
| 400 | `VALIDATION_FAILED` | `answer must be one of registered, pending, waitlisted, refused, dropped, re_rated; got 'enrolled'.` | the school's own word passed through | map it to `registered` |
| 400 | `VALIDATION_FAILED` | `If-Match is required: send the ETag of the booking version this answer applies to.` | | read the booking, send its ETag |
| 401 | `UNAUTHENTICATED` | `A school adapter token is required.` | a learner token, or none | use the adapter token |
| 403 | `SCOPE_MISMATCH` | `This adapter token is for school 228529; booking bkg_01J8Z42M7R is at school 225070.` | wrong school's adapter | route to the right adapter |
| 404 | `BOOKING_NOT_FOUND` | `No booking bkg_01J8Z42M7X is held at school 225070.` | | check the id |
| 409 | `INVALID_TRANSITION` | `Booking bkg_01J8Z42M7R is DROPPED; answer registered cannot apply to it.` | an out-of-date message from the school | ignore, or raise with the school |
| 409 | `INVALID_TRANSITION` | `Booking bkg_01J8Z42M7R is REJECTED; answer dropped cannot apply to it.` | | ignore |
| 412 | `VERSION_CONFLICT` | `Booking bkg_01J8Z42M7R changed since version "c3d49d0e2b7c41a5f6" (now "e4d29d0e2b7c41a5f6"). Read it again and resend the answer if it still applies.` | another change landed first | read again, then resend if still true |
| 500 | `INTERNAL_ERROR` | `GoldWire could not record the answer. Nothing was changed. Reference 8d9e0f1a-2b3c-4d4e-8f5a-6b7c8d9e0f1a.` | | resend |

## What gets recorded in S3

1. A **new version** of `…/bookings/bkg_01J8Z42M7R.json`, written only if the version is still the one in `If-Match`.
2. On `registered`: `goldwire/receipts/lrn_8f3a2c91d7/bkg_01J8Z42M7R.json`, locked.
