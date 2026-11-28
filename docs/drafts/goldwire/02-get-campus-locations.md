# Call 2 — Get campus locations

[← Call 1 — Get academic terms](01-get-academic-terms.md) · [Summary](README.md) · Next: [Call 3 — Get sections →](03-get-sections.md)

> **Draft.** Nothing here is live.

## What it does

Lists a school's **campuses and teaching locations** — including a virtual campus for online classes
if the school uses one — with their codes, addresses and buildings. It gives the `campus_id` used to
narrow [Get sections](03-get-sections.md).

It needs no learner. It changes nothing.

## Where it sits

- **Before:** the school is known (`unitid 225070`).
- **This call:** "Where does this school teach?"
- **After:** pick a campus → [Call 3 — Get sections](03-get-sections.md) with `campus_id`.

---

## The request

```http
GET /goldwire/v1/schools/225070/campuses?include_buildings=true HTTP/1.1
Host: api.goldseam.example
X-Api-Key: gsk_live_4f7b2c9e1a8d
X-Request-Id: 2b3c4d5e-6f7a-4b8c-9d0e-1f2a3b4c5d6e
Accept: application/json
```

## Every field you send

| Field | This example | Required? | What it means | Format and allowed values |
|---|---|---|---|---|
| `unitid` (in the path) | `225070` | yes | the school | 6 digits |
| `X-Api-Key` (header) | `gsk_live_4f7b2c9e1a8d` | yes | identifies the app | issued to each app |
| `campus_id` | *(not sent)* | no | return just this campus | the school's campus code, e.g. `MAIN` |
| `include_buildings` | `true` | no (default `false`) | also list each campus's buildings | `true` / `false` |

---

## The reply — `200 OK`

```json
{
  "contract": "goldwire_v1",
  "request_id": "2b3c4d5e-6f7a-4b8c-9d0e-1f2a3b4c5d6e",
  "status": "ok",
  "as_of": "2026-09-26T06:00:00Z",
  "source": { "kind": "school_sis_feed", "unitid": "225070", "feed": "banner", "read_at": "2026-09-26T06:00:00Z" },
  "not_held": [],
  "unitid": "225070",
  "campuses": [
    {
      "campus_id": "MAIN",
      "name": "Main Campus",
      "type": "physical",
      "address": { "line1": "100 College Way", "city": "Example City", "state": "TX", "postal_code": "75000", "country": "US" },
      "geo": { "lat": 33.1000, "lon": -97.2000 },
      "timezone": "America/Chicago",
      "buildings": [
        { "building_id": "LA",  "name": "Liberal Arts Building" },
        { "building_id": "SCI", "name": "Science Center" },
        { "building_id": "FA",  "name": "Fine Arts Center" }
      ],
      "native": { "campus_code": "M", "name": "Main Campus" }
    },
    {
      "campus_id": "SOUTH",
      "name": "South Campus",
      "type": "physical",
      "address": { "line1": "2500 South Parkway", "city": "Example City", "state": "TX", "postal_code": "75002", "country": "US" },
      "geo": { "lat": 33.0500, "lon": -97.2100 },
      "timezone": "America/Chicago",
      "buildings": [
        { "building_id": "S", "name": "South Campus Classroom Building" }
      ],
      "native": { "campus_code": "S", "name": "South Campus" }
    },
    {
      "campus_id": "WEB",
      "name": "Online",
      "type": "virtual",
      "address": null,
      "geo": null,
      "timezone": "America/Chicago",
      "buildings": [],
      "native": { "campus_code": "W", "name": "Web Campus" }
    }
  ]
}
```

In words: two physical campuses and one online "campus", `WEB`.

## Every field you get back

| Field | This example | What it means |
|---|---|---|
| `contract` / `request_id` / `as_of` | | rule version; tracking number; read once a day |
| `status` | `ok` | or `not_held` if GoldSeam holds no campus list for the school |
| `source` | `school_sis_feed`, `banner` | from the school's system |
| `not_held` | `[]` | e.g. `{ "what": "campus X", "note": "no mapping configured" }` |
| `campuses[].campus_id` | `MAIN` | **GoldWire's code — send it to Call 3** |
| `campuses[].name` | `Main Campus` | for display |
| `campuses[].type` | `physical` | `physical` or `virtual` (online) |
| `campuses[].address` | 100 College Way… | street address; `null` for virtual |
| `campuses[].geo` | lat / lon | for maps and distance; `null` for virtual |
| `campuses[].timezone` | `America/Chicago` | class times at this campus are in this zone |
| `campuses[].buildings[]` | `LA` — Liberal Arts Building | only with `include_buildings=true`; `building_id` matches `meetings[].building_id` in Call 3 |
| `campuses[].native.campus_code` | `M` | the school's own code — for staff only |

---

## Other replies you can get (not errors)

**No campus list held** — `200`, `status: "not_held"`, `campuses: []`.

---

## Errors

Example:

```json
{
  "contract": "goldwire_v1",
  "request_id": "2b3c4d5e-6f7a-4b8c-9d0e-1f2a3b4c5d6e",
  "status": "error",
  "error": {
    "code": "CAMPUS_NOT_FOUND",
    "message": "School 225070 has no campus NORTH. Its campuses are MAIN, SOUTH, WEB.",
    "retry": "fix_request",
    "field": "campus_id",
    "details": { "campus_id": "NORTH", "campuses": ["MAIN", "SOUTH", "WEB"] }
  }
}
```

| HTTP | Code | The message, as it appears | Happens when | Do this |
|---|---|---|---|---|
| 400 | `VALIDATION_FAILED` | `unitid must be exactly 6 digits; got '22507'.` | | fix it |
| 400 | `VALIDATION_FAILED` | `campus_id must be 1–10 letters, digits or _; got 'Main Campus'.` | a name sent | send `MAIN` |
| 400 | `VALIDATION_FAILED` | `include_buildings must be true or false; got 'yes'.` | | `true` |
| 401 | `CLIENT_UNAUTHENTICATED` | `An API key is required. Send X-Api-Key: <key>.` | | use the app's key |
| 404 | `SCHOOL_NOT_FOUND` | `No institution with UNITID 999999 is held. Find the school with CourseShelf find_institution.` | | look it up |
| 404 | `CAMPUS_NOT_FOUND` | `School 225070 has no campus NORTH. Its campuses are MAIN, SOUTH, WEB.` | | pick one listed |
| 429 | `RATE_LIMITED` | `Too many requests. Try again after 12 seconds.` | | wait |
| 500 | `INTERNAL_ERROR` | `GoldWire could not complete the request. Nothing was changed. Reference 2b3c4d5e-6f7a-4b8c-9d0e-1f2a3b4c5d6e.` | | try again |

## Behind the call

Adapter operation **A2 `list_campuses`**. Banner campus code `M` → `MAIN` through the school's
campus mapping. **Nothing is recorded in S3.**
