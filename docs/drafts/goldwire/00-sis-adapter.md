# How GoldWire talks to any SIS

[← GoldWire terms and formats](00-terms-and-formats.md) · [Summary](README.md) · Next: [Call 1 — Get academic terms →](01-get-academic-terms.md)

> **Draft.** Nothing here is live. The vendor field names below are **starting points** for each
> adapter build — every school configures its SIS differently, so each mapping is confirmed with that
> school's registrar and IT staff during onboarding.

## The idea in one picture

```
 Learner app ──▶ GoldWire ──▶ School adapter ──▶ the school's SIS
                 (one language:        (translates:          (Banner, PeopleSoft,
                  GoldWire terms)       terms, codes,          Colleague, Workday,
                                        times, days)           Jenzabar, homegrown …)
```

- **GoldWire never speaks a vendor's language.** It asks the adapter for things in GoldWire terms ([terms and formats](00-terms-and-formats.md)) and gets answers in GoldWire terms.
- **Each school has one adapter configuration.** Schools on the same vendor share adapter code; each has its own mapping (its term codes, campus codes, modality codes).
- **A homegrown SIS** implements the adapter operations below directly — no vendor code needed.

---

## What an adapter must do — nine operations

Each GoldWire call uses one or more of these. The input and output are always in GoldWire terms.

| # | Adapter operation | Used by | Given | Returns | How fresh |
|---|---|---|---|---|---|
| A1 | `list_terms` | [Call 1](01-get-academic-terms.md) | school | terms with dates, registration windows, sessions | refreshed daily |
| A2 | `list_campuses` | [Call 2](02-get-campus-locations.md) | school | campuses with codes, names, addresses | refreshed daily |
| A3 | `list_sections` | [Call 3](03-get-sections.md) | term, course, filters | sections, meetings, **live seat counts** | live (8-second limit), or cached no more than 60 seconds |
| A4 | `get_fee_rules` | [Call 4](04-get-section-fees.md) | term, course, section | tuition rates and fees | daily; most schools' fees come from their **published** policies in CourseShelf instead |
| A5 | `find_person` | [Call 5](05-get-person.md) | name, date of birth, contact details, school ID if known | match result and the school's person ID | live |
| A6 | `register` | [Call 10](10-book-seat.md) | school person ID, term, native section key | the school's answer | live, with retries |
| A7 | `drop` / `waitlist_add` / `waitlist_remove` | [Calls 7](07-hold-seat.md), [12](12-release-seat.md) | school person ID, native section key | the school's answer | live, with retries |
| A8 | `push_answer` | [Call 13](13-record-school-answer.md) | — (the SIS side starts it) | a later answer on a booking | when the school decides |
| A9 | `create_non_degree_student` | [Call 6](06-enroll-non-degree.md) | name, date of birth, contact, address, first term, reason, residency | created / pending review / refused / already known, and the new person ID | live, with retries |

---

## Three ways to connect

| Style | When to use | How it works |
|---|---|---|
| **Vendor API** | the school licenses its vendor's integration platform | the adapter calls the vendor's published API. Preferred: supported by the vendor, respects the SIS's own business rules (prerequisites, holds, time conflicts) |
| **Read views + vendor API for writes** | the API is slow or limited for searching | read-only database views feed A1–A4; registration (A6, A7) still goes through the vendor's own registration API so the SIS's rules run. **GoldWire never writes to SIS tables directly** |
| **Homegrown contract** | a homegrown or niche SIS | the school exposes the nine operations as a small web service in exactly the GoldWire shapes on these pages. For read-only schools, nightly files for A1–A2 plus a live seat endpoint for A3 are enough to list sections (`bookable: false` until A6 exists) |

**Integration platforms by vendor** (to confirm per school):

| SIS | Integration platform the adapter would use |
|---|---|
| Ellucian Banner | Ellucian Ethos Integration (Ethos Data Model resources), Banner Student APIs |
| Ellucian Colleague | Ellucian Ethos Integration, Colleague Web API |
| Oracle PeopleSoft Campus Solutions | Campus Solutions web services / Integration Broker (Class Search and Enrollment services) |
| Workday Student | Workday web services (SOAP) and REST APIs |
| Jenzabar, Anthology Student, CAMS, others | the vendor's API where one exists; otherwise the homegrown contract |

Where a school runs **OneRoster 1.2** (1EdTech) or Ethos, GoldWire's objects line up with theirs:

| GoldWire | Ethos Data Model resource | OneRoster 1.2 |
|---|---|---|
| academic term | `academic-periods` | `academicSessions` (type `term`) |
| campus | `sites` | `orgs` (type `school` / site) |
| course | `courses` | `courses` |
| section | `sections` | `classes` |
| person lookup | `person-matching-requests` | `users` (lookup only, no matching) |
| registration | `section-registrations` | `enrollments` |

---

## The crosswalk — GoldWire term to each SIS

What each GoldWire term is usually called in each system. **Examples are illustrative**; the adapter
configuration holds each school's actual codes.

### School, campus and calendar

| GoldWire | Banner | PeopleSoft Campus Solutions | Colleague | Workday Student |
|---|---|---|---|---|
| `unitid` `225070` | institution (one per database, or VPDI code) | `INSTITUTION` e.g. `INST1` | institution in `INSTITUTIONS` | Institution / Academic Unit |
| `campus_id` `MAIN` | campus code (`STVCAMP`) e.g. `M` | `CAMPUS` e.g. `MAIN` | location code (`LOCATIONS`) e.g. `MC` | Campus Location |
| `academic_year` `2026-27` | aid year / academic year setup | `ACAD_YEAR` e.g. `2027` | reporting year e.g. `2026` | Academic Year e.g. "2026-2027" |
| `term_id` `2027-SP` | term code (`STVTERM`) e.g. `202720` | `STRM` e.g. `2272` | term code (`TERMS`) e.g. `2027SP` | Academic Period e.g. "Spring 2027 Semester" |
| `session_id` `8W1` | part of term (`PTRM`) e.g. `2` | `SESSION_CODE` e.g. `8W1` | term sessions / sub-terms | Academic Period (child period) |

### Course and section

| GoldWire | Banner | PeopleSoft Campus Solutions | Colleague | Workday Student |
|---|---|---|---|---|
| `subject` `ART` | subject code | `SUBJECT` | subject (`SEC.SUBJECT`) | Course Subject |
| `course_number` `101` | course number | `CATALOG_NBR` (often space-padded, e.g. `  101`) | course number (`SEC.COURSE.NO`) | Course Number |
| `course_id` `ART 101` | subject + course number | `SUBJECT` + `CATALOG_NBR` (internal key `CRSE_ID` is not shown) | `ART-101` | Course Listing "ART 101" |
| `section_number` `001` | sequence number, e.g. `001` | `CLASS_SECTION` e.g. `01` | section number (`SEC.NO`) e.g. `01` | Course Section number |
| `native.section_ref` | **CRN** e.g. `21457` | **class number** `CLASS_NBR` e.g. `4312` | section name e.g. `ART-101-01` | Course Section reference ID |
| `modality` `in_person` | instructional method code + schedule type | `INSTRUCTION_MODE` e.g. `P`, `OL`, `HY` | instructional method (`SEC.INSTR.METHODS`) e.g. `LEC`, `ONL` | Delivery Mode |

### Meetings and seats

| GoldWire | Banner | PeopleSoft Campus Solutions | Colleague | Workday Student |
|---|---|---|---|---|
| `days` `["TUE","THU"]` | a flag per day, e.g. `T`, `R` | `MON` … `SUN`, each `Y`/`N` | days list on the section meeting | Meeting Pattern days |
| `start` / `end` `09:30` | `0930` (HHMM) | time value | time value | Meeting Pattern time |
| `capacity` | maximum enrollment | `ENRL_CAP` | section capacity | Enrollment Capacity |
| `enrolled` | actual enrollment | `ENRL_TOT` | active enrollment count | enrolled count |
| waitlist | wait capacity / wait count | `WAIT_CAP` / `WAIT_TOT` | section waitlist settings | Waitlist Capacity |

### People

| GoldWire | Banner | PeopleSoft Campus Solutions | Colleague | Workday Student |
|---|---|---|---|---|
| `school_person_id` | Banner ID (`SPRIDEN_ID`) e.g. `A00482913` | `EMPLID` e.g. `1048291` | Colleague person ID e.g. `0482913` | Student ID e.g. `S0048291` |
| `student_type` `degree_seeking` / `non_degree` | student type and degree/program on the general student record | academic program on the student's career (a non-degree program code such as `NDEG`) | academic program (a non-degree program) | Program of Study / Student Type |
| `admission_status` | admissions application and decision | admission application and program action | application status | Admission / Application status |
| `load` `full_time` / `part_time` | computed from term credits against the school's rule | academic load, computed per term | load computed per term | Academic Load |
| person lookup ([Call 5](05-get-person.md)) | common matching rules / Ethos `person-matching-requests` | Search/Match | Ethos `person-matching-requests` | find-student / matching services |

---

## What a school's mapping looks like

One school's adapter configuration — the only place vendor codes live. This is the concrete answer to
"how does `202720` become `2027-SP`":

```json
{
  "unitid": "225070",
  "sis": "banner",
  "connection": "ethos",
  "timezone": "America/Chicago",
  "terms": {
    "202710": { "term_id": "2026-FA", "academic_year": "2026-27" },
    "202720": { "term_id": "2027-SP", "academic_year": "2026-27" },
    "202730": { "term_id": "2027-SU", "academic_year": "2026-27" }
  },
  "sessions": {
    "1": "FULL",
    "2": "8W1",
    "3": "8W2"
  },
  "campuses": {
    "M": "MAIN",
    "S": "SOUTH",
    "W": "WEB"
  },
  "modality": {
    "TRAD": "in_person",
    "HYBR": "hybrid",
    "OLSY": "online_sync",
    "WEB":  "online_async"
  },
  "days": { "M": "MON", "T": "TUE", "W": "WED", "R": "THU", "F": "FRI", "S": "SAT", "U": "SUN" },
  "time_format": "HHMM",
  "person_id_format": "^A\\d{8}$",
  "registration": { "supports_register": true, "supports_waitlist": true, "non_degree_enrollment": true, "non_degree_credit_limit": 12, "full_time_at_credits": { "FA": 12, "SP": 12, "SU": 6 } }
}
```

**Rules the adapter follows when translating**

| Situation | Rule |
|---|---|
| a code with no mapping (a new term, a new campus) | the item is left out and reported in the reply's `not_held` list, e.g. `{ "what": "term 202740", "note": "no mapping configured" }` — **never guessed** |
| two SIS codes map to one GoldWire value (e.g. two online method codes) | allowed; the reply shows the original under `native` |
| times | converted to `HH:MM` in the campus's local time zone |
| course numbers | trailing padding removed; **leading zeros kept** |
| a section with no meeting times | `meetings: []` — normal for `online_async` |
| a value the SIS does not hold (e.g. no waitlist feature) | the field is `null` and the reason goes in `not_held` |
| the SIS is unreachable | GoldWire replies `SCHOOL_OFFLINE`; nothing is made up from old data except where a page says a cache is allowed |

## Adapter errors, and what the learner sees

| The adapter hits | GoldWire replies |
|---|---|
| timeout, connection refused, SIS maintenance window | `503 SCHOOL_OFFLINE` |
| authentication to the SIS failed | `503 SCHOOL_OFFLINE` to the learner; GoldWire staff are alerted |
| a code with no mapping | the item left out, listed in `not_held` |
| the SIS rejects a registration (prerequisite, hold, time conflict, closed) | not an error — `booking_state: REJECTED` with the school's reason, verbatim ([Call 10](10-book-seat.md)) |
