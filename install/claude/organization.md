# Claude Team or Enterprise — add GoldSeam once, for everyone

**Only an Owner can do this**, and it is the difference between every member configuring a connector
by hand and nobody having to.

1. Sign in as an **Owner** of the organization.
2. Open the organization's **settings → Connectors**.
3. **Add a connector** with `https://goldseam.ksaworks.com/mcp`, authentication **none**.
4. Members then switch **GoldSeam** on per chat from **+ → Connectors**.

## What does not work, and why it is worth knowing

- **A plugin or a skill does not attach a connector.** Tested 2026-09-23: a member-created plugin carrying `mcp.json` installed its skill, and the chat held no GoldSeam tools. A skill teaches an AI how to ask; only a connector grants access.
- **A member cannot add an organization connector.** If you are a member rather than an Owner, ask your Owner to do step 3 — it takes a minute and it is the whole ask.
- **Seats are not distribution.** An organization connector serves the people already in that organization. Everyone else — students, counselors, researchers, the public — uses the same address on their own account, or the browser at [KSAWorks](https://ksaworks.com/where-do-you-want-to-go.html). Nobody needs a seat to use GoldSeam.
