# KSA Dictionary — the shared vocabulary

**What it answers:** what a word means inside the KSA model, where it places, what it sits under, and
how a program code crosses to occupations and back.

Contract tag on every answer: `ksa_dictionary_v1`.

## Why a dictionary at all

Education and work describe the same human capability in different words. The dictionary is the layer
that lets a course description, an occupation's requirements and a person's own account of their
experience be compared without one being quietly rewritten into another's terms.

It answers in **layers**, and every answer says which layer answered:
- a **governed lexicon** — the terms KSAWorks itself governs, each placed in the model;
- the **O\*NET content model** — knowledge, skills, abilities and work activities as the federal model defines them;
- a **WordNet-derived index** — general English, for words the first two layers do not hold.

## The tools

- `resolve_term(term)` — where a word places in the KSA model, attributed to the layer that answered.
- `expand_term(term)` — walk a word up its is-a chain, so a narrow term can be matched to a broader one.
- `crosswalk_code(code)` — cross a **CIP** program code to **SOC** occupations, or a SOC back to CIPs.
- `get_dictionary_summary` — what the dictionary itself holds, by layer, each with its version and build date.

## What it will not do

- It does not invent a meaning. A word the layers do not hold comes back **not held**.
- It does not flatten the layers into one voice: an answer from general English is labelled as such and is not presented as a governed term.
