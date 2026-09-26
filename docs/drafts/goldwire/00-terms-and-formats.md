# GoldWire terms and formats

[← Summary](README.md) · Next: [How GoldWire talks to any SIS →](00-sis-adapter.md)

> **Draft.** Nothing here is live.

## Why this page exists

Every student information system (SIS) names the same things differently. Banner calls a term
`202720`, PeopleSoft calls it `2272`, Colleague `2027SP`, and Workday "Spring 2027 Semester". A section is
a CRN in one, a class number in another, and a reference ID in a third.

**GoldWire speaks one language.** Every value sent to GoldWire and every value it returns uses the
formats on this page, whatever system the school runs. The school's adapter translates (see
[How GoldWire talks to any SIS](00-sis-adapter.md)). Where it helps, replies also show the school's
own value under `native`, so a registrar can recognize it — but clients never have to send it.

**Input is forgiving; output is exact.** Where noted, GoldWire accepts a few common spellings and
always answers in the one exact form.

---

## The school and where it teaches

### School ID — `unitid`

| | |
|---|---|
| **What it is** | The school, by its federal IPEDS UNITID — the same id CourseShelf and GoldCheck use |
| **Format** | exactly 6 digits |
| **Examples ✓** | `225070` · `228529` · `100654` |
| **Not accepted ✗** | `22507` (5 digits) · `Grayson College` (a name — look it up with CourseShelf `find_institution`) · `OPEID 00354800` (a different federal id) |

### Campus ID — `campus_id`

| | |
|---|---|
| **What it is** | One campus or teaching location of a school, including a virtual "campus" for online classes if the school uses one |
| **Format** | the school's own campus code, 1–10 characters, `A–Z 0–9 _`, **unique within the school**. GoldWire returns it in capitals and accepts any case |
| **Examples ✓** | `MAIN` · `SOUTH` · `DTWN` · `WEB` · `C01` |
| **Not accepted ✗** | `Main Campus` (a name, not a code) · `MAIN-CAMPUS` (hyphen) · `225070-MAIN` (the school is sent separately) |
| **Where it comes from** | [Call 2 — Get campus locations](02-get-campus-locations.md) |

---

## When it is taught

### Academic year — `academic_year`

| | |
|---|---|
| **What it is** | The school's academic year, named by the calendar years it spans |
| **Format** | `YYYY-YY` — the year it starts, a hyphen, the last two digits of the year it ends |
| **Examples ✓** | `2026-27` (Fall 2026 through Summer 2027) · `2027-28` |
| **Not accepted ✗** | `2026/2027` · `AY27` · `2027` (ambiguous — is that the year starting or ending?) |

### Term ID — `term_id`

| | |
|---|---|
| **What it is** | One term of the school's calendar — a semester, quarter or intersession |
| **Format** | `YYYY-TT` — the **calendar year the term starts in**, a hyphen, and a two-letter term code |
| **Term codes** | `FA` fall · `WI` winter · `SP` spring · `SU` summer · `IN` intersession (e.g. a January or May mini-term) |
| **Examples ✓** | `2026-FA` (Fall 2026, academic year 2026-27) · `2027-SP` (Spring 2027, also 2026-27) · `2027-SU` · `2027-IN` |
| **Not accepted ✗** | `202720` (Banner's own code — shown as `native`, never sent) · `2027SP` (hyphen missing) · `Spring 2027` (a name) · `2027-S` (one letter) |
| **Where it comes from** | [Call 1 — Get academic terms](01-get-academic-terms.md) |

Semester and quarter schools use the same codes: a quarter school's Winter quarter is `2027-WI`.

### Session (part of term) — `session_id`

| | |
|---|---|
| **What it is** | A part of a term with its own dates, such as the first or second eight weeks, or a late-start session |
| **Format** | 1–8 characters, `A–Z 0–9`. `FULL` always means the whole term; other codes are the school's own |
| **Examples ✓** | `FULL` · `8W1` (first 8 weeks) · `8W2` · `LATE` · `SU1` · `SU2` |
| **Not accepted ✗** | `First 8 Weeks` (a name) · `8-W-1` (hyphens) |

---

## What is taught

### Subject — `subject`

| | |
|---|---|
| **What it is** | The subject prefix of a course, as the school publishes it |
| **Format** | 2–7 characters: capital letters, and `&` where the school uses it. Accepted in any case, returned in capitals |
| **Examples ✓** | `ART` · `ENGL` · `MATH` · `BIOL` · `HIST` · `M&E` |
| **Not accepted ✗** | `Art` → accepted, returned as `ART` · `ART.` (punctuation) · `English` (a name) |

### Course number — `course_number`

| | |
|---|---|
| **What it is** | The number part of a course, as the school publishes it |
| **Format** | 3–6 characters: digits, optionally followed by letters. **Leading zeros are kept** |
| **Examples ✓** | `101` · `1301` · `101L` (a lab) · `0310` (a developmental course) · `2425H` (honors) |
| **Not accepted ✗** | `310` when the school publishes `0310` (the zero matters) · `101-01` (that is a section) |

### Course ID — `course_id`

| | |
|---|---|
| **What it is** | One course at one school, written the way it appears in the catalog: **subject, one space, course number** |
| **Format** | `{subject} {course_number}` |
| **Examples ✓** | `ART 101` · `ENGL 1301` · `BIOL 1406L` · `MATH 0310` |
| **Also accepted** | `ART101`, `art 101`, `ART-101`, `ART  101` — all read as `ART 101`. Replies always say `ART 101` |
| **Not accepted ✗** | `ART 101 01` (includes a section) · `Intro to Drawing` (a title) · `lu_225070_ART101` (a CourseShelf id — replies include it as `learning_unit_id` for linking, but it is not a course ID) |
| **In a URL** | the space is written `%20`: `course_id=ART%20101` |

A course ID is only unique **within one school**; `ART 101` at two schools are two different courses.

### Section number — `section_number`

| | |
|---|---|
| **What it is** | The school's label for one offering of a course in one term |
| **Format** | 1–6 characters, `A–Z 0–9`. **Leading zeros are kept** |
| **Examples ✓** | `001` · `002` · `W01` (web) · `H01` (honors) · `N50` (night) |
| **Not accepted ✗** | `1` when the school publishes `001` |

### Section ID — `section_id`

| | |
|---|---|
| **What it is** | GoldWire's own id for one section (school + term + course + section number). Returned by Get Sections, used by every later call |
| **Format** | `sec_` followed by letters, digits and underscores. **Opaque — do not build it or take it apart**; always use the one GoldWire returned |
| **Example ✓** | `sec_225070_2027SP_ENGL1301_002` |

### The school's own section key — `native.section_ref`

| | |
|---|---|
| **What it is** | The key the school's own system uses for the section. Shown so staff can find it; never needed by clients |
| **Examples** | Banner CRN `21457` · PeopleSoft class number `4312` · Colleague section name `ART-101-01` · Workday section reference `ART 101-01 - Intro to Drawing` |

---

## How and when it meets

### Modality — `modality`

| Value | Means | Typical school labels it covers |
|---|---|---|
| `in_person` | all meetings on campus | "Traditional", "Face to face", "Lecture" |
| `hybrid` | some meetings on campus, some online | "Blended", "Hybrid", "Flex" |
| `online_sync` | online, at scheduled meeting times | "Live online", "Synchronous", "Virtual scheduled" |
| `online_async` | online, no scheduled meeting times | "Online", "Web", "Asynchronous", "Anytime" |

**Not accepted ✗** `online` (say which: `online_sync` or `online_async`) · `remote` · `F2F`.

### Days of the week — `days`

| | |
|---|---|
| **Format** | three-letter codes `MON TUE WED THU FRI SAT SUN`. In a query: comma-separated, `days=TUE,THU`. In a reply: a list, `["TUE", "THU"]` |
| **Examples ✓** | `MON,WED,FRI` · `TUE,THU` · `SAT` |
| **Not accepted ✗** | `TR` or `MWF` (single-letter codes differ between systems — R, H and Th all mean Thursday somewhere) · `Tuesday` · `T/Th` |

### Time and time window — `start`, `end`, `time_window`

| | |
|---|---|
| **Format** | `HH:MM`, 24-hour, **the campus's local time**. A window is `HH:MM-HH:MM`, start before end |
| **Examples ✓** | `09:30` · `18:00` · window `17:00-22:00` |
| **Not accepted ✗** | `9:30am` · `0930` (Banner's form — the adapter converts it) · `22:00-17:00` (backwards) |

### Dates and instants

| | Format | Example |
|---|---|---|
| a day | `YYYY-MM-DD` | `2027-01-19` |
| a moment | ISO-8601 in UTC, ending `Z` | `2026-09-25T14:02:11Z` |

---

## Seats

| Field | Means | Example |
|---|---|---|
| `capacity` | seats the school allows | `25` |
| `enrolled` | registered now, per the school | `21` |
| `held` | seats held by GoldWire learners who are paying right now (up to 20 minutes) | `2` |
| `available` | `capacity − enrolled − held`, never below 0 | `2` |
| `seat_status` | `open` (seats available) · `waitlist` (full, waitlist open) · `closed` (full, no waitlist) · `cancelled` | `open` |

---

## People

| Term | What it is | Format | Examples ✓ | Not accepted ✗ |
|---|---|---|---|---|
| **Learner ID** `learner_id` | the person in GoldSeam — private, the same at every school | `lrn_` + 8–40 letters/digits | `lrn_8f3a2c91d7` | an email, name or SSN |
| **School person ID** `school_person_id` | the person's ID **at one school**, in that school's own format | whatever the school uses, 1–20 characters | Banner `A00482913` · PeopleSoft EMPLID `1048291` · Colleague `0482913` · Workday `S0048291` | an SSN (never accepted) |
| **Person link ID** `person_link_id` | GoldWire's link between one learner and their record at one school, made by [Get Person](05-get-person.md) | `psl_` + 10 letters/digits | `psl_3N8QK2WD7F` | |
| **Date of birth** `date_of_birth` | | `YYYY-MM-DD` | `2001-04-17` | `04/17/2001` |
| **Postal code** `postal_code` | | US: 5 digits or 5+4 | `75090` · `75090-1234` | `7509` |

## Money

Always `{ "amount": "241.00", "currency": "USD" }` — the amount as text with exactly two decimal
places. ✗ `241`, `241.0`, `"$241.00"`.
