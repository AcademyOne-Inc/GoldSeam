# Call 3 — Get sections

[← Call 2 — Get campus locations](02-get-campus-locations.md) · [Summary](README.md) · Next: [Call 4 — Get section fees →](04-get-section-fees.md)

> **Draft.** Nothing here is live.

## What it does

Lists every **section** of one course at one school in one term — when and where it meets, how it is
taught, and **how many seats are left right now**. It is the flight-search screen: which flights
(sections) exist, when they leave, and how many seats remain.

It is a **public schedule search**: it takes **no learner** and knows nothing about who is asking. The
learner only appears later, when a seat is priced and held. It changes nothing.

## Where it sits

- **Before:** [Call 1](01-get-academic-terms.md) gave the term (`2027-SP`); [Call 2](02-get-campus-locations.md) gave the campus (`MAIN`). The course is known by its catalog form, `ENGL 1301`.
- **This call:** "When and where is ENGL 1301 taught in person on Main Campus this spring, and which sections have seats?"
- **After:** the learner picks section 002 → [Call 4 — Get section fees](04-get-section-fees.md) with its `section_id`.

---

## The request

```http
GET /goldwire/v1/schools/225070/sections?term_id=2027-SP&course_id=ENGL%201301&modality=in_person&campus_id=MAIN HTTP/1.1
Host: api.goldseam.example
X-Api-Key: gsk_live_4f7b2c9e1a8d
X-Request-Id: 7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47
Accept: application/json
```

## Every field you send

| Field | This example | Required? | What it means | Format and allowed values |
|---|---|---|---|---|
| `unitid` (in the path) | `225070` | yes | the school (school ID) | 6 digits |
| `X-Api-Key` (header) | `gsk_live_4f7b2c9e1a8d` | yes | identifies the **app**; no learner is sent | issued to each app |
| `term_id` | `2027-SP` | yes | the term | `YYYY-TT`, from [Call 1](01-get-academic-terms.md) |
| `course_id` | `ENGL 1301` (sent as `ENGL%201301`) | yes | the course, **subject + number** as in the catalog | `ART 101`, `ENGL 1301`, `BIOL 1406L`. `ENGL1301`, `engl 1301`, `ENGL-1301` are also accepted |
| `modality` | `in_person` | no (default: all) | how it is taught | `in_person`, `hybrid`, `online_sync`, `online_async`; several allowed, comma-separated: `in_person,hybrid` |
| `campus_id` | `MAIN` | no (default: all) | where | a code from [Call 2](02-get-campus-locations.md); several allowed: `MAIN,SOUTH` |
| `days` | *(not sent)* | no | only sections whose meetings fall **only** on these days | `MON,TUE,WED,THU,FRI,SAT,SUN`, comma-separated, e.g. `TUE,THU` |
| `time_window` | *(not sent)* | no | only sections whose every meeting starts and ends inside this window | `HH:MM-HH:MM`, 24-hour, campus local time, e.g. `17:00-22:00` |

**No `learner_id`.** Sending one is an error (`learner_id is not used by Get sections…`) so that no
learner's identity ever travels with a schedule search.

**Sections with no meeting times** (`online_async`) pass the `days` and `time_window` filters — they
never clash with anything. To leave them out, set `modality` without `online_async`.

**Every section is always returned, full ones included**, each with a `seat_status` of `open`,
`waitlist`, `closed` or `cancelled`. (Earlier drafts had an `include_full` switch to hide full
sections; it is gone — the app filters on `seat_status` if it wants to.)

Formats are defined on [GoldWire terms and formats](00-terms-and-formats.md).

---

## The reply — `200 OK`

```json
{
  "contract": "goldwire_v1",
  "request_id": "7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47",
  "status": "ok",
  "as_of": "2026-09-25T14:02:11Z",
  "source": { "kind": "school_sis_feed", "unitid": "225070", "feed": "banner", "read_at": "2026-09-25T14:02:09Z" },
  "not_held": [],
  "unitid": "225070",
  "term": { "term_id": "2027-SP", "name": "Spring 2027" },
  "course": {
    "course_id": "ENGL 1301",
    "subject": "ENGL",
    "course_number": "1301",
    "title": "Composition I",
    "credits": 3,
    "learning_unit_id": "lu_225070_ENGL1301"
  },
  "filters_applied": { "modality": ["in_person"], "campus_id": ["MAIN"], "days": null, "time_window": null },
  "sections": [
    {
      "section_id": "sec_225070_2027SP_ENGL1301_002",
      "section_number": "002",
      "session": { "session_id": "FULL", "starts_on": "2027-01-19", "ends_on": "2027-05-14" },
      "modality": "in_person",
      "campus": { "campus_id": "MAIN", "name": "Main Campus" },
      "meetings": [
        { "days": ["TUE", "THU"], "start": "09:30", "end": "10:50", "building_id": "LA", "room": "114" }
      ],
      "instructor": "Staff",
      "seats": {
        "capacity": 25,
        "enrolled": 21,
        "held": 2,
        "available": 2,
        "seat_status": "open",
        "waitlist": { "open": true, "length": 0, "capacity": 5 }
      },
      "booking": {
        "bookable": true,
        "not_bookable_reason": null,
        "registration_opens_at": null,
        "requires": ["prerequisite_check"],
        "hold_window_minutes": 20
      },
      "add_deadline": "2027-01-26",
      "drop_deadline_full_refund": "2027-02-02",
      "native": { "section_ref": "21457", "modality_code": "TRAD", "campus_code": "M" }
    },
    {
      "section_id": "sec_225070_2027SP_ENGL1301_004",
      "section_number": "004",
      "session": { "session_id": "FULL", "starts_on": "2027-01-19", "ends_on": "2027-05-14" },
      "modality": "in_person",
      "campus": { "campus_id": "MAIN", "name": "Main Campus" },
      "meetings": [
        { "days": ["TUE", "THU"], "start": "13:00", "end": "14:20", "building_id": "LA", "room": "120" }
      ],
      "instructor": "J. Whitfield",
      "seats": {
        "capacity": 25,
        "enrolled": 25,
        "held": 0,
        "available": 0,
        "seat_status": "waitlist",
        "waitlist": { "open": true, "length": 3, "capacity": 5 }
      },
      "booking": {
        "bookable": true,
        "not_bookable_reason": null,
        "registration_opens_at": null,
        "requires": ["prerequisite_check"],
        "hold_window_minutes": 20
      },
      "add_deadline": "2027-01-26",
      "drop_deadline_full_refund": "2027-02-02",
      "native": { "section_ref": "21459", "modality_code": "TRAD", "campus_code": "M" }
    },
    {
      "section_id": "sec_225070_2027SP_ENGL1301_005",
      "section_number": "005",
      "session": { "session_id": "8W2", "starts_on": "2027-03-22", "ends_on": "2027-05-14" },
      "modality": "in_person",
      "campus": { "campus_id": "MAIN", "name": "Main Campus" },
      "meetings": [
        { "days": ["MON", "WED"], "start": "18:00", "end": "20:50", "building_id": "LA", "room": "210" }
      ],
      "instructor": "R. Alvarez",
      "seats": {
        "capacity": 22,
        "enrolled": 14,
        "held": 0,
        "available": 8,
        "seat_status": "open",
        "waitlist": { "open": false, "length": 0, "capacity": 0 }
      },
      "booking": {
        "bookable": false,
        "not_bookable_reason": "registration_not_open",
        "registration_opens_at": "2026-11-02T13:00:00Z",
        "requires": ["prerequisite_check"],
        "hold_window_minutes": 20
      },
      "add_deadline": "2027-03-24",
      "drop_deadline_full_refund": "2027-03-29",
      "native": { "section_ref": "21460", "modality_code": "TRAD", "campus_code": "M" }
    }
  ]
}
```

In words — three in-person sections on Main Campus:

| Section | When | Seats | Can book now? |
|---|---|---|---|
| **002** | Tue/Thu 9:30–10:50, Liberal Arts 114, full term | **2 left** | yes |
| **004** | Tue/Thu 1:00–2:20, Liberal Arts 120 | full — waitlist has 3 of 5 | yes, onto the waitlist |
| **005** | Mon/Wed 6:00–8:50 pm, second 8 weeks | 8 left | not until 2 November |

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` | | rule version; tracking number |
| `status` | `ok` | `ok`, or `not_held` (GoldSeam has no schedule for this term — see below) |
| `as_of` | `2026-09-25T14:02:11Z` | when the seat counts were read — they can change a minute later |
| `source.kind` | `school_sis_feed` | live from the school's system. `school_published_schedule` = a published timetable with no live counts; nothing is bookable |
| `source.feed` / `read_at` | `banner` / 14:02:09 | the school's system, and when it answered |
| `not_held` | `[]` | e.g. `{ "what": "seat count for section 007" }` |
| `unitid` | `225070` | echoed |
| `term.term_id` / `term.name` | `2027-SP` / `Spring 2027` | echoed, with its name |
| `course.course_id` | `ENGL 1301` | always in the exact form, whatever spelling was sent |
| `course.subject` / `course.course_number` | `ENGL` / `1301` | the two parts |
| `course.title` / `course.credits` | `Composition I` / `3` | as the school publishes them |
| `course.learning_unit_id` | `lu_225070_ENGL1301` | the same course in CourseShelf and GoldCheck, for linking |
| `filters_applied` | modality, campus | what the filters were read as |
| `sections[].section_id` | `sec_225070_2027SP_ENGL1301_002` | **GoldWire's id — send it to Call 4.** Opaque; don't take it apart |
| `sections[].section_number` | `002` | the school's section number |
| `sections[].session` | `FULL`, 19 Jan – 14 May | the part of term it runs in, with its dates |
| `sections[].modality` | `in_person` | `in_person`, `hybrid`, `online_sync`, `online_async` |
| `sections[].campus` | `MAIN` — Main Campus | |
| `sections[].meetings[]` | `["TUE","THU"]` 09:30–10:50, `LA` 114 | each meeting pattern: days, start, end (campus local time), building, room. Empty for `online_async` |
| `sections[].instructor` | `Staff` | as published; "Staff" = not yet assigned |
| `sections[].seats.capacity` | `25` | seats the school allows |
| `sections[].seats.enrolled` | `21` | registered, per the school |
| `sections[].seats.held` | `2` | seats other learners are holding while they pay (up to 20 minutes) |
| `sections[].seats.available` | `2` | 25 − 21 − 2. Never below 0 |
| `sections[].seats.seat_status` | `open` | `open` · `waitlist` (full, can join the waitlist) · `closed` (full, no waitlist) · `cancelled` |
| `sections[].seats.waitlist` | open, 0 of 5 | the waitlist, if the school runs one |
| `sections[].booking.bookable` | `true` | can GoldWire hold and book it now |
| `sections[].booking.not_bookable_reason` | `registration_not_open` (005) | `registration_not_open`, `registration_closed`, `school_takes_no_bookings`, `section_cancelled`, `restricted_section` |
| `sections[].booking.registration_opens_at` | `2026-11-02T13:00:00Z` | when it will open |
| `sections[].booking.requires` | `["prerequisite_check"]` | what the school checks at registration |
| `sections[].booking.hold_window_minutes` | `20` | how long a hold lasts |
| `sections[].add_deadline` / `drop_deadline_full_refund` | | per the school, for this section's session |
| `sections[].native` | CRN `21457`, method `TRAD`, campus `M` | the school's own codes, for staff. Never sent back |

---

## Other replies you can get (not errors)

**The school's schedule for the term has not been captured** — `200 OK`:

```json
{
  "contract": "goldwire_v1",
  "request_id": "7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47",
  "status": "not_held",
  "as_of": "2026-09-25T14:02:11Z",
  "not_held": [ { "what": "schedule for term 2027-SP at school 225070", "note": "not yet captured" } ],
  "unitid": "225070",
  "term": { "term_id": "2027-SP", "name": "Spring 2027" },
  "course": { "course_id": "ENGL 1301", "subject": "ENGL", "course_number": "1301", "title": "Composition I", "credits": 3, "learning_unit_id": "lu_225070_ENGL1301" },
  "filters_applied": { "modality": ["in_person"], "campus_id": ["MAIN"], "days": null, "time_window": null },
  "sections": []
}
```

**The course isn't offered this term, or nothing matches the filters** — `200 OK`, `status: "ok"`,
`sections: []`. Try without `campus_id`, `days` or `time_window`.

---

## Errors

Example — the course was sent as a title:

```json
{
  "contract": "goldwire_v1",
  "request_id": "7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47",
  "status": "error",
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "course_id must be a subject and a course number, e.g. ART 101 or ENGL 1301; got 'Composition I'.",
    "retry": "fix_request",
    "field": "course_id",
    "details": { "value": "Composition I" }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `unitid must be exactly 6 digits; got '22507'.` | | fix it |
| 400 | `VALIDATION_FAILED` | `term_id is required. Get the school's terms with Get academic terms.` | left out | send it |
| 400 | `VALIDATION_FAILED` | `term_id must be YYYY-TT, e.g. 2027-SP; got '202720'.` | the school's own code sent | send `2027-SP` |
| 400 | `VALIDATION_FAILED` | `term_id must be YYYY-TT, e.g. 2027-SP; got 'Spring 2027'.` | a name sent | send `2027-SP` |
| 400 | `VALIDATION_FAILED` | `course_id is required, e.g. ART 101.` | left out | send it |
| 400 | `VALIDATION_FAILED` | `course_id must be a subject and a course number, e.g. ART 101 or ENGL 1301; got 'Composition I'.` | a title sent | send `ENGL 1301` |
| 400 | `VALIDATION_FAILED` | `course_id must be a subject and a course number, e.g. ART 101 or ENGL 1301; got 'ENGL 1301 002'.` | a section included | send `ENGL 1301`; the section comes back in the reply |
| 400 | `VALIDATION_FAILED` | `course_id must be a subject and a course number, e.g. ART 101 or ENGL 1301; got 'lu_225070_ENGL1301'.` | a CourseShelf id sent | send `ENGL 1301` |
| 400 | `VALIDATION_FAILED` | `modality may contain only in_person, hybrid, online_sync, online_async; got 'online'.` | | say `online_sync` or `online_async` |
| 400 | `VALIDATION_FAILED` | `campus_id must be 1–10 letters, digits or _; got 'Main Campus'.` | | send `MAIN` |
| 400 | `VALIDATION_FAILED` | `days may contain only MON, TUE, WED, THU, FRI, SAT, SUN, comma-separated; got 'TR'.` | single-letter codes | send `TUE,THU` |
| 400 | `VALIDATION_FAILED` | `time_window must be HH:MM-HH:MM, 24-hour, with the start before the end; got '5pm-10pm'.` | | send `17:00-22:00` |
| 400 | `VALIDATION_FAILED` | `learner_id is not used by Get sections. A schedule search carries no learner; remove it.` | a learner id sent | remove it |
| 400 | `VALIDATION_FAILED` | `Unknown parameter 'include_full'. Every section is returned with seat_status; filter on that.` | the old switch sent | remove it |
| 401 | `CLIENT_UNAUTHENTICATED` | `An API key is required. Send X-Api-Key: <key>.` | | use the app's key |
| 404 | `SCHOOL_NOT_FOUND` | `No institution with UNITID 999999 is held. Find the school with CourseShelf find_institution.` | | look it up |
| 404 | `TERM_NOT_FOUND` | `School 225070 has no term 2027-WI. Its current and upcoming terms are 2026-FA, 2027-SP, 2027-SU.` | no such term at this school | pick one listed |
| 404 | `COURSE_NOT_FOUND` | `School 225070 has no course ENGL 1399 in its catalog. Search its courses with CourseShelf learning_units.` | | check the course |
| 404 | `CAMPUS_NOT_FOUND` | `School 225070 has no campus NORTH. Its campuses are MAIN, SOUTH, WEB.` | | pick one listed |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | over 120 calls a minute per app | wait |
| 503 | `SCHOOL_OFFLINE` | `The registration system at school 225070 did not answer within 8 seconds. Seat counts could not be read. Try again shortly.` | | try again in a minute |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference 7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47.` | | try again |

## Behind the call

Adapter operation **A3 `list_sections`** ([how GoldWire talks to any SIS](00-sis-adapter.md)). The
adapter turns `2027-SP` into the school's `202720`, `ENGL 1301` into subject `ENGL` + number `1301`,
`MAIN` into campus `M`, and `in_person` into method `TRAD`; it reads the sections and their live
counts, and turns them back — Banner's `T`/`R` day flags into `["TUE","THU"]` and `0930` into `09:30`.
GoldWire then subtracts its own holds from `available`. **Nothing is recorded in S3.**
