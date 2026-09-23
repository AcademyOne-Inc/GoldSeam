# GoldCheck — will my credits transfer

**What it answers:** given what a person already holds — courses they have taken, exams they have
sat — what a **named school's own published record** says it may count as.

Contract tag on every answer: `goldcheck_v1`.

## The tool

`will_my_credits_transfer(held, target_unitid)` — optionally against one `checklist_id` so the answer
is scoped to a single program's requirements rather than the whole catalog.

- **`held`** is the person's own list: courses with their school, code and title, and exam results such as AP, CLEP, IB or DSST with their scores.
- **`target_unitid`** is the school being asked about, by IPEDS UNITID.
- It answers as a **card** (an MCP App) in clients that render one, and as text everywhere else.

## How an answer is built, and what it is worth

**A course** is matched against the target school's own catalog, and the match is grounded in a quoted
phrase from the course's own description. A match that cannot be grounded is not offered.

**An exam is not matched to a course.** It is answered by **the school's own published credit-by-exam
rules** — the score it requires and what it awards. A school that has published no rule for that exam
returns **not held**, which is an honest gap in the published record, not a refusal.

**Nothing here is a decision.** The school decides. This says what the school has published, with the
catalog edition and date behind it, so a person can ask the right question of the right office.

## What it will not do

- It does not promise credit, estimate a likelihood, or average across schools.
- It does not substitute one school's rule for another's.
- It does not invent an equivalency where the catalogs hold none.
