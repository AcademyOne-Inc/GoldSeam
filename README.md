# GoldSeam

**The marketplace for teaching, training and learning — to explore, compare and engage with, through
the AI you already use.**

```
https://goldseam.ksaworks.com/mcp
```

No authentication. No account. No sign-up. It is a public service, published by
[AcademyOne, Inc.](https://www.academyone.com), the company behind
[KSAWorks](https://ksaworks.com).

GoldSeam speaks the [Model Context Protocol](https://modelcontextprotocol.io) over streamable HTTP.
Every answer carries its source and the date of the snapshot that produced it, and **an absence is
stated as an absence** rather than filled in.

**What you can do here.** **Explore** what postsecondary organizations offer — programs, courses,
checklists, certifications, and the careers they lead to. **Compare** them on what actually differs:
what a program requires, what your credits and exams may count as, what a certification takes.
**Engage** — take an answer to the school that decides, with its source attached.

**Why it exists.** The advising conversation has moved. People ask the assistant on their phone, not a
portal, and what is missing there is not authoritative information about teaching, training and
learning — it is that the information is scattered across every institution that publishes it.
GoldSeam puts it in one seam, wherever the person already is: **assistant-agnostic by design**, so a
new surface is new reach rather than a new integration.

---

## Connect it

**In Claude** — open **Customize → Connectors**, click **+ → Add custom connector**, paste the address,
click **Add**. Then in any chat press **+ → Connectors** and switch **GoldSeam** on.

Using ChatGPT, Copilot, Gemini, VS Code or your own agent? Same address —
[`install/`](install/) has that client's steps, a `mcp.json` snippet, and a skill that teaches an AI
how to ask GoldSeam well.

**No AI app at all?** The same data answers questions in a browser at
[KSAWorks](https://ksaworks.com/where-do-you-want-to-go.html).

---

## What it covers

**CourseShelf** — colleges and what they teach, from the institutions' own published catalogs.
Identify a school by name or IPEDS UNITID, or list a state's; read its programs, its courses, its
checklists (the requirements for an award), its catalogs and its published policies; search programs
and courses across schools; compare two schools' checklists for the same program side by side.

**GoldCheck** — will my credits transfer. Give a course history or an exam score and see what a named
school's own published rules and catalog say it may count as.

**GoldRibbon** — your own record. Hand your AI transcripts, a resume, certifications or military
records, add life experiences in your own words, and get your Competency Selfie and what your prior
learning may earn as credit against a named program's checklist. **Your AI keeps the package;
GoldSeam stores nothing.**

**Industry Certifications** — search by a trade, field, job or company; everything held about one
certification (issuer, occupations served, exam and renewal requirements, Credential Engine identity);
the certifications that serve an occupation (SOC) or cross-reference an academic program (CIP); the
issuing bodies.

**Transitions** — where a path can lead: the destinations reachable from where a person stands, and
how they compare.

**KSA Dictionary** — the shared vocabulary. Resolve a word to where it places in the KSA model across
a governed lexicon, the O\*NET content model and a WordNet-derived index, each answer attributed to its
layer; walk a word up its is-a chain; cross a program code to occupations and back; ask what the
dictionary itself holds.

Each area has its own page under [`docs/areas/`](docs/areas/), and
[`examples/`](examples/) holds real runs — the prompt as asked and the answer as returned, stamped
with the build and the data publish that produced it.

---

## Reading an answer

| You see | It means |
|---|---|
| a named source | a school's own catalog page, a published rule, or the federal record |
| a snapshot date | which capture answered, so you always know how current it is |
| **not held** | we have not captured that yet. An honest gap, not an opinion |
| **offline** | the catalog service was unreachable at that moment. Try again |

GoldSeam never invents a school, a course or a requirement. If it cannot find one, it says so.

**Coverage is uneven while catalogs are being captured**, and the service publishes its own coverage
measure — ask it with `get_courseshelf_summary`. No number is written on this page, because the
service's own answer is always the current one.

---

## Using it beyond a first look

**Nothing is required to connect.** Anonymous use carries a daily allowance; past it, the answer
points at a free key-request form. **A key is authorization to keep using the service, not
authentication to reach it** — it raises the allowance and names your organization in the usage
record. Ask for one at https://goldseam.ksaworks.com/request-key.

## Privacy

The usage record keeps **counts only** — no personal record, and nothing a person types. Evidence
handed to GoldRibbon is held by your AI, not by this service. The full statement:
[ksaworks.com/privacy.html](https://ksaworks.com/privacy.html) · see also [SECURITY.md](SECURITY.md).

## Support

Open an issue here, or write to **guidedbyacademyone@gmail.com**.

## What is in this repository

This repository is GoldSeam's **public face**: what the services answer, how to connect, worked
examples and use cases. **It does not contain the server's source.**

| | |
|---|---|
| [`server.json`](server.json) | the manifest published to the [MCP Registry](https://registry.modelcontextprotocol.io) as `com.ksaworks/goldseam` |
| [`docs/`](docs/) | one page per area, how to read an answer, how to ask about coverage, use cases |
| [`examples/`](examples/) | recorded runs, each stamped with the build and publish that produced it |
| [`install/`](install/) | per-client connection steps, the `mcp.json` snippet, the skill |
| [`CHANGELOG.md`](CHANGELOG.md) | what changed for a person, at each release and each data publish |

Prose here is CC BY 4.0; snippets are MIT. See [LICENSE](LICENSE).
