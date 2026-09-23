# Industry Certifications — what they are, who issues them, what they serve

**What it answers:** which certifications exist in a field, who issues one, what it takes to earn and
renew it, which occupations it serves, and which academic programs cross-reference it.

Contract tag on every answer: `industry_certifications_v1`.

## The tools

**Start in plain words**
- `search_certifications(query)` — a trade, a field, a job or a company: *nursing*, *HVAC*, *IBM*, *welding inspector*. Narrow with `issuer`.

**Then follow an id**
- `get_certification(id)` — everything held about one: the issuing body, the occupations it serves, exam and renewal requirements, routes to earn it, and its Credential Engine identity where one exists.
- `occupations_for_certification(id)` — the occupations it serves, as SOC codes with titles.
- `related_certifications(id)` — certifications that serve the same work.

**Or start from a code**
- `certifications_for_occupation(soc)` — by occupation, e.g. `29-1141` for registered nurses.
- `certifications_for_program(cip)` — by academic program, e.g. `51.3801`.

**Or from the issuer**
- `list_issuers` — the issuing organizations, narrowable with `query`.

## How to read an answer

Every certification carries its **issuer** and the **source** the statement came from. Where a
certification is published in the Credential Engine registry, its CTID is carried, so the record can be
joined to the wider credential ecosystem rather than standing alone.

Codes are canonical: **SOC** dashed without a suffix (`11-1021`), **CIP** dotted six-digit (`52.0201`).

## What it will not do

- It does not rank certifications, endorse one, or say which is "best".
- It does not state a requirement the issuer has not published.
- An absence is **not held**, never a guess — see [answers.md](../answers.md).
