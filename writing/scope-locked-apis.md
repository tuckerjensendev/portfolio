# Scope-Locked APIs (Backend): Design It So It Fails Closed

Multi-tenant APIs usually fail the same way: you build 90% of the service correctly, then one endpoint trusts a client-supplied scope field and you’ve built a cross-tenant data leak path.

This is a practical design + testing checklist for “scope-locked” APIs: systems where **every read/write is implicitly bound to the caller’s scope**, and the service is designed to **fail closed** when something doesn’t match.

## TL;DR

- **Derive scope from auth, not the request body.** If a client can pick `tenant_id`, `org_id`, `user_id`, `namespace`, etc., you’ve created the most common class of authorization bugs.
- **Treat “scope selector” as a separate capability.** If you support it, it must be narrowing-only and explicitly authorized.
- **Lock every path, including the weird ones.** Not just your “main” `/items` endpoints, but also decode, status, downloads, exports, search, and admin/metrics.
- **Test with two tokens** (same tenant, different scope) and try to break it through *every* endpoint.
- **Prefer 404 on cross-scope reads** unless you have a strong reason not to. Avoid turning existence into an oracle.

## What I mean by “scope”

Scope is any partition you use to prevent cross-contamination:

- tenant / account
- user
- org / team
- project
- namespace / environment (`prod` vs `dev`, `workspace-a` vs `workspace-b`)

In practice, scope is usually represented by a tuple like:

```
(tenant_id, scope_user, scope_org, scope_project, namespace)
```

You can call them whatever you want. The rules below don’t change.

## The goal (clear, testable)

For any request path that touches user data:

1) The server must infer the caller’s scope **only** from the authenticated identity (token/session).  
2) The request must not be able to “reach across” scope boundaries by manipulating IDs, query params, body fields, or path segments.  
3) If there’s a mismatch, the server should behave predictably and fail closed.

If you can’t state those in one sentence, the system isn’t scope-locked.

## Core pattern: scope is derived, never accepted

The simplest correct pattern looks like this:

- Bearer token is presented to the API.
- API resolves the token to an identity record.
- Identity record includes the scope tuple.
- Every query is filtered by that scope tuple.

In pseudocode:

```ts
type Scope = {
  tenantId: string;
  scopeUser: string;
  scopeOrg: string;
  scopeProject: string;
  namespace: string;
};

type RequestContext = {
  scope: Scope;
  actorId: string;
  isAdmin: boolean;
};

function requireAuth(req): RequestContext {
  const token = parseBearerToken(req);
  const identity = lookupIdentityByTokenHash(token);
  if (!identity) throw unauthorized();

  return {
    scope: identity.scope, // authoritative
    actorId: identity.actorId,
    isAdmin: identity.isAdmin,
  };
}
```

Now the important rule:

> If the request body contains `tenantId` / `orgId` / `namespace` / etc., **reject it** by default.

Even if you “ignore it”, you’re training future you (or future teammates) to assume it’s safe to send it. That’s how it creeps back in.

### “But we need to query across projects…”

Cool. That is not the baseline API. That’s a *different capability*:

- either an admin endpoint,
- or a token with explicit selector privileges,
- or a backend-to-backend integration role.

Treat it as opt-in and separately reviewed.

## Scope selector (optional): narrowing-only or it’s a bug

Sometimes you want a “manager view” that can query multiple sub-scopes:

- a support console
- an org admin who can query all projects in the org
- a pipeline worker that needs to read across namespaces

If you add selector mode, do it like this:

1) Token grants selector privileges explicitly.  
2) The requested selector is validated against the token’s allowlist.  
3) The selector can only narrow; it can’t escalate beyond what the token is allowed to access.

Example allowlist patterns (conceptual):

```
namespace: "prod*"
scope_project: "proj:*"
scope_user: "user:*"
```

Now selector requests can be validated:

```ts
function validateSelector(identity, requestedScope): Scope {
  // Example intent: requestedScope must be within identity.allowedScopePatterns
  if (!withinAllowedPatterns(identity.allowedScopePatterns, requestedScope)) {
    throw forbidden(); // or 404, depending on your model
  }

  // Still bind tenantId, and still enforce that selector only narrows.
  return {
    tenantId: identity.scope.tenantId,
    ...requestedScope,
  };
}
```

The big gotcha:

- Selector mode must never accept “raw” `tenantId`/`orgId`/etc. without validation.
- Selector mode must not be silently enabled just because a param exists.

If you implement selector mode, write a conformance test for it. Treat it as a risky feature, because it is.

## Lock *every* path (the endpoints people forget)

Most teams scope-lock the obvious endpoints:

- `GET /items`
- `POST /items`
- `GET /items/:id`

Then a leak happens through a path that “isn’t really data access”… except it is.

Here are the usual suspects:

- `GET /download/:fileId`
- `GET /exports/:exportId`
- `GET /search?q=...`
- `GET /status?job_id=...`
- `GET /decode?evidence_id=...`
- `GET /admin/metrics` (if metrics are per-tenant, not global)
- `POST /webhooks/*` (writes scoped data and should be validated)
- `POST /bulk-import` (especially when it accepts “namespace” or “project” fields)

My rule: if an endpoint touches an internal ID that was generated from scoped data, it’s a data access path and must be scope-locked.

## Cross-scope behavior: 404 vs 403 (and why it matters)

If I try to read `itemId=abc123` and I’m not allowed, what should I get back?

Two common choices:

- `403 Forbidden`: “you are authenticated, but not authorized”
- `404 Not Found`: “as far as you’re concerned, this doesn’t exist”

If your IDs are guessable (or even if they’re just brute-forceable over time), `403` can become an existence oracle:

- `404` means “doesn’t exist”
- `403` means “exists but not yours”

That’s often enough for an attacker to enumerate IDs and confirm the presence of data.

In scope-locked APIs, I generally prefer:

- `404` for “wrong scope” reads
- `401` for “not authenticated”
- `403` for “authenticated but attempting a forbidden capability” (like selector mode) *when that distinction matters*

Whatever you choose, make it consistent and test it.

## Caching: the fastest way to reintroduce leaks

Scope bugs aren’t always in your DB queries. They can be in:

- in-memory caches
- CDN caches
- “query result” caches
- precomputed materializations

Two rules:

1) Scope must be part of the cache key.  
2) If you can’t include scope in the cache key, don’t cache it.

If you cache `GET /profile` by `userId` but “userId” is not globally unique across tenants, you’ve built a leak. If you cache by `path` only, you’ve built a leak.

## Background work: scope must follow the job

Any async workflow needs scope propagation:

- job rows must include the scope tuple
- workers must claim jobs within scope and execute with that scope
- job status endpoints must be scope-locked

If your job queue is global and workers can read any job, you need to enforce scope at claim time.

Also: don’t leak scope in logs or metrics in ways you wouldn’t want exposed. “Observability” is not an excuse to publish sensitive identifiers.

## Tests: the two-token harness (minimal and brutal)

Here’s the test strategy I like because it’s simple and it catches real bugs:

1) Mint token A and token B for the *same tenant*, but different scope.  
2) Ingest/write data using token A.  
3) Attempt to read/modify it using token B via every endpoint.

If anything comes back other than “not found” (or your chosen equivalent), you have a leak path.

### Example (curl-style)

This is intentionally generic; adapt it to your API.

```bash
export TOKEN_A="..."
export TOKEN_B="..."

# Create data under A
ITEM_ID=$(
  curl -sS -H "Authorization: Bearer $TOKEN_A" \
    -H "Content-Type: application/json" \
    -d '{"title":"secret"}' \
    https://api.example.com/v1/items \
  | jq -r .id
)

# Attempt cross-scope read under B
curl -i -H "Authorization: Bearer $TOKEN_B" \
  "https://api.example.com/v1/items/$ITEM_ID"
```

Repeat the same pattern for:

- downloads
- status endpoints
- search
- exports
- “decode”-style endpoints (if you have any)

If your service has admin endpoints, test that token B can’t reach them, and that admin endpoints are disabled by default unless intentionally configured.

## Checklist (what I’d look for in a review)

- [ ] No endpoint accepts `tenant_id` / `org_id` / `namespace` without explicit selector-mode validation.
- [ ] Every read and write query is filtered by the full scope tuple.
- [ ] Cross-scope reads return consistent “not found” semantics (or consistent `403`, if you choose that).
- [ ] Cache keys include scope; CDN caching is disabled or explicitly scoped.
- [ ] Background jobs store and enforce scope; status endpoints are scope-locked.
- [ ] Debug/admin endpoints are disabled by default; when enabled, they’re gated (token + network allowlist).
- [ ] Two-token tests exist and run in CI.

## If you want a concrete reference implementation

I built and documented a scope-locked API with a protocol-level conformance suite here:

- [Conformance spec (normative)](../docs/lam-v2/conformance-spec.md)
- [Example “what to falsify” packet (systems/security)](../docs/lam-v2/csail-review-packet.md)

## SEO notes (optional)

If I were publishing this as a blog post, I’d ship with:

- Suggested title: “Scope-Locked APIs: How to Prevent Cross-Tenant Data Leaks (Fail Closed)”
- Meta description: “A practical checklist for designing multi-tenant APIs that derive scope from auth, lock every endpoint, avoid existence leaks, and catch regressions with two-token tests.”
- Internal links:
  - To an article on idempotency (webhooks, retries)
  - To an ops/runbook article (how to verify isolation in production)
