# DECISIONS.md: why the starter does what it does

Short records of choices made while building on the platform, so the next
builder inherits the reasoning and not just the code. One entry per decision.
Harvest sessions with experienced builders feed this file (CORE-10259).

Status: v0, 14 Sep 2026. First entries come from the written field notes of the
SparkSustain build (harvest 1, CORE-10259). More follow from the other
working-group answers and the 16 Sep call.

---

## Template

```
## <Decision, stated as what we do>

- **Date / source**: harvest session N, or the build it came from.
- **Context**: what forced the choice.
- **Decision**: what we do.
- **Why**: the reason, including what we tried or rejected.
- **Consequences**: what this makes easier and what it makes harder.
- **Revisit when**: the condition under which this should be reconsidered.
```

---

## Entries from harvest 1 (SparkSustain, desktop presence app)

## The client calls the API directly; there is no middle layer

- **Date / source**: harvest 1, SparkSustain build.
- **Context**: a desktop app that drives brightness on the canvases around one
  person's desk from that person's laptop.
- **Decision**: the app authenticates as the user and calls the public API over
  HTTPS. No proxy, no worker, no backend of ours in the flow. State lives in a
  local config file (mode 0600) and on the devices themselves.
- **Why**: the API accepts the individual's own credentials and enforces scope
  per user, so a proxy would add nothing except a place to leak a shared secret.
- **Consequences**: every user needs their own API key. Fine for an internal
  tool. Wrong for a customer-facing product, where a service account behind a
  proxy keeps the estate from being pinned to one person's account.
- **Revisit when**: the build is offered to customers, or a scoped-key model
  exists (see CORE-10271 hosting guidance).

## `id_token` is the Bearer; re-authenticate instead of refreshing

- **Date / source**: harvest 1.
- **Context**: the token endpoint returns three tokens; only one works
  downstream and the error does not say which.
- **Decision**: send `id_token`. Cache it with a five-minute margin against
  `expires_in`; invalidate on any 401 so the next call re-authenticates.
  `refresh_token` is not used.
- **Why**: a client that already holds the credentials gains nothing from a
  refresh flow and has one less secret to store.
- **Consequences**: simple; one more credential round-trip per hour.
- **Revisit when**: the client no longer holds the password (browser or
  customer-facing builds).

## Re-assert intended device state on a timer; never trust a 200

- **Date / source**: harvest 1.
- **Context**: platform brightness schedules overrode the app; state drifted
  after reboots; `sync_command` returns HTTP 200 whatever happened; setting
  `brightness` acknowledged SUCCESS and did nothing.
- **Decision**: on every state change and every ten minutes, re-read device
  settings, force `brightness_schedule_enabled: false`, and re-apply the
  intended `display_on` and `set_brightness`. Branch on `response_code` and
  verify against `current_brightness`.
- **Why**: there is no push channel and no schedule "suspend"; polling and
  reconciliation is the only way to hold a state.
- **Consequences**: robust to reboots and out-of-band portal edits; costs a
  few calls per device per ten minutes; ad-hoc manual commands need explicit
  hold semantics or the loop reverts them within seconds.
- **Revisit when**: the platform offers webhooks or a schedule pause flag.

## Take over schedules by deleting them, with a warning

- **Date / source**: harvest 1.
- **Context**: device-side and publisher-side schedules fight any live
  brightness command.
- **Decision**: at setup, delete publisher schedule events for the managed
  canvases (after warning the user that this is permanent) and keep the
  device-side schedule flag forced off.
- **Why**: no suspend flag exists; deletion is the only takeover available.
- **Consequences**: destructive and irreversible; the delete path is confirmed
  reachable but has only been exercised against canvases with zero events.
- **Revisit when**: a "suspend schedules" flag or read-then-restore pattern
  exists.

## Probe first, then read: ship a probe script before framework code

- **Date / source**: harvest 1, "what I would change".
- **Context**: every real discovery came from a short script that
  authenticated with real credentials and hit one endpoint; none came from
  reading documentation.
- **Decision**: the starter kit ships `videri-probe` (CORE-10294) and the
  Start here page puts it before any framework step.
- **Why**: it moves failures from "a human notices weeks later" to "the first
  test call tells you".
- **Consequences**: the probe is the first thing to maintain when the API
  changes.
- **Revisit when**: contracts and articles are generated from one source and
  stop drifting.

## One core, thin shells (recommendation, not yet applied)

- **Date / source**: harvest 1, "what I would change".
- **Context**: SparkSustain implemented every feature twice (Swift and C#),
  and the second implementation is where the bugs lived.
- **Decision**: for multi-platform builds, put auth, API client and state in
  one shared core and keep platform UIs thin.
- **Why**: would have halved the work from about v1.2 onward.
- **Consequences**: the starter kit's client layer should be the reusable
  core; native shells stay out of scope for the kit.
- **Revisit when**: a second multi-platform recipe appears.

## Ship source, compile on the user's machine (Windows internal tools)

- **Date / source**: harvest 1.
- **Context**: distributing an internal tool to non-developers on Windows.
- **Decision**: the release zip ships C# source and the installer compiles it
  with the compiler already present in Windows.
- **Why**: no downloads, no admin rights, no toolchain, no code-signing
  problem; when a build fails on a colleague's machine the exact compiler
  error is on screen.
- **Consequences**: not appropriate for customer distribution; macOS still
  needs real code signing (a Developer ID certificate) to avoid the
  right-click-to-open step.
- **Revisit when**: a build is offered outside Videri.

## Updates compare for difference, not for newer

- **Date / source**: harvest 1.
- **Context**: a fleet of testers needed a way to receive fixes.
- **Decision**: apps poll a manifest on the repo's main branch every four
  hours, compare the referenced version with their own, and install the
  release asset after verifying its SHA-256. Difference, not newer.
- **Why**: repointing one file rolls the whole fleet backward as easily as
  forward.
- **Consequences**: one file to edit for a rollback; the manifest URL is
  estate-specific and must be replaced in any starter derived from it.
- **Revisit when**: a signed update channel is available.

---

## Entries from harvest 2 (menu-board family)

## Customer-facing builds put an integration layer between browser and API

- **Date / source**: harvest 2, menu-board family.
- **Context**: interfaces for operators (QSR, hotel, reseller, media network)
  where the end user must never hold Videri credentials.
- **Decision**: browser -> application server -> operations worker -> Videri
  services. The worker holds the service credentials, normalises data,
  enforces scope, runs asynchronous publishing jobs and does readback and
  recovery. It is an integration layer, not a transparent proxy.
- **Why**: keeps credentials out of the browser, keeps working after the user
  closes the page, and gives one place for compensation logic.
- **Consequences**: a substantial shared codebase that several apps depend on;
  regressions and deployment coordination grow with each app. Contrast with
  harvest 1, where an internal single-user tool calls the API directly with
  the user's own key: both are right for their audience (CORE-10271).
- **Revisit when**: the platform offers scoped service keys or a hosted
  publishing job API.

## One typed client, separate from sector UI

- **Date / source**: harvest 2, "what I would change".
- **Context**: multiple build paths (Next.js, Vite, esbuild) and a shared
  worker accumulated across eight interfaces.
- **Decision**: one frontend/build approach, one shared typed Videri client,
  and the integration package kept apart from sector-specific UI. Tenant,
  workspace, screen, storage and host configuration explicit from the first
  build. UI, worker, migrations and player versioned together in a release
  manifest.
- **Why**: reuse was quick at first and expensive later.
- **Consequences**: this is the shape of the starter kit (CORE-10267): client
  and adapters in one package, recipes as thin consumers.
- **Revisit when**: a recipe needs a different runtime.

## Verify one end-to-end flow before building the editor

- **Date / source**: harvest 2, "the biggest simplification".
- **Context**: menu editor, wall orchestration and enterprise workflows were
  built before one verified upload -> playlist -> event -> evidence ->
  rollback path existed.
- **Decision**: the first deliverable of any build is that single verified
  flow on one screen; everything else is layered on it.
- **Why**: every later problem (auth context, publishing contract, delivery
  evidence) would have surfaced in that flow first.
- **Consequences**: the menu-board recipe (CORE-10268) is written as that flow
  first, with the editor as a later section.
- **Revisit when**: never; this is the same lesson as harvest 1's "probe first".

## Publishing is a durable job with compensation, not a request

- **Date / source**: harvest 2.
- **Context**: CMS, Publisher and the application database do not share a
  transaction; responses can be delayed or ambiguous.
- **Decision**: publishing runs as durable job steps: capture target and
  existing schedule, ensure asset, create playlist, patch assetlist, read
  and retain the events being replaced, create new events via batch, read
  back and verify, then retire the old events, then check delivery. A timeout
  never creates a second job. Compensation removes new events and restores
  prior ones.
- **Why**: the only way to offer rollback and avoid duplicate schedules.
- **Consequences**: no atomic visual switch across displays can be promised;
  the UI must say so. Verified on 12 Sep (five events created and rolled back
  exactly) and 14 Sep (owner -> approval -> native schedule -> fresh capture
  -> rollback on one screen).
- **Revisit when**: Publisher offers transactional or idempotent batch
  creation.

## Delivery evidence is a separate concern from scheduling success

- **Date / source**: harvest 2.
- **Context**: accepted schedule, downloaded content, online status and a
  stale screenshot told different stories.
- **Decision**: treat delivery as its own lifecycle with its own evidence
  (fresh screenshot with a credible Last-Modified, proof-of-play report,
  event readback) and its own UI states.
- **Why**: false "it is live" conclusions cost more debugging time than any
  other class of problem after auth.
- **Consequences**: the starter's "one live render" (CORE-10267) must include
  an evidence step, not just a 200.
- **Revisit when**: the platform publishes a delivery-state API.

## Operational defaults are configuration, not assumptions

- **Date / source**: harvest 2, "what must change for a generic starter".
- **Context**: timezone, currency, dayparts, priorities, duration limits,
  replacement defaults, target-count limits and retention were baked in as
  one customer's conventions.
- **Decision**: the starter ships example configuration, schema and seed
  scripts, a tenant setup check and a fixture estate with no production
  identifiers; a new logo and API key are not enough.
- **Why**: several services had the same mapping hard-coded, so changing one
  value was never sufficient.
- **Consequences**: CORE-10267 acceptance includes a config-only bring-up on
  a second tenant.
- **Revisit when**: never.

---

## Decisions still to capture (16 Sep call)

The five questions originally listed here (stack, hosting, data source, the
role of the worker, what is estate-specific) are answered by the harvest 2
entries above. Still open for the call:

- Which of the two hosting models (direct client with the user's key, or an
  integration layer holding a service credential) the starter kit ships as
  its default, and how the recipe README flags the other (CORE-10271).
- Whether status and metrics really require `access_token` (harvest 2) or
  `id_token` works everywhere (harvest 1); one probe settles it.
- The publish permission model: root vs child workspace, single vs batch
  event creation (CORE-10302).
