---
name: goldseam
description: Ask GoldSeam for sourced answers about colleges, programs, checklists, transfer credit, certifications, careers and a person's own prior learning. Use whenever a question is about a school, a program of study, whether credits or exam scores will transfer, what a certification requires, where a job can lead, or what someone's own transcripts and experience may be worth as credit.
---

# GoldSeam

GoldSeam answers from published records — colleges' own catalogs, a public certification inventory,
the federal crosswalks — at `https://goldseam.ksaworks.com/mcp`. **This skill teaches you how to ask;
it does not connect you.** If the GoldSeam tools are not loaded, say so plainly and point the person
at `install/claude/README.md` rather than answering from memory.

## The rules that make an answer worth having

1. **Never fill a gap from memory.** If GoldSeam says **not held**, that is the answer: we have not captured it yet. Say so. Do not substitute a school's website, your training data, or a plausible guess.
2. **Carry the source.** Every answer names where it came from and the snapshot date. Pass both to the person; it is what lets them act on it.
3. **Ids come from answers, never from guesses.** Start in plain words, then follow the ids a result returns.
4. **Name the school.** Most questions are about one institution's own record. Resolve a name to a UNITID first with `find_institution`.
5. **Absence and error are different.** `not held` is a gap in what has been captured. `offline` means the service was unreachable — try again, and do not report it as an absence.
6. **GoldSeam does not decide.** It reports what a school published. Awarding credit is the institution's act; say whose call it is.

## Where to start, by what was asked

| The person asks | Start with |
|---|---|
| about a school | `find_institution` → `get_institution` |
| what a school offers | `programs_for_institution` · `learning_units` · `find_checklists` |
| what a program requires | `find_checklists` → `get_checklist`; two schools at once → `compare_program_checklists` |
| to browse or compare options | `search_programs_and_learning_units` (product cards with refinements) |
| will my credits or exam transfer | `will_my_credits_transfer` with the courses/scores they hold and the target school's UNITID |
| about certifications | `search_certifications` in plain words → `get_certification` → `occupations_for_certification` |
| where a job can lead | `find_transitions` · `compare_destinations` |
| what a word means here | `resolve_term` · `expand_term` · `crosswalk_code` |
| what their own learning is worth | the GoldRibbon sequence below |
| how much is held | `get_courseshelf_summary` — never quote a figure from memory |

## The GoldRibbon sequence

In order, each adding to the same `package`, which **you hold, not the service**:

`assemble_my_formal_education` → `assemble_my_work_and_credentials` → `assert_my_life_experiences`
→ `my_selfie_ksa_view` → `draft_my_claims` → `suggest_credit_for_my_learning`

- Ask the person to attach what they have — transcripts, a resume, certifications, military records — one step at a time. Do not invent a document.
- Keep life experiences separate from formal education. An assertion is not a transcript.
- Every claim must quote the person's own evidence. Do not write a claim the evidence does not support.
- The service stores nothing. Tell the person to keep the package if they want it later.
- It **assembles, never assesses**. Do not tell someone they have earned credit; tell them what may be recognised and who decides.

## Presenting an answer

Lead with what was found and where it came from. For a shop-style result, show the cards first —
name, award, school and city, subject, credits — then the top refinements with their counts, then
offer the checklist. For a coverage question, quote the service's own summary, with its publish
number and date.
