# Security and what this service keeps

## Reporting

Write to **guidedbyacademyone@gmail.com**, or open an issue if it is not sensitive. We respond to
security reports promptly and will tell you what we changed.

## What GoldSeam keeps

**The usage record holds counts only.** Per call: a timestamp, the tool name, a derived caller
identifier, whether the answer was a hit or a miss, and the result count. Raw entries are deleted
after 90 days; anything kept beyond that carries no caller-linked detail.

**Raw IP addresses are never stored.** An anonymous caller is identified by a one-way hash of the
connecting address.

**Nothing a person types is kept.** For the services that accept a person's own documents —
transcripts, a resume, certifications, military records, life experiences — the evidence is
**caller-supplied and retained by nothing**: your AI holds the package, GoldSeam returns the answer,
and its usage record logs counts alone, never the content.

**No personal records are stored, and there is no learner account.** The service serves the same
answer to every caller.

The full statement, including how information is used and shared:
[ksaworks.com/privacy.html](https://ksaworks.com/privacy.html).

## Connecting

One public address, `https://goldseam.ksaworks.com/mcp`, over HTTPS with streamable HTTP. **No
authentication, no key, no account.** Every tool reads public data; none writes to a person's record
anywhere.

If you are offered a GoldSeam address that is not the one above, it is not ours.
