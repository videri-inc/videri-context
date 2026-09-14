# DECISIONS.md: why the starter does what it does

Short records of choices made while building on the platform, so the next
builder inherits the reasoning and not just the code. One entry per decision.
Harvest sessions with experienced builders feed this file (CORE-10259).

Status: v0 template, 14 Sep 2026. First entries land after harvest session 1.

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

## Decisions to capture in harvest session 1

These are the questions the session should answer; each becomes an entry.

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
