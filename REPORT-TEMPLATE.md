# GraphQL Finding Report Template

Use one copy of this template per finding. Replace every placeholder in `<ANGLE BRACKETS>` with your own in-scope details. **Do not include real credentials, tokens, session cookies, or real customer data in a report** — redact them and use placeholders.

---

## Title

`GraphQL <VULNERABILITY TYPE> — <Impact One-Liner>`

Examples:

- `GraphQL Introspection Enabled — Full Schema Disclosure on api.example.com`
- `GraphQL Batching Allows OTP Brute Force — Account Takeover via Alias Bombing`
- `GraphQL IDOR via Direct Object Queries — Access Any User's Private Data`

## Summary

| Field | Value |
|---|---|
| Target | `https://api.example.com/graphql` (replace with the authorized in-scope endpoint) |
| Vulnerability type | `<e.g. IDOR / Introspection / Batching / Missing depth limit / Injection>` |
| Severity | `<Critical / High / Medium / Low>` |
| CVSS (approximate) | `<e.g. 7.5>` |
| Affected field / operation | `<e.g. query { user(id:) }>` |
| Authenticated? | `<Yes — as a low-privilege test user / No — unauthenticated>` |
| Date tested | `<YYYY-MM-DD>` |
| Tester | `<Your name / handle>` |

## Description

`<What the issue is, in plain language. What the endpoint allows that it should not, and which data or action is exposed.>`

## Steps to Reproduce

Numbered steps that work from a clean session. Include the exact request and the observed response.

### 1. `<First step — e.g. authenticate as test user A>`

```bash
curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <TEST_TOKEN>' \
  -d '{"query":"<YOUR QUERY HERE>"}'
```

### 2. `<Second step — e.g. request another user's object>`

```bash
curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <TEST_TOKEN>' \
  -d '{"query":"{ user(id: \"<OTHER_TEST_USER_ID>\") { email phone } }"}'
```

### 3. `<What you observed>`

```json
{
  "data": {
    "user": {
      "email": "<REDACTED — other test user's email>",
      "phone": "<REDACTED>"
    }
  }
}
```

## Impact

`<What an attacker can actually do RIGHT NOW with this — no hypotheticals. Who is affected, what data or actions are exposed, and what it chains into (e.g. batching + OTP brute force = account takeover).>`

## Evidence

- Request: `<paste the raw request, with tokens and secrets redacted>`
- Response: `<paste the relevant portion of the response, with real personal data redacted>`
- Screenshots / logs: `<file names, if any>`

## CVSS Guidance

| Finding | Approximate CVSS |
|---|---|
| Introspection only (schema disclosure) | 5.3 (Medium) |
| IDOR, cross-user data access | 7.5–8.5 (High) |
| Batching chained to account takeover (e.g. OTP brute force) | 9.0+ (Critical) |
| Unauthenticated privileged mutation | 9.8 (Critical) |
| No depth/complexity limits (DoS) | Medium |

Adjust the score to the actual impact on the target — the table is a starting point, not a verdict.

## Remediation

Concrete fixes for this finding (pick the ones that apply):

- [ ] Disable introspection in production (allow it only in development environments).
- [ ] Remove field suggestions / verbose error messages in production.
- [ ] Enforce a per-query depth limit (recommended: 10 or less).
- [ ] Enforce query complexity limits and a maximum query size.
- [ ] Disable query batching, or add per-batch operation limits and rate limiting.
- [ ] Validate object ownership in **every** resolver, not just at the route level.
- [ ] Enforce field-level authorization for privileged fields (`role`, `isAdmin`, internal fields).
- [ ] Scope subscriptions per user; never deliver another user's events.
- [ ] Rate-limit mutations (login, password reset, verification codes) per account and per IP.
- [ ] Sanitize and parameterize resolver inputs; never interpolate arguments into SQL/NoSQL queries or templates.
- [ ] Re-check deprecated fields — apply the same authorization as current fields, or remove them.

## Retest Notes

`<Leave blank until retest. Record the date, the fix observed, and whether the finding is resolved.>`

- Retest date: `<YYYY-MM-DD>`
- Result: `<Resolved / Partially fixed / Still present>`
- Notes: `<...>`

---

*Template version: 1.0 — for authorized GraphQL audits only. See the "Scope & Authorization" section in [README.md](README.md).*
