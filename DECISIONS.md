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

## Decisions still to capture (16 Sep call)

These are the questions the other working-group answers should settle; each
becomes an entry.

- Starter kit stack: which framework and tooling, and why it is a strip-down of
  the menu-board build rather than a rewrite.
- Hosting for prototypes: what is acceptable for public-API-only builds with a
  test key, and where the line is.
- How the menu-board build supplies its data today (static file vs live
  source), and why.
- What the Cloudflare Worker in the current demos is for, and whether the
  starter needs an equivalent.
- What is Beetley-estate-specific and must be removed for the starter and the
  first recipe.
