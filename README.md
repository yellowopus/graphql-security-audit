# GraphQL Security Audit

A focused, practical playbook for auditing a GraphQL API: schema discovery, authorization testing, batching and depth abuse, and denial-of-service checks. It distills the techniques from two internal skill notes (GraphQL audit and GraphQL/hidden-parameter testing) into a single repo you can work through on an in-scope target.

## Scope & Authorization

> **Only test targets you own or have explicit written authorization for.**
> Stay strictly inside the agreed scope — endpoints, accounts, and data that the authorization covers. Do not test, probe, or enumerate anything outside it, and do not use these techniques against any third-party service you do not have permission to test.
>
> Every example in this repo uses the generic placeholder endpoint `https://api.example.com/graphql` and placeholder values such as `USER_ID`, `TOKEN`, and `example.com`. They do not target any real third party. Replace the placeholders with your own in-scope target before running anything, and never point these examples at a live service you are not authorized to test.

## Who This Is For

- Security researchers and bug-bounty hunters auditing a GraphQL API they are authorized to test.
- Application security engineers reviewing a GraphQL service before launch or after a change.
- Developers who want to understand what an attacker can do with a single GraphQL endpoint, so they can defend it.

## Why GraphQL Needs Its Own Audit

GraphQL flips the usual threat model: instead of many fixed REST endpoints, there is one endpoint and the client drives the query. That means:

- **Introspection can hand over the whole schema** — every type, field, argument, and mutation — in one request.
- **Even with introspection off**, error messages and field suggestions can leak most of the schema back.
- **Batching and aliases** turn one HTTP request into hundreds of resolver calls, amplifying brute force and DoS.
- **Authorization is per-field, per-resolver**, not per-route — a field that is safe on one object may be unprotected on another.

## Contents

| File | What it is |
|---|---|
| [README.md](README.md) | This file — intro, scope, and contents |
| [METHODOLOGY.md](METHODOLOGY.md) | The audit workflow, step by step |
| [QUERIES.md](QUERIES.md) | Generic example queries and mutations for testing |
| [CHECKLIST.md](CHECKLIST.md) | Copy-paste checklist for a GraphQL audit |
| [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md) | Report template for a GraphQL finding |
| [LICENSE](LICENSE) | MIT License |

## Quick Start

1. Read [METHODOLOGY.md](METHODOLOGY.md) to understand the workflow.
2. Work through [CHECKLIST.md](CHECKLIST.md) against your in-scope target.
3. Use the examples in [QUERIES.md](QUERIES.md) — with your own authorized endpoint substituted for the placeholder.
4. Record each confirmed issue with [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md).

## Tools Worth Knowing

These are referenced in the methodology; install and use them only against authorized targets.

| Tool | Purpose | Source |
|---|---|---|
| graphw00f | Fingerprint the GraphQL engine (different engines, different CVEs) | Install from source: `https://github.com/dolevf/graphw00f` |
| clairvoyance | Recover field names when introspection is off | `pipx install clairvoyance` |
| graphql-cop | Automated attack checklist (introspection, batching, depth, and more) | Install from source: `https://github.com/dolevf/graphql-cop` |
| GraphQLmap | SQL/NoSQL injection testing via GraphQL arguments (interactive) | `pipx install git+https://github.com/swisskyrepo/GraphQLmap.git` |
| inql | Burp Suite extension for schema exploration and IDOR testing | Burp BApp Store |
| graphql-voyager | Visual schema explorer | Browser tool |
| wscat | WebSocket subscription testing | `npm i -g wscat` |

> Note: some of these tools' names on package registries are third-party placeholders or unclaimed — install graphw00f and graphql-cop from their source repositories, not from a package registry.

## High-Level Finding Chains

These are the combinations that turn a single weakness into a serious finding:

| Chain | Severity |
|---|---|
| Introspection → admin mutation found → called without auth | Critical |
| Batching → OTP brute force → account takeover | Critical |
| Field suggestions → hidden field discovered → IDOR | High |
| Alias bomb → rate limit bypassed → credential stuffing | High |
| Unauthenticated subscription → real-time data leak | High |
| No depth/complexity limits → query depth bomb → DoS | Medium |
| Deprecated field → sensitive data exposure | Medium |

## License

MIT — see [LICENSE](LICENSE).
