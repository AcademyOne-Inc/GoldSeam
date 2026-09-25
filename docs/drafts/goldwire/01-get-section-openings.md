# Call 1 — Get section openings

[← Summary](README.md) · Next: [Call 2 — Get section fees →](02-get-section-fees.md)

> **Draft.** Nothing here is live.

## What it does

Shows every section of one course at one school for one term, and how many seats each section has
**right now**. It is the flight-search screen: which flights (sections) exist, when they leave
(meeting times), and how many seats are left.

It changes nothing and records nothing.

## Where it sits

- **Before:** GoldCheck has said what the course counts toward (`goldcheck_ref: gck_5TR20P`). CourseShelf gave the school (`225070`) and the course (`lu_225070_ENGL1301`).
- **This call:** "Which sections of ENGL 1301 are open for Spring 2027?"
- **After:** the learner picks section 002 → [Call 2 — Get section fees](02-get-section-fees.md).

---

## The request

```http
GET /goldwire/v1/sections?learner_id=lrn_8f3a2c91d7&unitid=225070&learning_unit_id=lu_225070_ENGL1301&term=2027SP&modality=in_person&goldcheck_ref=gck_5TR20P HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fOGYzYTJjOTFkNyJ9.sig
X-Request-Id: 7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47
Accept: application/json
```

## Every field you send

| Field | This example | Required? | What it means | Allowed |
|---|---|---|---|---|
| `Authorization` (header) | `Bearer eyJ…` | yes | the learner's sign-in token | must belong to `lrn_8f3a2c91d7` |
| `X-Request-Id` (header) | `7c1e0b52-…-6f0c2d8e1a47` | no | a tracking number for this request; GoldWire makes one if you don't | a UUID |
| `learner_id` | `lrn_8f3a2c91d7` | yes | who is asking — a private GoldSeam id, never a name or SSN | `lrn_` + 8–40 letters/digits |
| `unitid` | `225070` | yes | the school, by its federal IPEDS number | exactly 6 digits |
| `learning_unit_id` | `lu_225070_ENGL1301` | yes | the course (ENGL 1301 Composition I at that school) | an id from CourseShelf |
| `term` | `2027SP` | yes | the school's own code for Spring 2027 | whatever code the school uses |
| `modality` | `in_person` | no | only show in-person sections | `in_person`, `online`, `hybrid`, `any` (default) |
| `campus` | *(not sent)* | no | only show one campus | the school's campus name, e.g. `Main` |
| `days` | *(not sent)* | no | only sections that meet on these days | letters M T W R F S U — R is Thursday, U is Sunday. `TR` = Tue/Thu |
| `time_window` | *(not sent)* | no | only sections whose classes fall inside these hours | `17:00-22:00` |
| `include_full` | *(not sent — so `false`)* | no | also show sections with no seats left, with their waitlist | `true` / `false` |
| `goldcheck_ref` | `gck_5TR20P` | no | the GoldCheck answer; if sent, each section says what it counts toward | `gck_` + letters/digits |

---

## The reply — `200 OK`

```json
{
  "contract": "goldwire_v1",
  "request_id": "7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47",
  "status": "ok",
  "as_of": "2026-09-25T14:02:11Z",
  "source": {
    "kind": "school_sis_feed",
    "unitid": "225070",
    "feed": "banner-ssb",
    "read_at": "2026-09-25T14:02:09Z"
  },
  "not_held": [],
  "learning_unit": {
    "learning_unit_id": "lu_225070_ENGL1301",
    "code": "ENGL 1301",
    "title": "Composition I",
    "credits": 3
  },
  "term": "2027SP",
  "sections": [
    {
      "section_id": "sec_225070_2027SP_ENGL1301_002",
      "section_number": "002",
      "crn": "21457",
      "modality": "in_person",
      "campus": "Main",
      "meetings": [
        { "days": "TR", "start": "09:30", "end": "10:50", "room": "LA 114" }
      ],
      "instructor": "Staff",
      "starts_on": "2027-01-19",
      "ends_on": "2027-05-14",
      "seats": {
        "capacity": 25,
        "enrolled": 21,
        "held": 2,
        "available": 2,
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
      "goldcheck_note": "Counts toward: English Composition requirement, AA General Studies (checklist chk_225070_AA_GS)"
    },
    {
      "section_id": "sec_225070_2027SP_ENGL1301_005",
      "section_number": "005",
      "crn": "21460",
      "modality": "in_person",
      "campus": "South",
      "meetings": [
        { "days": "MW", "start": "18:00", "end": "19:20", "room": "S 210" }
      ],
      "instructor": "R. Alvarez",
      "starts_on": "2027-01-19",
      "ends_on": "2027-05-14",
      "seats": {
        "capacity": 22,
        "enrolled": 14,
        "held": 0,
        "available": 8,
        "waitlist": { "open": false, "length": 0, "capacity": 0 }
      },
      "booking": {
        "bookable": false,
        "not_bookable_reason": "registration_not_open",
        "registration_opens_at": "2026-11-02T13:00:00Z",
        "requires": ["prerequisite_check"],
        "hold_window_minutes": 20
      },
      "add_deadline": "2027-01-26",
      "drop_deadline_full_refund": "2027-02-02",
      "goldcheck_note": "Counts toward: English Composition requirement, AA General Studies (checklist chk_225070_AA_GS)"
    }
  ]
}
```

In words: two in-person sections. **002** (Tue/Thu 9:30, Main campus) has **2 seats left** and can be
booked now. **005** (Mon/Wed evening, South campus) has 8 seats but registration for it does not open
until 2 November.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` | `goldwire_v1` | which version of the rules produced this answer |
| `request_id` | `7c1e0b52-…` | the tracking number, echoed back |
| `status` | `ok` | `ok` = full answer · `not_held` = GoldSeam has not captured this school's schedule for this term (see below) |
| `as_of` | `2026-09-25T14:02:11Z` | when the seat counts were read — they can change a minute later |
| `source.kind` | `school_sis_feed` | the counts came live from the school's registration system. `school_published_schedule` means the school only publishes a timetable, with no live counts — then nothing can be booked |
| `source.unitid` | `225070` | the school |
| `source.feed` | `banner-ssb` | which registration system the school runs |
| `source.read_at` | `2026-09-25T14:02:09Z` | when the school's system answered |
| `not_held` | `[]` | anything the school did not publish, e.g. a seat count for one section |
| `learning_unit.code` / `.title` / `.credits` | `ENGL 1301` / `Composition I` / `3` | the course, as the school publishes it |
| `term` | `2027SP` | echoed |
| `sections[].section_id` | `sec_225070_2027SP_ENGL1301_002` | GoldWire's id for this section — **send this to Call 2** |
| `sections[].section_number` | `002` | the school's section number |
| `sections[].crn` | `21457` | the school's course reference number |
| `sections[].modality` | `in_person` | in person, online or hybrid |
| `sections[].campus` | `Main` | where |
| `sections[].meetings[]` | Tue/Thu 09:30–10:50, room LA 114 | when and where it meets (school's local time) |
| `sections[].instructor` | `Staff` | as the school publishes it; "Staff" means not yet assigned |
| `sections[].starts_on` / `ends_on` | 19 Jan – 14 May 2027 | first and last class day |
| `sections[].seats.capacity` | `25` | seats the school allows |
| `sections[].seats.enrolled` | `21` | already registered, per the school |
| `sections[].seats.held` | `2` | seats other GoldWire learners are holding right now (for up to 20 minutes) |
| `sections[].seats.available` | `2` | 25 − 21 − 2 = **2 seats you can hold**. Never below 0 |
| `sections[].seats.waitlist` | open, 0 waiting, room for 5 | whether you can join a waitlist if it fills |
| `sections[].booking.bookable` | `true` | can GoldWire hold and book this section now |
| `sections[].booking.not_bookable_reason` | `null` (002) · `registration_not_open` (005) | why not, when `bookable` is false. Other reasons: `registration_closed`, `school_takes_no_bookings`, `section_cancelled`, `restricted_section` |
| `sections[].booking.registration_opens_at` | `2026-11-02T13:00:00Z` (005) | when registration opens, if it hasn't |
| `sections[].booking.requires` | `["prerequisite_check"]` | the school will check prerequisites; the learner will be asked to confirm they meet them at booking |
| `sections[].booking.hold_window_minutes` | `20` | how long a hold lasts once taken |
| `sections[].add_deadline` | `2027-01-26` | last day to add, per the school |
| `sections[].drop_deadline_full_refund` | `2027-02-02` | last day to drop with a full refund, per the school |
| `sections[].goldcheck_note` | "Counts toward: English Composition requirement…" | only because `goldcheck_ref` was sent |

---

## Other replies you can get (not errors)

**The school's Spring 2027 schedule has not been captured by GoldSeam** — `200 OK`:

```json
{
  "contract": "goldwire_v1",
  "request_id": "7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47",
  "status": "not_held",
  "as_of": "2026-09-25T14:02:11Z",
  "not_held": [ { "what": "schedule for term 2027SP at school 225070", "note": "not yet captured" } ],
  "learning_unit": { "learning_unit_id": "lu_225070_ENGL1301", "code": "ENGL 1301", "title": "Composition I", "credits": 3 },
  "term": "2027SP",
  "sections": []
}
```

This is an honest gap, not a failure. Tell the learner GoldSeam does not have that schedule yet.

**Nothing matches the filters** — `200 OK`, `status: "ok"`, `sections: []`. Try without `modality`,
`days` or `time_window`, or with `include_full=true`.

---

## Errors

Every error comes back in this shape. Example — the course belongs to a different school:

```json
{
  "contract": "goldwire_v1",
  "request_id": "7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47",
  "status": "error",
  "error": {
    "code": "COURSE_NOT_AT_SCHOOL",
    "message": "Course lu_228529_ENGL1301 belongs to school 228529, not 225070.",
    "retry": "fix_request",
    "field": "learning_unit_id",
    "details": { "learning_unit_id": "lu_228529_ENGL1301", "actual_unitid": "228529", "unitid": "225070" }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `learner_id is required.` | `learner_id` left out | send it |
| 400 | `VALIDATION_FAILED` | `learner_id must look like lrn_ followed by 8–40 letters or digits; got 'maria.lopez@example.com'.` | an email or name sent instead of the id | send the `lrn_` id |
| 400 | `VALIDATION_FAILED` | `unitid is required.` | left out | send it |
| 400 | `VALIDATION_FAILED` | `unitid must be exactly 6 digits; got '22507'.` | a digit missing | fix it |
| 400 | `VALIDATION_FAILED` | `learning_unit_id is required.` | left out | send it |
| 400 | `VALIDATION_FAILED` | `learning_unit_id must look like lu_…; got 'ENGL 1301'. Use an id from CourseShelf learning_units.` | the course code sent instead of the id | look the id up in CourseShelf |
| 400 | `VALIDATION_FAILED` | `term is required. Use the school's own term code, e.g. 2027SP.` | left out | send it |
| 400 | `VALIDATION_FAILED` | `modality must be one of in_person, online, hybrid, any; got 'remote'.` | a word the service doesn't use | use `online` |
| 400 | `VALIDATION_FAILED` | `days may contain only the letters M T W R F S U; got 'TTh'.` | "Th" for Thursday | use `TR` |
| 400 | `VALIDATION_FAILED` | `time_window must be HH:MM-HH:MM with the start before the end; got '5pm-10pm'.` | wrong format | use `17:00-22:00` |
| 400 | `VALIDATION_FAILED` | `goldcheck_ref must look like gck_…; got '5TR20P'.` | prefix missing | send `gck_5TR20P` |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | no token, or it expired | sign the learner in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_8f3a2c91d7.` | token for one learner, `learner_id` for another | send the matching pair |
| 404 | `SCHOOL_NOT_FOUND` | `No institution with UNITID 999999 is held. Find the school with CourseShelf find_institution.` | not a real school number | look the school up |
| 404 | `COURSE_NOT_FOUND` | `No course lu_225070_ENGL1399 is held. Find the course with CourseShelf learning_units.` | course id doesn't exist | look the course up |
| 404 | `GOLDCHECK_REF_NOT_FOUND` | `No GoldCheck answer gck_9ZZ00Q is held for learner lrn_8f3a2c91d7.` | the ref is wrong or someone else's | drop it or fix it |
| 422 | `COURSE_NOT_AT_SCHOOL` | `Course lu_228529_ENGL1301 belongs to school 228529, not 225070.` | course and school don't match | fix one of them |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | more than 60 calls a minute | wait 12 seconds |
| 503 | `SCHOOL_OFFLINE` | `The registration system at school 225070 did not answer within 8 seconds. Seat counts could not be read. Try again shortly.` | the school's system is down or slow | try again in a minute |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference 7c1e0b52-3d4f-4a8e-9b21-6f0c2d8e1a47.` | a fault inside GoldWire | try again; quote the reference if it repeats |

## What gets recorded in S3

Nothing. This call only reads.
