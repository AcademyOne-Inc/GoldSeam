# Call 6 — Enroll as a non-degree student

[← Call 5 — Get person](05-get-person.md) · [Summary](README.md) · Next: [Call 7 — Hold a seat →](07-hold-seat.md)

> **Draft.** Nothing here is live.

## What it does

Creates a **non-degree student record** at the school for a learner the school has never had, so they
can register for courses **without applying for admission**.

Students come in two kinds, and they get in differently:

| | **Degree-seeking** | **Non-degree** |
|---|---|---|
| Who | working toward a degree or certificate, full-time or part-time | taking courses without a program: personal interest, job skills, a visiting student taking a course to transfer home |
| How they get a student record | **apply for admission**; the school admits them into a program. GoldWire does not submit applications | **this call** — a short enrollment form, no application |
| Limits | the program's requirements | the school's non-degree rules, often a credit cap (e.g. 12 credit hours) before they must apply |
| Full-time / part-time | by credits each term (e.g. 12+ = full-time) | the same, by credits each term |

Full-time and part-time are not a kind of student — they are the **load in a term**, counted from
registered credits against the school's threshold. [Get person](05-get-person.md) and
[Book the seat](10-book-seat.md) report it.

## Where it sits

- **Before:** [Get person](05-get-person.md) found **no record** (`match_status: no_match`) and returned `lookup_ref: plk_7R2M9X4QTA` with a `non_degree` path available.
- **This call:** "Enroll me as a non-degree student."
- **After:** the learner has a `person_link_id` → [Call 7 — Hold a seat](07-hold-seat.md), then [Call 10 — Book the seat](10-book-seat.md).

This page follows a **different learner** from the rest of the pages: someone new to the school who
wants to take ART 101 for personal interest.

---

## The request

```http
PUT /goldwire/v1/schools/225070/non-degree-students/01J8Z5B2C4D6E8F0G2H4J6K8M0 HTTP/1.1
Host: api.goldseam.example
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9.eyJzdWIiOiJscm5fMmI3ZDRlOWExYyJ9.sig
X-Request-Id: 6d7e8f9a-0b1c-4d2e-9f3a-4b5c6d7e8f9a
Content-Type: application/json

{
  "learner_id": "lrn_2b7d4e9a1c",
  "lookup_ref": "plk_9W4T6Y8U0I",
  "name": { "first": "James", "middle": null, "last": "Okafor", "suffix": null },
  "preferred_first_name": "Jim",
  "date_of_birth": "1968-11-03",
  "email": "jim.okafor@example.com",
  "phone": "+19035550198",
  "address": {
    "line1": "412 Pecan Street",
    "line2": null,
    "city": "Example City",
    "state": "TX",
    "postal_code": "75090",
    "country": "US"
  },
  "first_term_id": "2027-SP",
  "non_degree_reason": "personal_enrichment",
  "high_school": "diploma",
  "residency_declared": "in_district",
  "citizenship": "us_citizen",
  "attestations": {
    "information_true": true,
    "policies_acknowledged": ["pol_225070_nondegree_2026_27"]
  }
}
```

## Every field you send

| Field | This example | Required? | What it means | Format and allowed values |
|---|---|---|---|---|
| `unitid` (in the path) | `225070` | yes | the school | 6 digits |
| idempotency key (in the path) | `01J8Z5B2C4D6E8F0G2H4J6K8M0` | yes | makes retries safe — **a retry never creates a second record** | 16–64 letters, digits, `_` or `-` |
| `Authorization` (header) | `Bearer eyJ…` | yes | the learner's sign-in token | must belong to `lrn_2b7d4e9a1c` |
| `learner_id` | `lrn_2b7d4e9a1c` | yes | who is enrolling | `lrn_` + 8–40 letters/digits |
| `lookup_ref` | `plk_9W4T6Y8U0I` | yes | proof that [Get person](05-get-person.md) found no record — so no one gets two records | `plk_` + 10 letters/digits; from a `no_match` lookup by this learner at this school in the last 24 hours |
| `name.first` / `name.last` | `James` / `Okafor` | yes | legal name | 1–60 characters each |
| `name.middle` / `name.suffix` | `null` | no | | suffix: `Jr`, `Sr`, `II`, `III`, `IV` |
| `preferred_first_name` | `Jim` | no | the name the learner goes by | 1–60 characters |
| `date_of_birth` | `1968-11-03` | yes | | `YYYY-MM-DD` |
| `email` | `jim.okafor@example.com` | yes | the school will write here | |
| `phone` | `+19035550198` | yes | | E.164: `+` and 8–15 digits |
| `address.line1` / `line2` / `city` / `state` / `postal_code` / `country` | 412 Pecan Street… | yes (`line2` no) | home address — schools need it for residency and billing | `state`: 2 letters · `postal_code`: US 5 or 5+4 digits · `country`: 2 letters |
| `first_term_id` | `2027-SP` | yes | the first term they will take classes | `YYYY-TT`, a term open for registration |
| `non_degree_reason` | `personal_enrichment` | yes | why they are taking courses | `personal_enrichment` · `job_skills` · `visiting_student` (taking a course to transfer to another school) · `prerequisite_prep` (preparing for a program elsewhere) |
| `high_school` | `diploma` | yes | high-school completion | `diploma` · `ged_or_equivalent` · `homeschool_completed` · `still_in_high_school` (dual credit — **not** through GoldWire, see errors) |
| `residency_declared` | `in_district` | yes | which tuition rate the learner believes applies; the school may review it | `in_district` · `in_state` · `out_of_state` · `international` |
| `citizenship` | `us_citizen` | yes | some schools must review non-citizens before enrolling them | `us_citizen` · `permanent_resident` · `other` |
| `attestations.information_true` | `true` | yes | the learner confirms the details are true | must be `true` |
| `attestations.policies_acknowledged` | `["pol_225070_nondegree_2026_27"]` | yes | the school's non-degree policy, shown to the learner | must include it |

**Never accepted:** a Social Security number, in any field.

## What GoldWire does, in order

1. Checks the key, token and fields.
2. Checks `lookup_ref`: this learner's, this school's, `no_match`, under 24 hours old.
3. Checks the school lets GoldWire enroll non-degree students, and the learner has no person link at this school already.
4. Sends the form to the school through adapter operation **A9 `create_non_degree_student`**. The school runs its **own duplicate check** on the fuller details first.
5. Records the result and, if a record exists now, the person link.

---

## The reply — `201 Created`

```json
{
  "contract": "goldwire_v1",
  "request_id": "6d7e8f9a-0b1c-4d2e-9f3a-4b5c6d7e8f9a",
  "status": "ok",
  "as_of": "2026-09-25T15:12:44Z",
  "source": { "kind": "school_sis_feed", "unitid": "225070", "feed": "banner", "read_at": "2026-09-25T15:12:43Z" },
  "not_held": [],
  "unitid": "225070",
  "learner_id": "lrn_2b7d4e9a1c",
  "enrollment_status": "created",
  "person_link_id": "psl_8H2J4K6L8M",
  "school_person_id_masked": "A*****377",
  "student_record": {
    "student_type": "non_degree",
    "admission_status": "not_required",
    "program": null,
    "first_term_id": "2027-SP"
  },
  "limits": {
    "credit_limit": 12,
    "credits_used": 0,
    "note": "Non-degree students may take up to 12 credit hours. To take more, apply for admission.",
    "source": "pol_225070_nondegree_2026_27"
  },
  "registration_readiness": {
    "status": "ready",
    "message": "Enrolled as a non-degree student. You can register for Spring 2027."
  },
  "next": { "service": "hold_seat" }
}
```

In words: **enrolled as a non-degree student**, school ID ending 377, can take up to 12 credit hours
before applying, ready to register for Spring 2027.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` / `as_of` | | rule version; tracking number; when the school answered |
| `status` | `ok` | `ok`, or `offline` if the school did not answer (below) |
| `source` | `school_sis_feed`, `banner` | |
| `unitid` / `learner_id` | | echoed |
| `enrollment_status` | `created` | `created` · `pending_school_review` (a person at the school must check it; answer within the school's stated time) · `refused` (with the school's reason) · `already_known` (the school's own check found an existing record — that record is linked instead) |
| `person_link_id` | `psl_8H2J4K6L8M` | **the link — send it to Call 10 (Book the seat).** `null` when `pending_school_review` or `refused` |
| `school_person_id_masked` | `A*****377` | the new ID at the school, masked |
| `student_record.student_type` | `non_degree` | |
| `student_record.admission_status` | `not_required` | non-degree students are not admitted into a program |
| `student_record.program` | `null` | no program |
| `student_record.first_term_id` | `2027-SP` | |
| `limits.credit_limit` | `12` | the school's cap on non-degree credit hours; `null` if it has none |
| `limits.credits_used` | `0` | credits taken so far as a non-degree student |
| `limits.note` / `limits.source` | | in plain words, and the policy it comes from |
| `registration_readiness.status` | `ready` | `ready` · `pending_school_review` · `holds_on_account` |
| `registration_readiness.message` | | plain words for the learner |
| `next.service` | `hold_seat` | what to call next |

**Same request sent twice:** `200 OK`, header `Idempotent-Replayed: true`, the same reply. No second record.

---

## Other replies you can get (not errors)

**The school must review it first** (e.g. `citizenship: "other"`) — `201`:

```json
{
  "contract": "goldwire_v1",
  "request_id": "6d7e8f9a-0b1c-4d2e-9f3a-4b5c6d7e8f9a",
  "status": "ok",
  "as_of": "2026-09-25T15:12:44Z",
  "not_held": [],
  "unitid": "225070",
  "learner_id": "lrn_2b7d4e9a1c",
  "enrollment_status": "pending_school_review",
  "person_link_id": null,
  "school_person_id_masked": null,
  "review": { "expected_by": "2026-09-30", "contact": "Office of Admissions and Records" },
  "registration_readiness": {
    "status": "pending_school_review",
    "message": "The school is reviewing your enrollment. You'll be notified by 2026-09-30."
  },
  "next": { "service": null }
}
```

When the school decides, the learner is notified and Get person will find the new record.

**The school already had them** — its own check matched on the fuller details: `201`,
`enrollment_status: "already_known"`, a `person_link_id` for the existing record, and
`student_record` as the school holds it. No new record is created.

**The school refused** — `201`, `enrollment_status: "refused"`, `person_link_id: null`,
`school_reason: "Non-degree enrollment for Spring 2027 closed on 2027-01-12."` in the school's own words.

**The school didn't answer in 8 seconds** — `201`, `status: "offline"`,
`enrollment_status: "pending_school_review"`, and a `note` that GoldWire will resend every 5 minutes
for 24 hours and notify the learner.

---

## Errors

Example — the learner already has a record:

```json
{
  "contract": "goldwire_v1",
  "request_id": "6d7e8f9a-0b1c-4d2e-9f3a-4b5c6d7e8f9a",
  "status": "error",
  "error": {
    "code": "LOOKUP_FOUND_RECORD",
    "message": "Lookup plk_9W4T6Y8U0I found a record at school 225070, so no new record can be created. Use person link psl_3N8QK2WD7F.",
    "retry": "no",
    "field": "lookup_ref",
    "details": { "lookup_ref": "plk_9W4T6Y8U0I", "person_link_id": "psl_3N8QK2WD7F" }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `lookup_ref is required. Look the learner up with Get person first.` | left out | Call 5 |
| 400 | `VALIDATION_FAILED` | `address.postal_code is required.` | left out (same message for every required field) | send it |
| 400 | `VALIDATION_FAILED` | `non_degree_reason must be one of personal_enrichment, job_skills, visiting_student, prerequisite_prep; got 'fun'.` | | pick one |
| 400 | `VALIDATION_FAILED` | `high_school must be one of diploma, ged_or_equivalent, homeschool_completed, still_in_high_school; got 'yes'.` | | pick one |
| 400 | `VALIDATION_FAILED` | `attestations.information_true must be true.` | | the learner must confirm |
| 400 | `VALIDATION_FAILED` | `attestations.policies_acknowledged must include pol_225070_nondegree_2026_27.` | | show the policy, then send it |
| 400 | `VALIDATION_FAILED` | `Social Security numbers are not accepted by GoldWire. The value was discarded.` | an SSN sent | remove it |
| 401 | `UNAUTHENTICATED` | `A learner token is required. Send Authorization: Bearer <token>.` | | sign in again |
| 403 | `LEARNER_MISMATCH` | `This token belongs to a different learner than lrn_2b7d4e9a1c.` | | send the matching pair |
| 404 | `LOOKUP_REF_NOT_FOUND` | `No lookup plk_9W4T6Y8U0X is held for learner lrn_2b7d4e9a1c at school 225070.` | wrong ref, another learner's, another school's | Call 5 again |
| 404 | `TERM_NOT_FOUND` | `School 225070 has no term 2027-WI. Its current and upcoming terms are 2026-FA, 2027-SP, 2027-SU.` | | pick one listed |
| 409 | `LOOKUP_FOUND_RECORD` | `Lookup plk_9W4T6Y8U0I found a record at school 225070, so no new record can be created. Use person link psl_3N8QK2WD7F.` | the lookup matched someone | use that link |
| 409 | `ALREADY_LINKED` | `Learner lrn_2b7d4e9a1c is already linked to a record at school 225070 (psl_8H2J4K6L8M).` | enrolled before | use that link |
| 409 | `NON_DEGREE_NOT_OFFERED` | `School 225070 does not enroll non-degree students through GoldWire. Contact the school's Admissions office.` | school setting | contact the school |
| 409 | `TERM_NOT_OPEN` | `Non-degree enrollment for 2027-SU at school 225070 opens 2027-03-29.` | too early | come back then |
| 410 | `LOOKUP_REF_EXPIRED` | `Lookup plk_9W4T6Y8U0I expired at 2026-09-26T15:05:20Z. Look the learner up again with Get person.` | over 24 hours old | Call 5 again |
| 422 | `DUAL_CREDIT_NOT_SUPPORTED` | `Students still in high school enroll through the school's dual-credit program, which needs high-school approval. GoldWire cannot enroll them.` | `still_in_high_school` | contact the school |
| 422 | `IDEMPOTENCY_CONFLICT` | `Key 01J8Z5B2C4D6E8F0G2H4J6K8M0 was already used for a different enrollment request on 2026-09-25T15:12:44Z. Use a new key.` | | new key |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | | wait |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was created. Reference 6d7e8f9a-0b1c-4d2e-9f3a-4b5c6d7e8f9a.` | | retry with the same key |

## What gets recorded in S3

1. `goldwire/idempotency/01J8Z5B2C4D6E8F0G2H4J6K8M0.json` — written once; the result only.
2. On `created` or `already_known`: `goldwire/persons/225070/lrn_2b7d4e9a1c.json` (encrypted) — the person link and the school's person ID, exactly as in [Get person](05-get-person.md#what-gets-recorded-in-s3).

The name, date of birth, address and other details go to the school and are **not kept** by GoldWire.
