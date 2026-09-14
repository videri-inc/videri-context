# GOTCHAS.md: what the portal does not tell you

Things every fast builder knows and the documentation does not say. Each entry
has the same four parts so an agent can act on it: **symptom**, **cause**,
**what to do**, **example**. Point at the contract in `/openapi/index.json`
rather than retyping endpoints.

Status: v0 seed, 14 Sep 2026. The entries below come from Videri's internal
reference client and from the developer portal's own articles. Entries from
harvest sessions with the fastest builders follow (CORE-10259, CORE-10265).
This file is curated by the owners listed in `CODEOWNERS`; if you hit a gap,
open an issue using the template at the bottom and a curator will add it.

---

## Pagination differs by service

- **Symptom**: your list helper works for canvases and returns nothing, or
  the wrong count, for assets or playlists.
- **Cause**: services paginate differently. Canvas Service returns `content`
  plus `totalElements` / `totalPages` / `size` / `number` (zero-based `page`,
  `size` up to 1000). CMS endpoints return `data` plus `meta.totalItems`. The
  V2 assets listing paginates with `page` and `limit`.
- **What to do**: one adapter per service behind a common interface (the
  starter kit ships these). Never assume the shape; read it from the spec.
- **Example**: `GET /canvas-service/canvases?...&page=0&size=10` yields
  `{ content: [...], totalElements: 42, ... }`; a CMS list yields
  `{ data: [...], meta: { totalItems: 42 } }`.

## Field casing is inconsistent

- **Symptom**: `presence_status` is undefined on some canvases and
  `presenceStatus` on others.
- **Cause**: canvas payloads can arrive in snake_case or camelCase depending
  on the endpoint (`presence_status` / `presenceStatus`, `device_id` /
  `deviceId`, `group_name` / `groupName`, or a nested `group.name`).
- **What to do**: read both spellings, in a single accessor, once.
- **Example**: `status = c.presence_status ?? c.presenceStatus`.

## Count from totals, never from array length

- **Symptom**: "You have 10 canvases" when the tenant has 400.
- **Cause**: arrays are one page, or a capped subset.
- **What to do**: read `totalElements`, `total` or `meta.totalItems` and page
  through when you need the full set.
- **Example**: report `totalElements`, not `content.length`.

## `assigned_to_group` is required on the canvas list

- **Symptom**: `400 Missing required query parameter: assigned_to_group`.
- **Cause**: the canvas list needs to know whether you want canvases assigned
  to a workspace (`true`) or not yet assigned (`false`).
- **What to do**: always pass it. `true` is what almost every UI wants.
- **Example**: `/canvas-service/canvases?assigned_to_group=true&with_status=true`.

## Canvas Service scopes by `group_id`, not by `x-group`

- **Symptom**: you send `x-group` and still get every workspace's canvases.
- **Cause**: Canvas Service does not read the `x-group` header; it takes the
  workspace as the `group_id` query parameter. Other services (for example
  Publisher writes) do require `x-group`.
- **What to do**: for canvases, pass `group_id=<workspace uuid>`; keep sending
  `x-group` where the spec asks for it.
- **Example**: `GET /canvas-service/canvases?assigned_to_group=true&group_id=9f1c...`.

## Brightness is 0 to 255 on the device

- **Symptom**: setting brightness to `80` makes the screen nearly black.
- **Cause**: the device range is 0 to 255; users think in 0 to 100 percent.
- **What to do**: convert at the edge: `round(pct / 100 * 255)` on the way in,
  the inverse on the way out.
- **Example**: 80 percent is `204`.

## Durations are milliseconds

- **Symptom**: a "10 second" playlist item flashes by, or filters return
  nothing.
- **Cause**: durations in the API are milliseconds; people say seconds.
- **What to do**: multiply by 1000 on input, divide on display. Filter
  operators on durations are `$lt`, `$gt`, `$btw`.
- **Example**: 10 s is `10000`.

## Event priority is 100 / 200 / 300

- **Symptom**: a "priority 1" event never plays.
- **Cause**: priorities are the values 100, 200 and 300, not a free integer.
- **What to do**: offer exactly those three; read the semantics from the
  Publisher contract.
- **Example**: default scheduling at `100`.

## A stale or foreign token breaks even a spec fetch

- **Symptom**: `401 RSA key with id ... not found` or `401 invalid jwt` when
  downloading an OpenAPI spec.
- **Cause**: spec URLs are public, but any `Authorization` header that is
  present is validated. A token from another environment, or an expired one,
  fails before the spec is served.
- **What to do**: fetch specs with no `Authorization` header at all.
- **Example**: `curl https://api.sandbox.videri.com/canvas-service/v3/api-docs`
  with no headers.

## Key type does not limit reach

- **Symptom**: a `sk_test_` key reads a production-tenant's data.
- **Cause**: what a key can reach is decided by the account's tenants and
  roles, not by the key type. Test and production keys differ in prefix and in
  how you manage them, not in permissions.
- **What to do**: build against the sandbox environment with a sandbox account.
  Treat every key as if it were production.
- **Example**: none needed; this is a rule, not a trick.

## Scheduling and deleting change real screens

- **Symptom**: a demo event replaces what a store is playing.
- **Cause**: `create event` is a live scheduling change; deletes are live
  removals.
- **What to do**: list the target canvases first and confirm with the person
  before creating an event. Confirm before deleting a playlist or event or
  removing an asset from a playlist. Decide explicitly whether device commands
  (brightness, volume, reboot, overlay) need a confirmation in your UI; the
  reference client runs them without one.
- **Example**: "This will schedule *Lunch specials* on 6 canvases in
  *Lobby* from 11:00 to 14:00 daily. Proceed?"

## Error bodies are short; log the status

- **Symptom**: an error message is cut off.
- **Cause**: the reference client caps error bodies to keep them readable.
- **What to do**: always log and surface the HTTP status code alongside the
  message; the status is what the errors table in `AUTH.md` keys on.
- **Example**: `403 No access to ACME01 tenant.`

---

## Template for a new entry

```
## <One-line title, the way a builder would say it>

- **Symptom**: what you saw.
- **Cause**: why it happens, in one or two sentences.
- **What to do**: the fix or the habit. Point at the spec; do not retype endpoints.
- **Example**: one concrete request, value or message.
```

Open an issue with the entry filled in; a curator turns it into a pull request.
If the gap is in the portal itself (an article or the Start here page), say so
so a portal ticket can be opened as well.
