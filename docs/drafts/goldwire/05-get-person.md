# Call 5 — Get person

[← Call 4 — Get section fees](04-get-section-fees.md) · [Summary](README.md) · Next: [Call 6 — Hold a seat →](06-hold-seat.md)

> **Draft.** Nothing here is live.

## What it does

Finds the learner's **own record at the school** — from their name and whatever else they know,
**with or without their student ID** — and links the GoldSeam learner to it. A school can only
register someone it can identify, so a booking needs this link.

- **ID known:** the learner gives their student ID; GoldWire checks it belongs to the same person as the name and date of birth.
- **ID unknown:** the learner gives their name, date of birth and one more detail; GoldWire asks the school to find the one record that matches.

What comes back is a **match result and a link** (`person_link_id`) — never the school's record
itself, never anyone else's details, and the student ID only in masked form.

## Where it sits

- **Before:** the learner has a price for section 002 ([Call 4](04-get-section-fees.md)). Any time before booking works; it only needs doing once per learner per school.
- **This call:** "Find me in this school's records."
- **After:** [Call 6 — Hold a seat](06-hold-seat.md), then [Call 9 — Book the seat](09-book-seat.md) with the `person_link_id`.

---

## The request

**Why `POST` and not `GET`:** this is a lookup, but it carries a name and date of birth. Anything in a
URL is written into server and proxy logs, so personal details travel only in the request body. It
changes nothing at the school.

```http
POST /goldwire/v1/schools/225070/persons/lookup HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fOGYzYTJjOTFkNyJ9.sig
X-Request-Id: 5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f
Content-Type: application/json

{
  "learner_id": "lrn_8f3a2c91d7",
  "school_person_id": null,
  "name": { "first": "Maria", "middle": "Elena", "last": "Lopez", "suffix": null },
  "former_last_names": ["Garza"],
  "date_of_birth": "2001-04-17",
  "email": "maria.lopez@example.com",
  "phone": "+19035550142",
  "postal_code": "75090"
}
```

## Every field you send

| Field | This example | Required? | What it means | Format and rules |
|---|---|---|---|---|
| `unitid` (in the path) | `225070` | yes | the school to look in | 6 digits |
| `Authorization` (header) | `Bearer eyJ…` | yes | the **learner's** sign-in token — a learner can only look up themselves | must belong to `lrn_8f3a2c91d7` |
| `learner_id` | `lrn_8f3a2c91d7` | yes | who is being linked | `lrn_` + 8–40 letters/digits |
| `school_person_id` | `null` — **unknown** | no | the learner's student ID at this school, if they know it | the school's own format, 1–20 characters: `A00482913` (Banner), `1048291` (PeopleSoft), `0482913` (Colleague) |
| `name.first` | `Maria` | yes | legal first name | 1–60 characters |
| `name.middle` | `Elena` | no | middle name or initial | |
| `name.last` | `Lopez` | yes | legal last name | 1–60 characters |
| `name.suffix` | `null` | no | | `Jr`, `Sr`, `II`, `III`, `IV` |
| `former_last_names` | `["Garza"]` | no | earlier last names — the school may hold one of these | list, up to 5 |
| `date_of_birth` | `2001-04-17` | yes | | `YYYY-MM-DD` |
| `email` | `maria.lopez@example.com` | one of these three **if no `school_person_id`** | an email the school may have | |
| `phone` | `+19035550142` | ″ | a phone number the school may have | E.164: `+` and digits |
| `postal_code` | `75090` | ″ | home postal code | US: `75090` or `75090-1234` |

**Minimum to look someone up**

| The learner knows their ID | They must also send |
|---|---|
| yes | `name.last` + `date_of_birth` |
| no | `name.first` + `name.last` + `date_of_birth` + at least one of `email`, `phone`, `postal_code` |

**Never accepted:** a Social Security number, in any field. Sending one is an error and the value is
discarded unread.

## What GoldWire does, in order

1. Checks the token (the learner can only look up themselves) and the fields.
2. Checks the learner has not used up today's attempts at this school (5 per 24 hours — stops anyone guessing IDs).
3. Asks the school's system, through adapter operation **A5 `find_person`**, using the school's own matching rules.
4. Decides:
   - **ID sent:** the record with that ID must also match the last name and date of birth → `matched`; otherwise → `id_mismatch`.
   - **No ID:** exactly one record matches → `matched`; more than one → `multiple_possible`; none → `no_match`.
5. On `matched`: records the link (encrypted) and returns a `person_link_id`, and says whether the school sees anything that would stop registration.

---

## The reply — `200 OK`

The learner didn't know their ID; the school found exactly one record:

```json
{
  "contract": "goldwire_v1",
  "request_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "status": "ok",
  "as_of": "2026-09-25T14:05:20Z",
  "source": { "kind": "school_sis_feed", "unitid": "225070", "feed": "banner", "read_at": "2026-09-25T14:05:19Z" },
  "not_held": [],
  "unitid": "225070",
  "learner_id": "lrn_8f3a2c91d7",
  "match_status": "matched",
  "person_link_id": "psl_3N8QK2WD7F",
  "school_person_id_masked": "A*****913",
  "id_was": "found",
  "matched_on": ["first_name", "last_name", "date_of_birth", "email"],
  "person_type": "former_student",
  "registration_readiness": {
    "status": "ready",
    "message": "No holds on the account. The learner can be registered."
  },
  "attempts_left_today": 4,
  "next": { "service": "hold_seat" }
}
```

In words: **found** — a former student at this school, ID ending in 913, nothing blocking registration.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` / `as_of` | | rule version; tracking number; when the school answered |
| `status` | `ok` | `ok`, or `not_held` if the school isn't connected for lookups (below) |
| `unitid` / `learner_id` | | echoed |
| `match_status` | `matched` | `matched` · `multiple_possible` · `no_match` · `id_mismatch` (below) |
| `person_link_id` | `psl_3N8QK2WD7F` | **the link — send it to Call 9 (Book the seat).** Only when `matched` |
| `school_person_id_masked` | `A*****913` | the learner's ID at the school, masked. The full ID never leaves GoldWire except to the school itself |
| `id_was` | `found` | `found` (the learner didn't know it) · `supplied_and_verified` (they sent it and it checked out) |
| `matched_on` | first name, last name, date of birth, email | which details agreed |
| `person_type` | `former_student` | what the school has: `current_student`, `former_student`, `applicant`, `person` (known, never enrolled) |
| `registration_readiness.status` | `ready` | `ready` · `holds_on_account` (the school will refuse until cleared) · `admission_required` (must apply first) |
| `registration_readiness.message` | "No holds on the account…" | plain words for the learner. **Never names the hold** — the school tells the learner that itself |
| `attempts_left_today` | `4` | lookups left at this school in the next 24 hours |
| `next.service` | `hold_seat` | what to call next |

---

## Other replies you can get (not errors)

All `200 OK`.

**The learner sent their ID and it checks out**

```json
{
  "contract": "goldwire_v1",
  "request_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "status": "ok",
  "as_of": "2026-09-25T14:05:20Z",
  "not_held": [],
  "unitid": "225070",
  "learner_id": "lrn_8f3a2c91d7",
  "match_status": "matched",
  "person_link_id": "psl_3N8QK2WD7F",
  "school_person_id_masked": "A*****913",
  "id_was": "supplied_and_verified",
  "matched_on": ["school_person_id", "last_name", "date_of_birth"],
  "person_type": "current_student",
  "registration_readiness": { "status": "ready", "message": "No holds on the account. The learner can be registered." },
  "attempts_left_today": 4,
  "next": { "service": "hold_seat" }
}
```

**More than one record could be the learner** — nothing about the candidates is returned; only what
would tell them apart:

```json
{
  "contract": "goldwire_v1",
  "request_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "status": "ok",
  "as_of": "2026-09-25T14:05:20Z",
  "not_held": [],
  "unitid": "225070",
  "learner_id": "lrn_8f3a2c91d7",
  "match_status": "multiple_possible",
  "person_link_id": null,
  "message": "More than one record at this school matches. Add your student ID, or another email, phone or postal code the school may have.",
  "more_details_would_help": ["school_person_id", "email", "phone", "postal_code"],
  "attempts_left_today": 3,
  "next": { "service": "get_person" }
}
```

**No record at the school** — a new learner here:

```json
{
  "contract": "goldwire_v1",
  "request_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "status": "ok",
  "as_of": "2026-09-25T14:05:20Z",
  "not_held": [],
  "unitid": "225070",
  "learner_id": "lrn_8f3a2c91d7",
  "match_status": "no_match",
  "person_link_id": null,
  "message": "No record at this school matches. You will need to apply to the school before registering.",
  "new_person_allowed": false,
  "apply_url": "https://www.example.edu/apply",
  "attempts_left_today": 3,
  "next": { "service": null }
}
```

`new_person_allowed: true` would mean the school lets GoldWire create a new non-degree student record
at booking; this school does not.

**The ID and the details don't belong to the same record** — the reply never says which part is
wrong, so no one can use it to test IDs:

```json
{
  "contract": "goldwire_v1",
  "request_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "status": "ok",
  "as_of": "2026-09-25T14:05:20Z",
  "not_held": [],
  "unitid": "225070",
  "learner_id": "lrn_8f3a2c91d7",
  "match_status": "id_mismatch",
  "person_link_id": null,
  "message": "The student ID and the details given do not match the same record. Check them, or leave the ID out and try again.",
  "attempts_left_today": 2,
  "next": { "service": "get_person" }
}
```

**Found, but the school has a hold on the account** — `match_status: "matched"`, a `person_link_id`, and
`registration_readiness: { "status": "holds_on_account", "message": "The school has a hold on this account that will stop registration. Contact the school's Registrar or Business Office." }`.
Booking will come back `REJECTED` until the school clears it.

**The school isn't connected for lookups** — `status: "not_held"`, `match_status: null`,
`not_held: [ { "what": "person lookup at school 225070" } ]`. Seats at such a school are not bookable
through GoldWire.

---

## Errors

Example — an SSN was sent:

```json
{
  "contract": "goldwire_v1",
  "request_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "status": "error",
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Social Security numbers are not accepted by GoldWire. The value was discarded. Use your student ID, or your name, date of birth and an email, phone or postal code.",
    "retry": "fix_request",
    "field": "ssn",
    "details": {}
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `name.last is required.` | left out (same for `name.first` when no ID, `date_of_birth`, `learner_id`) | send it |
| 400 | `VALIDATION_FAILED` | `date_of_birth must be YYYY-MM-DD; got '04/17/2001'.` | | send `2001-04-17` |
| 400 | `VALIDATION_FAILED` | `date_of_birth 2031-04-17 is in the future.` | | fix it |
| 400 | `VALIDATION_FAILED` | `Without school_person_id, send at least one of email, phone or postal_code as well as first name, last name and date of birth.` | too little to match on | add one |
| 400 | `VALIDATION_FAILED` | `phone must be + and 8–15 digits, e.g. +19035550142; got '(903) 555-0142'.` | | send E.164 |
| 400 | `VALIDATION_FAILED` | `postal_code must be 5 digits or 5+4 digits; got '7509'.` | | fix it |
| 400 | `VALIDATION_FAILED` | `school_person_id may be at most 20 characters.` | | check it |
| 400 | `VALIDATION_FAILED` | `school_person_id A0048291 is not in this school's format (a letter A followed by 8 digits).` | a digit missing | check the ID |
| 400 | `VALIDATION_FAILED` | `Social Security numbers are not accepted by GoldWire. The value was discarded. Use your student ID, or your name, date of birth and an email, phone or postal code.` | an `ssn` field, or a 9-digit SSN-shaped value in `school_person_id` | remove it |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_8f3a2c91d7. A learner can only look up themselves.` | | send the matching pair |
| 404 | `SCHOOL_NOT_FOUND` | `No institution with UNITID 999999 is held. Find the school with CourseShelf find_institution.` | | look it up |
| 429 | `LOOKUP_LIMIT_REACHED` | `Learner lrn_8f3a2c91d7 has used all 5 lookups at school 225070 for today. Try again after 2026-09-26T14:05:20Z, or contact the school's Registrar.` | 5 tries in 24 hours | wait, or contact the school |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | | wait |
| 503 | `SCHOOL_OFFLINE` | `The records system at school 225070 did not answer within 8 seconds. No lookup was counted. Try again shortly.` | | try again |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the lookup. No lookup was counted. Reference 5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f.` | | try again |

## What gets recorded in S3

| File | Holds | Never holds |
|---|---|---|
| `goldwire/persons/225070/lrn_8f3a2c91d7.json` (encrypted; written only on `matched`) | `person_link_id`, the full school person ID, `id_was`, `matched_on`, when | name, date of birth, email, phone, postal code |
| `goldwire/persons/225070/attempts/lrn_8f3a2c91d7/2026-09-25.json` | the count of lookups and each result (`matched`, `no_match` …) | any detail that was sent |

The name, date of birth and contact details are passed to the school to match on and then
**discarded**; GoldWire does not keep them. A new lookup that matches a different record replaces the
link, and the old one is kept as an earlier version.
