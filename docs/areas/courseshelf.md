# CourseShelf — colleges and what they teach

**What it answers:** what a school offers, what a program requires, what a course is, and what the
school has published about transferring into it — from the institutions' **own catalogs**, not a
survey and not an aggregator's summary.

Contract tag on every answer: `courseshelf_v1`.

## The model, and why the tools are shaped this way

CourseShelf holds objects, and there is a **`find_` / `get_` pair per object**: `find_` searches and
returns candidates, `get_` returns one record in full. Learn the six objects and you know the area.

| Object | What it is |
|---|---|
| **Institution** | a postsecondary provider, spined on its IPEDS UNITID |
| **Program** | a program of study offered by one institution |
| **Checklist** | the requirements for an award — the generic itinerary, group by group, item by item |
| **Learning unit** | a course, or anything else that can satisfy a requirement |
| **Catalog / policy** | the published edition an answer came from, and the school's own rules |
| **Agreement / pathway / comparability / framework** | what one school has published about another school's work |

## The tools

**Find a school**
- `find_institution` — by `query` (a name fragment or a UNITID), by `state` for that state's schools, by `city`, or with nothing at all for the states themselves.
- `get_institution` — everything held about one, by `unitid` or `organization_id`.

**What a school offers**
- `programs_for_institution(unitid)` — its programs, narrowable with `query`.
- `learning_units(unitid)` — its courses, narrowable by `subject`, `code`, `unit_type` or `query`.
- `find_checklists(unitid)` — its checklists, optionally for one `program_id`.
- `find_catalogs(unitid)` · `get_catalog(catalog_id)` — the published editions behind the answers.
- `find_policies(unitid)` · `get_policy(policy_id)` — the school's published rules.

**Across schools**
- `find_programs(query)` — programs anywhere, narrowable by `state`, `city` or `unitid`.
- `search_programs_and_learning_units` — the shop: product cards with counts to refine by (`subject`, `award`, `credential`, `state`, `place`, `sector`, `cip`, `type`). Courses across every school need words of three characters or more.
- `compare_program_checklists(unitids[], program)` — the same program at several schools, side by side: requirement groups, items and published credit values beside each school's published tuition and fee policies.

**One record**
- `get_program(program_id)` · `get_checklist(checklist_id)` · `get_learning_unit(learning_unit_id)`.
- `checklists_requiring_learning_unit(learning_unit_id)` — which programs require this course.

**What schools publish about each other**
- `find_agreements` · `get_agreement` — articulation agreements.
- `find_pathways` · `get_pathway` — published routes, by agreement or by checklist.
- `find_comparability` · `get_comparability` · `comparability_for_learning_unit` — equivalencies, including by `code`, `title`, `school` and `direction`.
- `find_transfer_frameworks` · `get_transfer_framework` — statewide and system frameworks.

**The service's own coverage**
- `get_courseshelf_summary` — what is held, by state and by area, with the census reading. See [coverage.md](../coverage.md).

## What it will not do

- It does not **decide** that a course transfers. It reports what a school published. The judgment belongs to the school — see [goldcheck.md](goldcheck.md) for what can be said and how.
- It does not invent a program, a course or a requirement. If nothing is held, the answer says **not held** — see [answers.md](../answers.md).
- It does not fill a missing title or description with a placeholder. A field with nothing behind it stays empty.

## Coverage, honestly

Catalogs are still being captured, so coverage is uneven: some states and schools are complete, others
are not started. That is why **not held** appears, and it is why no figure is printed on this page —
`get_courseshelf_summary` always holds the current one.
