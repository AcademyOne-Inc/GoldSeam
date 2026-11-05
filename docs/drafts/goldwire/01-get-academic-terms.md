# Call 1 — Get academic terms

[← How GoldWire talks to any SIS](00-sis-adapter.md) · [Summary](README.md) · Next: [Call 2 — Get campus locations →](02-get-campus-locations.md)

> **Draft.** Nothing here is live.

## What it does

Lists a school's **terms** — by academic year and term — with their dates, registration windows and
parts of term (sessions). It gives the `term_id` (e.g. `2027-SP`) every later call uses, whatever the
school's own system calls that term.

It needs no learner: it is the school's public calendar. It changes nothing.

## Where it sits

- **Before:** the school is known (`unitid 225070`, from CourseShelf).
- **This call:** "Which terms can I register for at this school?"
- **After:** pick a term → [Call 3 — Get sections](03-get-sections.md) with its `term_id`.

---

## The request

```http
GET /goldwire/v1/schools/225070/terms?academic_year=2026-27 HTTP/1.1
Host: api.goldseam.example
X-Api-Key: gsk_live_4f7b2c9e1a8d
X-Request-Id: 1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d
Accept: application/json
```

## Every field you send

| Field | This example | Required? | What it means | Format and allowed values |
|---|---|---|---|---|
| `unitid` (in the path) | `225070` | yes | the school | 6 digits |
| `X-Api-Key` (header) | `gsk_live_4f7b2c9e1a8d` | yes | identifies the **app** calling, not a person | issued to each app |
| `academic_year` | `2026-27` | no | only terms in this academic year | `YYYY-YY` |
| `year` | *(not sent)* | no | only terms that **start** in this calendar year | 4 digits, e.g. `2027` |
| `term_code` | *(not sent)* | no | only this kind of term | `FA`, `WI`, `SP`, `SU`, `IN` |
| `status` | *(not sent)* | no | only terms in this state; default is `current,upcoming` | comma-separated from `past`, `current`, `upcoming`, `registration_open` |

Formats are defined on [GoldWire terms and formats](00-terms-and-formats.md#when-it-is-taught).

---

## The reply — `200 OK`

```json
{
  "contract": "goldwire_v1",
  "request_id": "1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d",
  "status": "ok",
  "as_of": "2026-09-26T06:00:00Z",
  "source": { "kind": "school_sis_feed", "unitid": "225070", "feed": "banner", "read_at": "2026-09-26T06:00:00Z" },
  "not_held": [],
  "unitid": "225070",
  "calendar_type": "semester",
  "timezone": "America/Chicago",
  "terms": [
    {
      "term_id": "2026-FA",
      "year": 2026,
      "term_code": "FA",
      "name": "Fall 2026",
      "academic_year": "2026-27",
      "term_type": "semester",
      "status": "current",
      "starts_on": "2026-08-24",
      "ends_on": "2026-12-11",
      "registration": {
        "opens_at": "2026-04-06T13:00:00Z",
        "closes_at": "2026-08-31T04:59:59Z",
        "add_deadline": "2026-08-30",
        "drop_deadline_full_refund": "2026-09-08",
        "last_drop_date": "2026-11-06"
      },
      "bookable_through_goldwire": false,
      "sessions": [
        { "session_id": "FULL", "name": "Full term",     "starts_on": "2026-08-24", "ends_on": "2026-12-11" },
        { "session_id": "8W1",  "name": "First 8 weeks", "starts_on": "2026-08-24", "ends_on": "2026-10-16" },
        { "session_id": "8W2",  "name": "Second 8 weeks","starts_on": "2026-10-19", "ends_on": "2026-12-11" }
      ],
      "native": { "term_code": "202710", "name": "Fall 2026" }
    },
    {
      "term_id": "2027-SP",
      "year": 2027,
      "term_code": "SP",
      "name": "Spring 2027",
      "academic_year": "2026-27",
      "term_type": "semester",
      "status": "registration_open",
      "starts_on": "2027-01-19",
      "ends_on": "2027-05-14",
      "registration": {
        "opens_at": "2026-09-21T13:00:00Z",
        "closes_at": "2027-01-27T05:59:59Z",
        "add_deadline": "2027-01-26",
        "drop_deadline_full_refund": "2027-02-02",
        "last_drop_date": "2027-04-02"
      },
      "bookable_through_goldwire": true,
      "sessions": [
        { "session_id": "FULL", "name": "Full term",     "starts_on": "2027-01-19", "ends_on": "2027-05-14" },
        { "session_id": "8W1",  "name": "First 8 weeks", "starts_on": "2027-01-19", "ends_on": "2027-03-12" },
        { "session_id": "8W2",  "name": "Second 8 weeks","starts_on": "2027-03-22", "ends_on": "2027-05-14" }
      ],
      "native": { "term_code": "202720", "name": "Spring 2027" }
    },
    {
      "term_id": "2027-SU",
      "year": 2027,
      "term_code": "SU",
      "name": "Summer 2027",
      "academic_year": "2026-27",
      "term_type": "semester",
      "status": "upcoming",
      "starts_on": "2027-06-01",
      "ends_on": "2027-08-06",
      "registration": {
        "opens_at": "2027-03-29T13:00:00Z",
        "closes_at": "2027-06-03T04:59:59Z",
        "add_deadline": "2027-06-02",
        "drop_deadline_full_refund": "2027-06-04",
        "last_drop_date": "2027-07-16"
      },
      "bookable_through_goldwire": false,
      "sessions": [
        { "session_id": "SU1", "name": "Summer I",  "starts_on": "2027-06-01", "ends_on": "2027-07-02" },
        { "session_id": "SU2", "name": "Summer II", "starts_on": "2027-07-06", "ends_on": "2027-08-06" }
      ],
      "native": { "term_code": "202730", "name": "Summer 2027" }
    }
  ]
}
```

In words: in 2026-27 the school has Fall 2026 (under way), **Spring 2027 (registration open now)**, and
Summer 2027 (registration opens 29 March). Only Spring can be booked through GoldWire today.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` | | rule version; tracking number |
| `status` | `ok` | `ok`, or `not_held` if GoldSeam holds no calendar for the school |
| `as_of` | `2026-09-26T06:00:00Z` | calendars are read once a day |
| `source.kind` | `school_sis_feed` | from the school's system. `school_published_calendar` = from its published academic calendar (no live registration) |
| `source.feed` | `banner` | the school's system |
| `not_held` | `[]` | e.g. `{ "what": "term 202740", "note": "no mapping configured" }` — a term the school has but GoldWire can't yet name |
| `unitid` | `225070` | echoed |
| `calendar_type` | `semester` | `semester`, `quarter`, `trimester` |
| `timezone` | `America/Chicago` | all meeting times at this school are in this zone |
| `terms[].term_id` | `2027-SP` | **GoldWire's id for the term — send it to Call 3** |
| `terms[].year` | `2027` | calendar year the term starts |
| `terms[].term_code` | `SP` | `FA`, `WI`, `SP`, `SU`, `IN` |
| `terms[].name` | `Spring 2027` | for display |
| `terms[].academic_year` | `2026-27` | |
| `terms[].term_type` | `semester` | `semester`, `quarter`, `trimester`, `intersession` |
| `terms[].status` | `registration_open` | `past`, `current` (classes meeting), `registration_open`, `upcoming` |
| `terms[].starts_on` / `ends_on` | 19 Jan – 14 May 2027 | first and last class day |
| `terms[].registration.opens_at` / `closes_at` | | the registration window |
| `terms[].registration.add_deadline` | `2027-01-26` | last day to add a class |
| `terms[].registration.drop_deadline_full_refund` | `2027-02-02` | last day to drop with a full refund |
| `terms[].registration.last_drop_date` | `2027-04-02` | last day to drop at all (after this the school decides) |
| `terms[].bookable_through_goldwire` | `true` | whether GoldWire can hold and book seats in this term now |
| `terms[].sessions[]` | `FULL`, `8W1`, `8W2` | parts of term, each with its own dates |
| `terms[].native.term_code` | `202720` | the school's own code — for staff; never needed by clients |
| `terms[].native.name` | `Spring 2027` | the school's own name for it |

---

## Other replies you can get (not errors)

**No calendar held for the school** — `200`, `status: "not_held"`, `terms: []`,
`not_held: [ { "what": "academic calendar for school 225070" } ]`.

**A filter matches nothing** — `200`, `status: "ok"`, `terms: []`. E.g. `term_code=WI` at a semester school.

---

## Errors

Example:

```json
{
  "contract": "goldwire_v1",
  "request_id": "1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d",
  "status": "error",
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "academic_year must be YYYY-YY, e.g. 2026-27; got '2026/2027'.",
    "retry": "fix_request",
    "field": "academic_year",
    "details": { "value": "2026/2027" }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `unitid must be exactly 6 digits; got '22507'.` | | fix it |
| 400 | `VALIDATION_FAILED` | `academic_year must be YYYY-YY, e.g. 2026-27; got '2026/2027'.` | | send `2026-27` |
| 400 | `VALIDATION_FAILED` | `academic_year 2026-28 does not span one year; the second part must be 27.` | | send `2026-27` |
| 400 | `VALIDATION_FAILED` | `year must be 4 digits; got '27'.` | | send `2027` |
| 400 | `VALIDATION_FAILED` | `term_code must be one of FA, WI, SP, SU, IN; got 'SPR'.` | | send `SP` |
| 400 | `VALIDATION_FAILED` | `status may contain only past, current, upcoming, registration_open; got 'open'.` | | send `registration_open` |
| 401 | `CLIENT_UNAUTHENTICATED` | `An API key is required. Send X-Api-Key: <key>.` | no key, or a revoked key | use the app's key |
| 404 | `SCHOOL_NOT_FOUND` | `No institution with UNITID 999999 is held. Find the school with CourseShelf find_institution.` | | look the school up |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | over 120 calls a minute per app | wait |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference 1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d.` | | try again |

Terms are read once a day, so a school whose system is down still answers from the last daily read;
`as_of` says when that was.

## Behind the call

Adapter operation **A1 `list_terms`** ([how GoldWire talks to any SIS](00-sis-adapter.md)). Banner
`202720` → `2027-SP` through the school's term mapping. **Nothing is recorded in S3.**
