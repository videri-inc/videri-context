# GOTCHAS.md: what the portal does not tell you

Things every fast builder knows and the documentation does not say. Each entry
has the same four parts so an agent can act on it: **symptom**, **cause**,
**what to do**, **example**. Point at the contract in `/openapi/index.json`
rather than retyping endpoints.

Status: v0 seed, 14 Sep 2026. The first entries come from Videri's internal
reference client and from the developer portal's own articles. The entries
under "From harvest 1" come from the written field notes of the SparkSustain
build (a desktop presence app, macOS and Windows, 22 Aug to 14 Sep 2026) and
were reviewed by a curator (CORE-10259, CORE-10265). Entries marked
**unverified** were reported, not reproduced; verify with a probe call before
relying on them.
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
- **What to do**: for canvases, pass the workspace as a query parameter; keep
  sending `x-group` where the spec asks for it. The reference client uses
  `group_id`; the SparkSustain build used `group_uuid`, which accepts a
  comma-separated list and matches **direct** membership only, so a workspace
  with sub-workspaces has to be expanded client-side into all its descendants
  (see "The workspace tree field is `descendants`"). Check the spec for the
  parameter name your version exposes; do not assume one filter covers the
  subtree.
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

## From harvest 1 (SparkSustain field notes)

## Use `id_token` as the Bearer, not `access_token`

- **Symptom**: `401 jwt signature verification failed: 'tenants' claim is required`
  on the first call after a successful token exchange.
- **Cause**: the token endpoint returns `access_token`, `id_token` and
  `refresh_token`. Downstream services validate the `id_token`; the error
  describes a missing claim instead of telling you that you picked the wrong
  token.
- **What to do**: send `Authorization: Bearer <id_token>`. Cache it with a
  five-minute margin against `expires_in` and re-authenticate on any 401;
  a client that holds the credentials does not need `refresh_token`.
- **Example**: `expires_in: 3600` means refresh at 55 minutes.

## The `tenants` claim is a JSON string inside the JWT

- **Symptom**: `tenants` decodes to a string like `"[\"VIDERISALES\",\"TRADESHOW\"]"`
  and iterating it yields characters.
- **Cause**: the claim is a JSON-encoded string, not a JSON array.
- **What to do**: decode the JWT payload, then `JSON.parse` the `tenants`
  value a second time.
- **Example**: two decodes, then `["VIDERISALES","TRADESHOW"]`.

## `x-tenant` takes exactly one tenant code

- **Symptom**: `403 No access to ["VIDERISALES","GENESCO"] tenant`.
- **Cause**: you passed the whole `tenants` array; the header takes one code.
- **What to do**: for a multi-tenant account, one pass per tenant, results
  merged client-side. A missing header fails less helpfully:
  `403 {"code":403,"message":"Access is denied"}` with no hint about the
  header (see `AUTH.md` errors table).
- **Example**: `x-tenant: VIDERISALES`.

## Published `servers` blocks do not all match the live paths

- **Symptom**: `GET /rpm-service/v1/users/me/groups_access` returns 404 with
  no body; the same path under `/rpm/v1/` returns 200.
- **Cause**: some specs carry a `servers` block whose prefix differs from the
  path the API actually serves.
- **What to do**: when a spec path 404s with an empty body, try the prefix used
  by the other endpoints of that service in the portal's examples; report the
  mismatch so the spec gets fixed (CORE-10278 fixes the host, CORE-10291 the
  prefix).
- **Example**: `/rpm/v1/users/me/groups_access` works; `/rpm-service/v1/...` does not.

## The workspace tree field is `descendants`, not `children`

- **Symptom**: your code sees a flat one-level tree and concludes the tenant
  has no sub-workspaces.
- **Cause**: `GET .../users/me/groups_access` returns `{ groupAccess: [...] }`
  with nested workspaces under `descendants`. The published schema calls the
  field `children`, so code generated from the spec reads the wrong key and
  fails silently.
- **What to do**: read `descendants`; walk it recursively to expand a checked
  workspace into every workspace below it.
- **Example**: `node.descendants ?? node.children ?? []`.

## `assigned_to_group=true` and `=false` are disjoint sets

- **Symptom**: canvases you know exist are missing from the list.
- **Cause**: the two values return disjoint sets, and there is no "all".
- **What to do**: when you need every canvas, call both and merge.
- **Example**: `size=1000`, two calls, concatenate `content`.

## `batch_command` accepts only `demo_command`

- **Symptom**: `400 Only 'demo_command' is supported for batch_command` after
  building a whole multi-device layer on it.
- **Cause**: the endpoint schema lists about 25 `command_name` values; the
  service accepts exactly one of them.
- **What to do**: send `ops_*` commands through `sync_command`, one device per
  call, fanned out concurrently (five devices settle in about a second).
  Ticket CORE-10291 tracks the schema fix.
- **Example**: five parallel `sync_command` calls instead of one `batch_command`.

## `sync_command` always returns HTTP 200

- **Symptom**: a command "succeeded" and nothing happened on the device.
- **Cause**: the HTTP status is 200 whatever the outcome. The real result is
  the `response_code` string (`SUCCESS`, `TIME_OUT`, `DEVICE_OFFLINE`,
  `FAILED`, `INVALID_COMMAND`), sometimes `ERROR` with the detail at
  `others.message_json.error_msg`.
- **What to do**: branch on `response_code`, never on the HTTP status.
- **Example**: `response_code: "DEVICE_OFFLINE"` inside a 200.

## Setting `brightness` in `ops_set_settings` does not move the backlight

- **Symptom**: `ops_set_settings` returns `SUCCESS`, the value reads back, and
  the screen does not change.
- **Cause**: `brightness` in settings is the schedule setpoint and may be inert
  while the device-side schedule is active. The live backlight is driven by the
  `demo_command` string `set_brightness:=N`. Nothing in the response
  distinguishes "stored" from "applied".
- **What to do**: to change what the panel shows now, send `demo_command`
  with `set_brightness:=N`; to hold a value, also force
  `brightness_schedule_enabled: false` in settings. Verify against
  `current_brightness`, not `brightness`. Optional per-display targeting is
  `set_brightness:=N|[0,1]` (**unverified**).
- **Example**: 70 percent is `set_brightness:=179`.

## `ops_set_settings` params need the `system_properties` wrapper

- **Symptom**: HTTP 200 with `error_msg: "No value for system_properties"`.
- **Cause**: settings commands expect `command_params: { system_properties: {...} }`.
- **What to do**: wrap every settings payload.
- **Example**: `{ "system_properties": { "display_on": true, "brightness_schedule_enabled": false } }`.

## True "off" is `display_on: false`, not brightness 0

- **Symptom**: brightness 0 leaves a visible backlight floor on some models.
- **Cause**: zero brightness is not the same as the display being off.
- **What to do**: use `display_on: false` in `ops_set_settings` to turn a panel
  off, and `display_on: true` plus a `set_brightness` command to bring it back.
- **Example**: sleep = `display_on: false`; wake = `display_on: true` then `set_brightness:=179`.

## Device commands need three identifiers, and `player_id` is the numeric id

- **Symptom**: a command is refused or targets nothing.
- **Cause**: `sync_command` wants `device_id`, `device_jid` and `player_id`
  together. `player_id` is the canvas's numeric `id` (the same value Publisher
  calls `canvasIds`), not the `device_id` and not the JID. `device_jid` is the
  `xmpp_jid` field of the `/canvases` response and is required alongside
  `device_id`, not instead of it.
- **What to do**: keep the triplet from the canvas list and pass all three.
- **Example**: `{ "device_id": "...", "device_jid": "<xmpp_jid>", "player_id": 1027421 }`.

## Two payload shapes are documented for commands; the flat one works

- **Symptom**: a 400 from a command built from the Knowledge Base example.
- **Cause**: the Knowledge Base shows `{ targets: [...], command: { name, uuid, params } }`;
  the contract shows a flat `{ command_name, command_params, device_id, device_jid, player_id }`.
  The API accepts the flat shape.
- **What to do**: follow the contract in `/openapi/index.json`, not the article.
- **Example**: see the entry above.

## Settings times are `HHMM` strings and the timezone is per device

- **Symptom**: `turn_on_time: 900` is rejected or misread.
- **Cause**: `turn_on_time` / `turn_off_time` are strings like `"0900"`;
  the timezone is an IANA name stored per device.
- **What to do**: format as four digits; read the device's timezone before
  reasoning about local time. `ops_get_settings` returns the entire IANA
  timezone list inline in `available_timezones` on every call, so do not log
  the raw response.
- **Example**: `"turn_on_time": "0900"`.

## Deleting schedule events is permanent and returns counts

- **Symptom**: no undo, no export; the response is
  `{ numberOfSuccessfulEventsDeletion, numberOfFailedEventsDeletion }`.
- **Cause**: Publisher's delete of canvas settings events is a live, bulk,
  irreversible removal, keyed by numeric `canvasIds`.
- **What to do**: read the events first if you need to restore them; warn the
  person that a takeover is permanent. The count in the response tells you how
  much you destroyed; check it. **Unverified** against an estate that has
  events (the harvest build only ever saw `0` deletions).
- **Example**: `DELETE .../canvas_settings_events` with `{ "canvasIds": [1027421] }`.

## The platform pushes nothing; everything is polling

- **Symptom**: no webhook, no event stream for a canvas going offline or
  content changing.
- **Cause**: there is no push channel for device state in the public API.
- **What to do**: poll on a timer and re-assert your intended state
  periodically (the harvest build re-reads settings and re-asserts every ten
  minutes so a rebooted canvas or an out-of-band portal change is pulled back
  in line). No rate limit is documented; 5 to 10 concurrent `sync_command`
  calls have been fine, larger fleets are a guess.
- **Example**: ten-minute reconcile loop.

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
