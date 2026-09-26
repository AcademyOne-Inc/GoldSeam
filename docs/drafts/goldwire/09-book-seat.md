# Call 9 — Book the seat

[← Call 8 — GoldCard authorize](08-goldcard-authorize.md) · [Summary](README.md) · Next: [Call 10 — Get a booking →](10-get-booking.md)

> **Draft.** Nothing here is live.

## What it does

Turns a paid-for (or guaranteed) hold into a **registration request to the school**, sends it, and
returns **the school's answer**:

- **Confirmed** — the school registered the learner. A confirmation number is issued and the money is taken.
- **Submitted** — the school accepted the request but hasn't decided yet (or didn't answer in time).
- **Waitlisted** — the school put the learner on its waitlist instead.
- **Rejected** — the school said no, with its reason.

GoldWire never decides the learner is registered. **Only the school's own system can make a booking
`CONFIRMED`.**

## Where it sits

- **Before:** [Call 5](05-get-person.md) linked the learner to their record at the school (`psl_3N8QK2WD7F`); [Call 6](06-hold-seat.md) held `hld_01J8Z3XQ2K`; [Call 8](08-goldcard-authorize.md) authorized $241.00 as `auth_7HF2Q9`.
- **This call:** "Register me."
- **After:** show the confirmation. Check back any time with [Call 10 — Get a booking](10-get-booking.md). Drop with [Call 11](11-release-seat.md).

---

## The request

```http
PUT /goldwire/v1/bookings/01J8Z41V2S8K3N6P0D9F5G7H1J HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fOGYzYTJjOTFkNyJ9.sig
X-Request-Id: c51f2d8a-7e4b-4c19-b6a3-5d0e8f1a9c72
Content-Type: application/json

{
  "learner_id": "lrn_8f3a2c91d7",
  "hold_id": "hld_01J8Z3XQ2K",
  "payment_mode": "prepay",
  "goldcard_authorization_id": "auth_7HF2Q9",
  "person_link_id": "psl_3N8QK2WD7F",
  "attestations": {
    "prerequisites_met": true,
    "policies_acknowledged": ["pol_225070_refunds_2026_27"]
  }
}
```

## Every field you send

| Field | This example | Required? | What it means | Rules |
|---|---|---|---|---|
| idempotency key (in the path) | `01J8Z41V2S8K3N6P0D9F5G7H1J` | yes | makes retries safe — **a retry never books twice** | 16–64 letters, digits, `_` or `-` |
| `Authorization` (header) | `Bearer eyJ…` | yes | sign-in token | must belong to `lrn_8f3a2c91d7` |
| `learner_id` | `lrn_8f3a2c91d7` | yes | who is registering | must own the hold |
| `hold_id` | `hld_01J8Z3XQ2K` | yes | the held seat | must be `HELD` or `OFFERED`, not expired |
| `payment_mode` | `prepay` | yes | | must match the hold |
| `goldcard_authorization_id` | `auth_7HF2Q9` | **yes for prepay** | the money, from Call 8 | must be for this hold and the full amount due now |
| `goldcard_guarantee_id` | *(not sent)* | **yes for reserve** | the guarantee, from Call 8 | send one or the other, never both |
| `person_link_id` | `psl_3N8QK2WD7F` | yes | the learner's record at this school, from [Call 5 — Get person](05-get-person.md) | must be this learner's link at this school. The school's own person ID behind it goes only to the school and is **never returned in any reply** |
| `attestations.prerequisites_met` | `true` | yes | the learner confirms they meet the course prerequisites | must be `true` when the section lists `prerequisite_check`; the school still checks |
| `attestations.policies_acknowledged` | `["pol_225070_refunds_2026_27"]` | yes | the policies the learner was shown and accepted | must include the refund policy from the quote |

## What GoldWire does, in order

1. Checks the key, the token and every field.
2. Checks the hold: it exists and is this learner's → not expired → not already booked → `HELD` or `OFFERED`.
3. Asks GoldCard: is `auth_7HF2Q9` real, for this hold, for $241.00, and not expired?
4. Checks the attestations and the person link: it must be this learner's, at this school.
5. Records the booking as `SUBMITTED` and marks the hold `BOOKED`.
6. Sends the registration to the school's system, waiting up to 8 seconds: term `2027-SP`, CRN `21457`, the school's person ID behind `psl_3N8QK2WD7F` (`A00482913`), GoldWire booking `bkg_01J8Z42M7R`.
7. Records the school's answer, moves the money to match, and replies.

---

## The reply — `201 Created`

The school answered "registered" two seconds later:

```json
{
  "contract": "goldwire_v1",
  "request_id": "c51f2d8a-7e4b-4c19-b6a3-5d0e8f1a9c72",
  "status": "ok",
  "as_of": "2026-09-25T14:11:53Z",
  "not_held": [],
  "booking_id": "bkg_01J8Z42M7R",
  "booking_state": "CONFIRMED",
  "confirmation_number": "GW-225070-7Q4K-2M",
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "hold_id": "hld_01J8Z3XQ2K",
  "goldcheck_ref": "gck_5TR20P",
  "school": {
    "unitid": "225070",
    "answer": "registered",
    "reason": null,
    "registration_ref": "202720.21457",
    "waitlist_position": null,
    "answered_at": "2026-09-25T14:11:52Z"
  },
  "payment": {
    "mode": "prepay",
    "goldcard_authorization_id": "auth_7HF2Q9",
    "goldcard_guarantee_id": null,
    "amount": { "amount": "241.00", "currency": "USD" },
    "funds_status": "captured",
    "capture": "on_confirmation"
  },
  "refund_policy": { "full_refund_until": "2027-02-02", "source": "pol_225070_refunds_2026_27" },
  "s3": {
    "booking": {
      "key": "goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/bookings/bkg_01J8Z42M7R.json",
      "etag": "\"e4d29d0e2b7c41a5f6\"",
      "version_id": "3HL4kqtJlcpXroDTDmJ.rmSpXd3dIbrHY"
    },
    "receipt": {
      "key": "goldwire/receipts/lrn_8f3a2c91d7/bkg_01J8Z42M7R.json",
      "etag": "\"77ab9d0e2b7c41a5f6\""
    }
  }
}
```

In words: **registered in ENGL 1301 section 002, Spring 2027. Confirmation GW-225070-7Q4K-2M.**
$241.00 was charged to the card. Full refund if dropped by 2 February 2027.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` / `as_of` | | rule version, tracking number, when recorded |
| `status` | `ok` | `offline` if the school did not answer (see below) |
| `booking_id` | `bkg_01J8Z42M7R` | **the booking — use it in Calls 10 and 11** |
| `booking_state` | `CONFIRMED` | `CONFIRMED`, `SUBMITTED`, `WAITLISTED` or `REJECTED` |
| `confirmation_number` | `GW-225070-7Q4K-2M` | only when `CONFIRMED`; for the learner to keep |
| `section_id` / `hold_id` / `goldcheck_ref` | | carried through |
| `school.unitid` | `225070` | |
| `school.answer` | `registered` | the school's own answer: `registered`, `pending`, `waitlisted`, `refused`, `no_answer` |
| `school.reason` | `null` | the school's words, when it refused or is pending |
| `school.registration_ref` | `202720.21457` | the school's own reference |
| `school.waitlist_position` | `null` | when waitlisted |
| `school.answered_at` | `2026-09-25T14:11:52Z` | |
| `payment.mode` | `prepay` | |
| `payment.goldcard_authorization_id` | `auth_7HF2Q9` | |
| `payment.amount` | `241.00 USD` | |
| `payment.funds_status` | `captured` | the money was taken because the school confirmed. Others: `authorized`, `committed`, `pending`, `released` |
| `payment.capture` | `on_confirmation` | prepay: taken on confirmation · `at_school`: reserve, the school bills · `none`: nothing will be taken |
| `refund_policy` | until 2027-02-02 | |
| `s3.booking` | `…/bookings/bkg_01J8Z42M7R.json` | the booking record |
| `s3.receipt` | `goldwire/receipts/lrn_8f3a2c91d7/bkg_01J8Z42M7R.json` | the learner's receipt — locked, cannot be changed or deleted |

**Same request sent twice:** `200 OK`, header `Idempotent-Replayed: true`, the same reply. No second booking, no second charge.

---

## Other replies you can get (not errors)

Each of these is `201 Created`. A school saying no is **an answer**, not an error.

| The school says | `booking_state` | What changes in the reply | Money |
|---|---|---|---|
| **registered** | `CONFIRMED` | as above | prepay taken; reserve: the school bills by its due date |
| **pending** — e.g. "Awaiting 529 plan payment" | `SUBMITTED` | `confirmation_number: null`, `school.answer: "pending"`, `school.reason: "Awaiting 529 plan payment"` | authorization kept open |
| **waitlisted** | `WAITLISTED` | `school.answer: "waitlisted"`, `school.waitlist_position: 2` | authorization released; learner pays again if offered a seat |
| **refused** | `REJECTED` | `school.answer: "refused"`, `school.reason` in the school's words | authorization released; seat returned |
| **nothing within 8 seconds** | `SUBMITTED` | `status: "offline"`, `school.answer: "no_answer"`, `note` explains | nothing taken; GoldWire resends every 5 minutes for 24 hours |

**Refused** — the school has a hold on the learner's account:

```json
{
  "contract": "goldwire_v1",
  "request_id": "c51f2d8a-7e4b-4c19-b6a3-5d0e8f1a9c72",
  "status": "ok",
  "as_of": "2026-09-25T14:11:53Z",
  "not_held": [],
  "booking_id": "bkg_01J8Z42M7R",
  "booking_state": "REJECTED",
  "confirmation_number": null,
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "hold_id": "hld_01J8Z3XQ2K",
  "goldcheck_ref": "gck_5TR20P",
  "school": {
    "unitid": "225070",
    "answer": "refused",
    "reason": "Registration hold on student account: unpaid balance from Fall 2025. Contact the Bursar's Office.",
    "registration_ref": null,
    "waitlist_position": null,
    "answered_at": "2026-09-25T14:11:52Z"
  },
  "payment": {
    "mode": "prepay",
    "goldcard_authorization_id": "auth_7HF2Q9",
    "goldcard_guarantee_id": null,
    "amount": { "amount": "241.00", "currency": "USD" },
    "funds_status": "released",
    "capture": "none"
  },
  "refund_policy": { "full_refund_until": "2027-02-02", "source": "pol_225070_refunds_2026_27" },
  "s3": {
    "booking": { "key": "goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/bookings/bkg_01J8Z42M7R.json", "etag": "\"f5e39d0e2b7c41a5f6\"", "version_id": "7Jk2mNpQ4rStUvWx.yZ1aB3cD5eF7gH9" }
  }
}
```

**The school didn't answer in time** — `status: "offline"`, `booking_state: "SUBMITTED"`,
`school.answer: "no_answer"`, `payment.funds_status: "authorized"`, and:

```json
"note": "School 225070 did not answer within 8 seconds. The request is recorded and will be resent every 5 minutes for 24 hours. Check get_booking for the answer."
```

If the school still hasn't answered after 24 hours, the booking becomes `REJECTED` with reason
`The school did not answer within 24 hours.` and the authorization is released.

---

## Errors

Example — the hold ran out before booking:

```json
{
  "contract": "goldwire_v1",
  "request_id": "c51f2d8a-7e4b-4c19-b6a3-5d0e8f1a9c72",
  "status": "error",
  "error": {
    "code": "HOLD_EXPIRED",
    "message": "Hold hld_01J8Z3XQ2K expired at 2026-09-25T14:24:05Z. The seat was released. Hold a seat again.",
    "retry": "new_hold",
    "field": "hold_id",
    "details": { "hold_id": "hld_01J8Z3XQ2K", "expired_at": "2026-09-25T14:24:05Z" }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `hold_id is required.` | left out (same for `learner_id`, `payment_mode`, `attestations`) | send it |
| 400 | `VALIDATION_FAILED` | `goldcard_authorization_id is required when payment_mode is prepay.` | | send the id from Call 8 |
| 400 | `VALIDATION_FAILED` | `goldcard_guarantee_id is required when payment_mode is reserve.` | | send the id from Call 8 |
| 400 | `VALIDATION_FAILED` | `Send goldcard_authorization_id or goldcard_guarantee_id, not both.` | | send only the one for the mode |
| 400 | `VALIDATION_FAILED` | `attestations.policies_acknowledged must list at least one policy id.` | empty list | send the refund policy id |
| 400 | `VALIDATION_FAILED` | `Unknown field 'section_id'. book_seat accepts learner_id, hold_id, payment_mode, goldcard_authorization_id, goldcard_guarantee_id, person_link_id, attestations.` | | drop it — the section comes from the hold |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_8f3a2c91d7.` | | send the matching pair |
| 404 | `HOLD_NOT_FOUND` | `No hold hld_01J8Z3ZZ99 is held for learner lrn_8f3a2c91d7.` | | check the id |
| 404 | `AUTHORIZATION_NOT_FOUND` | `GoldCard has no authorization auth_7HF2Q0 for learner lrn_8f3a2c91d7.` | typo, or authorized for someone else | authorize again (Call 8) |
| 404 | `GUARANTEE_NOT_FOUND` | `GoldCard has no guarantee gtd_4KX81N for learner lrn_8f3a2c91d7.` | | Call 8 again |
| 409 | `HOLD_ALREADY_BOOKED` | `Hold hld_01J8Z3XQ2K was already booked as bkg_01J8Z42M7R on 2026-09-25T14:11:53Z.` | booked before under a different key | read that booking (Call 10) |
| 409 | `HOLD_NOT_ACTIVE` | `Hold hld_01J8Z3XQ2K is WAITLISTED; only a HELD or OFFERED hold can be booked.` | | wait for a seat |
| 409 | `HOLD_NOT_ACTIVE` | `Hold hld_01J8Z3XQ2K is RELEASED; only a HELD or OFFERED hold can be booked.` | | hold again |
| 410 | `HOLD_EXPIRED` | `Hold hld_01J8Z3XQ2K expired at 2026-09-25T14:24:05Z. The seat was released. Hold a seat again.` | more than 20 minutes | Call 6 again; release the old authorization |
| 410 | `AUTHORIZATION_EXPIRED` | `Authorization auth_7HF2Q9 expired at 2026-10-02T14:06:30Z. Authorize again with GoldCard.` | | Call 8 again |
| 422 | `PAYMENT_MISMATCH` | `Authorization auth_7HF2Q9 is for hld_01J8Z3W1AB / $186.00; this hold is hld_01J8Z3XQ2K / $241.00.` | authorization from another hold | authorize this hold |
| 422 | `PAYMENT_MODE_MISMATCH` | `Hold hld_01J8Z3XQ2K is for prepay, not reserve.` | | match the hold |
| 422 | `ATTESTATION_REQUIRED` | `Section sec_225070_2027SP_ENGL1301_002 requires a prerequisite check. attestations.prerequisites_met must be true; the school will verify it.` | sent `false` | ask the learner; if they don't meet them, don't book |
| 422 | `POLICY_NOT_ACKNOWLEDGED` | `The learner must acknowledge refund policy pol_225070_refunds_2026_27 before booking.` | refund policy missing | show it, then send it |
| 400 | `VALIDATION_FAILED` | `person_link_id is required. Find the learner's record at the school with Get person first.` | left out | Call 5 |
| 404 | `PERSON_LINK_NOT_FOUND` | `No person link psl_3N8QK2WD7X is held for learner lrn_8f3a2c91d7 at school 225070. Look the learner up with Get person.` | wrong link, another learner's, or another school's | Call 5 |
| 422 | `IDEMPOTENCY_CONFLICT` | `Key 01J8Z41V2S8K3N6P0D9F5G7H1J was already used for a different booking request on 2026-09-25T14:11:53Z. Use a new key.` | | new key |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | | wait |
| 503 | `PAYMENT_PROVIDER_OFFLINE` | `GoldCard could not confirm authorization auth_7HF2Q9. Nothing was booked. Try again shortly.` | | retry with the **same key** |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was booked. Reference c51f2d8a-7e4b-4c19-b6a3-5d0e8f1a9c72.` | | retry with the same key |

## What gets recorded in S3

1. `goldwire/idempotency/01J8Z41V2S8K3N6P0D9F5G7H1J.json` — written once; makes retries safe.
2. `goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/bookings/bkg_01J8Z42M7R.json` — created as `SUBMITTED` (`If-None-Match: *`); the school's answer is a **new version** (`If-Match` on the version just written). It refers to the person link, not the school's person ID.
3. A new version of `…/holds/hld_01J8Z3XQ2K.json` with state `BOOKED`.
4. On `CONFIRMED` only: `goldwire/receipts/lrn_8f3a2c91d7/bkg_01J8Z42M7R.json` — locked (Object Lock) so it can't be changed or deleted.
