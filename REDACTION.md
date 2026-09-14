# REDACTION.md: what must never appear in this repository

This pack describes how to build **on** the Videri platform. The internal
knowledge it is curated from describes how Videri **runs** the platform. Only
the first belongs here. This checklist is run by hand on every pull request
today; it becomes an automated lint when the nightly sync is built (Phase 2).

## Never include

1. **Internal hostnames.** Anything that is not a public `api.<env>.videri.com`
   or `developer.<env>.videri.com` host, or `developer.videri.com`.
   Source-control, CI, admin, monitoring and VPN hosts are internal.
2. **Ports, container or cluster names, deployment topology.**
3. **Database, schema, table or column names.** Describe fields as the API
   returns them, never as they are stored.
4. **Message queue names, exchanges, routing keys, worker or job names.**
5. **Service-to-service credentials or headers, platform-admin roles, internal
   role names.** The only roles that exist here are the ones a tenant admin can
   assign.
6. **Internal repository paths, branch names, CI job names, pipeline
   variables.** Ticket keys are fine; MR numbers are not.
7. **Infrastructure vendor and product names** (gateway, orchestration,
   identity provider, VPN, observability). Say what it does, not what it is.
8. **Customer names, customer tenant codes, device serials, real asset URLs.**
   The one exception is the designated internal builders' sandbox tenant named
   in `AUTH.md`.
9. **People's names in content.** Owners belong in `CODEOWNERS`, not in
   articles.

## Always fine

- Public URLs and paths that appear in the portal or in a public contract.
- Service names as they appear in public URL paths (`canvas-service`,
  `rpm-service`, `cms`).
- Header names, status codes and error messages the API actually returns.
- Units, ranges and conventions (brightness 0 to 255, durations in ms).
- Anything a logged-in portal user can already see in Swagger UI.

## Grep list for the future lint

A change fails if any of these match (case-insensitive, whole words where
marked with `\b` so that "redistributed" does not trip on "redis"). Extend as
leaks are found.

```
\bkong\b|rabbit|amqp|ejabberd|xmpp|kubernetes|\bk8s\b|argocd|\bhelm\b|terraform|terragrunt|netbird|wireguard|grafana|\bloki\b|gitlab|git\.ops|cognito|\blambda\b|postgres|\bredis\b|dynamodb|typeorm|liquibase|x-vle-service-token|videri-admin|super_role|:80[0-9][0-9]\b|glpat-|arn:aws
```

## How to record a run

In the pull request description:

```
Redaction checklist: run on <date> by <name>. Clean.
```

or list what was removed.
