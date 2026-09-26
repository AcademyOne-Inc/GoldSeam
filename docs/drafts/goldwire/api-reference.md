# GoldWire API reference

[← Summary](README.md) · [Terms and formats](00-terms-and-formats.md) · [How GoldWire talks to any SIS](00-sis-adapter.md)

> **Draft.** Nothing here is live. This page is the table view of every call: what is sent and what
> comes back, field by field. Each call's own page has full examples, the non-error variants and every
> error message.

**Contents:** [All calls](#all-calls) · [Types used](#types-used) · [Headers and reply envelope](#headers-and-reply-envelope) ·
[1](#1-get-academic-terms) · [2](#2-get-campus-locations) · [3](#3-get-sections) · [4](#4-get-section-fees) ·
[5](#5-get-person) · [6](#6-enroll-as-a-non-degree-student) · [7](#7-hold-a-seat) · [8](#8-get-a-hold) ·
[9](#9-goldcard-authorize) · [10](#10-book-the-seat) · [11](#11-get-a-booking) · [12](#12-release-a-seat) ·
[13](#13-record-the-schools-answer) · [Every error code](#every-error-code)

---

## All calls

Base URL `https://api.goldseam.example`. Arguments marked **\*** are required.

| # | Call | Method and path | Auth | Arguments sent | Response returns | Success |
|---|---|---|---|---|---|---|
| 1 | [Get academic terms](01-get-academic-terms.md) | `GET /goldwire/v1/schools/{unitid}/terms` | app key | `unitid`\*, `academic_year`, `year`, `term_code`, `status` | `terms[]`: `term_id`, year, term code, name, academic year, dates, registration window and deadlines, `sessions[]`, bookable flag | 200 |
| 2 | [Get campus locations](02-get-campus-locations.md) | `GET /goldwire/v1/schools/{unitid}/campuses` | app key | `unitid`\*, `campus_id`, `include_buildings` | `campuses[]`: `campus_id`, name, physical/virtual, address, geo, time zone, `buildings[]` | 200 |
| 3 | [Get sections](03-get-sections.md) | `GET /goldwire/v1/schools/{unitid}/sections` | app key — **no learner** | `unitid`\*, `term_id`\*, `course_id`\*, `modality`, `campus_id`, `days`, `time_window` | `course` (id, subject, number, title, credits); `sections[]`: `section_id`, number, session, modality, campus, `meetings[]`, instructor, `seats` (capacity, enrolled, held, available, `seat_status`, waitlist), `booking` (bookable, reason, requires, hold window), deadlines | 200 |
| 4 | [Get section fees](04-get-section-fees.md) | `GET /goldwire/v1/sections/{section_id}/fees` | learner token | `section_id`\*, `learner_id`\*, `payment_mode`\*, `residency`, `funding_source_type`, `currency` | `quote_id`, expiry, `line_items[]`, `totals`, `due` (now / at school), `refund_policy`, `no_show_fee`, `rate_options[]` | 200 |
| 5 | [Get person](05-get-person.md) | `POST /goldwire/v1/schools/{unitid}/persons/lookup` | learner token | `unitid`\*, `learner_id`\*, `name.first`, `name.last`\*, `name.middle`, `name.suffix`, `former_last_names`, `date_of_birth`\*, `school_person_id`, `email`, `phone`, `postal_code`, `intended_student_type`, `term_id` | `match_status`, `person_link_id`, masked school ID, `matched_on`, `person_type`, `student_record` (student type, admission status, program), `term_load` (credits, full/part-time), `registration_readiness`, `paths` (apply / enroll non-degree), `lookup_ref` | 200 |
| 6 | [Enroll as a non-degree student](06-enroll-non-degree.md) | `PUT /goldwire/v1/schools/{unitid}/non-degree-students/{idempotency_key}` | learner token | `unitid`\*, key\*, `learner_id`\*, `lookup_ref`\*, `name`\*, `preferred_first_name`, `date_of_birth`\*, `email`\*, `phone`\*, `address`\*, `first_term_id`\*, `non_degree_reason`\*, `high_school`\*, `residency_declared`\*, `citizenship`\*, `attestations`\* | `enrollment_status`, `person_link_id`, masked school ID, `student_record`, `limits` (credit cap), `registration_readiness` | 201 |
| 7 | [Hold a seat](07-hold-seat.md) | `PUT /goldwire/v1/holds/{idempotency_key}` | learner token | key\*, `learner_id`\*, `section_id`\*, `quote_id`\*, `payment_mode`\*, `goldcheck_ref`, `accept_waitlist` | `hold_id`, `hold_state`, `hold_expires_at`, `waitlist_position`, `seats_after_hold`, `next` (amount to authorize) | 201 |
| 8 | [Get a hold](08-get-hold.md) | `GET /goldwire/v1/holds/{hold_id}` | learner token | `hold_id`\*, `learner_id`\* | `hold_state`, `hold_expires_at`, `seconds_left`, `waitlist_position`, `booking_id`, `history[]` | 200 |
| 9 | [GoldCard authorize](09-goldcard-authorize.md) | `PUT /goldcard/v1/authorizations/{idempotency_key}` | learner token | key\*, `learner_id`\*, `hold_id`\*, `quote_id`\*, `payment_mode`\*, `funding_source`\*, `amount`\* | `goldcard_authorization_id` or `goldcard_guarantee_id`, `authorized_amount`, `funds_status`, expiry, expected settlement | 201 |
| 10 | [Book the seat](10-book-seat.md) | `PUT /goldwire/v1/bookings/{idempotency_key}` | learner token | key\*, `learner_id`\*, `hold_id`\*, `payment_mode`\*, `goldcard_authorization_id` or `goldcard_guarantee_id`\*, `person_link_id`\*, `attestations`\* | `booking_id`, `booking_state`, `confirmation_number`, `school` answer, `payment`, `refund_policy`, `term_load` (before / after) | 201 |
| 11 | [Get a booking](11-get-booking.md) | `GET /goldwire/v1/bookings/{booking_id}` | learner token | `booking_id`\*, `learner_id`\* | the booking as in 10, plus `course`, `section`, `price_changes[]`, `history[]` | 200 |
| 12 | [Release a seat](12-release-seat.md) | `PUT /goldwire/v1/releases/{idempotency_key}` | learner token | key\*, `learner_id`\*, `hold_id` or `booking_id`\*, `reason`\*, `note` | `released_state`, `refund` (amount, percent, policy, paid by), `fee_due`, `seats_after_release` | 201 |
| 13 | [Record the school's answer](13-record-school-answer.md) | `PUT /goldwire/v1/bookings/{booking_id}/school-answer` | school adapter token | `booking_id`\*, `If-Match`\*, `unitid`\*, `answer`\*, `reason`, `registration_ref`, `waitlist_position`, `new_total`, `answered_at`\* | the full booking, as in 11 | 200 |

---

## Types used

| Type | Means | Example |
|---|---|---|
| string | text | `"ENGL 1301"` |
| integer | whole number | `25` |
| number | number with decimals | `3.0` |
| boolean | `true` / `false` | `true` |
| date | `YYYY-MM-DD` | `"2027-01-19"` |
| datetime | ISO-8601 UTC | `"2026-09-25T14:02:11Z"` |
| time | `HH:MM`, 24-hour, campus local | `"09:30"` |
| uuid | a UUID | `"7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47"` |
| enum | one of the listed words | `"in_person"` |
| Money | object `{ amount: string with 2 decimals, currency: 3 letters }` | `{ "amount": "241.00", "currency": "USD" }` |
| array\<T\> | a list of T | `["TUE", "THU"]` |
| object | named fields, listed below it | |
| `\| null` | may be `null` | `string \| null` |

Every identifier format (`unitid`, `term_id`, `course_id` …) is defined on
[terms and formats](00-terms-and-formats.md).

## Headers and reply envelope

**Request headers**

| Header | Calls | Required | Value |
|---|---|---|---|
| `X-Api-Key` | 1, 2, 3 | yes | the app's key, e.g. `gsk_live_4f7b2c9e1a8d` |
| `Authorization` | 4–12 | yes | `Bearer <learner token>`; the token's learner must equal `learner_id` |
| `Authorization` | 13 | yes | `Bearer <school adapter token>`, scoped to one `unitid` |
| `X-Request-Id` | all | no | a uuid; generated if absent |
| `Content-Type` | 5, 6, 7, 9, 10, 12, 13 | yes | `application/json` |
| `If-Match` | 13 | yes | the booking's ETag |

**Reply envelope — the first fields of every successful reply**

| Field | Type | Present | Values |
|---|---|---|---|
| `contract` | enum | always | `goldwire_v1` (call 9: `goldcard_v1`) |
| `request_id` | uuid | always | the `X-Request-Id` |
| `status` | enum | always | `ok` · `not_held` · `offline` |
| `as_of` | datetime | always (not call 9) | when the numbers were read |
| `source` | object | when data came from the school | `kind` enum (`school_sis_feed`, `school_published_schedule`, `school_published_calendar`, `school_fee_policy`, `goldwire_ledger`), `unitid`, `feed`, `policy_id`, `edition`, `published` date, `read_at` datetime |
| `not_held` | array\<object\> | always (may be empty) | each `{ what: string, note: string \| null }` |

**Error reply**

| Field | Type | Values |
|---|---|---|
| `contract` | enum | `goldwire_v1` / `goldcard_v1` |
| `request_id` | uuid | |
| `status` | enum | `error` |
| `error.code` | enum | see [every error code](#every-error-code) |
| `error.message` | string | the exact text on each call's page |
| `error.retry` | enum | `no` · `fix_request` · `same_request` · `new_quote` · `new_hold` · `reauthorize` · `contact_school` |
| `error.field` | string \| null | the field at fault |
| `error.details` | object | the values named in the message |

---

## 1. Get academic terms

`GET /goldwire/v1/schools/{unitid}/terms` · app key · [full page](01-get-academic-terms.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `unitid` | path | string | ✓ | 6 digits | `225070` |
| `academic_year` | query | string | | `YYYY-YY` | `2026-27` |
| `year` | query | integer | | 4 digits; the calendar year a term starts | `2027` |
| `term_code` | query | enum | | `FA` `WI` `SP` `SU` `IN` | `SP` |
| `status` | query | enum list | | comma-separated: `past` `current` `upcoming` `registration_open`; default `current,upcoming` | `registration_open` |

**Response `200`** — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `unitid` | string | always | | `225070` |
| `calendar_type` | enum | always | `semester` `quarter` `trimester` | `semester` |
| `timezone` | string | always | IANA zone | `America/Chicago` |
| `terms` | array\<object\> | always | may be empty | |
| `terms[].term_id` | string | always | `YYYY-TT` | `2027-SP` |
| `terms[].year` | integer | always | | `2027` |
| `terms[].term_code` | enum | always | `FA` `WI` `SP` `SU` `IN` | `SP` |
| `terms[].name` | string | always | | `Spring 2027` |
| `terms[].academic_year` | string | always | `YYYY-YY` | `2026-27` |
| `terms[].term_type` | enum | always | `semester` `quarter` `trimester` `intersession` | `semester` |
| `terms[].status` | enum | always | `past` `current` `registration_open` `upcoming` | `registration_open` |
| `terms[].starts_on` / `ends_on` | date | always | | `2027-01-19` / `2027-05-14` |
| `terms[].registration.opens_at` / `closes_at` | datetime | always | | `2026-09-21T13:00:00Z` |
| `terms[].registration.add_deadline` | date | always | | `2027-01-26` |
| `terms[].registration.drop_deadline_full_refund` | date | always | | `2027-02-02` |
| `terms[].registration.last_drop_date` | date | always | | `2027-04-02` |
| `terms[].bookable_through_goldwire` | boolean | always | | `true` |
| `terms[].sessions` | array\<object\> | always | | |
| `terms[].sessions[].session_id` | string | always | 1–8 `A–Z 0–9`; `FULL` = whole term | `8W1` |
| `terms[].sessions[].name` | string | always | | `First 8 weeks` |
| `terms[].sessions[].starts_on` / `ends_on` | date | always | | `2027-01-19` / `2027-03-12` |
| `terms[].native.term_code` | string | always | the school's own code | `202720` |
| `terms[].native.name` | string | always | | `Spring 2027` |

**Errors:** `VALIDATION_FAILED` 400 · `CLIENT_UNAUTHENTICATED` 401 · `SCHOOL_NOT_FOUND` 404 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500

---

## 2. Get campus locations

`GET /goldwire/v1/schools/{unitid}/campuses` · app key · [full page](02-get-campus-locations.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `unitid` | path | string | ✓ | 6 digits | `225070` |
| `campus_id` | query | string | | 1–10 `A–Z 0–9 _`, any case | `MAIN` |
| `include_buildings` | query | boolean | | default `false` | `true` |

**Response `200`** — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `unitid` | string | always | | `225070` |
| `campuses` | array\<object\> | always | | |
| `campuses[].campus_id` | string | always | capitals | `MAIN` |
| `campuses[].name` | string | always | | `Main Campus` |
| `campuses[].type` | enum | always | `physical` `virtual` | `physical` |
| `campuses[].address` | object \| null | always | `line1`, `line2`, `city`, `state` (2 letters), `postal_code`, `country` (2 letters); `null` for virtual | `100 College Way, Example City, TX 75000, US` |
| `campuses[].geo` | object \| null | always | `lat` number, `lon` number | `33.1000, -97.2000` |
| `campuses[].timezone` | string | always | IANA zone | `America/Chicago` |
| `campuses[].buildings` | array\<object\> | with `include_buildings=true` | each `building_id` string, `name` string | `LA` — Liberal Arts Building |
| `campuses[].native.campus_code` / `name` | string | always | the school's own | `M` / `Main Campus` |

**Errors:** `VALIDATION_FAILED` 400 · `CLIENT_UNAUTHENTICATED` 401 · `SCHOOL_NOT_FOUND` 404 · `CAMPUS_NOT_FOUND` 404 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500

---

## 3. Get sections

`GET /goldwire/v1/schools/{unitid}/sections` · app key, **no learner** · [full page](03-get-sections.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `unitid` | path | string | ✓ | 6 digits | `225070` |
| `term_id` | query | string | ✓ | `YYYY-TT` | `2027-SP` |
| `course_id` | query | string | ✓ | subject + space + number; `ART101`, `art 101`, `ART-101` also read as `ART 101`; space sent as `%20` | `ENGL 1301` |
| `modality` | query | enum list | | comma-separated: `in_person` `hybrid` `online_sync` `online_async`; default all | `in_person` |
| `campus_id` | query | string list | | comma-separated campus codes; default all | `MAIN` |
| `days` | query | enum list | | comma-separated: `MON` `TUE` `WED` `THU` `FRI` `SAT` `SUN`; sections meeting **only** on these days | `TUE,THU` |
| `time_window` | query | string | | `HH:MM-HH:MM`, start before end, campus local time | `08:00-12:00` |
| ~~`learner_id`~~ | | | | **not accepted** — rejected with `VALIDATION_FAILED` | |

**Response `200`** — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `unitid` | string | always | | `225070` |
| `term.term_id` / `term.name` | string | always | | `2027-SP` / `Spring 2027` |
| `course.course_id` | string | always | exact form | `ENGL 1301` |
| `course.subject` | string | always | 2–7 capitals | `ENGL` |
| `course.course_number` | string | always | 3–6, leading zeros kept | `1301` |
| `course.title` | string | always | | `Composition I` |
| `course.credits` | number | always | | `3` |
| `course.learning_unit_id` | string \| null | always | CourseShelf id, for linking | `lu_225070_ENGL1301` |
| `filters_applied` | object | always | `modality`, `campus_id`, `days` arrays or `null`; `time_window` string or `null` | |
| `sections` | array\<object\> | always | every section, full ones included | |
| `sections[].section_id` | string | always | `sec_…`, opaque | `sec_225070_2027SP_ENGL1301_002` |
| `sections[].section_number` | string | always | | `002` |
| `sections[].session.session_id` | string | always | | `FULL` |
| `sections[].session.starts_on` / `ends_on` | date | always | | `2027-01-19` / `2027-05-14` |
| `sections[].modality` | enum | always | `in_person` `hybrid` `online_sync` `online_async` | `in_person` |
| `sections[].campus.campus_id` / `name` | string | always | | `MAIN` / `Main Campus` |
| `sections[].meetings` | array\<object\> | always | empty for `online_async` | |
| `sections[].meetings[].days` | array\<enum\> | always | `MON`…`SUN` | `["TUE","THU"]` |
| `sections[].meetings[].start` / `end` | time | always | | `09:30` / `10:50` |
| `sections[].meetings[].building_id` / `room` | string \| null | always | | `LA` / `114` |
| `sections[].instructor` | string \| null | always | as published | `Staff` |
| `sections[].seats.capacity` | integer | always | | `25` |
| `sections[].seats.enrolled` | integer | always | | `21` |
| `sections[].seats.held` | integer | always | GoldWire's un-expired holds | `2` |
| `sections[].seats.available` | integer | always | capacity − enrolled − held, ≥ 0 | `2` |
| `sections[].seats.seat_status` | enum | always | `open` `waitlist` `closed` `cancelled` | `open` |
| `sections[].seats.waitlist` | object \| null | always | `open` boolean, `length` integer, `capacity` integer | `true, 0, 5` |
| `sections[].booking.bookable` | boolean | always | | `true` |
| `sections[].booking.not_bookable_reason` | enum \| null | always | `registration_not_open` `registration_closed` `school_takes_no_bookings` `section_cancelled` `restricted_section` | `null` |
| `sections[].booking.registration_opens_at` | datetime \| null | always | | `null` |
| `sections[].booking.requires` | array\<enum\> | always | `prerequisite_check` `instructor_permission` `placement_score` `advisor_approval` | `["prerequisite_check"]` |
| `sections[].booking.hold_window_minutes` | integer | always | | `20` |
| `sections[].add_deadline` / `drop_deadline_full_refund` | date | always | | `2027-01-26` / `2027-02-02` |
| `sections[].native` | object | always | `section_ref`, `modality_code`, `campus_code` — the school's own | `21457`, `TRAD`, `M` |

**Errors:** `VALIDATION_FAILED` 400 · `CLIENT_UNAUTHENTICATED` 401 · `SCHOOL_NOT_FOUND` · `TERM_NOT_FOUND` · `COURSE_NOT_FOUND` · `CAMPUS_NOT_FOUND` 404 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500 · `SCHOOL_OFFLINE` 503

---

## 4. Get section fees

`GET /goldwire/v1/sections/{section_id}/fees` · learner token · [full page](04-get-section-fees.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `section_id` | path | string | ✓ | `sec_…` from call 3 | `sec_225070_2027SP_ENGL1301_002` |
| `learner_id` | query | string | ✓ | `lrn_…` | `lrn_8f3a2c91d7` |
| `payment_mode` | query | enum | ✓ | `reserve` `prepay` | `prepay` |
| `residency` | query | enum | | `in_district` `in_state` `out_of_state` `international` `unknown` (default; no quote issued) | `in_district` |
| `funding_source_type` | query | enum | | `bank_ach` `credit_card` `loan` `plan_529` | `credit_card` |
| `currency` | query | string | | `USD` only | `USD` |

**Response `200`** — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `quote_id` | string \| null | always | `qt_…`; `null` when residency is `unknown` or fees not held | `qt_01J8Z3V6N4` |
| `quote_expires_at` | datetime \| null | always | issue + 30 minutes | `2026-09-25T14:33:40Z` |
| `section_id` | string | always | | |
| `payment_mode` | enum | always | | `prepay` |
| `residency_applied` | enum | always | | `in_district` |
| `rate_options` | array\<object\> | residency `unknown` only | each `residency` enum, `rate_per_credit` Money, `tuition` Money, `source` string | `in_state`, `104.00`, `312.00` |
| `line_items` | array\<object\> | with a quote | | |
| `line_items[].code` | enum | | `TUITION` `GEN_FEE` `COURSE_FEE` `LAB_FEE` `TECH_FEE` `FUNDING_SURCHARGE` `GW_BOOKING` | `TUITION` |
| `line_items[].label` | string | | | `Tuition, 3 credits × $62.00` |
| `line_items[].amount` | Money | | | `186.00 USD` |
| `line_items[].payee` | enum | | `school` `goldwire` `goldcard` | `school` |
| `line_items[].source` | string \| null | | policy or course id | `pol_225070_tuition_2026_27` |
| `totals.school` / `goldwire` / `total` | Money | with a quote | | `241.00` / `0.00` / `241.00` |
| `totals.totals_complete` | boolean | with a quote | `false` if a fee is not held | `true` |
| `due.now` | Money | with a quote | prepay: total; reserve: `0.00` | `241.00` |
| `due.at_school` | Money | with a quote | prepay: `0.00`; reserve: total | `0.00` |
| `due.school_due_date` | date \| null | with a quote | reserve only | `null` |
| `refund_policy.full_refund_until` | date | with a quote | | `2027-02-02` |
| `refund_policy.source` | string | with a quote | policy id | `pol_225070_refunds_2026_27` |
| `no_show_fee` | object \| null | with a quote | `amount` Money, `applies_after` date, `source` string | `null` |
| `s3.key` / `s3.etag` | string | with a quote | | `goldwire/225070/2027-SP/…/quotes/qt_01J8Z3V6N4.json` |

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `LEARNER_MISMATCH` 403 · `SECTION_NOT_FOUND` 404 · `SECTION_CANCELLED` · `SECTION_NOT_BOOKABLE` 409 · `FUNDING_SOURCE_NOT_ACCEPTED` 422 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500 · `SCHOOL_OFFLINE` 503

---

## 5. Get person

`POST /goldwire/v1/schools/{unitid}/persons/lookup` · learner token · [full page](05-get-person.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `unitid` | path | string | ✓ | 6 digits | `225070` |
| `learner_id` | body | string | ✓ | `lrn_…`; the token's own learner | `lrn_8f3a2c91d7` |
| `school_person_id` | body | string \| null | | the school's own format, 1–20; `null` = unknown. **Never an SSN** | `null` |
| `name.first` | body | string | ✓ if no ID | 1–60 | `Maria` |
| `name.middle` | body | string \| null | | | `Elena` |
| `name.last` | body | string | ✓ | 1–60 | `Lopez` |
| `name.suffix` | body | enum \| null | | `Jr` `Sr` `II` `III` `IV` | `null` |
| `former_last_names` | body | array\<string\> | | up to 5 | `["Garza"]` |
| `date_of_birth` | body | date | ✓ | not in the future | `2001-04-17` |
| `email` | body | string | one of these three if no ID | | `maria.lopez@example.com` |
| `phone` | body | string | ″ | E.164 | `+19035550142` |
| `postal_code` | body | string | ″ | 5 or 5+4 digits | `75090` |
| `intended_student_type` | body | enum | | `degree_seeking` `non_degree` | `degree_seeking` |
| `term_id` | body | string | | `YYYY-TT`; adds `term_load` to the reply | `2027-SP` |

**Response `200`** — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `unitid` / `learner_id` | string | always | echoed | |
| `match_status` | enum \| null | always | `matched` `multiple_possible` `no_match` `id_mismatch`; `null` if not held | `matched` |
| `person_link_id` | string \| null | always | `psl_…`; only when `matched` | `psl_3N8QK2WD7F` |
| `school_person_id_masked` | string | `matched` | | `A*****913` |
| `id_was` | enum | `matched` | `found` `supplied_and_verified` | `found` |
| `matched_on` | array\<enum\> | `matched` | `school_person_id` `first_name` `last_name` `date_of_birth` `email` `phone` `postal_code` | `["first_name","last_name","date_of_birth","email"]` |
| `person_type` | enum | `matched` | `current_student` `former_student` `applicant` `person` | `former_student` |
| `student_record.student_type` | enum \| null | `matched` | `degree_seeking` `non_degree` | `degree_seeking` |
| `student_record.admission_status` | enum | `matched` | `admitted` `applied` `not_applied` `not_required` | `admitted` |
| `student_record.program` | object \| null | `matched` | `program_code` string, `name` string | `AA-GS`, Associate of Arts, General Studies |
| `student_record.last_term_attended` | string \| null | `matched` | `YYYY-TT` | `2025-FA` |
| `term_load.term_id` | string | `matched` + `term_id` sent | | `2027-SP` |
| `term_load.credits_registered` | number | ″ | before any new booking | `6` |
| `term_load.load` | enum | ″ | `full_time` `part_time` | `part_time` |
| `term_load.full_time_at_credits` | number | ″ | the school's threshold | `12` |
| `registration_readiness.status` | enum | `matched` | `ready` `holds_on_account` `application_required` `readmission_required` `non_degree_enrollment_needed` | `ready` |
| `registration_readiness.message` | string | `matched` | never names a hold | `No holds on the account…` |
| `message` | string | not `matched` | plain words | |
| `more_details_would_help` | array\<enum\> | `multiple_possible` | `school_person_id` `email` `phone` `postal_code` | |
| `lookup_ref` | string | `no_match` | `plk_…`, valid 24 hours; needed by call 6 | `plk_7R2M9X4QTA` |
| `paths.degree_seeking` | object | `no_match` | `action` `apply`, `apply_url` string, `note` string | |
| `paths.non_degree` | object | `no_match` | `action` `enroll_non_degree`, `available` boolean, `service` string, `credit_limit` integer \| null, `note` string | `true`, `12` |
| `attempts_left_today` | integer | always | 0–5 | `4` |
| `next.service` | string \| null | always | `hold_seat` `get_person` `enroll_non_degree` | `hold_seat` |

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `LEARNER_MISMATCH` 403 · `SCHOOL_NOT_FOUND` 404 · `LOOKUP_LIMIT_REACHED` · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500 · `SCHOOL_OFFLINE` 503

---

## 6. Enroll as a non-degree student

`PUT /goldwire/v1/schools/{unitid}/non-degree-students/{idempotency_key}` · learner token · [full page](06-enroll-non-degree.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `unitid` | path | string | ✓ | 6 digits | `225070` |
| `idempotency_key` | path | string | ✓ | 16–64 `A–Z a–z 0–9 _ -` | `01J8Z5B2C4D6E8F0G2H4J6K8M0` |
| `learner_id` | body | string | ✓ | `lrn_…` | `lrn_2b7d4e9a1c` |
| `lookup_ref` | body | string | ✓ | `plk_…` from a `no_match` lookup, < 24 hours | `plk_9W4T6Y8U0I` |
| `name.first` / `name.last` | body | string | ✓ | 1–60 | `James` / `Okafor` |
| `name.middle` / `name.suffix` | body | string \| null | | suffix `Jr` `Sr` `II` `III` `IV` | `null` |
| `preferred_first_name` | body | string \| null | | 1–60 | `Jim` |
| `date_of_birth` | body | date | ✓ | | `1968-11-03` |
| `email` | body | string | ✓ | | `jim.okafor@example.com` |
| `phone` | body | string | ✓ | E.164 | `+19035550198` |
| `address.line1` / `city` / `state` / `postal_code` / `country` | body | string | ✓ | `state` 2 letters; `country` 2 letters | `412 Pecan Street`, `Example City`, `TX`, `75090`, `US` |
| `address.line2` | body | string \| null | | | `null` |
| `first_term_id` | body | string | ✓ | `YYYY-TT`, open for registration | `2027-SP` |
| `non_degree_reason` | body | enum | ✓ | `personal_enrichment` `job_skills` `visiting_student` `prerequisite_prep` | `personal_enrichment` |
| `high_school` | body | enum | ✓ | `diploma` `ged_or_equivalent` `homeschool_completed` `still_in_high_school` (refused) | `diploma` |
| `residency_declared` | body | enum | ✓ | `in_district` `in_state` `out_of_state` `international` | `in_district` |
| `citizenship` | body | enum | ✓ | `us_citizen` `permanent_resident` `other` | `us_citizen` |
| `attestations.information_true` | body | boolean | ✓ | must be `true` | `true` |
| `attestations.policies_acknowledged` | body | array\<string\> | ✓ | must include the school's non-degree policy | `["pol_225070_nondegree_2026_27"]` |

**Response `201`** (replay `200`, header `Idempotent-Replayed: true`) — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `unitid` / `learner_id` | string | always | echoed | |
| `enrollment_status` | enum | always | `created` `pending_school_review` `refused` `already_known` | `created` |
| `person_link_id` | string \| null | always | `psl_…` when `created` or `already_known` | `psl_8H2J4K6L8M` |
| `school_person_id_masked` | string \| null | always | | `A*****377` |
| `student_record.student_type` | enum | `created` / `already_known` | `non_degree` (or the existing type) | `non_degree` |
| `student_record.admission_status` | enum | ″ | `not_required` | `not_required` |
| `student_record.program` | object \| null | ″ | | `null` |
| `student_record.first_term_id` | string | ″ | | `2027-SP` |
| `limits.credit_limit` | integer \| null | ″ | the school's non-degree cap | `12` |
| `limits.credits_used` | number | ″ | | `0` |
| `limits.note` / `limits.source` | string | ″ | | policy `pol_225070_nondegree_2026_27` |
| `review.expected_by` / `review.contact` | date / string | `pending_school_review` | | `2026-09-30` / `Office of Admissions and Records` |
| `school_reason` | string | `refused` | the school's words | |
| `registration_readiness.status` | enum | always | `ready` `pending_school_review` `holds_on_account` | `ready` |
| `registration_readiness.message` | string | always | | |
| `next.service` | string \| null | always | `hold_seat` or `null` | `hold_seat` |

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `LEARNER_MISMATCH` 403 · `LOOKUP_REF_NOT_FOUND` · `TERM_NOT_FOUND` 404 · `LOOKUP_FOUND_RECORD` · `ALREADY_LINKED` · `NON_DEGREE_NOT_OFFERED` · `TERM_NOT_OPEN` 409 · `LOOKUP_REF_EXPIRED` 410 · `DUAL_CREDIT_NOT_SUPPORTED` · `IDEMPOTENCY_CONFLICT` 422 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500

---

## 7. Hold a seat

`PUT /goldwire/v1/holds/{idempotency_key}` · learner token · [full page](07-hold-seat.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `idempotency_key` | path | string | ✓ | 16–64 `A–Z a–z 0–9 _ -` | `01J8Z3XH7TQ5W2A9R6C4M0PBNE` |
| `learner_id` | body | string | ✓ | `lrn_…` | `lrn_8f3a2c91d7` |
| `section_id` | body | string | ✓ | the quote's section | `sec_225070_2027SP_ENGL1301_002` |
| `quote_id` | body | string | ✓ | this learner's, un-expired, priced | `qt_01J8Z3V6N4` |
| `payment_mode` | body | enum | ✓ | `reserve` `prepay`; must match the quote | `prepay` |
| `goldcheck_ref` | body | string \| null | | `gck_…` | `gck_5TR20P` |
| `accept_waitlist` | body | boolean | | default `false` | `true` |

**Response `201`** (replay `200`) — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `hold_id` | string | not `SECTION_FULL` | `hld_…` | `hld_01J8Z3XQ2K` |
| `hold_state` | enum | always | `HELD` `WAITLISTED` `SECTION_FULL` | `HELD` |
| `section_id` / `quote_id` / `payment_mode` / `goldcheck_ref` | | always | echoed | |
| `hold_expires_at` | datetime \| null | always | `HELD`: now + 20 min | `2026-09-25T14:24:05Z` |
| `waitlist_position` | integer \| null | always | `WAITLISTED` only | `null` |
| `waitlist_full` | boolean \| null | always | `SECTION_FULL` only | `null` |
| `seats_after_hold` | object | always | `capacity`, `enrolled`, `held`, `available` integers | `25, 21, 3, 1` |
| `next.service` | string \| null | always | `goldcard.authorize` when `HELD` | `goldcard.authorize` |
| `next.payment_mode` | enum | always | | `prepay` |
| `next.amount_due_now` | Money | always | the quote's `due.now` | `241.00 USD` |
| `s3.key` / `s3.etag` | string | not `SECTION_FULL` | | `…/holds/hld_01J8Z3XQ2K.json` |

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `LEARNER_MISMATCH` 403 · `QUOTE_NOT_FOUND` · `SECTION_NOT_FOUND` · `GOLDCHECK_REF_NOT_FOUND` 404 · `PRICE_CHANGED` · `ACTIVE_HOLD_EXISTS` · `ALREADY_BOOKED` · `HOLD_LIMIT_REACHED` · `SECTION_CANCELLED` · `SECTION_NOT_BOOKABLE` 409 · `QUOTE_EXPIRED` 410 · `QUOTE_SECTION_MISMATCH` · `PAYMENT_MODE_MISMATCH` · `QUOTE_HAS_NO_PRICE` · `IDEMPOTENCY_CONFLICT` 422 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500 · `SEAT_STORE_UNAVAILABLE` · `SCHOOL_OFFLINE` 503

---

## 8. Get a hold

`GET /goldwire/v1/holds/{hold_id}` · learner token · [full page](08-get-hold.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `hold_id` | path | string | ✓ | `hld_…` | `hld_01J8Z3XQ2K` |
| `learner_id` | query | string | ✓ | must own the hold | `lrn_8f3a2c91d7` |

**Response `200`** — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `hold_id` | string | always | | `hld_01J8Z3XQ2K` |
| `hold_state` | enum | always | `HELD` `WAITLISTED` `OFFERED` `BOOKED` `EXPIRED` `RELEASED` | `HELD` |
| `section_id` / `quote_id` / `payment_mode` / `goldcheck_ref` | | always | as recorded | |
| `hold_expires_at` | datetime \| null | always | `HELD` / `OFFERED` | `2026-09-25T14:24:05Z` |
| `seconds_left` | integer \| null | always | ≥ 0 | `900` |
| `waitlist_position` | integer \| null | always | `WAITLISTED` | `null` |
| `booking_id` | string \| null | always | `BOOKED` | `null` |
| `history` | array\<object\> | always | each `at` datetime, `state` enum, `by` enum (`learner` `goldwire` `school`) | |
| `s3.key` / `etag` / `version_id` | string | always | | |

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `LEARNER_MISMATCH` 403 · `HOLD_NOT_FOUND` 404 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500

---

## 9. GoldCard authorize

`PUT /goldcard/v1/authorizations/{idempotency_key}` · learner token · [full page](09-goldcard-authorize.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `idempotency_key` | path | string | ✓ | 16–64 | `01J8Z3ZP4M6Q8S0U2W4Y6A8C0E` |
| `learner_id` | body | string | ✓ | | `lrn_8f3a2c91d7` |
| `hold_id` | body | string | ✓ | `HELD` or `OFFERED`, un-expired | `hld_01J8Z3XQ2K` |
| `quote_id` | body | string | ✓ | the hold's quote | `qt_01J8Z3V6N4` |
| `payment_mode` | body | enum | ✓ | `reserve` `prepay` | `prepay` |
| `funding_source.type` | body | enum | ✓ | `bank_ach` `credit_card` `loan` `plan_529` | `credit_card` |
| `funding_source.token` | body | string | ✓ | `fs_…` — never raw numbers | `fs_Q2w8Lk3mZ9` |
| `amount` | body | Money | ✓ | equals the quote's `due.now` (`0.00` for reserve) | `241.00 USD` |

**Response `201`** — `contract` `goldcard_v1`, `request_id`, `status`, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `payment_mode` | enum | always | | `prepay` |
| `goldcard_authorization_id` | string \| null | always | prepay | `auth_7HF2Q9` |
| `goldcard_guarantee_id` | string \| null | always | reserve | `null` |
| `hold_id` / `quote_id` | string | always | echoed | |
| `authorized_amount` | Money | always | | `241.00 USD` |
| `funding_source.type` | enum | always | | `credit_card` |
| `funding_source.display` | string | always | safe to show | `Visa ending 4242` |
| `funds_status` | enum | always | `authorized` `committed` `pending` | `authorized` |
| `authorization_expires_at` | datetime | always | | `2026-10-02T14:06:30Z` |
| `expected_settlement_on` | date \| null | always | ACH, 529, loan | `null` |

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `PAYMENT_DECLINED` 402 · `LEARNER_MISMATCH` 403 · `HOLD_NOT_FOUND` · `FUNDING_SOURCE_NOT_FOUND` 404 · `HOLD_NOT_ACTIVE` 409 · `HOLD_EXPIRED` 410 · `AMOUNT_MISMATCH` · `QUOTE_HOLD_MISMATCH` · `PAYMENT_MODE_MISMATCH` · `FUNDING_SOURCE_NOT_ACCEPTED` · `IDEMPOTENCY_CONFLICT` 422 · `INTERNAL_ERROR` 500 · `PAYMENT_PROVIDER_OFFLINE` 503

---

## 10. Book the seat

`PUT /goldwire/v1/bookings/{idempotency_key}` · learner token · [full page](10-book-seat.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `idempotency_key` | path | string | ✓ | 16–64 | `01J8Z41V2S8K3N6P0D9F5G7H1J` |
| `learner_id` | body | string | ✓ | owns the hold | `lrn_8f3a2c91d7` |
| `hold_id` | body | string | ✓ | `HELD` / `OFFERED`, un-expired | `hld_01J8Z3XQ2K` |
| `payment_mode` | body | enum | ✓ | matches the hold | `prepay` |
| `goldcard_authorization_id` | body | string | ✓ if prepay | `auth_…` | `auth_7HF2Q9` |
| `goldcard_guarantee_id` | body | string | ✓ if reserve | `gtd_…`; never both | |
| `person_link_id` | body | string | ✓ | `psl_…` from call 5 or 6, this learner, this school | `psl_3N8QK2WD7F` |
| `attestations.prerequisites_met` | body | boolean | ✓ | `true` if the section requires a prerequisite check | `true` |
| `attestations.policies_acknowledged` | body | array\<string\> | ✓ | includes the quote's refund policy | `["pol_225070_refunds_2026_27"]` |

**Response `201`** (replay `200`) — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `booking_id` | string | always | `bkg_…` | `bkg_01J8Z42M7R` |
| `booking_state` | enum | always | `CONFIRMED` `SUBMITTED` `WAITLISTED` `REJECTED` | `CONFIRMED` |
| `confirmation_number` | string \| null | always | `CONFIRMED` only | `GW-225070-7Q4K-2M` |
| `section_id` / `hold_id` / `goldcheck_ref` | string | always | | |
| `school.unitid` | string | always | | `225070` |
| `school.answer` | enum | always | `registered` `pending` `waitlisted` `refused` `no_answer` | `registered` |
| `school.reason` | string \| null | always | the school's words | `null` |
| `school.registration_ref` | string \| null | always | the school's own | `202720.21457` |
| `school.waitlist_position` | integer \| null | always | | `null` |
| `school.answered_at` | datetime \| null | always | | `2026-09-25T14:11:52Z` |
| `payment.mode` | enum | always | | `prepay` |
| `payment.goldcard_authorization_id` / `goldcard_guarantee_id` | string \| null | always | | `auth_7HF2Q9` |
| `payment.amount` | Money | always | | `241.00 USD` |
| `payment.funds_status` | enum | always | `authorized` `committed` `pending` `captured` `released` | `captured` |
| `payment.capture` | enum | always | `on_confirmation` `at_school` `none` | `on_confirmation` |
| `refund_policy.full_refund_until` / `source` | date / string | always | | `2027-02-02` |
| `term_load.term_id` | string | `CONFIRMED` | | `2027-SP` |
| `term_load.credits_before` / `credits_after` | number | `CONFIRMED` | | `6` / `9` |
| `term_load.load_before` / `load_after` | enum | `CONFIRMED` | `full_time` `part_time` | `part_time` / `part_time` |
| `term_load.full_time_at_credits` | number | `CONFIRMED` | | `12` |
| `note` | string | `status: offline` | | |
| `s3.booking.key` / `etag` / `version_id` | string | always | | |
| `s3.receipt.key` / `etag` | string | `CONFIRMED` | locked | `goldwire/receipts/lrn_8f3a2c91d7/bkg_01J8Z42M7R.json` |

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `LEARNER_MISMATCH` 403 · `HOLD_NOT_FOUND` · `AUTHORIZATION_NOT_FOUND` · `GUARANTEE_NOT_FOUND` · `PERSON_LINK_NOT_FOUND` 404 · `HOLD_ALREADY_BOOKED` · `HOLD_NOT_ACTIVE` 409 · `HOLD_EXPIRED` · `AUTHORIZATION_EXPIRED` 410 · `PAYMENT_MISMATCH` · `PAYMENT_MODE_MISMATCH` · `ATTESTATION_REQUIRED` · `POLICY_NOT_ACKNOWLEDGED` · `IDEMPOTENCY_CONFLICT` 422 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500 · `PAYMENT_PROVIDER_OFFLINE` 503

---

## 11. Get a booking

`GET /goldwire/v1/bookings/{booking_id}` · learner token · [full page](11-get-booking.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `booking_id` | path | string | ✓ | `bkg_…` | `bkg_01J8Z42M7R` |
| `learner_id` | query | string | ✓ | owns the booking | `lrn_8f3a2c91d7` |

**Response `200`**, header `ETag` — envelope, then every field of call 10's reply except `note`, plus:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `booking_state` | enum | always | `SUBMITTED` `CONFIRMED` `WAITLISTED` `REJECTED` `DROP_REQUESTED` `DROPPED` | `CONFIRMED` |
| `course.code` / `title` / `credits` | string / string / number | always | copied at booking | `ENGL 1301`, `Composition I`, `3` |
| `section.section_number` / `crn` / `term_id` | string | always | | `002`, `21457`, `2027-SP` |
| `section.meetings` | array\<object\> | always | as in call 3 | `["TUE","THU"] 09:30–10:50` |
| `section.starts_on` | date | always | | `2027-01-19` |
| `price_changes` | array\<object\> | always | each `from` Money, `to` Money, `reason` string, `at` datetime, `approved` boolean | `[]` |
| `history` | array\<object\> | always | each `at` datetime, `state` enum, `by` enum (`learner` `school` `goldwire` `goldcard`), `version_id` string, `note` string \| null | |
| `s3.key` / `etag` / `version_id` | string | always | the current version | |

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `LEARNER_MISMATCH` 403 · `BOOKING_NOT_FOUND` 404 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500

---

## 12. Release a seat

`PUT /goldwire/v1/releases/{idempotency_key}` · learner token · [full page](12-release-seat.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `idempotency_key` | path | string | ✓ | 16–64 | `01J9A7K3M5P7R9T1V3X5Z7B9D1` |
| `learner_id` | body | string | ✓ | owns the target | `lrn_8f3a2c91d7` |
| `hold_id` | body | string | one of these two | `hld_…` | |
| `booking_id` | body | string | one of these two | `bkg_…` | `bkg_01J8Z42M7R` |
| `reason` | body | enum | ✓ | `changed_mind` `schedule_conflict` `chose_other_section` `financial` `other` | `schedule_conflict` |
| `note` | body | string \| null | ✓ if reason `other` | ≤ 500 | `Took a job with morning hours.` |

**Response `201`** (replay `200`) — envelope, then:

| Field | Type | Present | Values / format | Example |
|---|---|---|---|---|
| `hold_id` / `booking_id` | string \| null | always | whichever was released | `bkg_01J8Z42M7R` |
| `released_state` | enum | always | `RELEASED` `DROP_REQUESTED` `DROPPED` | `DROPPED` |
| `note` | string \| null | always | set when the school didn't answer | `null` |
| `refund.amount` | Money \| null | always | `null` for a hold | `241.00 USD` |
| `refund.percent` | integer \| null | always | 0–100 | `100` |
| `refund.source` | string \| null | always | refund policy id | `pol_225070_refunds_2026_27` |
| `refund.paid_by` | enum \| null | always | `goldcard` `school` | `goldcard` |
| `fee_due` | object \| null | always | reserve: `amount` Money, `source` string | `null` |
| `seats_after_release` | object | always | `capacity`, `enrolled`, `held`, `available` | `25, 24, 0, 1` |
| `s3.key` / `etag` / `version_id` | string | always | | |

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `LEARNER_MISMATCH` 403 · `HOLD_NOT_FOUND` · `BOOKING_NOT_FOUND` 404 · `ALREADY_RELEASED` · `HOLD_ALREADY_BOOKED` · `DROP_DEADLINE_PASSED` 409 · `IDEMPOTENCY_CONFLICT` 422 · `RATE_LIMITED` 429 · `INTERNAL_ERROR` 500

---

## 13. Record the school's answer

`PUT /goldwire/v1/bookings/{booking_id}/school-answer` · school adapter token · internal · [full page](13-record-school-answer.md)

**Request**

| Argument | In | Type | Req | Format / allowed values | Example |
|---|---|---|---|---|---|
| `booking_id` | path | string | ✓ | a booking at this school | `bkg_01J8Z42M7R` |
| `If-Match` | header | string | ✓ | the booking's ETag | `"c3d49d0e2b7c41a5f6"` |
| `unitid` | body | string | ✓ | the token's school | `225070` |
| `answer` | body | enum | ✓ | `registered` `pending` `waitlisted` `refused` `dropped` `re_rated` | `registered` |
| `reason` | body | string \| null | ✓ for `refused` `pending` `re_rated` | ≤ 500, the school's words | `null` |
| `registration_ref` | body | string \| null | ✓ for `registered` | | `202720.21457` |
| `waitlist_position` | body | integer \| null | ✓ for `waitlisted` | ≥ 1 | `null` |
| `new_total` | body | Money \| null | ✓ for `re_rated` | | `null` |
| `answered_at` | body | datetime | ✓ | | `2026-10-09T15:40:10Z` |

**Response `200`** — exactly call 11's reply.

**Errors:** `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `SCOPE_MISMATCH` 403 · `BOOKING_NOT_FOUND` 404 · `INVALID_TRANSITION` 409 · `VERSION_CONFLICT` 412 · `INTERNAL_ERROR` 500

---

## Every error code

The exact message for each, with real values, is on each call's page.

| Code | HTTP | Means | Calls |
|---|---|---|---|
| `VALIDATION_FAILED` | 400 | a field is missing or in the wrong form | all |
| `CLIENT_UNAUTHENTICATED` | 401 | no valid app API key | 1, 2, 3 |
| `UNAUTHENTICATED` | 401 | no valid learner token (adapter token for 13) | 4–13 |
| `PAYMENT_DECLINED` | 402 | the card, bank, plan or lender said no | 9 |
| `LEARNER_MISMATCH` | 403 | token is for another learner | 4–12 |
| `SCOPE_MISMATCH` | 403 | school adapter used for the wrong school | 13 |
| `SCHOOL_NOT_FOUND` | 404 | no such UNITID | 1, 2, 3, 5 |
| `TERM_NOT_FOUND` | 404 | the school has no such term | 3, 6 |
| `CAMPUS_NOT_FOUND` | 404 | the school has no such campus | 2, 3 |
| `COURSE_NOT_FOUND` | 404 | the school has no such course | 3 |
| `SECTION_NOT_FOUND` | 404 | no such section | 4, 7 |
| `QUOTE_NOT_FOUND` | 404 | no such quote for this learner | 7 |
| `LOOKUP_REF_NOT_FOUND` | 404 | no such lookup for this learner at this school | 6 |
| `PERSON_LINK_NOT_FOUND` | 404 | no such person link for this learner at this school | 10 |
| `HOLD_NOT_FOUND` | 404 | no such hold for this learner | 8, 9, 10, 12 |
| `BOOKING_NOT_FOUND` | 404 | no such booking for this learner | 11, 12, 13 |
| `GOLDCHECK_REF_NOT_FOUND` | 404 | no such GoldCheck answer | 7 |
| `FUNDING_SOURCE_NOT_FOUND` | 404 | card/account/plan not saved in GoldCard | 9 |
| `AUTHORIZATION_NOT_FOUND` | 404 | no such GoldCard authorization | 10 |
| `GUARANTEE_NOT_FOUND` | 404 | no such GoldCard guarantee | 10 |
| `SECTION_CANCELLED` | 409 | the school cancelled the section | 4, 7 |
| `SECTION_NOT_BOOKABLE` | 409 | registration not open, closed, restricted, or school not connected | 4, 7 |
| `LOOKUP_FOUND_RECORD` | 409 | the lookup matched someone; no new record | 6 |
| `ALREADY_LINKED` | 409 | the learner already has a record link at this school | 6 |
| `NON_DEGREE_NOT_OFFERED` | 409 | the school doesn't enroll non-degree students through GoldWire | 6 |
| `TERM_NOT_OPEN` | 409 | non-degree enrollment for that term isn't open | 6 |
| `PRICE_CHANGED` | 409 | the school changed its fees after the quote | 7 |
| `ACTIVE_HOLD_EXISTS` | 409 | already holding a seat in this course this term | 7 |
| `ALREADY_BOOKED` | 409 | already booked in this course this term | 7 |
| `HOLD_LIMIT_REACHED` | 409 | 5 active holds already | 7 |
| `HOLD_NOT_ACTIVE` | 409 | the hold is waitlisted or released | 9, 10 |
| `HOLD_ALREADY_BOOKED` | 409 | the hold was booked already | 10, 12 |
| `ALREADY_RELEASED` | 409 | already released, expired or dropped | 12 |
| `DROP_DEADLINE_PASSED` | 409 | past the school's last drop date | 12 |
| `INVALID_TRANSITION` | 409 | the school's answer doesn't fit the booking's state | 13 |
| `QUOTE_EXPIRED` | 410 | more than 30 minutes since the quote | 7 |
| `LOOKUP_REF_EXPIRED` | 410 | more than 24 hours since the lookup | 6 |
| `HOLD_EXPIRED` | 410 | more than 20 minutes since the hold | 9, 10 |
| `AUTHORIZATION_EXPIRED` | 410 | the GoldCard authorization ran out | 10 |
| `VERSION_CONFLICT` | 412 | the booking changed before this answer landed | 13 |
| `FUNDING_SOURCE_NOT_ACCEPTED` | 422 | the school doesn't take that kind of money through GoldCard | 4, 9 |
| `DUAL_CREDIT_NOT_SUPPORTED` | 422 | high-school students use the school's dual-credit process | 6 |
| `QUOTE_SECTION_MISMATCH` | 422 | quote is for another section | 7 |
| `QUOTE_HOLD_MISMATCH` | 422 | quote is not the hold's quote | 9 |
| `QUOTE_HAS_NO_PRICE` | 422 | the quote request had no residency | 7 |
| `PAYMENT_MODE_MISMATCH` | 422 | reserve vs prepay doesn't match | 7, 9, 10 |
| `AMOUNT_MISMATCH` | 422 | amount isn't the amount due now | 9 |
| `PAYMENT_MISMATCH` | 422 | the authorization is for another hold or amount | 10 |
| `ATTESTATION_REQUIRED` | 422 | prerequisites must be confirmed | 10 |
| `POLICY_NOT_ACKNOWLEDGED` | 422 | the refund policy must be acknowledged | 10 |
| `IDEMPOTENCY_CONFLICT` | 422 | the retry key was already used for something else | 6, 7, 9, 10, 12 |
| `LOOKUP_LIMIT_REACHED` | 429 | 5 person lookups at this school in 24 hours | 5 |
| `RATE_LIMITED` | 429 | too many calls | 1–12 |
| `INTERNAL_ERROR` | 500 | a fault inside GoldWire; nothing was changed | all |
| `SCHOOL_OFFLINE` | 503 | the school's system didn't answer | 3, 4, 5, 7 |
| `SEAT_STORE_UNAVAILABLE` | 503 | seats couldn't be counted; nothing held | 7 |
| `PAYMENT_PROVIDER_OFFLINE` | 503 | the card network, bank or GoldCard didn't answer | 9, 10 |
