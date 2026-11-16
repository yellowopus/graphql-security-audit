# GraphQL Audit Checklist

Copy-paste checklist for a GraphQL audit. Check each box as you complete it against an in-scope, authorized target. All endpoints are the placeholder `https://api.example.com/graphql`.

## Setup

- [ ] Written authorization confirmed; scope (endpoints, accounts, data) recorded
- [ ] Endpoint confirmed: `https://api.example.com/graphql` (POST; GET also tested)
- [ ] Baseline query (`{ __typename }`) sent with and without auth; status, timing, and error format noted
- [ ] Engine fingerprinted (graphw00f) and engine-specific issues noted

## Schema Discovery

- [ ] Introspection query (`__schema`) attempted
- [ ] Full schema dumped and saved (if introspection is on)
- [ ] If introspection is off: bypass attempts tried (newline injection, fragment trick, `__type`, GET, WebSocket)
- [ ] Field suggestions probed with deliberate typos
- [ ] Clairvoyance run for automated field discovery (if introspection is off)
- [ ] Client-side sources checked (JS bundles, mobile code) for schema and query documents
- [ ] Schema reviewed for sensitive mutations (`updateUser`, `deleteAccount`, `changeEmail`, `changePassword`)
- [ ] Schema reviewed for sensitive fields (`internalNote`, `adminOnly`, `role`, `isAdmin`, `apiKey`)
- [ ] Schema reviewed for sensitive types (`AdminUser`, `InternalConfig`, `DebugInfo`)
- [ ] Deprecated fields listed
- [ ] Subscription types listed

## Field Mapping

- [ ] Every query, mutation, and subscription listed with its arguments
- [ ] Each field marked: public / authenticated / owner-only / admin-only / internal
- [ ] Hidden or undocumented fields flagged (in schema or client code, not in public docs)
- [ ] Nested/related-object fields flagged (auth may be weaker one level down)

## Authorization — Unauthenticated

- [ ] Sensitive list queries attempted with no credentials
- [ ] Object-by-ID queries attempted with no credentials
- [ ] Privileged mutations attempted with no credentials

## Authorization — Horizontal (IDOR)

- [ ] Another test user's object requested by ID (user, order, message, invoice, appointment)
- [ ] Enumeration attempted via aliases (many IDs in one request)
- [ ] Related-object traversal tested (`me { orders { user { email } } }`)

## Authorization — Field-Level and Vertical

- [ ] Privileged fields queried on own object (`role`, `isAdmin`, `internalNote`)
- [ ] Privileged mutation attempted on another user
- [ ] Role-escalation mutation attempted as a regular user
- [ ] Deprecated fields tested with the privileged-field set

## Authorization — Subscriptions

- [ ] Subscription replayed with another user's ID over WebSocket
- [ ] Subscription checked for missing per-user scoping (events for all users?)

## Batching and Rate Limiting

- [ ] Array batching tested (100 identical queries in one POST); response time compared to a single query
- [ ] Alias batching tested (500 aliases in one query)
- [ ] Brute-force amplification assessed (login / OTP / password-reset mutations in a batch)
- [ ] Mutation rate limiting tested (repeated login, reset, verification mutations)

## Depth and Complexity (DoS)

- [ ] Circular-reference depth query tested to increasing depth; response time recorded
- [ ] Depth limit confirmed present or absent (recommended limit: 10 or less)
- [ ] Query complexity limit confirmed present or absent
- [ ] Query character/size limit confirmed present or absent

## Injection via Arguments

- [ ] SQL injection probed in `search`, `filter`, and `id` arguments
- [ ] Time-based blind SQL injection probed
- [ ] NoSQL operator injection probed (where backend accepts object-typed arguments)
- [ ] Template injection probed in writable text fields (bio, description)
- [ ] GraphQLmap used for systematic injection testing (interactive)

## WAF / Filter Bypass (Where In Scope)

- [ ] Content-Type switching tested (`application/graphql`)
- [ ] GET introspection tested
- [ ] Comment injection in keywords tested
- [ ] Fragment-based introspection tested

## Reporting

- [ ] Each confirmed issue written up with [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md)
- [ ] Impact stated as what an attacker can do now, not hypothetically
- [ ] Approximate CVSS assigned
- [ ] Remediation written for each finding
- [ ] No credentials, tokens, or real customer data included in the report or evidence
