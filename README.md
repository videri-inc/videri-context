# videri-context

The context pack for building on the Videri platform: what a builder or a
coding agent needs to know that the API contracts do not say.

**Status: internal preview.** Private to Videri while the developer program
validates it with internal builders. It becomes public after two cold-start
tests pass and the redaction checklist has run clean for two weeks.

## Files

| File | What it is |
|---|---|
| `AGENTS.md` | Canonical instructions for any coding agent: read order, where things are, non-negotiable rules |
| `CLAUDE.md` | One-line pointer to `AGENTS.md` |
| `AUTH.md` | Getting access, API keys, key to token, required headers, first call, errors |
| `GOTCHAS.md` | Traps that cost builders hours, in symptom / cause / what to do / example form |
| `DECISIONS.md` | Why the starter kit does what it does |
| `PLATFORM.md` | (coming) Domain glossary in builder terms |
| `CAPABILITIES.md` | (coming) What is supported, constrained, unverified or unsupported, per domain |
| `REDACTION.md` | The checklist every change to this repository must pass |

## What this repository is not

It is not a copy of the API reference. The developer portal
(`developer.<env>.videri.com`) owns the contracts, published at
`/openapi/index.json`; this pack points at them and never retypes an endpoint.
Every duplicated contract is a future contradiction.

## Contributing

The pack is curated by the owners in `CODEOWNERS`. If you are one of them:
open a pull request, run `REDACTION.md` against your change and tick the boxes
in the PR template. Everyone else: open an issue with what you found, using the
template at the bottom of `GOTCHAS.md`, and a curator will add it.

## Related

- Developer portal: https://developer.sandbox.videri.com (internal builders)
- Starter kit: https://github.com/videri-inc/videri-starter
- Recipes: https://github.com/videri-inc/videri-recipes
