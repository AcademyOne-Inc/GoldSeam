# GoldWire POC — service specification (DRAFT)

> **Status: draft for discussion.** Nothing here is live. This is the written specification of every
> GoldWire call: what is sent, what comes back, what the service does in between, and every error
> with its exact message. The machine form (`goldwire-openapi.yaml`) will be regenerated from this
> document once it is agreed; until then, **this document wins** wherever the two differ.
>
> Companion: [goldwire-booking.md](goldwire-booking.md) explains the flow and the reasoning.

---

## Contents

1. [The flow in one page](#1-the-flow-in-one-page)
2. [Rules that apply to every call](#2-rules-that-apply-to-every-call)
3. [Call 1 — get_section_openings](#call-1--get_section_openings)
4. [Call 2 — get_section_fees](#call-2--get_section_fees)
5. [Call 3 — hold_seat](#call-3--hold_seat)
6. [Call 4 — get_hold](#call-4--get_hold)
7. [Call 5 — goldcard_authorize (GoldCard)](#call-5--goldcard_authorize-goldcard)
8. [Call 6 — book_seat](#call-6--book_seat)
9. [Call 7 — get_booking](#call-7--get_booking)
10. [Call 8 — release_seat](#call-8--release_seat)
11. [Call 9 — record_school_answer (school adapter, internal)](#call-9--record_school_answer-school-adapter-internal)
12. [Background processes](#12-background-processes)
13. [States](#13-states)
14. [Error catalog — every code in one place](#14-error-catalog--every-code-in-one-place)
15. [S3 records written, call by call](#15-s3-records-written-call-by-call)

---

## 1. The flow in one page

| Step | Who calls | Call | What the learner gets |
|---|---|---|---|
| 0 | Client → GoldCheck | `will_my_credits_transfer` (existing) | `goldcheck_ref` — what the school's record says the course counts toward |
| 1 | Client → GoldWire | **get_section_openings** | the sections of the course this term, with seat counts |
| 2 | Client → GoldWire | **get_section_fees** | a price quote (`quote_id`) for one section, due now vs. due at the school |
| 3 | Client → GoldWire | **hold_seat** | a seat kept for 20 minutes (`hold_id`) — or a waitlist place, or "full" |
| 4 | Client → GoldWire | **get_hold** (optional) | the hold's current state and time left |
| 5 | Client → GoldCard | **goldcard_authorize** | prepay: an authorization (`auth_…`); reserve: a guarantee (`gtd_…`) |
| 6 | Client → GoldWire | **book_seat** | a booking (`booking_id`) with the school's answer: confirmed, submitted, waitlisted or rejected |
| 7 | Client → GoldWire | **get_booking** | the booking as recorded, with its full history |
| 8 | Client → GoldWire | **release_seat** | a hold let go, or a drop requested from the school, with the refund the school's policy gives |
| 9 | School adapter → GoldWire | **record_school_answer** | (internal) the school's later answer lands on the booking |

A seat that is **held** is not a **registration**. A booking is `CONFIRMED` only when the school's own
system says the learner is registered.

---

## 2. Rules that apply to every call

These are stated once here **and repeated inside each call** so that each call can be read alone.

**Transport.** HTTPS, JSON bodies (`Content-Type: application/json; charset=utf-8`). Base path
`/goldwire/v1`. Reads are `GET`; anything that changes state is `PUT`.

**Authentication.** Every learner call carries `Authorization: Bearer <learner token>`. The token's
subject must equal the `learner_id` sent in the call.

**Correlation.** The client may send `X-Request-Id: <uuid>`. If it does not, GoldWire makes one. It is
returned as `request_id` in every reply and written into every S3 record the call makes.

**Idempotency.** Every `PUT` goes to a path ending in a client-chosen `idempotency_key`
(16–64 characters of `A–Z a–z 0–9 _ -`; a UUID or ULID is fine). Keys are kept 24 hours.
- First time: the call runs; reply `201 Created`.
- Same key, same body: the call does **not** run again; the first reply is returned with `200 OK` and header `Idempotent-Replayed: true`.
- Same key, different body: `422 IDEMPOTENCY_CONFLICT`.

**Identifiers.**

| Id | Format | Example | Issued by |
|---|---|---|---|
| `learner_id` | `lrn_` + 8–40 letters/digits | `lrn_8f3a2c91d7` | GoldSeam (pseudonymous — never a name, SSN or school student ID) |
| `unitid` | exactly 6 digits | `225070` | IPEDS, via CourseShelf |
| `learning_unit_id` | `lu_` + letters/digits/underscores | `lu_225070_ENGL1301` | CourseShelf |
| `section_id` | `sec_` + letters/digits/underscores | `sec_225070_2027SP_ENGL1301_002` | GoldWire |
| `quote_id` | `qt_` + 10–26 character ULID | `qt_01J8Z3V6N4` | GoldWire |
| `hold_id` | `hld_` + ULID | `hld_01J8Z3XQ2K` | GoldWire |
| `booking_id` | `bkg_` + ULID | `bkg_01J8Z42M7R` | GoldWire |
| `confirmation_number` | `GW-` + unitid + `-` + 4 + `-` + 2 characters | `GW-225070-7Q4K-2M` | GoldWire, only on CONFIRMED |
| `goldcheck_ref` | `gck_` + 6–40 letters/digits | `gck_5TR20P` | GoldCheck |
| `goldcard_authorization_id` | `auth_` + 6–40 | `auth_7HF2Q9` | GoldCard (prepay) |
| `goldcard_guarantee_id` | `gtd_` + 6–40 | `gtd_4KX81M` | GoldCard (reserve) |
| `funding_source.token` | `fs_` + 8–40 | `fs_Q2w8Lk3mZ9` | GoldCard (tokenized bank account, card, loan or 529 plan) |
| `policy_id` | `pol_` + letters/digits/underscores | `pol_225070_refunds_2026_27` | CourseShelf (the school's published policy) |

**Money.** Always an object: `{ "amount": "241.00", "currency": "USD" }`. `amount` is a string with
exactly two decimal places (never a floating-point number). `currency` is ISO 4217, default `USD`.

**Times.** Instants are ISO-8601 UTC (`2026-09-25T14:02:11Z`). Dates are `YYYY-MM-DD`. Clock times of
a class meeting are the school's local `HH:MM`, 24-hour.

**Reply envelope — on every successful reply:**

| Field | Always present | Meaning |
|---|---|---|
| `contract` | yes | `"goldwire_v1"` (GoldCard replies say `"goldcard_v1"`) |
| `request_id` | yes | the correlation id |
| `status` | yes | `ok` — answered in full · `not_held` — GoldSeam has not captured the record asked for (an honest gap, HTTP 200) · `offline` — the school's system did not answer; what is returned is what GoldWire already knew |
| `as_of` | yes | when the numbers in the reply were read |
| `source` | when numbers came from the school | `kind` (`school_sis_feed`, `school_published_schedule`, `school_fee_policy`, `goldwire_ledger`), plus `unitid`, `feed`, `policy_id`, `edition`, `published`, `read_at` as they apply |
| `not_held` | yes (may be empty) | list of `{ "what": "...", "note": "..." }` — each thing the school has not published |

**Error reply — on every error:**

```json
{
  "contract": "goldwire_v1",
  "request_id": "0b6d2f4e-1a3c-4e5f-8a7b-9c0d1e2f3a4b",
  "status": "error",
  "error": {
    "code": "QUOTE_EXPIRED",
    "message": "Quote qt_01J8Z3V6N4 expired at 2026-09-25T14:33:40Z. Request a new quote for section sec_225070_2027SP_ENGL1301_002.",
    "retry": "new_quote",
    "field": null,
    "details": { "quote_id": "qt_01J8Z3V6N4", "expired_at": "2026-09-25T14:33:40Z" }
  }
}
```

- `code` — fixed, machine-readable; clients branch on this, never on the message.
- `message` — the exact human text in the tables below, with `{placeholders}` filled in.
- `retry` — what the client should do: `no` · `fix_request` · `same_request` (retry as is, later) · `new_quote` · `new_hold` · `reauthorize` · `contact_school`.
- `field` — the request field at fault, for validation errors; otherwise `null`.
- `details` — the values named in the message, so a client never has to parse text.

**Errors every call can return** (listed again under each call):

| HTTP | Code | Message | When |
|---|---|---|---|
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | no token, or token expired/invalid |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than {learner_id}.` | token subject ≠ `learner_id` |
| 400 | `VALIDATION_FAILED` | one message per field, listed under each call | a field is missing or malformed |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after {retry_after_seconds} seconds.` | more than 60 calls a minute from one learner |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference {request_id}.` | an unexpected fault; any partial write is rolled back |

**Privacy.** Records name the learner only by `learner_id`. A school student reference, when a school
requires one, goes to the school and is stored encrypted; it is never returned in any reply.

---

## Call 1 — get_section_openings

**Purpose.** List the sections of one course at one school for one term, with how many seats each has
right now.

**Method and path.** `GET /goldwire/v1/sections`

**Called by.** The client, after GoldCheck, when the learner wants to see where and when the course runs.

**Changes nothing.** Writes nothing to S3.

### Request — query parameters

| Field | Required | Type / format | Allowed values | Rules and meaning |
|---|---|---|---|---|
| `learner_id` | yes | string, `lrn_…` | | must match the token |
| `unitid` | yes | string, 6 digits | | the school |
| `learning_unit_id` | yes | string, `lu_…` | | the course; must belong to `unitid` |
| `term` | yes | string, 2–16 characters | the school's own term code | passed to the school unchanged, e.g. `2027SP` or `202710` |
| `goldcheck_ref` | no | string, `gck_…` | | if sent, each section carries `goldcheck_note` |
| `modality` | no | string | `in_person`, `online`, `hybrid`, `any` | default `any` |
| `campus` | no | string, ≤ 80 | | exact match on the school's campus name |
| `days` | no | string | letters from `M T W R F S U` (R = Thursday, U = Sunday) | returns sections meeting **only** on these days |
| `time_window` | no | string `HH:MM-HH:MM` | | returns sections whose every meeting falls inside the window |
| `include_full` | no | boolean | `true`, `false` | default `false`; `true` also returns sections with 0 seats, with their waitlist |

Headers: `Authorization: Bearer <token>` (required), `X-Request-Id` (optional).

### What the service does

1. Checks the token and every field (errors below).
2. Looks up `unitid`. Not a known institution → `404 SCHOOL_NOT_FOUND`.
3. Looks up `learning_unit_id`. Unknown → `404 COURSE_NOT_FOUND`. Belongs to another school → `422 COURSE_NOT_AT_SCHOOL`.
4. Checks GoldSeam holds a schedule for this school and term. If not: reply `200`, `status: "not_held"`, `sections: []`, and a `not_held` entry `{ "what": "schedule for term {term} at school {unitid}" }`.
5. Reads live enrollment from the school's system (8-second timeout). If the school does not answer: `503 SCHOOL_OFFLINE`.
6. For each section: `held` = GoldWire's un-expired holds on it; `available` = `capacity − enrolled − held`, and never less than 0.
7. Applies the filters. Drops sections with `available = 0` unless `include_full=true`.
8. Sets `booking.bookable` and, when false, `booking.not_bookable_reason`.

### Success reply — `200 OK`

| Field | Type | Meaning |
|---|---|---|
| `contract` | string | `"goldwire_v1"` |
| `request_id` | uuid | correlation id |
| `status` | string | `ok` or `not_held` |
| `as_of` | instant | when seat counts were read |
| `source.kind` | string | `school_sis_feed` (live counts) or `school_published_schedule` (school gives no live counts; every section has `bookable: false`) |
| `source.unitid` | string | the school |
| `source.feed` | string | the school's system, e.g. `banner-ssb`, `colleague`, `peoplesoft`, `workday` |
| `source.read_at` | instant | when the school answered |
| `not_held` | list | anything missing, e.g. `{ "what": "seat count for section 004" }` |
| `learning_unit.learning_unit_id` | string | the course |
| `learning_unit.code` | string | e.g. `ENGL 1301` |
| `learning_unit.title` | string | e.g. `Composition I` |
| `learning_unit.credits` | number | credit hours as published |
| `term` | string | echoed |
| `sections[]` | list | one entry per section, below; may be empty |
| `sections[].section_id` | string | GoldWire's id for the section |
| `sections[].section_number` | string | the school's section number, e.g. `002` |
| `sections[].crn` | string | the school's course reference number, if it has one |
| `sections[].modality` | string | `in_person`, `online`, `hybrid` |
| `sections[].campus` | string | as published |
| `sections[].meetings[]` | list | each `{ days, start, end, room }`; empty for fully online, self-paced sections |
| `sections[].instructor` | string | as published; `Staff` if the school says so |
| `sections[].starts_on` / `ends_on` | date | first and last class day |
| `sections[].seats.capacity` | integer | seats the school allows |
| `sections[].seats.enrolled` | integer | registered, per the school |
| `sections[].seats.held` | integer | GoldWire's own un-expired holds |
| `sections[].seats.available` | integer | `capacity − enrolled − held`, ≥ 0 |
| `sections[].seats.waitlist.open` | boolean | the school runs a waitlist for this section |
| `sections[].seats.waitlist.length` | integer | people on it now |
| `sections[].seats.waitlist.capacity` | integer | its limit, if published |
| `sections[].booking.bookable` | boolean | GoldWire can hold and book this section |
| `sections[].booking.not_bookable_reason` | string or null | `registration_not_open`, `registration_closed`, `school_takes_no_bookings`, `section_cancelled`, `restricted_section` |
| `sections[].booking.registration_opens_at` | instant or null | when `registration_not_open` |
| `sections[].booking.requires[]` | list | what the school checks: `prerequisite_check`, `instructor_permission`, `placement_score`, `advisor_approval` |
| `sections[].booking.hold_window_minutes` | integer | how long a hold on this section lasts (POC: 20) |
| `sections[].add_deadline` | date | last day to add, per the school |
| `sections[].drop_deadline_full_refund` | date | last day to drop with a full refund, per the school |
| `sections[].goldcheck_note` | string | only when `goldcheck_ref` was sent, e.g. `Counts toward: English Composition requirement, AA General Studies (checklist chk_225070_AA_GS)` |

### Example

`GET /goldwire/v1/sections?learner_id=lrn_8f3a2c91d7&unitid=225070&learning_unit_id=lu_225070_ENGL1301&term=2027SP&modality=in_person`

```json
{
  "contract": "goldwire_v1",
  "request_id": "7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47",
  "status": "ok",
  "as_of": "2026-09-25T14:02:11Z",
  "source": { "kind": "school_sis_feed", "unitid": "225070", "feed": "banner-ssb", "read_at": "2026-09-25T14:02:09Z" },
  "not_held": [],
  "learning_unit": { "learning_unit_id": "lu_225070_ENGL1301", "code": "ENGL 1301", "title": "Composition I", "credits": 3 },
  "term": "2027SP",
  "sections": [
    {
      "section_id": "sec_225070_2027SP_ENGL1301_002",
      "section_number": "002",
      "crn": "21457",
      "modality": "in_person",
      "campus": "Main",
      "meetings": [ { "days": "TR", "start": "09:30", "end": "10:50", "room": "LA 114" } ],
      "instructor": "Staff",
      "starts_on": "2027-01-19",
      "ends_on": "2027-05-14",
      "seats": { "capacity": 25, "enrolled": 21, "held": 2, "available": 2,
                 "waitlist": { "open": true, "length": 0, "capacity": 5 } },
      "booking": { "bookable": true, "not_bookable_reason": null, "registration_opens_at": null,
                   "requires": ["prerequisite_check"], "hold_window_minutes": 20 },
      "add_deadline": "2027-01-26",
      "drop_deadline_full_refund": "2027-02-02"
    }
  ]
}
```

### Errors

| HTTP | Code | Exact message | When | Client should |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `learner_id is required.` | missing | fix request |
| 400 | `VALIDATION_FAILED` | `learner_id must look like lrn_ followed by 8–40 letters or digits; got '{value}'.` | malformed | fix request |
| 400 | `VALIDATION_FAILED` | `unitid is required.` | missing | fix request |
| 400 | `VALIDATION_FAILED` | `unitid must be exactly 6 digits; got '{value}'.` | malformed | fix request |
| 400 | `VALIDATION_FAILED` | `learning_unit_id is required.` | missing | fix request |
| 400 | `VALIDATION_FAILED` | `learning_unit_id must look like lu_…; got '{value}'. Use an id from CourseShelf learning_units.` | malformed | fix request |
| 400 | `VALIDATION_FAILED` | `term is required. Use the school's own term code, e.g. 2027SP.` | missing | fix request |
| 400 | `VALIDATION_FAILED` | `modality must be one of in_person, online, hybrid, any; got '{value}'.` | bad value | fix request |
| 400 | `VALIDATION_FAILED` | `days may contain only the letters M T W R F S U; got '{value}'.` | bad value | fix request |
| 400 | `VALIDATION_FAILED` | `time_window must be HH:MM-HH:MM with the start before the end; got '{value}'.` | bad value | fix request |
| 400 | `VALIDATION_FAILED` | `goldcheck_ref must look like gck_…; got '{value}'.` | malformed | fix request |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than {learner_id}.` | | fix request |
| 404 | `SCHOOL_NOT_FOUND` | `No institution with UNITID {unitid} is held. Find the school with CourseShelf find_institution.` | unknown UNITID | fix request |
| 404 | `COURSE_NOT_FOUND` | `No course {learning_unit_id} is held. Find the course with CourseShelf learning_units.` | unknown course | fix request |
| 404 | `GOLDCHECK_REF_NOT_FOUND` | `No GoldCheck answer {goldcheck_ref} is held for learner {learner_id}.` | bad or foreign ref | drop the ref or fix it |
| 422 | `COURSE_NOT_AT_SCHOOL` | `Course {learning_unit_id} belongs to school {actual_unitid}, not {unitid}.` | mismatch | fix request |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after {retry_after_seconds} seconds.` | | wait |
| 503 | `SCHOOL_OFFLINE` | `The registration system at school {unitid} did not answer within 8 seconds. Seat counts could not be read. Try again shortly.` | school timeout | same request, later |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference {request_id}.` | | same request, later |

A term GoldSeam has not captured is **not** an error: `200`, `status: "not_held"`, `sections: []`.

---

## Call 2 — get_section_fees

**Purpose.** Price one seat in one section, split into what is due now and what is due at the school,
and fix that price in a quote the hold will use.

**Method and path.** `GET /goldwire/v1/sections/{section_id}/fees`

**Called by.** The client, once the learner has picked a section and a way to pay.

**Writes** one quote record to S3 (the only GET that writes; the price must be provable later).
Calling again with the same learner, section, payment mode, residency and funding source type while
the quote is live returns the **same** `quote_id`.

### Request

| Field | In | Required | Type / format | Allowed values | Rules and meaning |
|---|---|---|---|---|---|
| `section_id` | path | yes | `sec_…` | | from Call 1 |
| `learner_id` | query | yes | `lrn_…` | | must match the token |
| `payment_mode` | query | yes | string | `reserve`, `prepay` | reserve = book now, the school bills later · prepay = GoldCard pays at booking |
| `residency` | query | no | string | `in_district`, `in_state`, `out_of_state`, `international`, `unknown` | default `unknown`. The learner's own declaration; the school may re-rate it later (see Call 9) |
| `funding_source_type` | query | no | string | `bank_ach`, `credit_card`, `loan`, `plan_529` | only used to add a surcharge the school publishes, or to refuse a source the school does not accept |
| `currency` | query | no | 3 capital letters | | default `USD`; POC accepts only `USD` |

### What the service does

1. Checks the token and fields.
2. Finds the section. Unknown → `404 SECTION_NOT_FOUND`. Cancelled → `409 SECTION_CANCELLED`. Not bookable → `409 SECTION_NOT_BOOKABLE`.
3. If `funding_source_type` is sent and the school does not accept it through GoldCard → `422 FUNDING_SOURCE_NOT_ACCEPTED`.
4. Reads the school's published tuition and fee policies for the term. No tuition policy held → `200`, `status: "not_held"`, no quote.
5. **If `residency` is `unknown`:** returns every published tuition rate in `rate_options[]` and **no quote** (`quote_id: null`). A hold needs a quote, so the client asks again with residency set.
6. Builds line items: tuition (credits × the rate), each fee the school publishes for the section, any funding-source surcharge, and the GoldWire booking fee. A fee the school names but does not price goes in `not_held` and `totals.totals_complete` becomes `false`.
7. Splits into `due.now` and `due.at_school`: prepay puts the whole total in `now`; reserve puts it in `at_school` with the school's due date.
8. Issues a `quote_id` valid for 30 minutes, writes it to S3 (write-once), and replies.

### Success reply — `200 OK`

| Field | Type | Meaning |
|---|---|---|
| `contract` | string | `"goldwire_v1"` |
| `request_id` | uuid | |
| `status` | string | `ok` or `not_held` |
| `as_of` | instant | when the fee record was read |
| `source.kind` | string | `school_fee_policy` |
| `source.policy_id` | string | the tuition policy used |
| `source.edition` | string | e.g. `2026–27` |
| `source.published` | date | when the school published it |
| `not_held` | list | e.g. `{ "what": "lab fee", "note": "named in the catalog, amount not published" }` |
| `quote_id` | string or null | null when residency is `unknown` or the fee record is not held |
| `quote_expires_at` | instant or null | 30 minutes after issue |
| `section_id` | string | echoed |
| `payment_mode` | string | echoed |
| `residency_applied` | string | the residency the tuition line used |
| `rate_options[]` | list | only when residency is `unknown`: each `{ residency, rate_per_credit (Money), tuition (Money), source }` |
| `line_items[]` | list | each item below |
| `line_items[].code` | string | `TUITION`, `GEN_FEE`, `COURSE_FEE`, `LAB_FEE`, `TECH_FEE`, `FUNDING_SURCHARGE`, `GW_BOOKING` |
| `line_items[].label` | string | human text, e.g. `Tuition, 3 credits × $62.00` |
| `line_items[].amount` | Money | |
| `line_items[].payee` | string | `school`, `goldwire`, `goldcard` |
| `line_items[].source` | string | the policy or course id the amount is published in |
| `totals.school` | Money | sum of items paid to the school |
| `totals.goldwire` | Money | sum of items paid to GoldWire |
| `totals.total` | Money | everything |
| `totals.totals_complete` | boolean | false if any fee is not held |
| `due.now` | Money | prepay: `totals.total`; reserve: `0.00` |
| `due.at_school` | Money | prepay: `0.00`; reserve: `totals.total` |
| `due.school_due_date` | date or null | reserve: the school's payment due date |
| `refund_policy.full_refund_until` | date | |
| `refund_policy.source` | string | the refund policy id |
| `no_show_fee` | object or null | `{ amount (Money), applies_after (date), source }` if the school publishes a late-drop / no-show fee; otherwise `null` |
| `s3.key` / `s3.etag` | string | where the quote is recorded |

### Example

`GET /goldwire/v1/sections/sec_225070_2027SP_ENGL1301_002/fees?learner_id=lrn_8f3a2c91d7&payment_mode=prepay&residency=in_district`

```json
{
  "contract": "goldwire_v1",
  "request_id": "9a4d6e13-8c2b-4f71-a5d0-3e9b7c1f2a64",
  "status": "ok",
  "as_of": "2026-09-25T14:03:40Z",
  "source": { "kind": "school_fee_policy", "policy_id": "pol_225070_tuition_2026_27", "edition": "2026–27", "published": "2026-06-01" },
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
  "s3": { "key": "goldwire/225070/2027SP/sec_225070_2027SP_ENGL1301_002/quotes/qt_01J8Z3V6N4.json", "etag": "\"3f1c9d0e2b7c41a5f6\"" }
}
```

### Errors

| HTTP | Code | Exact message | When | Client should |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `learner_id is required.` / `learner_id must look like lrn_ followed by 8–40 letters or digits; got '{value}'.` | | fix request |
| 400 | `VALIDATION_FAILED` | `section_id must look like sec_…; got '{value}'. Use an id from get_section_openings.` | malformed | fix request |
| 400 | `VALIDATION_FAILED` | `payment_mode is required: reserve or prepay.` | missing | fix request |
| 400 | `VALIDATION_FAILED` | `payment_mode must be reserve or prepay; got '{value}'.` | bad value | fix request |
| 400 | `VALIDATION_FAILED` | `residency must be one of in_district, in_state, out_of_state, international, unknown; got '{value}'.` | bad value | fix request |
| 400 | `VALIDATION_FAILED` | `funding_source_type must be one of bank_ach, credit_card, loan, plan_529; got '{value}'.` | bad value | fix request |
| 400 | `VALIDATION_FAILED` | `currency {value} is not supported. This service quotes in USD.` | not USD | fix request |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than {learner_id}.` | | fix request |
| 404 | `SECTION_NOT_FOUND` | `No section {section_id} is held. List sections with get_section_openings.` | | fix request |
| 409 | `SECTION_CANCELLED` | `Section {section_number} of {course_code} ({section_id}) was cancelled by the school on {cancelled_on}.` | | choose another section |
| 409 | `SECTION_NOT_BOOKABLE` | `Section {section_id} cannot be booked through GoldWire: {reason_text}.` where `{reason_text}` is one of: `registration opens {registration_opens_at}` · `registration closed on {closed_on}` · `the school does not take bookings through GoldWire; register with the school directly` · `the section is restricted to {restriction}` | | per reason |
| 422 | `FUNDING_SOURCE_NOT_ACCEPTED` | `School {unitid} does not accept {funding_source_type} through GoldCard. Accepted: {accepted_list}.` | | choose another source |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after {retry_after_seconds} seconds.` | | wait |
| 503 | `SCHOOL_OFFLINE` | `School {unitid}'s fee record could not be read just now. Try again shortly.` | | same request, later |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference {request_id}.` | | same request, later |

Not errors: fee policy not captured → `200`, `status: "not_held"`, `quote_id: null`. Residency `unknown`
→ `200`, `rate_options[]`, `quote_id: null`.

---

## Call 3 — hold_seat

**Purpose.** Take one seat out of `available` for 20 minutes while the learner arranges payment — the
airline "hold this fare" step.

**Method and path.** `PUT /goldwire/v1/holds/{idempotency_key}`

**Called by.** The client, with a live quote.

### Request

Path: `idempotency_key` (required, 16–64 of `A–Z a–z 0–9 _ -`). Headers: `Authorization` (required),
`X-Request-Id` (optional). Body:

| Field | Required | Type / format | Allowed values | Rules and meaning |
|---|---|---|---|---|
| `learner_id` | yes | `lrn_…` | | must match the token |
| `section_id` | yes | `sec_…` | | must be the quote's section |
| `quote_id` | yes | `qt_…` | | must be this learner's, un-expired, with a non-null price |
| `payment_mode` | yes | string | `reserve`, `prepay` | must equal the quote's |
| `goldcheck_ref` | recommended | `gck_…` | | recorded on the hold and the booking; not re-judged |
| `accept_waitlist` | no | boolean | | default `false`. If the section filled since the quote: `true` joins the waitlist, `false` returns `SECTION_FULL` |

No other fields are accepted (`VALIDATION_FAILED` names any unknown field).

### What the service does

1. Idempotency: if the key was used before, replay or conflict (see §2).
2. Checks the token and fields.
3. Loads the quote. Not found (or another learner's) → `404 QUOTE_NOT_FOUND`. Expired → `410 QUOTE_EXPIRED`. For another section → `422 QUOTE_SECTION_MISMATCH`. Different payment mode → `422 PAYMENT_MODE_MISMATCH`.
4. Re-reads the school's fee record. If the total changed → `409 PRICE_CHANGED` (the old quote is dead; a new quote is needed).
5. Checks the learner's other holds and bookings. Already holding or booked in **any** section of this course this term → `409 ACTIVE_HOLD_EXISTS` or `409 ALREADY_BOOKED`. Five active holds already → `409 HOLD_LIMIT_REACHED`.
6. Checks the section is still bookable → else `409 SECTION_NOT_BOOKABLE`.
7. Takes the seat with an atomic "decrement if available > 0" in the seat store.
   - Seat taken → `hold_state: HELD`, `hold_expires_at` = now + 20 minutes.
   - No seat, `accept_waitlist: true`, waitlist has room → joins the school's waitlist through GoldWire: `hold_state: WAITLISTED`, `waitlist_position`.
   - No seat otherwise → `hold_state: SECTION_FULL`, no `hold_id`, nothing held. **This is a normal reply (201), not an error.**
8. Writes the hold record to S3, then replies. If the S3 write fails, the seat is given back and the call returns `500`.

### Success reply — `201 Created` (first time) or `200 OK` with `Idempotent-Replayed: true` (replay)

| Field | Type | Meaning |
|---|---|---|
| `contract` | string | `"goldwire_v1"` |
| `request_id` | uuid | |
| `status` | string | `ok` |
| `as_of` | instant | when the seat was taken |
| `not_held` | list | normally empty |
| `hold_id` | string | absent when `SECTION_FULL` |
| `hold_state` | string | `HELD`, `WAITLISTED`, `SECTION_FULL` |
| `section_id` | string | echoed |
| `quote_id` | string | echoed |
| `payment_mode` | string | echoed |
| `goldcheck_ref` | string or null | echoed |
| `hold_expires_at` | instant or null | `HELD` only: now + 20 minutes |
| `waitlist_position` | integer or null | `WAITLISTED` only |
| `waitlist_full` | boolean | `SECTION_FULL` only: true if the waitlist is also full |
| `seats_after_hold` | object | `{ capacity, enrolled, held, available }` after this call |
| `next.service` | string | `goldcard.authorize` when `HELD`; null otherwise |
| `next.payment_mode` | string | echoed |
| `next.amount_due_now` | Money | the quote's `due.now` — what GoldCard must authorize (`0.00` for reserve) |
| `s3.key` / `s3.etag` | string | the hold record; absent when `SECTION_FULL` |

### Example

`PUT /goldwire/v1/holds/01J8Z3XH7TQ5W2A9R6C4M0PBNE`

```json
{
  "learner_id": "lrn_8f3a2c91d7",
  "section_id": "sec_225070_2027SP_ENGL1301_002",
  "quote_id": "qt_01J8Z3V6N4",
  "payment_mode": "prepay",
  "goldcheck_ref": "gck_5TR20P",
  "accept_waitlist": true
}
```

Reply `201 Created`:

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
  "seats_after_hold": { "capacity": 25, "enrolled": 21, "held": 3, "available": 1 },
  "next": { "service": "goldcard.authorize", "payment_mode": "prepay", "amount_due_now": { "amount": "241.00", "currency": "USD" } },
  "s3": { "key": "goldwire/225070/2027SP/sec_225070_2027SP_ENGL1301_002/holds/hld_01J8Z3XQ2K.json", "etag": "\"a90b9d0e2b7c41a5f6\"" }
}
```

### Errors

| HTTP | Code | Exact message | When | Client should |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `idempotency_key must be 16–64 characters of letters, digits, _ or -; got '{value}'.` | bad key | fix request |
| 400 | `VALIDATION_FAILED` | `{field} is required.` — for each of `learner_id`, `section_id`, `quote_id`, `payment_mode` | missing | fix request |
| 400 | `VALIDATION_FAILED` | `{field} has the wrong format; got '{value}'.` | malformed id | fix request |
| 400 | `VALIDATION_FAILED` | `payment_mode must be reserve or prepay; got '{value}'.` | bad value | fix request |
| 400 | `VALIDATION_FAILED` | `accept_waitlist must be true or false.` | bad value | fix request |
| 400 | `VALIDATION_FAILED` | `Unknown field '{field}'. hold_seat accepts learner_id, section_id, quote_id, payment_mode, goldcheck_ref, accept_waitlist.` | extra field | fix request |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than {learner_id}.` | | fix request |
| 404 | `QUOTE_NOT_FOUND` | `No quote {quote_id} is held for learner {learner_id}.` | unknown or another learner's | new quote |
| 404 | `SECTION_NOT_FOUND` | `No section {section_id} is held. List sections with get_section_openings.` | | fix request |
| 404 | `GOLDCHECK_REF_NOT_FOUND` | `No GoldCheck answer {goldcheck_ref} is held for learner {learner_id}.` | | drop the ref or fix it |
| 409 | `PRICE_CHANGED` | `The school's price for section {section_id} changed after quote {quote_id} was issued: was {old_total}, now {new_total}. Request a new quote.` | fee record changed | new quote, show the difference |
| 409 | `ACTIVE_HOLD_EXISTS` | `Learner {learner_id} already holds a seat in {course_code} for {term} (hold {existing_hold_id}, section {existing_section_number}, expires {expires_at}). Release it first or book it.` | | release or book the other |
| 409 | `ALREADY_BOOKED` | `Learner {learner_id} already has booking {existing_booking_id} ({booking_state}) in {course_code} for {term}.` | | nothing, or release the booking first |
| 409 | `HOLD_LIMIT_REACHED` | `Learner {learner_id} has 5 active holds, the most allowed. Book or release one first.` | | release a hold |
| 409 | `SECTION_CANCELLED` | `Section {section_number} of {course_code} ({section_id}) was cancelled by the school on {cancelled_on}.` | | choose another |
| 409 | `SECTION_NOT_BOOKABLE` | `Section {section_id} cannot be booked through GoldWire: {reason_text}.` (reasons as in Call 2) | | per reason |
| 410 | `QUOTE_EXPIRED` | `Quote {quote_id} expired at {expired_at}. Request a new quote for section {section_id}.` | | new quote |
| 422 | `QUOTE_SECTION_MISMATCH` | `Quote {quote_id} is for section {quote_section_id}, not {section_id}.` | | fix request |
| 422 | `PAYMENT_MODE_MISMATCH` | `Quote {quote_id} was priced for {quote_payment_mode}, not {payment_mode}. Request a quote for {payment_mode}.` | | new quote |
| 422 | `QUOTE_HAS_NO_PRICE` | `Quote request for section {section_id} had residency unknown, so no price was fixed. Request a quote with residency set.` | a residency-unknown fee reply was used | new quote |
| 422 | `IDEMPOTENCY_CONFLICT` | `Key {idempotency_key} was already used for a different hold request on {first_used_at}. Use a new key.` | | new key |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after {retry_after_seconds} seconds.` | | wait |
| 503 | `SEAT_STORE_UNAVAILABLE` | `Seats could not be counted just now, so no seat was held. Try again shortly.` | seat store down | same request, later |
| 503 | `SCHOOL_OFFLINE` | `School {unitid} did not answer, so the waitlist could not be joined. No seat was held. Try again shortly.` | waitlist path only | same request, later |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference {request_id}.` | | same request, later |

---

## Call 4 — get_hold

**Purpose.** Read a hold: its state and how long is left. Lets a client show a countdown and learn
that a waitlist place has become a seat.

**Method and path.** `GET /goldwire/v1/holds/{hold_id}`

**Changes nothing.** Reads the hold record from S3.

### Request

| Field | In | Required | Type / format | Rules |
|---|---|---|---|---|
| `hold_id` | path | yes | `hld_…` | |
| `learner_id` | query | yes | `lrn_…` | must match the token and own the hold |

### Success reply — `200 OK`

| Field | Type | Meaning |
|---|---|---|
| `contract`, `request_id`, `status`, `as_of`, `not_held` | | as in §2 (`status` is `ok`) |
| `hold_id` | string | |
| `hold_state` | string | `HELD`, `WAITLISTED`, `OFFERED` (a waitlist place became a seat — see §12), `BOOKED`, `EXPIRED`, `RELEASED` |
| `section_id`, `quote_id`, `payment_mode`, `goldcheck_ref` | | as recorded |
| `hold_expires_at` | instant or null | for `HELD` and `OFFERED` |
| `seconds_left` | integer or null | seconds until `hold_expires_at`, never negative |
| `waitlist_position` | integer or null | `WAITLISTED` only |
| `booking_id` | string or null | `BOOKED` only |
| `history[]` | list | each `{ at, state, by }` where `by` is `learner`, `goldwire`, `school` |
| `s3.key`, `s3.etag`, `s3.version_id` | string | the current record |

### Errors

| HTTP | Code | Exact message | When | Client should |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `hold_id must look like hld_…; got '{value}'.` / `learner_id is required.` | | fix request |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than {learner_id}.` | | fix request |
| 404 | `HOLD_NOT_FOUND` | `No hold {hold_id} is held for learner {learner_id}.` | unknown or another learner's | fix request |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after {retry_after_seconds} seconds.` | polling faster than once a second | wait |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference {request_id}.` | | same request, later |

---

## Call 5 — goldcard_authorize (GoldCard)

**Purpose.** GoldCard's step between hold and book: authorize the amount due now from the learner's
funding source (prepay), or record a guarantee with no charge (reserve). GoldWire never touches money;
this call is specified here so the references line up.

**Method and path.** `PUT /goldcard/v1/authorizations/{idempotency_key}`

### Request

| Field | Required | Type / format | Allowed values | Rules and meaning |
|---|---|---|---|---|
| `learner_id` | yes | `lrn_…` | | must match the token |
| `hold_id` | yes | `hld_…` | | must be `HELD` or `OFFERED`, un-expired, this learner's |
| `quote_id` | yes | `qt_…` | | must be the hold's quote |
| `payment_mode` | yes | string | `reserve`, `prepay` | must equal the hold's |
| `funding_source.type` | yes | string | `bank_ach`, `credit_card`, `loan`, `plan_529` | must be accepted by the school |
| `funding_source.token` | yes | `fs_…` | | a GoldCard token the learner set up earlier; never raw account or card numbers |
| `amount` | yes | Money | | must equal the quote's `due.now` exactly (`0.00` for reserve) |

### What the service does

1. Checks the token, fields, the hold (reads it from GoldWire) and that `amount` equals `due.now`.
2. **prepay**, by source:
   - `credit_card` — authorization hold on the card; captured only when the booking is `CONFIRMED`. `funds_status: authorized`.
   - `bank_ach` — debit authorized; settles in 1–3 business days. `funds_status: authorized`.
   - `plan_529` — withdrawal request sent to the plan, payable to the school. `funds_status: committed`, `expected_settlement_on` from the plan.
   - `loan` — a private loan disbursement request, or a record that the learner relies on aid the school disburses. `funds_status: committed` or `pending`.
3. **reserve** — records the funding source as a guarantee against a published no-show / late-drop fee. Nothing is charged. `funds_status: pending`.
4. Authorization lasts 7 days (card, ACH) or until the plan or lender answers (529, loan).

### Success reply — `201 Created`

| Field | Type | Meaning |
|---|---|---|
| `contract` | string | `"goldcard_v1"` |
| `request_id` | uuid | |
| `status` | string | `ok` |
| `payment_mode` | string | echoed |
| `goldcard_authorization_id` | string or null | prepay only |
| `goldcard_guarantee_id` | string or null | reserve only |
| `hold_id`, `quote_id` | string | echoed |
| `authorized_amount` | Money | equals `amount` |
| `funding_source.type` | string | echoed |
| `funding_source.display` | string | e.g. `Visa ending 4242`, `529 plan — Example State Plan` |
| `funds_status` | string | `authorized`, `committed`, `pending` |
| `authorization_expires_at` | instant | |
| `expected_settlement_on` | date or null | ACH, 529, loan |

### Errors

| HTTP | Code | Exact message | When | Client should |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `{field} is required.` for each required field | | fix request |
| 400 | `VALIDATION_FAILED` | `amount must be {"amount":"0.00","currency":"USD"} form with two decimal places; got '{value}'.` | float or bad form | fix request |
| 400 | `VALIDATION_FAILED` | `funding_source.type must be one of bank_ach, credit_card, loan, plan_529; got '{value}'.` | | fix request |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than {learner_id}.` | | fix request |
| 404 | `HOLD_NOT_FOUND` | `No hold {hold_id} is held for learner {learner_id}.` | | fix request |
| 404 | `FUNDING_SOURCE_NOT_FOUND` | `No funding source {token} is on file for learner {learner_id}. Add it in GoldCard first.` | | add source |
| 409 | `HOLD_NOT_ACTIVE` | `Hold {hold_id} is {hold_state}; only a HELD or OFFERED hold can be paid for.` | released, booked, waitlisted | per state |
| 410 | `HOLD_EXPIRED` | `Hold {hold_id} expired at {expired_at}. Hold a seat again.` | | new hold |
| 422 | `AMOUNT_MISMATCH` | `amount {amount} does not equal the amount due now on quote {quote_id}: {due_now}.` | | fix request |
| 422 | `QUOTE_HOLD_MISMATCH` | `Quote {quote_id} is not the quote on hold {hold_id} ({hold_quote_id}).` | | fix request |
| 422 | `PAYMENT_MODE_MISMATCH` | `Hold {hold_id} is for {hold_payment_mode}, not {payment_mode}.` | | fix request |
| 422 | `FUNDING_SOURCE_NOT_ACCEPTED` | `School {unitid} does not accept {type} through GoldCard. Accepted: {accepted_list}.` | | another source |
| 402 | `PAYMENT_DECLINED` | `{display} was declined: {decline_text}.` where `{decline_text}` is one of: `insufficient funds` · `the card has expired` · `the issuer declined without a reason; contact the card issuer` · `the 529 plan refused the withdrawal: {plan_reason}` · `the lender declined the disbursement: {lender_reason}` · `the bank account could not be verified` | | another source, or contact the issuer |
| 422 | `IDEMPOTENCY_CONFLICT` | `Key {idempotency_key} was already used for a different authorization request on {first_used_at}. Use a new key.` | | new key |
| 503 | `PAYMENT_PROVIDER_OFFLINE` | `The {type} provider did not answer. Nothing was authorized. Try again shortly.` | | same request, later |
| 500 | `INTERNAL_ERROR` | `GoldCard could not complete the request. Nothing was authorized. Reference {request_id}.` | | same request, later |

---

## Call 6 — book_seat

**Purpose.** Turn a paid-for (or guaranteed) hold into a registration request to the school, and
report the school's answer.

**Method and path.** `PUT /goldwire/v1/bookings/{idempotency_key}`

### Request

Path: `idempotency_key`. Headers: `Authorization`, `X-Request-Id` (optional). Body:

| Field | Required | Type / format | Allowed values | Rules and meaning |
|---|---|---|---|---|
| `learner_id` | yes | `lrn_…` | | must match the token and own the hold |
| `hold_id` | yes | `hld_…` | | must be `HELD` or `OFFERED` and un-expired |
| `payment_mode` | yes | string | `reserve`, `prepay` | must equal the hold's |
| `goldcard_authorization_id` | if prepay | `auth_…` | | must be for this hold and the full `due.now` |
| `goldcard_guarantee_id` | if reserve | `gtd_…` | | must be for this hold |
| `school_student_ref` | if the school requires it | string, ≤ 40 | | the learner's student ID at the school. Sent to the school only; stored encrypted; never returned |
| `attestations.prerequisites_met` | yes | boolean | | must be `true` when the section lists `prerequisite_check` |
| `attestations.policies_acknowledged` | yes | list of `pol_…`, ≥ 1 | | must include the refund policy on the quote |

Sending both GoldCard ids, or the one that does not match `payment_mode`, is `VALIDATION_FAILED`.

### What the service does

1. Idempotency (§2), token and fields.
2. Loads the hold. Not found → `404 HOLD_NOT_FOUND`. Expired → `410 HOLD_EXPIRED`. Already booked under another key → `409 HOLD_ALREADY_BOOKED`. Released or waitlisted → `409 HOLD_NOT_ACTIVE`.
3. Checks the GoldCard reference with GoldCard: it exists, is for this hold, this mode and the full amount, and is not expired.
4. Checks attestations and, if the school needs one, `school_student_ref`.
5. Writes the booking to S3 as `SUBMITTED` (write-once), marks the hold `BOOKED`.
6. Sends the registration to the school (8-second timeout) with: the school's term code, CRN or section number, the student reference, and the GoldWire `booking_id`.
7. Records the school's answer and replies with the result:

| School answers | `booking_state` | `school.answer` | Money | Reply |
|---|---|---|---|---|
| registered | `CONFIRMED` + `confirmation_number` | `registered` | prepay: GoldCard captures · reserve: guarantee kept to the school's due date | 201 |
| accepted, pending its own review or funds (e.g. a 529 in transit) | `SUBMITTED` | `pending` | authorization kept open | 201 |
| waitlisted | `WAITLISTED` + `waitlist_position` | `waitlisted` | prepay authorization released; learner re-pays if a seat is offered | 201 |
| refused | `REJECTED` + the school's `reason`, verbatim | `refused` | authorization or guarantee released; seat returned | 201 |
| no answer in 8 seconds | `SUBMITTED`, `status: "offline"` | `no_answer` | nothing captured; GoldWire resubmits every 5 minutes for 24 hours, then `REJECTED` with reason `The school did not answer within 24 hours.` | 201 |

A school's refusal is **an answer, not an error**: it comes back `201` with `booking_state: REJECTED`.

### Success reply — `201 Created` (first time) or `200 OK` with `Idempotent-Replayed: true`

| Field | Type | Meaning |
|---|---|---|
| `contract` | string | `"goldwire_v1"` |
| `request_id` | uuid | |
| `status` | string | `ok`, or `offline` when the school did not answer |
| `as_of` | instant | |
| `not_held` | list | normally empty |
| `booking_id` | string | |
| `booking_state` | string | `CONFIRMED`, `SUBMITTED`, `WAITLISTED`, `REJECTED` |
| `confirmation_number` | string or null | `CONFIRMED` only |
| `section_id` | string | |
| `hold_id` | string | |
| `goldcheck_ref` | string or null | carried from the hold |
| `school.unitid` | string | |
| `school.answer` | string | `registered`, `pending`, `waitlisted`, `refused`, `no_answer` |
| `school.reason` | string or null | the school's own words, when refused or pending |
| `school.registration_ref` | string or null | the school's reference for the registration |
| `school.waitlist_position` | integer or null | |
| `school.answered_at` | instant or null | |
| `payment.mode` | string | |
| `payment.goldcard_authorization_id` / `payment.goldcard_guarantee_id` | string | whichever applies |
| `payment.amount` | Money | amount authorized (`0.00` for reserve) |
| `payment.funds_status` | string | `authorized`, `committed`, `pending`, `captured`, `released` |
| `payment.capture` | string | `on_confirmation` (prepay), `at_school` (reserve), `none` (rejected) |
| `refund_policy.full_refund_until` | date | |
| `refund_policy.source` | string | |
| `s3.booking.key` / `.etag` / `.version_id` | string | the booking record |
| `s3.receipt.key` / `.etag` | string | the learner's receipt (write-once, locked) — only when `CONFIRMED` |

### Example

`PUT /goldwire/v1/bookings/01J8Z41V2S8K3N6P0D9F5G7H1J`

```json
{
  "learner_id": "lrn_8f3a2c91d7",
  "hold_id": "hld_01J8Z3XQ2K",
  "payment_mode": "prepay",
  "goldcard_authorization_id": "auth_7HF2Q9",
  "attestations": {
    "prerequisites_met": true,
    "policies_acknowledged": ["pol_225070_refunds_2026_27"]
  }
}
```

Reply `201 Created`:

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
  "school": { "unitid": "225070", "answer": "registered", "reason": null, "registration_ref": "21457/2027SP",
              "waitlist_position": null, "answered_at": "2026-09-25T14:11:52Z" },
  "payment": { "mode": "prepay", "goldcard_authorization_id": "auth_7HF2Q9",
               "amount": { "amount": "241.00", "currency": "USD" }, "funds_status": "captured", "capture": "on_confirmation" },
  "refund_policy": { "full_refund_until": "2027-02-02", "source": "pol_225070_refunds_2026_27" },
  "s3": {
    "booking": { "key": "goldwire/225070/2027SP/sec_225070_2027SP_ENGL1301_002/bookings/bkg_01J8Z42M7R.json", "etag": "\"e4d29d0e2b7c41a5f6\"", "version_id": "3HL4kqtJlcpXroDTDmJ.rmSpXd3dIbrHY" },
    "receipt": { "key": "goldwire/receipts/lrn_8f3a2c91d7/bkg_01J8Z42M7R.json", "etag": "\"77ab9d0e2b7c41a5f6\"" }
  }
}
```

### Errors

| HTTP | Code | Exact message | When | Client should |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `{field} is required.` for `learner_id`, `hold_id`, `payment_mode`, `attestations` | missing | fix request |
| 400 | `VALIDATION_FAILED` | `goldcard_authorization_id is required when payment_mode is prepay.` | | fix request |
| 400 | `VALIDATION_FAILED` | `goldcard_guarantee_id is required when payment_mode is reserve.` | | fix request |
| 400 | `VALIDATION_FAILED` | `Send goldcard_authorization_id or goldcard_guarantee_id, not both.` | | fix request |
| 400 | `VALIDATION_FAILED` | `attestations.policies_acknowledged must list at least one policy id.` | empty list | fix request |
| 400 | `VALIDATION_FAILED` | `Unknown field '{field}'. book_seat accepts learner_id, hold_id, payment_mode, goldcard_authorization_id, goldcard_guarantee_id, school_student_ref, attestations.` | | fix request |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than {learner_id}.` | | fix request |
| 404 | `HOLD_NOT_FOUND` | `No hold {hold_id} is held for learner {learner_id}.` | | fix request |
| 404 | `AUTHORIZATION_NOT_FOUND` | `GoldCard has no authorization {goldcard_authorization_id} for learner {learner_id}.` | | reauthorize |
| 404 | `GUARANTEE_NOT_FOUND` | `GoldCard has no guarantee {goldcard_guarantee_id} for learner {learner_id}.` | | reauthorize |
| 409 | `HOLD_ALREADY_BOOKED` | `Hold {hold_id} was already booked as {booking_id} on {booked_at}.` | booked under another key | read that booking |
| 409 | `HOLD_NOT_ACTIVE` | `Hold {hold_id} is {hold_state}; only a HELD or OFFERED hold can be booked.` | | per state |
| 410 | `HOLD_EXPIRED` | `Hold {hold_id} expired at {expired_at}. The seat was released. Hold a seat again.` | | new hold |
| 410 | `AUTHORIZATION_EXPIRED` | `Authorization {goldcard_authorization_id} expired at {expired_at}. Authorize again with GoldCard.` | | reauthorize |
| 422 | `PAYMENT_MISMATCH` | `Authorization {goldcard_authorization_id} is for {auth_hold_id} / {auth_amount}; this hold is {hold_id} / {due_now}.` | wrong hold or amount | reauthorize |
| 422 | `PAYMENT_MODE_MISMATCH` | `Hold {hold_id} is for {hold_payment_mode}, not {payment_mode}.` | | fix request |
| 422 | `ATTESTATION_REQUIRED` | `Section {section_id} requires a prerequisite check. attestations.prerequisites_met must be true; the school will verify it.` | false on a section that requires it | confirm with the learner |
| 422 | `POLICY_NOT_ACKNOWLEDGED` | `The learner must acknowledge refund policy {policy_id} before booking.` | refund policy missing from the list | show the policy, fix request |
| 422 | `STUDENT_REF_REQUIRED` | `School {unitid} requires the learner's student ID there (school_student_ref) to register.` | | ask the learner |
| 422 | `IDEMPOTENCY_CONFLICT` | `Key {idempotency_key} was already used for a different booking request on {first_used_at}. Use a new key.` | | new key |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after {retry_after_seconds} seconds.` | | wait |
| 503 | `PAYMENT_PROVIDER_OFFLINE` | `GoldCard could not confirm authorization {goldcard_authorization_id}. Nothing was booked. Try again shortly.` | | same request, later |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was booked. Reference {request_id}.` | | same request, later |

Not errors: the school refused (`201`, `REJECTED`); the school did not answer (`201`, `SUBMITTED`, `status: "offline"`).

---

## Call 7 — get_booking

**Purpose.** "Am I in?" — the booking as last recorded, with every state it has been in. It reads the
record; it never recomputes or re-asks the school.

**Method and path.** `GET /goldwire/v1/bookings/{booking_id}`

### Request

| Field | In | Required | Type / format | Rules |
|---|---|---|---|---|
| `booking_id` | path | yes | `bkg_…` | |
| `learner_id` | query | yes | `lrn_…` | must match the token and own the booking |

### Success reply — `200 OK`, header `ETag` = the booking record's S3 ETag

| Field | Type | Meaning |
|---|---|---|
| `contract`, `request_id`, `status`, `as_of`, `not_held` | | as in §2; `source.kind` is `goldwire_ledger` |
| `booking_id` | string | |
| `booking_state` | string | `SUBMITTED`, `CONFIRMED`, `WAITLISTED`, `REJECTED`, `DROP_REQUESTED`, `DROPPED` |
| `confirmation_number` | string or null | |
| `section_id`, `hold_id`, `goldcheck_ref` | string | |
| `course.code`, `course.title`, `section.section_number`, `section.meetings[]`, `section.starts_on` | | copied into the record at booking time so the booking reads alone |
| `school.*` | | as in Call 6 |
| `payment.*` | | as in Call 6, with the current `funds_status` |
| `refund_policy.*` | | as in Call 6 |
| `history[]` | list | oldest first; each `{ at, state, by, version_id, note }` — `by` is `learner`, `school`, `goldwire` or `goldcard`; `version_id` is the S3 version that holds that state |
| `s3.key`, `s3.etag`, `s3.version_id` | string | the current record |

Example `history`:

```json
[
  { "at": "2026-09-25T14:11:50Z", "state": "SUBMITTED", "by": "learner",  "version_id": "0pQ1rT7uVx2yZa3bCd4eFg5hIj6kLm7nO", "note": null },
  { "at": "2026-09-25T14:11:52Z", "state": "CONFIRMED", "by": "school",   "version_id": "3HL4kqtJlcpXroDTDmJ.rmSpXd3dIbrHY", "note": "registration_ref 21457/2027SP" }
]
```

### Errors

| HTTP | Code | Exact message | When | Client should |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `booking_id must look like bkg_…; got '{value}'.` / `learner_id is required.` | | fix request |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than {learner_id}.` | | fix request |
| 404 | `BOOKING_NOT_FOUND` | `No booking {booking_id} is held for learner {learner_id}.` | unknown or another learner's (never says which) | fix request |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after {retry_after_seconds} seconds.` | | wait |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference {request_id}.` | | same request, later |

---

## Call 8 — release_seat

**Purpose.** Let go of a hold, leave a waitlist, or ask the school to drop a booking — and say what
the school's published refund policy gives back today.

**Method and path.** `PUT /goldwire/v1/releases/{idempotency_key}`

### Request

| Field | Required | Type / format | Allowed values | Rules and meaning |
|---|---|---|---|---|
| `learner_id` | yes | `lrn_…` | | must match the token and own the target |
| `hold_id` | one of these two | `hld_…` | | to let go of a hold or a waitlist place |
| `booking_id` | one of these two | `bkg_…` | | to drop a booking |
| `reason` | yes | string | `changed_mind`, `schedule_conflict`, `chose_other_section`, `financial`, `other` | recorded; sent to the school on a drop |
| `note` | no | string, ≤ 500 | | free text; required when `reason` is `other` |

### What the service does

| Target and its state | What happens | `released_state` | Money |
|---|---|---|---|
| hold `HELD` or `OFFERED` | seat returned at once | `RELEASED` | GoldCard authorization (if any) released |
| hold `WAITLISTED` | removed from the school's waitlist | `RELEASED` | none held |
| booking `SUBMITTED` | withdrawal sent to the school | `DROP_REQUESTED` | authorization released when the school confirms |
| booking `WAITLISTED` | removed from the school's waitlist | `RELEASED` | none held |
| booking `CONFIRMED` | drop sent to the school; if the school answers at once → `DROPPED`, else `DROP_REQUESTED` until Call 9 | `DROPPED` or `DROP_REQUESTED` | refund per the school's refund policy on today's date; reserve: `fee_due` if a published no-show / late-drop fee applies |

The refund is computed from the school's published refund policy for today's date and cites it. The
school issues it; GoldWire reports what the policy gives and never promises more.

### Success reply — `201 Created` (first time) or `200 OK` with `Idempotent-Replayed: true`

| Field | Type | Meaning |
|---|---|---|
| `contract`, `request_id`, `status`, `as_of`, `not_held` | | as in §2 |
| `hold_id` / `booking_id` | string | whichever was released |
| `released_state` | string | `RELEASED`, `DROP_REQUESTED`, `DROPPED` |
| `refund.amount` | Money or null | what the policy gives today; null for a hold (nothing was captured) |
| `refund.percent` | integer or null | 0–100 |
| `refund.source` | string or null | the refund policy id |
| `note` | string or null | set when `status` is `offline` |
| `refund.paid_by` | string or null | `school` (tuition) or `goldcard` (a captured prepay reversed through the funding source) |
| `fee_due` | object or null | reserve only: `{ amount (Money), source }` when a published late-drop fee applies |
| `seats_after_release` | object | `{ capacity, enrolled, held, available }` |
| `s3.key`, `s3.etag`, `s3.version_id` | string | the updated hold or booking record |

### Errors

| HTTP | Code | Exact message | When | Client should |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `Send exactly one of hold_id or booking_id.` | neither or both | fix request |
| 400 | `VALIDATION_FAILED` | `reason must be one of changed_mind, schedule_conflict, chose_other_section, financial, other; got '{value}'.` | | fix request |
| 400 | `VALIDATION_FAILED` | `note is required when reason is other.` | | fix request |
| 400 | `VALIDATION_FAILED` | `note may be at most 500 characters.` | | fix request |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than {learner_id}.` | | fix request |
| 404 | `HOLD_NOT_FOUND` | `No hold {hold_id} is held for learner {learner_id}.` | | fix request |
| 404 | `BOOKING_NOT_FOUND` | `No booking {booking_id} is held for learner {learner_id}.` | | fix request |
| 409 | `ALREADY_RELEASED` | `{target_id} is already {state} (since {since}). Nothing was changed.` | released, expired, dropped, rejected | nothing |
| 409 | `HOLD_ALREADY_BOOKED` | `Hold {hold_id} was booked as {booking_id}. To give up the seat, release the booking.` | | release the booking |
| 409 | `DROP_DEADLINE_PASSED` | `The last day to drop {course_code} section {section_number} at school {unitid} was {last_drop_date}. A withdrawal now is the school's decision; contact the registrar.` | past the school's published drop date | contact the school |
| 422 | `IDEMPOTENCY_CONFLICT` | `Key {idempotency_key} was already used for a different release request on {first_used_at}. Use a new key.` | | new key |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after {retry_after_seconds} seconds.` | | wait |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference {request_id}.` | | same request, later |

Not an error: if the school does not answer a drop, the reply is `201` with `status: "offline"`,
`released_state: DROP_REQUESTED`, and the message field `note: "School {unitid} did not answer. The drop
is recorded and will be resent every 5 minutes; check get_booking."`

---

## Call 9 — record_school_answer (school adapter, internal)

**Purpose.** The school's system gives its answer on a booking after the fact — registered, refused,
waitlisted, dropped, or re-rated — and GoldWire moves the booking and the money to match.

**Method and path.** `PUT /goldwire/v1/bookings/{booking_id}/school-answer`

**Called by.** The school adapter only, with a token scoped to one `unitid`.

### Request

Headers: `Authorization: Bearer <school adapter token>` (required), `If-Match: <booking ETag>`
(required — the version this answer applies to). Path: `booking_id`. Body:

| Field | Required | Type / format | Allowed values | Rules and meaning |
|---|---|---|---|---|
| `unitid` | yes | 6 digits | | must equal the token's school and the booking's school |
| `answer` | yes | string | `registered`, `pending`, `waitlisted`, `refused`, `dropped`, `re_rated` | |
| `reason` | when `refused`, `pending` or `re_rated` | string, ≤ 500 | | the school's own words; shown to the learner verbatim |
| `registration_ref` | when `registered` | string | | the school's reference |
| `waitlist_position` | when `waitlisted` | integer ≥ 1 | | |
| `new_total` | when `re_rated` | Money | | the school's corrected price (e.g. residency re-rated) |
| `answered_at` | yes | instant | | when the school decided |

### What the service does

| Booking now | `answer` | Booking becomes | Money |
|---|---|---|---|
| `SUBMITTED` | `registered` | `CONFIRMED`, issues `confirmation_number`, writes the receipt | prepay captured |
| `SUBMITTED` | `pending` | stays `SUBMITTED`, reason recorded | unchanged |
| `SUBMITTED` | `waitlisted` | `WAITLISTED` | authorization released |
| `SUBMITTED` | `refused` | `REJECTED` | authorization / guarantee released; seat returned |
| `CONFIRMED` or `DROP_REQUESTED` | `dropped` | `DROPPED` | refund per policy, as in Call 8 |
| `SUBMITTED` or `CONFIRMED` | `re_rated` | unchanged; `price_changes[]` gains `{ from, to, reason, at }` | prepay: GoldCard asks the learner to approve the difference; never charged without approval |

Each change is a new S3 version written with `If-Match` on the version read, and the learner is
notified.

### Success reply — `200 OK`

The full booking record, exactly as `get_booking` returns it (every field listed in Call 7).

### Errors

| HTTP | Code | Exact message | When |
|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `{field} is required when answer is {answer}.` | e.g. `registration_ref` missing on `registered` |
| 400 | `VALIDATION_FAILED` | `answer must be one of registered, pending, waitlisted, refused, dropped, re_rated; got '{value}'.` | |
| 400 | `VALIDATION_FAILED` | `If-Match is required: send the ETag of the booking version this answer applies to.` | |
| 401 | `UNAUTHENTICATED` | `A school adapter token is required.` | |
| 403 | `SCOPE_MISMATCH` | `This adapter token is for school {token_unitid}; booking {booking_id} is at school {booking_unitid}.` | |
| 404 | `BOOKING_NOT_FOUND` | `No booking {booking_id} is held at school {unitid}.` | |
| 409 | `INVALID_TRANSITION` | `Booking {booking_id} is {booking_state}; answer {answer} cannot apply to it.` | e.g. `registered` on a `DROPPED` booking |
| 412 | `VERSION_CONFLICT` | `Booking {booking_id} changed since version {if_match} (now {current_etag}). Read it again and resend the answer if it still applies.` | |
| 500 | `INTERNAL_ERROR` | `GoldWire could not record the answer. Nothing was changed. Reference {request_id}.` | |

---

## 12. Background processes

These run inside GoldWire; no one calls them, but they change what the calls above return.

| Process | Runs | What it does | Learner sees |
|---|---|---|---|
| **Hold expiry** | every minute | each `HELD` or `OFFERED` hold past `hold_expires_at` → `EXPIRED`; seat returned; any GoldCard authorization on it released | `get_hold` shows `EXPIRED`; `book_seat` returns `410 HOLD_EXPIRED` |
| **Waitlist offer** | when a seat opens in a section with a waitlist | the first waitlisted learner's hold → `OFFERED` with a 24-hour window; learner notified | `get_hold` shows `OFFERED`; they pay (Call 5) and book (Call 6) as for a new hold |
| **School resubmission** | every 5 minutes, up to 24 hours | resends `SUBMITTED` bookings and `DROP_REQUESTED` drops that got no answer | eventually `CONFIRMED` / `REJECTED` / `DROPPED` |
| **Quote expiry** | on read | a quote past `quote_expires_at` is refused by `hold_seat`; the S3 record stays | `410 QUOTE_EXPIRED` |
| **Seat reconciliation** | every 15 minutes | re-reads `enrolled` from the school and corrects the seat store; never removes an active hold | counts in Call 1 stay true |

---

## 13. States

**Hold**

| State | Meaning | Goes to |
|---|---|---|
| `HELD` | a seat is kept for this learner until `hold_expires_at` | `BOOKED`, `EXPIRED`, `RELEASED` |
| `WAITLISTED` | on the school's waitlist, no seat | `OFFERED`, `RELEASED` |
| `OFFERED` | a waitlist place became a seat, kept 24 hours | `BOOKED`, `EXPIRED`, `RELEASED` |
| `SECTION_FULL` | reply only; no hold exists | — |
| `BOOKED` | turned into a booking | (final for the hold) |
| `EXPIRED` | window ran out; seat returned | (final) |
| `RELEASED` | the learner let it go | (final) |

**Booking**

| State | Meaning | Goes to |
|---|---|---|
| `SUBMITTED` | sent to the school; waiting for its answer | `CONFIRMED`, `WAITLISTED`, `REJECTED`, `DROP_REQUESTED` |
| `CONFIRMED` | the school says the learner is registered | `DROP_REQUESTED`, `DROPPED` |
| `WAITLISTED` | the school put the learner on its waitlist instead | (a new `OFFERED` hold when a seat opens), `RELEASED` |
| `REJECTED` | the school refused; its reason is recorded | (final) |
| `DROP_REQUESTED` | a drop was sent; waiting for the school | `DROPPED` |
| `DROPPED` | the school dropped the registration | (final) |

---

## 14. Error catalog — every code in one place

| Code | HTTP | Calls | Retry |
|---|---|---|---|
| `VALIDATION_FAILED` | 400 | all | fix_request |
| `UNAUTHENTICATED` | 401 | all | fix_request |
| `LEARNER_MISMATCH` | 403 | 1–8 | fix_request |
| `SCOPE_MISMATCH` | 403 | 9 | no |
| `SCHOOL_NOT_FOUND` | 404 | 1 | fix_request |
| `COURSE_NOT_FOUND` | 404 | 1 | fix_request |
| `SECTION_NOT_FOUND` | 404 | 2, 3 | fix_request |
| `QUOTE_NOT_FOUND` | 404 | 3 | new_quote |
| `HOLD_NOT_FOUND` | 404 | 4, 5, 6, 8 | fix_request |
| `BOOKING_NOT_FOUND` | 404 | 7, 8, 9 | fix_request |
| `GOLDCHECK_REF_NOT_FOUND` | 404 | 1, 3 | fix_request |
| `FUNDING_SOURCE_NOT_FOUND` | 404 | 5 | fix_request |
| `AUTHORIZATION_NOT_FOUND` | 404 | 6 | reauthorize |
| `GUARANTEE_NOT_FOUND` | 404 | 6 | reauthorize |
| `PAYMENT_DECLINED` | 402 | 5 | reauthorize |
| `SECTION_CANCELLED` | 409 | 2, 3 | no |
| `SECTION_NOT_BOOKABLE` | 409 | 2, 3 | per reason |
| `PRICE_CHANGED` | 409 | 3 | new_quote |
| `ACTIVE_HOLD_EXISTS` | 409 | 3 | no |
| `ALREADY_BOOKED` | 409 | 3 | no |
| `HOLD_LIMIT_REACHED` | 409 | 3 | no |
| `HOLD_NOT_ACTIVE` | 409 | 5, 6 | per state |
| `HOLD_ALREADY_BOOKED` | 409 | 6, 8 | no |
| `ALREADY_RELEASED` | 409 | 8 | no |
| `DROP_DEADLINE_PASSED` | 409 | 8 | contact_school |
| `INVALID_TRANSITION` | 409 | 9 | no |
| `QUOTE_EXPIRED` | 410 | 3 | new_quote |
| `HOLD_EXPIRED` | 410 | 5, 6 | new_hold |
| `AUTHORIZATION_EXPIRED` | 410 | 6 | reauthorize |
| `VERSION_CONFLICT` | 412 | 9 | fix_request |
| `COURSE_NOT_AT_SCHOOL` | 422 | 1 | fix_request |
| `FUNDING_SOURCE_NOT_ACCEPTED` | 422 | 2, 5 | fix_request |
| `QUOTE_SECTION_MISMATCH` | 422 | 3 | fix_request |
| `QUOTE_HOLD_MISMATCH` | 422 | 5 | fix_request |
| `QUOTE_HAS_NO_PRICE` | 422 | 3 | new_quote |
| `PAYMENT_MODE_MISMATCH` | 422 | 3, 5, 6 | fix_request / new_quote |
| `AMOUNT_MISMATCH` | 422 | 5 | fix_request |
| `PAYMENT_MISMATCH` | 422 | 6 | reauthorize |
| `ATTESTATION_REQUIRED` | 422 | 6 | fix_request |
| `POLICY_NOT_ACKNOWLEDGED` | 422 | 6 | fix_request |
| `STUDENT_REF_REQUIRED` | 422 | 6 | fix_request |
| `IDEMPOTENCY_CONFLICT` | 422 | 3, 5, 6, 8 | fix_request |
| `RATE_LIMITED` | 429 | 1–8 | same_request |
| `INTERNAL_ERROR` | 500 | all | same_request |
| `SCHOOL_OFFLINE` | 503 | 1, 2, 3 | same_request |
| `SEAT_STORE_UNAVAILABLE` | 503 | 3 | same_request |
| `PAYMENT_PROVIDER_OFFLINE` | 503 | 5, 6 | same_request |

**Answers that are not errors:** `status: "not_held"` (record not captured) · `hold_state: SECTION_FULL`
· `booking_state: REJECTED` (the school refused) · `status: "offline"` on `book_seat` and
`release_seat` (recorded, waiting on the school).

---

## 15. S3 records written, call by call

Bucket `goldwire-ledger-{env}`: versioning on, SSE-KMS encryption, public access blocked, Object Lock
(governance mode) on `receipts/`. Nothing is edited in place and nothing is deleted.

| Call | PUT (write) | Conditional header | GET (read) |
|---|---|---|---|
| 1 get_section_openings | — | — | — |
| 2 get_section_fees | `goldwire/{unitid}/{term}/{section_id}/quotes/{quote_id}.json` | `If-None-Match: *` (write-once) | — |
| 3 hold_seat | `goldwire/idempotency/{key}.json`, then `…/holds/{hold_id}.json` | `If-None-Match: *` on both | `…/quotes/{quote_id}.json` |
| 4 get_hold | — | — | `…/holds/{hold_id}.json` + version list for `history` |
| 5 goldcard_authorize | GoldCard's own store (not this bucket) | — | `…/quotes/{quote_id}.json`, `…/holds/{hold_id}.json` |
| 6 book_seat | `goldwire/idempotency/{key}.json`, `…/bookings/{booking_id}.json`; new version of `…/holds/{hold_id}.json` (`BOOKED`); on `CONFIRMED`, `goldwire/receipts/{learner_id}/{booking_id}.json` | `If-None-Match: *` to create; `If-Match: <etag>` for each new version | the hold and its quote |
| 7 get_booking | — | — | `…/bookings/{booking_id}.json` + version list for `history` |
| 8 release_seat | `goldwire/idempotency/{key}.json`; new version of the hold or booking | `If-None-Match: *`; `If-Match: <etag>` | the hold or booking |
| 9 record_school_answer | new version of the booking; receipt on `CONFIRMED` | `If-Match: <etag>` from the caller | the booking |
| background | new versions of holds (`EXPIRED`, `OFFERED`) and bookings (resubmission results) | `If-Match: <etag>` | holds and bookings |

Every object carries metadata `x-amz-meta-contract: goldwire_v1` and `x-amz-meta-request-id`.
Live seat counts are **not** in S3: they are in a store that can decrement atomically (e.g. DynamoDB
with a conditional update on `available > 0`). S3 is the ledger that proves each step.
