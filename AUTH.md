# AUTH.md: access, keys, tokens, headers, first call

Everything a builder or a coding agent needs to make one correct authenticated
call against the Videri REST API. The human-readable version is the developer
portal article "Authenticating with your API Key" (Knowledge Base). That article
is canonical; if this file and the article disagree, the article wins and this
file has a bug. Open a pull request.

Status: v0, drafted 14 Sep 2026 (CORE-10261). Sections marked TBD are waiting
on CORE-10260.

## Getting access

- **A developer portal account is a Videri Portal account.** Same username and
  password, same environment. If you can sign in to the Videri Portal for an
  environment, you can sign in to that environment's developer portal.
- **If you do not have a user yet**, either ask someone in your organization
  who already has portal access to create one for you, or request one from
  Videri customer support, who will validate the request with an admin of your
  tenant.
- **Internal Videri builders** work on the **sandbox** environment against the
  internal builders' sandbox tenant.
  Tenant code: **TBD (CORE-10260)**. Tenant admin who creates users: **TBD
  (CORE-10260)**. Never build or test against a customer's production tenant.
- **Your tenant code(s)**: the Profile tab of your developer portal dashboard
  lists the tenants you belong to. The same list is in the `tenants` claim of
  your `id_token`. Use the code exactly as shown.

## Environments

| Environment | Developer portal | API base URL |
|---|---|---|
| Sandbox (internal builders, partner previews) | `https://developer.sandbox.videri.com` | `https://api.sandbox.videri.com` |
| Production (US) | `https://developer.videri.com` | `https://api.go.videri.com` |

Other environments exist for Videri's internal use and are not documented here.
A key, a token and a tenant belong to exactly one environment. Each portal
publishes its own environment's contract index at `/openapi/index.json`.

## API keys

- Create one from **Dashboard > API Keys** on the developer portal. The full
  key is shown once, at creation. Copy it then; it is never shown again.
- Two types, which set the key's prefix: `sk_test_` for **test**, `sk_prod_`
  for **production**. Both are exchanged at the same token endpoint. **What a
  key can reach is decided by your account's tenants and roles, not by its
  type.** Build with a test key so production keys stay easy to find and
  revoke.
- A key works only against the environment it was created in.
- An expiry date is optional. Past its expiry, the key is refused at the token
  endpoint.
- Never put a key in a client bundle, in source control, in a log or in a chat.
  A browser app that must call the API needs a server-side proxy that holds the
  key and forwards the user's own token.

## Key to token

An API key is not a bearer token. Exchange it:

```
POST {API_BASE_URL}/rpm-service/v2/auth/token
Content-Type: application/json

{
  "username": "your-username",
  "password": "your-password",
  "api_key":  "your-api-key"
}
```

```bash
curl -X POST "{API_BASE_URL}/rpm-service/v2/auth/token" \
  -H "Content-Type: application/json" \
  -d '{"username":"your-username","password":"your-password","api_key":"your-api-key"}'
```

Response, `200 OK`:

```json
{
  "access_token":  "eyJraWQiOiJ...",
  "id_token":      "eyJraWQiOiJ...",
  "refresh_token": "eyJjdHkiOiJ...",
  "expires_in":    3600,
  "token_type":    "Bearer"
}
```

- Use **`id_token`** as the Bearer token on every API call. Not `access_token`,
  and never the API key itself.
- `expires_in` is seconds, typically 3600 (one hour). Cache the token and fetch
  a new one as it nears expiry rather than re-authenticating on every request.
- `refresh_token` is long-lived and more sensitive than the key. Keep it
  server-side. The refresh flow is defined in the RPM Service V2 contract;
  read it there, do not guess the endpoint.
- Contract for this service: `{API_BASE_URL}/rpm-service/api-json` (RPM Service
  V2). Fetch it with no `Authorization` header. It is being added to
  `/openapi/index.json`; until then use the URL above.

## Required headers on every call after the token call

```
Authorization: Bearer <id_token>
x-tenant: <tenant_code>
x-group: <workspace_uuid>        (only where the spec requires a workspace)
```

- **`Authorization`**: `Bearer` plus the `id_token` from the token response,
  obtained from the token endpoint of the **same environment** as the API base
  URL you are calling.
- **`x-tenant`**: the tenant code whose data you are working with. Required on
  every call except the token call itself. From your Profile tab or the
  `tenants` claim.
- **`x-group`**: the UUID of a workspace (group) inside the tenant. Include it
  to scope reads and writes to one workspace; omit it to work across every
  workspace you can access. Some write endpoints, such as creating a schedule,
  require it and answer `403` without it. **Canvas Service does not read this
  header**; it takes the workspace as the `group_id` query parameter. Every
  canvas returned by the first call below carries the `group_id` and
  `group_name` of its workspace.
- A few user-scoped endpoints, such as your own API keys and MFA settings, need
  only `Authorization`.

## Your first call

Lists the canvases (displays) assigned to a workspace in your tenant, with live
status. A page of canvases back means your key, your token and your tenant code
are all correct.

```bash
curl -X GET "{API_BASE_URL}/canvas-service/canvases?assigned_to_group=true&with_status=true&page=0&size=10" \
  -H "Authorization: Bearer <id_token>" \
  -H "x-tenant: <tenant_code>"
```

Expected: `200 OK` with `content` (the canvases) and `totalElements`,
`totalPages`, `size`, `number` describing the page. Each canvas carries
`device_id`, `name`, `orientation`, `group_id`, `group_name`, `presence_status`
(`online`, `offline`, `unavailable`), `content_status` and `last_online_time`,
among other fields; the full schema is in the Canvas Service contract.

- `assigned_to_group` is **required**. Omit it and you get
  `400 Missing required query parameter: assigned_to_group`.
- `with_status=true` adds the real-time status fields.
- `group_id=<workspace uuid>` limits the list to one workspace.
- `page` is zero-based; `size` defaults to 10, maximum 1000.

## Errors you will see

| Status | Message you will see | What happened | What to do |
|---|---|---|---|
| 401 | `Unrecognised API key`, `Invalid credentials` | Token endpoint rejected the key (not yours, or only the prefix sent), or the username or password is wrong | Copy the full key from Dashboard > API Keys; check the credentials you use to sign in to the portal |
| 401 | `'exp' claim expired at ...` | The `id_token` has expired (one hour) | Request a new token, or refresh, and retry |
| 401 | `jwt signature verification failed`, `RSA key with id ... not found` | Token malformed, truncated, altered, or issued by a **different environment** | Send the complete `id_token` from the token endpoint of the same environment as the API base URL |
| 403 | `Access is denied`, `Authorization header missing` | No `Authorization` header | Add it |
| 403 or 400 | `Access is denied`, `Missing required header: x-tenant` | No `x-tenant` header (some services answer 400 with the explicit message) | Add `x-tenant` with a tenant code from your Profile |
| 403 | `No access to <TENANT> tenant.` | `x-tenant` names a tenant you do not belong to, or has a typo | Use a tenant code exactly as listed on your Profile; ask a tenant admin for access if you need another |
| 403 | `Forbidden resource`, `Insufficient role permissions` | You belong to the tenant but your role lacks the permission, or the endpoint is reserved for platform administrators | Ask a tenant admin for a role with the required permission, or use an endpoint your role allows |
| 403 | `Not authorized. Context group required` | Endpoint requires a workspace context and no valid `x-group` was sent | Add `x-group` with the UUID of a workspace you belong to |

Messages arrive either as the `message` field of a JSON body or as plain text.

## Rules for agents (authentication)

1. Fetch spec URLs with **no** `Authorization` header. Tokens are for API
   calls, not for contracts; a stale or foreign-environment token on a spec
   fetch gets a `401` on the spec itself.
2. Key, token and API base URL from the same environment, always.
3. Read credentials from environment variables. Never print a token or key,
   never write one into code, and never paste one into a chat.
4. On `401` with an expiry message, refresh once and retry. On `403`, stop and
   show the person the message; do not retry in a loop.
5. Cache the token for its lifetime. One token per hour is the expected shape,
   not one per request.
