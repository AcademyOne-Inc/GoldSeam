# Call 4 — Get section fees

[← Call 3 — Get sections](03-get-sections.md) · [Summary](README.md) · Next: [Call 5 — Get person →](05-get-person.md)

> **Draft.** Nothing here is live.

## What it does

Prices **one seat in one section**, line by line from the school's published tuition and fee
policies, and splits it into **what is due now** and **what is due to the school later**. It returns a
**quote** — a price fixed for 30 minutes — so the price the learner sees is the price the seat is held
at.

- **Prepay** (pay with GoldCard now): everything is due now.
- **Reserve** (the OpenTable model — book now, pay the school on its bill): nothing is due now.

It records the quote in S3 so the price can be proved later. Asking again with the same choices within
30 minutes returns the **same** quote.

## Where it sits

- **Before:** [Call 3](03-get-sections.md) showed section 002 has 2 seats: `sec_225070_2027SP_ENGL1301_002`.
- **This call:** "What does a seat in section 002 cost if I prepay, as an in-district student?"
- **After:** the learner accepts the price → [Call 7 — Hold a seat](07-hold-seat.md) with the `quote_id`.

---

## The request

```http
GET /goldwire/v1/sections/sec_225070_2027SP_ENGL1301_002/fees?learner_id=lrn_8f3a2c91d7&payment_mode=prepay&residency=in_district&funding_source_type=credit_card HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fOGYzYTJjOTFkNyJ9.sig
X-Request-Id: 9a4d6e13-8c2b-4f71-a5d0-3e9b7c1f2a64
Accept: application/json
```

## Every field you send

| Field | This example | Required? | What it means | Allowed |
|---|---|---|---|---|
| `Authorization` (header) | `Bearer eyJ…` | yes | the learner's sign-in token | must belong to `lrn_8f3a2c91d7` |
| `section_id` (in the path) | `sec_225070_2027SP_ENGL1301_002` | yes | the section, from Call 3 | a `sec_` id |
| `learner_id` | `lrn_8f3a2c91d7` | yes | who is asking | `lrn_` + 8–40 letters/digits |
| `payment_mode` | `prepay` | yes | how the seat will be paid for | `prepay` = GoldCard pays at booking · `reserve` = book now, the school bills later |
| `residency` | `in_district` | no, but **needed for a quote** | which tuition rate applies — the learner's own statement; the school may correct it later | `in_district`, `in_state`, `out_of_state`, `international`, `unknown` (default) |
| `funding_source_type` | `credit_card` | no | the kind of money; used only to add a surcharge the school publishes, or to say the school won't take it | `bank_ach`, `credit_card`, `loan`, `plan_529` |
| `currency` | *(not sent — so `USD`)* | no | | only `USD` in the POC |

---

## The reply — `200 OK`

```json
{
  "contract": "goldwire_v1",
  "request_id": "9a4d6e13-8c2b-4f71-a5d0-3e9b7c1f2a64",
  "status": "ok",
  "as_of": "2026-09-25T14:03:40Z",
  "source": {
    "kind": "school_fee_policy",
    "policy_id": "pol_225070_tuition_2026_27",
    "edition": "2026–27",
    "published": "2026-06-01"
  },
  "not_held": [],
  "quote_id": "qt_01J8Z3V6N4",
  "quote_expires_at": "2026-09-25T14:33:40Z",
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "payment_mode": "prepay",
  "residency_applied": "in_district",
  "line_items": [
    { "code": "TUITION",    "label": "Tuition, 3 credits × $62.00", "amount": { "amount": "186.00", "currency": "USD" }, "payee": "school",   "source": "pol_225070_tuition_2026_27" },
    { "code": "GEN_FEE",    "label": "General service fee",         "amount": { "amount": "45.00",  "currency": "USD" }, "payee": "school",   "source": "pol_225070_fees_2026_27" },
    { "code": "COURSE_FEE", "label": "Course fee (ENGL 1301)",      "amount": { "amount": "10.00",  "currency": "USD" }, "payee": "school",   "source": "lu_225070_ENGL1301" },
    { "code": "GW_BOOKING", "label": "GoldWire booking fee",        "amount": { "amount": "0.00",   "currency": "USD" }, "payee": "goldwire", "source": null }
  ],
  "totals": {
    "school":   { "amount": "241.00", "currency": "USD" },
    "goldwire": { "amount": "0.00",   "currency": "USD" },
    "total":    { "amount": "241.00", "currency": "USD" },
    "totals_complete": true
  },
  "due": {
    "now":       { "amount": "241.00", "currency": "USD" },
    "at_school": { "amount": "0.00",   "currency": "USD" },
    "school_due_date": null
  },
  "refund_policy": { "full_refund_until": "2027-02-02", "source": "pol_225070_refunds_2026_27" },
  "no_show_fee": null,
  "s3": {
    "key": "goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/quotes/qt_01J8Z3V6N4.json",
    "etag": "\"3f1c9d0e2b7c41a5f6\""
  }
}
```

In words: **$241.00, all due now** — $186 tuition, $45 general fee, $10 course fee, no GoldWire fee.
The price is fixed until **14:33:40**. Full refund if dropped by **2 February 2027**.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` / `as_of` | | as in every reply: rule version, tracking number, when read |
| `status` | `ok` | `ok`, or `not_held` if GoldSeam has not captured the school's fee policy |
| `source.policy_id` | `pol_225070_tuition_2026_27` | the school's tuition policy the price came from |
| `source.edition` / `.published` | `2026–27` / `2026-06-01` | which edition, and when the school published it |
| `not_held` | `[]` | any fee the school names but doesn't price, e.g. `{ "what": "lab fee" }` |
| `quote_id` | `qt_01J8Z3V6N4` | **the price, fixed — send this to Call 7** |
| `quote_expires_at` | `2026-09-25T14:33:40Z` | 30 minutes from now; after this, ask again |
| `section_id` | `sec_225070_2027SP_ENGL1301_002` | echoed |
| `payment_mode` | `prepay` | echoed |
| `residency_applied` | `in_district` | the rate the tuition line used |
| `line_items[].code` | `TUITION`, `GEN_FEE`, `COURSE_FEE`, `GW_BOOKING` | what the charge is. Others that can appear: `LAB_FEE`, `TECH_FEE`, `FUNDING_SURCHARGE` |
| `line_items[].label` | `Tuition, 3 credits × $62.00` | in plain words, for showing the learner |
| `line_items[].amount` | `{ "amount": "186.00", "currency": "USD" }` | money is always written this way: text with two decimals, never a floating-point number |
| `line_items[].payee` | `school` | who gets it: `school`, `goldwire` or `goldcard` |
| `line_items[].source` | `pol_225070_tuition_2026_27` | where the school published that amount |
| `totals.school` | `241.00` | everything going to the school |
| `totals.goldwire` | `0.00` | GoldWire's fee |
| `totals.total` | `241.00` | everything |
| `totals.totals_complete` | `true` | `false` would mean a fee is missing from the school's record and the total is short |
| `due.now` | `241.00` | **what GoldCard must authorize in Call 9** |
| `due.at_school` | `0.00` | what the school will bill later |
| `due.school_due_date` | `null` | the school's payment due date — used for reserve |
| `refund_policy.full_refund_until` | `2027-02-02` | per the school's refund policy |
| `refund_policy.source` | `pol_225070_refunds_2026_27` | the policy — the learner must acknowledge it in Call 10 |
| `no_show_fee` | `null` | the school publishes no no-show fee. If it did: `{ "amount": {…}, "applies_after": "2027-01-19", "source": "pol_…" }` |
| `s3.key` / `s3.etag` | `goldwire/225070/…/quotes/qt_01J8Z3V6N4.json` | where the quote is recorded |

---

## Other replies you can get (not errors)

**Same seat, reserve instead of prepay** — `payment_mode=reserve`. The line items and totals are the
same; only `due` changes:

```json
"due": {
  "now":       { "amount": "0.00",   "currency": "USD" },
  "at_school": { "amount": "241.00", "currency": "USD" },
  "school_due_date": "2027-01-12"
},
"no_show_fee": null
```

In words: nothing now; the school bills $241.00, due 12 January 2027.

**Residency not given** — `residency` left out or `unknown`. No quote is issued (a hold needs one fixed
price), and every published rate is shown so the learner can choose:

```json
{
  "contract": "goldwire_v1",
  "request_id": "9a4d6e13-8c2b-4f71-a5d0-3e9b7c1f2a64",
  "status": "ok",
  "as_of": "2026-09-25T14:03:40Z",
  "source": { "kind": "school_fee_policy", "policy_id": "pol_225070_tuition_2026_27", "edition": "2026–27", "published": "2026-06-01" },
  "not_held": [],
  "quote_id": null,
  "quote_expires_at": null,
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "payment_mode": "prepay",
  "residency_applied": "unknown",
  "rate_options": [
    { "residency": "in_district",  "rate_per_credit": { "amount": "62.00",  "currency": "USD" }, "tuition": { "amount": "186.00", "currency": "USD" }, "source": "pol_225070_tuition_2026_27" },
    { "residency": "in_state",     "rate_per_credit": { "amount": "104.00", "currency": "USD" }, "tuition": { "amount": "312.00", "currency": "USD" }, "source": "pol_225070_tuition_2026_27" },
    { "residency": "out_of_state", "rate_per_credit": { "amount": "158.00", "currency": "USD" }, "tuition": { "amount": "474.00", "currency": "USD" }, "source": "pol_225070_tuition_2026_27" }
  ]
}
```

**A fee is named but not priced by the school** — `totals.totals_complete: false` and
`not_held: [ { "what": "lab fee", "note": "named in the catalog, amount not published" } ]`. The
learner should be told the total is short by that fee.

**The school's fee policy has not been captured** — `200`, `status: "not_held"`, `quote_id: null`.

---

## Errors

Example — the school does not take 529 plans through GoldCard:

```json
{
  "contract": "goldwire_v1",
  "request_id": "9a4d6e13-8c2b-4f71-a5d0-3e9b7c1f2a64",
  "status": "error",
  "error": {
    "code": "FUNDING_SOURCE_NOT_ACCEPTED",
    "message": "School 225070 does not accept plan_529 through GoldCard. Accepted: credit_card, bank_ach.",
    "retry": "fix_request",
    "field": "funding_source_type",
    "details": { "unitid": "225070", "funding_source_type": "plan_529", "accepted": ["credit_card", "bank_ach"] }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `learner_id is required.` | left out | send it |
| 400 | `VALIDATION_FAILED` | `section_id must look like sec_…; got '002'. Use an id from get_sections.` | the section number sent instead of the id | use the `section_id` from Call 3 |
| 400 | `VALIDATION_FAILED` | `payment_mode is required: reserve or prepay.` | left out | send it |
| 400 | `VALIDATION_FAILED` | `payment_mode must be reserve or prepay; got 'card'.` | the funding type sent as the mode | `prepay`, and put `credit_card` in `funding_source_type` |
| 400 | `VALIDATION_FAILED` | `residency must be one of in_district, in_state, out_of_state, international, unknown; got 'resident'.` | | pick one |
| 400 | `VALIDATION_FAILED` | `funding_source_type must be one of bank_ach, credit_card, loan, plan_529; got '529'.` | | use `plan_529` |
| 400 | `VALIDATION_FAILED` | `currency EUR is not supported. This service quotes in USD.` | | drop it or send `USD` |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_8f3a2c91d7.` | | send the matching pair |
| 404 | `SECTION_NOT_FOUND` | `No section sec_225070_2027SP_ENGL1301_009 is held. List sections with get_sections.` | wrong id | run Call 3 again |
| 409 | `SECTION_CANCELLED` | `Section 002 of ENGL 1301 (sec_225070_2027SP_ENGL1301_002) was cancelled by the school on 2026-10-14.` | the school cancelled it | choose another section |
| 409 | `SECTION_NOT_BOOKABLE` | `Section sec_225070_2027SP_ENGL1301_005 cannot be booked through GoldWire: registration opens 2026-11-02T13:00:00Z.` | too early | come back then |
| 409 | `SECTION_NOT_BOOKABLE` | `Section sec_225070_2027SP_ENGL1301_002 cannot be booked through GoldWire: registration closed on 2027-01-26.` | too late | contact the school |
| 409 | `SECTION_NOT_BOOKABLE` | `Section sec_225070_2027SP_ENGL1301_002 cannot be booked through GoldWire: the school does not take bookings through GoldWire; register with the school directly.` | school not connected | register at the school |
| 409 | `SECTION_NOT_BOOKABLE` | `Section sec_225070_2027SP_ENGL1301_H01 cannot be booked through GoldWire: the section is restricted to Honors Program students.` | restricted section | choose another |
| 422 | `FUNDING_SOURCE_NOT_ACCEPTED` | `School 225070 does not accept plan_529 through GoldCard. Accepted: credit_card, bank_ach.` | | pick an accepted source, or use `reserve` and pay the school directly |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | | wait |
| 503 | `SCHOOL_OFFLINE` | `School 225070's fee record could not be read just now. Try again shortly.` | | try again in a minute |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference 9a4d6e13-8c2b-4f71-a5d0-3e9b7c1f2a64.` | | try again |

## What gets recorded in S3

One file, written once and never changed:

`goldwire/225070/2027-SP/sec_225070_2027SP_ENGL1301_002/quotes/qt_01J8Z3V6N4.json`

It holds the whole reply above, plus the learner id, the request id, and when it was written. It is
written with `If-None-Match: *`, so a quote can never be overwritten. No quote is written when
`quote_id` is `null`.
