# Transitions — where a path can lead

**What it answers:** from where a person stands — an occupation, a field of work — which destinations
are reachable, and how they compare.

Contract tag on every answer: `transitions_v1`.

## The tools

- `find_transitions` — the destinations reachable from a starting point: `from` in plain words or `from_soc` as an occupation code, narrowable by `state`, by `class` of transition, and by what to `prefer`.
- `compare_destinations(from_soc, to_socs[])` — several destinations set against each other from the same starting point.

## How to read an answer

A transition is a **statement about published relationships** — between occupations, the programs that
prepare for them, and the credentials that serve them. It is not a prediction about a person, and it
is not advice. It says where the road network goes, not who should travel it.

Occupations are SOC, dashed and without a suffix (`11-1021`); programs are CIP, dotted six-digit
(`52.0201`). Use [`crosswalk_code`](dictionary.md) to move between them.

## What it will not do

- It does not forecast a salary for an individual or promise an outcome.
- It does not rank people, or score their likelihood of succeeding.
- Where no relationship is held, the answer is **not held**.
