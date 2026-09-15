# GOTCHAS.md: what the portal does not tell you

Things every fast builder knows and the documentation does not say. Each entry
has the same four parts so an agent can act on it: **symptom**, **cause**,
**what to do**, **example**. Point at the contract in `/openapi/index.json`
rather than retyping endpoints.

Status: v0 seed, 14 Sep 2026. The first entries come from Videri's internal
reference client and from the developer portal's own articles. The entries
under "From harvest 1" come from the written field notes of the SparkSustain
build (a desktop presence app, macOS and Windows, 22 Aug to 14 Sep 2026); the
entries under "From harvest 2" come from the written business feedback on the
menu-board family of builds (QSR, hotel, corporate, reseller, venue and media
network interfaces sharing one integration layer, Aug to Sep 2026). All were
reviewed by a curator (CORE-10259, CORE-10265). Entries marked
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

## From harvest 2 (menu-board family field notes)

## Some services want `access_token`, not `id_token` (unverified)

- **Symptom**: Canvas Status `/status/fetch_all` and Metrics `/metrics/fetch_all`
  answer 401 or 403 to a Bearer `id_token` that every other service accepts.
- **Cause**: reported by one build, which retries those two calls with the
  `access_token` and succeeds. Not reproduced by a curator yet.
- **What to do**: keep `id_token` as the default (see harvest 1). If a status
  or metrics call fails with 401/403 and the tenant header is right, try the
  `access_token` once and report the case so the auth matrix can be fixed.
- **Example**: `Authorization: Bearer <access_token>` on `/status/fetch_all` only.

## Reading a workspace does not mean you can publish to it

- **Symptom**: `GET` canvases in a child workspace works; `POST` a Publisher
  event with the same `x-group` returns 403.
- **Cause**: event creation is authorised at a different level than reads.
  Some builds only succeed with the root workspace or with tenant-level
  context.
- **What to do**: when a publish 403s, retry once with the workspace you are
  actually entitled to publish in (root, then tenant context), and only
  following an explicit 403. Do not widen scope by default in a multi-tenant
  product. Ask for the permission matrix (CORE-10302) rather than guessing.
- **Example**: 403 on child `x-group`, 200 on root `x-group`, same token.

## Single-event `POST` can 403 where `/events/batch` succeeds

- **Symptom**: creating one Publisher event is forbidden; the same credentials
  create it through the batch endpoint.
- **Cause**: unknown; observed and recorded in one build's source comments.
- **What to do**: create events through `POST .../events/batch` even for one
  event; read the created events back afterwards.
- **Example**: batch body with a one-element `events` array.

## A batch response is not proof the events exist

- **Symptom**: the batch call returns 200 and a later list does not show the
  events, or shows different targets.
- **Cause**: responses can be arrays or nested records under several keys, and
  processing can be delayed.
- **What to do**: persist the returned identifiers, then `GET .../events/{uuid}`
  or the target's event list and verify identity, content and window before
  retiring the events you are replacing.
- **Example**: create, read back, then delete the old ones, in that order.

## Library image URLs can 403; use the asset's canonical blob

- **Symptom**: two preview image URLs from a CMS list return 403 with a valid
  token.
- **Cause**: list payloads can carry URLs that are not the authoritative media
  reference.
- **What to do**: read the scoped asset detail and select its canonical
  original blob; use that URL for preview and native playback.
- **Example**: `GET /cms/api/v1/assets/{uuid}` then the original blob URL.

## `save_url` returns 403 for tenant users

- **Symptom**: the documented `save_url` demo command is forbidden whatever
  credentials you try.
- **Cause**: it is on the platform's privileged-command list (with the shell,
  clock, relay, server, package and firmware verbs), which ordinary tenant
  roles cannot call.
- **What to do**: do not build background preload on it. Use native
  scheduling and cached playback; if you need silent preload, raise it as a
  platform request.
- **Example**: 403 on `save_url`; `set_brightness:=N` on the same device works.

## CMS playlist lists stop at 100 unless you page

- **Symptom**: a library with more than 100 playlists shows exactly 100.
- **Cause**: the default read is one page; there is no "all".
- **What to do**: page with a stable ordering (`uuid:ASC` worked), 50 rows at
  a time, deduplicate by id, and keep the pages you already have if a later
  page fails. Flag results as partial when you stop early.
- **Example**: `size=50&sort=uuid:ASC&page=0..n`.

## Playlist durations go in the `assetlist`, in milliseconds

- **Symptom**: a playlist plays but every item shows for the default time.
- **Cause**: item order and duration live on
  `PATCH /cms/api/v1/playlists/{uuid}/assetlist`, not on playlist creation.
  Each entry needs `asset_type`, `childUuid` and `duration` in ms. A video's
  duration comes from its native metadata, not from an image dwell time.
- **What to do**: create the playlist, persist its UUID at once, then PATCH
  the ordered list.
- **Example**: `{ "asset_type": "image", "childUuid": "...", "duration": 10000 }`.

## An upload is not usable until processing finishes

- **Symptom**: an asset you just created is missing from a playlist or fails
  to render.
- **Cause**: `POST /cms/api/v1/assets` creates an upload record; bytes go to
  the returned signed destination; the asset is then processed.
- **What to do**: wait for the processed state before referencing the asset,
  verify its identity and tags, and reuse the existing asset after a delay
  rather than uploading a second copy.
- **Example**: poll the asset detail until it is ready, then build the playlist.

## Proof of play is an asynchronous query

- **Symptom**: the proof-of-play call returns "processing", not a report.
- **Cause**: you request a query (`/canvas/proof_of_play/{hardwareDeviceId}?start=&end=`)
  and poll it by `queryId` until the report is ready.
- **What to do**: treat it as evidence of what played, fetched later; it is
  not a synchronous "is the screen correct" answer, and it does not measure
  audience or attention.
- **Example**: request, receive `queryId`, poll `/{hardwareDeviceId}/{queryId}`.

## A screenshot is only evidence if it is fresh

- **Symptom**: the screenshot shows the right content; the screen does not.
- **Cause**: stored screenshots can be old. Only `Last-Modified` tells you.
- **What to do**: apply a freshness threshold (one build uses five minutes),
  and request a new capture with the `get_screenshot:=true` demo command when
  stale. Installation photos and thumbnails are not live evidence.
- **Example**: reject a capture whose `Last-Modified` is older than 5 minutes.

## Five identifiers, five APIs

- **Symptom**: joins between inventory, status, metrics and publisher silently
  mismatch.
- **Cause**: canvas id (numeric), device JID, hardware serial, player id and
  workspace (group) id serve different services and are not interchangeable.
  Matching everything by one field produced fragile joins.
- **What to do**: keep all of them on your canvas record; join each service on
  the field it actually uses.
- **Example**: status by player identity, metrics by hardware id, publisher
  by numeric canvas id.

## Walls are native topology, not a label

- **Symptom**: four screens you named after venues are still one wall, and a
  wall event targets all of them.
- **Cause**: wall membership and geometry live in Canvas Service
  (`POST /walls` with an `ids` array); venue or workspace names do not change
  it.
- **What to do**: resolve walls explicitly from inventory, never infer
  topology from names, and keep wall membership visible even when no wall
  event is active. Wall content needs verified geometry and member order.
- **Example**: `POST /canvas-service/walls?page=0&size=500` with `{ "ids": [...] }`.

## Never add 24 hours to make "tomorrow"

- **Symptom**: an overnight or multi-day schedule is off by an hour twice a year.
- **Cause**: daylight-saving changes. Recurrence uses `MON`..`SUN` with hour
  parts and `startTime` / `endTime`; the user's inclusive last day becomes an
  exclusive next-midnight boundary in your interval maths.
- **What to do**: do all date arithmetic in the estate's IANA timezone with a
  proper library; move the date and the weekday together for overnight windows.
- **Example**: Europe/London, last Sunday in October.

## Layout schemas disagree with each other

- **Symptom**: `components` is an object in one CMS layout schema and a
  nullable array in another; the PATCH prose says fields are optional while
  `EditLayoutDTO` requires `name` and `orientation`.
- **Cause**: the published layout DTOs are inconsistent and do not define
  component internals (types, transforms, fonts, groups, media, widgets).
- **What to do**: work from a native example layout read back from the API,
  not from the schema alone. Browser fonts, animation and crop behaviour do
  not automatically survive native conversion; a saved layout may reference a
  `referAsset` of type layout with no media blob. CORE-10291 tracks the fix.
- **Example**: read an existing layout with `GET /cms/api/v1/layouts/{uuid}`
  and copy its component shape.

## "Published" is not "on the screen"

- **Symptom**: schedule accepted, content downloaded, device online, and the
  screen shows something else.
- **Cause**: scheduling success, delivery and playback are separate states
  with separate evidence.
- **What to do**: verify delivery separately (fresh screenshot, proof of play,
  event readback) and keep your UI language honest: "scheduled", not
  "playing", until you have evidence.
- **Example**: a rolled-back job must not still say "switching".

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
