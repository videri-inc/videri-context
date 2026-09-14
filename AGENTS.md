# AGENTS.md: building on the Videri platform

This file is for coding agents (Claude, Cursor, Copilot and similar) and for the
people who direct them. It is the canonical instruction file for this repository
and for building on the Videri REST API. `CLAUDE.md` in this repository is a
one-line pointer here; do not maintain a separate copy.

Status: internal preview (Videri employees). Not yet public.

## What Videri is, in three sentences

Videri is a multi-tenant digital signage platform. One REST API, at
`api.<env>.videri.com`, manages canvases (the displays), the assets and
playlists shown on them, the events that schedule that content, the walls and
workspaces that group them, and the live status each device reports. Every call
is scoped to a tenant, and a key, a token and a tenant belong to exactly one
environment.

## Read in this order

1. This file.
2. `AUTH.md`: getting access, API key to token, required headers, a known-good
   first call, the errors you will see.
3. The contract of every service you are about to call, from the environment's
   index at `https://developer.<env>.videri.com/openapi/index.json`
   (sandbox: `https://developer.sandbox.videri.com/openapi/index.json`).
   Fetch spec URLs with no `Authorization` header.
4. `GOTCHAS.md`: the traps that cost builders hours.
5. `DECISIONS.md`: why the starter kit does what it does.
6. When they are published: `PLATFORM.md` (domain glossary) and
   `CAPABILITIES.md` (what is supported, constrained, unverified or
   unsupported). Until then, treat any capability that neither the starter kit
   nor a recipe exercises as unverified, and say so to the person you are
   building for before you design around it.

## Where things are

| Thing | Where |
|---|---|
| Developer portal (accounts, API keys, articles, Swagger) | `https://developer.<env>.videri.com`, agent index at `/llms.txt` |
| API contracts | `/openapi/index.json` on the portal, one entry per service with an absolute spec URL |
| Token endpoint | `POST {API_BASE_URL}/rpm-service/v2/auth/token`, see `AUTH.md` |
| Starter kit | `github.com/videri-inc/videri-starter` (internal preview) |
| Validated recipes | `github.com/videri-inc/videri-recipes` (internal preview) |
| Conformance checklist | `videri-recipes/CONFORMANCE.md` |

Never retype an endpoint, parameter, field name or enum value from memory or
from this repository. Read it from the spec.

## Non-negotiable rules

1. **Read the spec before you call.** Every endpoint you use must exist in a
   spec listed in `/openapi/index.json`. If it is not there, it is not public;
   stop and ask.
2. **`Authorization` and `x-tenant` on every call after the token call.**
   Add `x-group` only where the spec requires a workspace. Canvas Service
   takes the workspace as the `group_id` query parameter instead of the header.
3. **One environment.** The API key, the token and the API base URL must come
   from the same environment. A token minted on one environment is rejected by
   every other, including when you fetch a spec with it.
4. **Never invent identifiers.** Tenant codes, workspace ids, device ids and
   asset ids come from the person you are building for or from an API
   response. Never from a guess.
5. **Scheduling and deleting change real screens.** List the target canvases
   and confirm with the person before creating an event. Confirm before
   deleting a playlist or event or removing an asset from a playlist. Device
   commands (brightness, volume, reboot, overlay) take effect immediately;
   decide with the person whether your UI confirms them.
6. **Count from totals, not from arrays.** Read `totalElements`, `total` or
   `meta.totalItems`; arrays may be one page or a capped subset. Handle
   pagination through the starter's adapters; the shapes differ by service.
7. **No secrets anywhere visible.** API keys, passwords, tokens and refresh
   tokens live in environment variables or a secrets manager. Never in source,
   a client bundle, a log line, a screenshot or a chat message. A browser app
   that needs a key needs a server-side proxy that holds it.
8. **Prefer the starter over greenfield.** Start from `videri-starter`; when a
   recipe matches the job, start from the recipe.
9. **Say "unverified" out loud.** When the person asks for something no spec,
   recipe or capability entry covers, tell them it is unverified rather than
   building a plausible interface over an endpoint that may not exist.
10. **Environment matters for where you build.** Internal builders work on
    sandbox. Never build or test against a customer's production tenant.

## Before you hand something over

Run the checklist in `videri-recipes/CONFORMANCE.md`. It is short and it is
what moves a build from prototype to validated recipe.

## Feedback loop

Every time the portal or this pack failed to tell you something, that gap is
worth recording: open an issue on this repository describing the symptom and
what you had to find out, using the template at the bottom of `GOTCHAS.md`.
The curators listed in `CODEOWNERS` turn issues into entries. The pack improves
only through use.

## Getting access

See the "Getting access" section of `AUTH.md`.
