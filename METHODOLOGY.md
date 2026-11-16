# GraphQL Audit Methodology

The audit workflow, in order. Work through the phases against an in-scope, authorized target only (see the "Scope & Authorization" section in [README.md](README.md)). All endpoints below are the generic placeholder `https://api.example.com/graphql`.

## Phase 0 — Recon and Setup

1. **Confirm the endpoint.** Common paths: `/graphql`, `/api/graphql`. Confirm which HTTP methods are accepted (POST, and sometimes GET).
2. **Confirm authorization.** Written permission, in-scope endpoints, test accounts, and what data you may touch. Record it.
3. **Establish a baseline.** Send a minimal query (`{ __typename }`) with and without authentication and note the status code, response time, and error format. A baseline lets you tell a real finding from normal behavior.
4. **Fingerprint the engine.** Different GraphQL engines have different known issues (e.g. Hasura, Apollo, GraphQL-Go, Graphene, Hot Chocolate, WPGraphQL). Use graphw00f (from source) to identify the engine before you test, so you know which engine-specific issues to check.

## Phase 1 — Schema Discovery

### 1a. Introspection on

Send the introspection query (see [QUERIES.md](QUERIES.md)) and save the full schema. Then study it for:

- Mutations that touch user data: `updateUser`, `deleteAccount`, `changeEmail`, `changePassword`.
- Queries that take an object by ID: `user(id: ...)`, `order(id: ...)`.
- Sensitive-looking fields: `internalNote`, `adminOnly`, `role`, `isAdmin`, `rawPassword`, `apiKey`.
- Sensitive-looking types: `AdminUser`, `InternalConfig`, `DebugInfo`.
- Deprecated fields — these are often forgotten, with auth checks removed or loosened.
- Subscription types — real-time data that may leak across users.

### 1b. Introspection off — recover the schema anyway

If `__schema` is blocked, try (in order):

1. **Bypass attempts** — newline injection inside the query, a fragment on `__Schema`, `__type(name: "User")` instead of `__schema` (often missed by blocklists), introspection over GET instead of POST, or introspection over the WebSocket subscription channel (a different code path that may be unrestricted).
2. **Field suggestions** — send a query with a deliberate typo on a known field. Engines often answer "Did you mean X?", leaking real field names.
3. **Clairvoyance** — automated field discovery from suggestions and error messages. It typically recovers roughly 80% of the schema even when introspection is blocked. Seed it with any type names you already know.
4. **Client-side sources** — JavaScript bundles and mobile app code often contain the full schema, query documents, and type names the UI uses.

If only `__typename` is reachable and no types leak at all, the schema is well locked down — note it and move on.

## Phase 2 — Field Mapping

Build a map of the schema before testing individual fields:

1. **List every query, mutation, and subscription** with its arguments.
2. **Mark each field's sensitivity**: public, authenticated-user-only, owner-only, admin-only, internal.
3. **Flag hidden or undocumented fields**: fields present in the schema (or in admin docs, frontend code, or mobile bundles) but not in the public documentation — especially fields carrying role, organization, feature-flag, or internal filter values, and fields whose request bodies are richer than the visible UI controls.
4. **Note nested fields**: authorization is often checked on the top-level object but not on related objects reached through it (e.g. `me { orders { user { email } } }`).

This map drives Phase 3 — every sensitive field gets its own authorization test.

## Phase 3 — Authorization Testing (Per Field)

Test authorization field by field, not just endpoint by endpoint. A GraphQL endpoint usually has no per-route auth; the checks live in individual resolvers, and one missed resolver is a finding.

### 3a. Unauthenticated access

Repeat the sensitive queries and mutations with **no** credentials at all:

- List queries (`users { id email role }`).
- Object-by-ID queries (`user(id: USER_ID)`).
- Mutations, including privileged ones (`createAdmin`, `updateUserRole`).

Any success without auth is a finding.

### 3b. Horizontal access (IDOR)

Authenticated as a low-privilege test user, request another user's objects by ID:

- `user(id: OTHER_USER_ID) { email phone address }`
- `order(id: OTHER_ORDER_ID) { total items { name } user { email } }`
- Messages, invoices, appointments — any object type that takes an ID.

Then enumerate efficiently with aliases: one request carrying many aliased object fetches (e.g. `u1: user(id: 1)`, `u2: user(id: 2)`, …) to see how far enumeration reaches.

### 3c. Field-level access on your own objects

An object may be yours, but not every field on it is:

- Query privileged fields on your own profile: `me { id email role isAdmin internalNote }`.
- Try privileged mutations on other users: `updateUser(id: OTHER_USER_ID, role: "admin")`.
- Try deprecated fields with the same privileged-field set — they are the most commonly forgotten.

### 3d. Vertical privilege escalation

Find a role-changing mutation in the schema, then call it as a regular user. If a low-privilege account can promote itself (or another account), that is a critical finding.

### 3e. Subscriptions

Capture a legitimate subscription (e.g. order updates) and replay it with another user's ID over WebSocket. Also check whether a subscription delivers events for **all** users with no scoping. This matters most for payment status, messages, and location updates.

## Phase 4 — Batching and Depth Analysis

### 4a. Query batching

GraphQL accepts an array of operations in one POST. Test it:

1. Send a batch of 100 identical trivial queries in one request and compare response time against a single query.
2. Send alias batching: many aliases in one query (e.g. 500 aliases, each a separate resolver call) — this bypasses per-query limits.
3. Assess the amplification: 100 login mutations per request bypasses per-IP lockouts; alias-batched OTP guesses bypass per-request OTP limits; batched password-reset mutations can spam many accounts at once.

### 4b. Depth and complexity

1. Build a circular-reference query (e.g. `User -> friends -> User -> friends -> ...`) to increasing depth and measure response time.
2. If response time grows with depth and there is no error at reasonable depth (a common recommendation is a depth limit of 10 or less), there is no effective depth limit.
3. Check for query complexity limits and character limits as well — their absence is part of the same finding.

### 4c. Rate limiting on mutations

Confirm whether mutations are rate-limited at all: repeated login, password-reset, or verification-code mutations in quick succession. Batching (Phase 4a) multiplied by missing rate limits is what turns a medium into a critical.

## Phase 5 — Injection via Arguments

GraphQL arguments flow straight into resolvers, which often pass them to SQL/NoSQL queries or templates without sanitization:

- **SQL injection**: classic probes in `search`, `filter`, and `id` arguments; time-based blind probes.
- **NoSQL injection**: operator injection via JSON coercion in login/filter arguments (e.g. `$gt`, `$regex` operators where the backend is MongoDB).
- **Server-side template injection**: if a field you can write (bio, description) is rendered in a template, test a harmless arithmetic expression and see whether it evaluates on read.

Use GraphQLmap (interactive) to drive injection testing systematically.

## Phase 6 — WAF and Filter Bypass (Only Where In Scope)

If a filter blocks `__schema` or other keywords, test whether the filter is only skin-deep — content-type switching (`application/graphql` instead of `application/json`), GET instead of POST, comments inside keywords, and fragment-based introspection. Document each bypass that works; each one is evidence the control is not effective.

## Phase 7 — Reporting

For every confirmed issue, write it up with [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md):

- One finding per report, with a precise title.
- Reproduction steps that work from a clean session.
- Impact stated as what an attacker can do **right now**, not hypothetically.
- Approximate CVSS and concrete remediation.

### Severity guide

| Finding | Approximate CVSS |
|---|---|
| Introspection only (schema disclosure) | 5.3 (Medium) |
| IDOR, cross-user data access | 7.5–8.5 (High) |
| Batching chained to account takeover (e.g. OTP brute force) | 9.0+ (Critical) |
| Unauthenticated privileged mutation | 9.8 (Critical) |
| No depth/complexity limits (DoS) | Medium |

## When to Walk Away

Stop a line of testing when the target shows it is hardened — it saves time and avoids noise:

- The endpoint consistently returns 404/410 (not active).
- All queries return a generic "Unauthorized" with no field suggestions.
- A rate limit fires on the second query.
- Only `__typename` is accessible and no types leak.
- The endpoint is an Apollo Federation gateway only — the downstream services are the real targets, not the gateway.
