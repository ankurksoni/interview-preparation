# WunderGraph Interview Questions & Answers

A **novice-to-medium** collection of WunderGraph Q&A — the API/BFF platform that turns GraphQL, REST, and other APIs into type-safe, generated JSON-RPC endpoints — with diagrams and examples, growing gradually from fundamentals to more complex patterns.

---

## Fundamentals

### 1. **What is WunderGraph and what problem does it solve?**

**Answer:** WunderGraph is an **open-source API platform / BFF (Backend-for-Frontend) framework** that lets you compose multiple APIs (GraphQL, REST, databases, gRPC) into **one unified, type-safe API layer** — but unlike a typical GraphQL gateway, it exposes that unified graph to the client as **auto-generated, type-safe JSON-RPC-style endpoints**, not raw GraphQL.

```
Without WunderGraph:                  With WunderGraph:

┌────────┐   direct calls             ┌────────┐
│ Client │──┬──► REST API A           │ Client │──► WunderGraph (single API)
└────────┘  ├──► GraphQL API B        └────────┘         │
            └──► Database C                    ┌──────────┼──────────┐
                                                 ▼          ▼          ▼
Client must know 3 different            REST API A   GraphQL API B  Database C
protocols, auth schemes, shapes         (all composed into ONE typed schema)
```

| Problem WunderGraph solves         | How                                                    |
| ------------------------------------ | --------------------------------------------------------- |
| Multiple APIs, inconsistent shapes    | Composes them into one WunderGraph schema                 |
| Clients need GraphQL knowledge        | Generates typed operations clients call like RPC functions |
| Exposing raw GraphQL is risky         | Only pre-defined, allowlisted operations are reachable      |
| Type safety across the stack          | Auto-generates TypeScript types end-to-end                  |

---

### 2. **How does WunderGraph differ from a typical GraphQL server like Apollo Server?**

**Answer:**

| Aspect                | Apollo Server                        | WunderGraph                                      |
| ----------------------- | --------------------------------------- | --------------------------------------------------- |
| Client-facing protocol   | Raw GraphQL (client sends query strings) | JSON-RPC-style HTTP endpoints (auto-generated)       |
| Ad-hoc client queries     | ✅ Allowed by default                    | ❌ Only pre-registered ".graphql" operations exposed |
| Schema composition        | Manual resolvers or Federation           | Declarative composition of upstream APIs (`wundergraph.config.ts`) |
| Type generation            | Optional (codegen tools)                 | Built-in, automatic (TypeScript client + hooks)      |
| Primary use case           | Building a GraphQL API from scratch      | Unifying/wrapping *existing* APIs as a BFF           |

```
Apollo model:                         WunderGraph model:

Client writes GraphQL query   ──►     Client calls a generated function
  query { books { title } }             wundergraph.query.GetBooks()
       │                                       │
       ▼                                       ▼
 Server resolves it live             Server executes a PRE-COMPILED,
 (query shape decided by client)     PRE-REGISTERED operation
                                      (query shape decided at build time)
```

> **Key mental model:** WunderGraph treats `.graphql` files in your project as **operations to compile**, not queries to run ad-hoc — similar to how a persisted-query allowlist works in Apollo, but as the default, first-class design.

---

### 3. **What is the WunderGraph architecture at a high level?**

**Answer:**

```
┌───────────────────────────────────────────────────────┐
│                     wundergraph.config.ts               │
│   (declares data sources: REST, GraphQL, DB, gRPC...)   │
└───────────────────────┬───────────────────────────────┘
                         │  wunderctl generate
                         ▼
        ┌────────────────────────────────┐
        │   Virtual Unified Graph (schema) │
        └────────────────┬────────────────┘
                          │
              operations/*.graphql (your queries)
                          │
                          ▼
        ┌────────────────────────────────┐
        │        WunderGraph Server        │  ← runs at request time
        │  (wunderctl / wunderNode)         │
        └────────────────┬────────────────┘
                          │  generates
                          ▼
        ┌────────────────────────────────┐
        │  Typed Client SDK (TS/React)     │  ← import { useQuery } from "./generated/client"
        └────────────────────────────────┘
```

| Component        | Role                                                      |
| ------------------ | ------------------------------------------------------------ |
| `wundergraph.config.ts` | Declares upstream APIs and global config                |
| `operations/`       | `.graphql` files — your app's allowed queries/mutations       |
| `wunderctl`          | CLI: generates schema, types, and runs the dev server          |
| Generated client      | Type-safe TS functions/hooks matching your operations          |

---

### 4. **How do you define a data source in `wundergraph.config.ts`?**

**Answer:**

```ts
import { configureWunderGraphApplication, introspect } from "@wundergraph/sdk";

const countries = introspect.graphql({
  apiNamespace: "countries",
  url: "https://countries.trevorblades.com/",
});

const jsonPlaceholder = introspect.openApi({
  apiNamespace: "jsonplaceholder",
  source: {
    kind: "file",
    filePath: "./jsonplaceholder.openapi.json",
  },
});

configureWunderGraphApplication({
  apis: [countries, jsonPlaceholder],
  server: {},
  cors: { allowedOrigins: ["http://localhost:3000"] },
});
```

```
introspect.graphql(...)   → pulls in an existing GraphQL API
introspect.openapi(...)   → pulls in a REST API described by OpenAPI/Swagger
introspect.postgresql(...)→ pulls in a Postgres database directly
introspect.mongodb(...)   → pulls in a MongoDB database directly

All of these become namespaced fields in ONE unified schema:
  query {
    countries_countries { name }
    jsonplaceholder_getPosts { title }
  }
```

> **`apiNamespace`** prevents naming collisions — if two APIs both have a `User` type, they become `countries_User` and `jsonplaceholder_User`.

---

### 5. **What is an "operation" in WunderGraph and how do you define one?**

**Answer:** An operation is a `.graphql` file in your `operations/` directory — WunderGraph compiles it at build time into a callable, typed API endpoint.

```graphql
# operations/GetCountry.graphql
query GetCountry($code: ID!) {
  countries_country(code: $code) {
    name
    capital
    currency
  }
}
```

This automatically becomes:

```
HTTP:  GET /operations/GetCountry?code=US
TS:    const { data } = useQuery({ operationName: "GetCountry", input: { code: "US" } });
```

```
operations/GetCountry.graphql
        │
        │  wunderctl generate
        ▼
1. Type-checked against the unified schema
2. Compiled into an executable plan
3. Exposed as /operations/GetCountry (REST-like JSON endpoint)
4. TypeScript types generated for input/output
```

> Only operations that exist as files are reachable — there's no way for a client to send an arbitrary GraphQL query to a WunderGraph server. This is a **security-by-design** default (see [Q14](#14-how-does-wundergraph-prevent-api-abuse-by-default)).

---

## Client Usage & Type Safety

### 6. **How do you call a WunderGraph operation from a React app?**

**Answer:**

```tsx
import { useQuery, useMutation } from "./generated/hooks";

function CountryPage({ code }: { code: string }) {
  const { data, isLoading, error } = useQuery({
    operationName: "GetCountry",
    input: { code },
  });

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return <h1>{data?.countries_country?.name}</h1>;
}

function AddReviewForm() {
  const { mutate } = useMutation({ operationName: "AddReview" });

  return (
    <button onClick={() => mutate({ input: { rating: 5, text: "Great!" } })}>
      Submit
    </button>
  );
}
```

```
Generated hooks are fully typed — TypeScript autocompletes:
  - operationName          (only valid operation names)
  - input                  (matches the .graphql variables exactly)
  - data / error shape     (matches the .graphql selection set exactly)

No manual type definitions, no codegen config to maintain.
```

---

### 7. **How does WunderGraph achieve end-to-end type safety?**

**Answer:**

```
   Upstream API schema           Your operation file           Generated types
  (introspected, e.g. GraphQL) ─►  GetCountry.graphql   ──►    GetCountryResponse
                                                                 GetCountryInput

┌──────────────┐   introspect   ┌──────────────┐   compile   ┌──────────────────┐
│ Countries API│ ─────────────► │ Unified Graph│ ──────────► │ operations/*.ts    │
└──────────────┘                └──────────────┘             │ (typed client SDK) │
                                                               └──────────────────┘
```

Because types flow automatically from the **real upstream schema** → **unified graph** → **your operation** → **generated client**, there's no manual type-writing step where drift can creep in. If the upstream API changes a field type, `wunderctl generate` regenerates and TypeScript immediately flags any now-incompatible usage in your frontend.

---

### 8. **What is the difference between a WunderGraph "Query" and "Mutation" operation?**

**Answer:** Same semantic distinction as GraphQL itself — WunderGraph just enforces it via the HTTP verb and file convention.

| Operation type | File location example         | HTTP method generated | Caching             |
| ----------------- | -------------------------------- | ------------------------ | ---------------------- |
| Query              | `operations/GetCountry.graphql`   | `GET`                     | Cacheable by default    |
| Mutation           | `operations/AddReview.graphql`    | `POST`                    | Never cached             |
| Subscription        | `operations/OnReviewAdded.graphql`| `GET` (SSE/WebSocket)      | N/A (live stream)         |

```graphql
# operations/AddReview.graphql
mutation AddReview($rating: Int!, $text: String!) {
  reviews_addReview(rating: $rating, text: $text) {
    id
    rating
  }
}
```

> Because queries map to `GET`, they're automatically **cacheable at the HTTP/CDN layer** — a benefit that's harder to get with raw GraphQL-over-POST.

---

### 9. **How do subscriptions work in WunderGraph?**

**Answer:**

```graphql
# operations/OnReviewAdded.graphql
subscription OnReviewAdded($productId: ID!) {
  reviews_reviewAdded(productId: $productId) {
    id
    rating
    text
  }
}
```

```tsx
function LiveReviews({ productId }: { productId: string }) {
  const { data } = useSubscription({
    operationName: "OnReviewAdded",
    input: { productId },
  });

  return <p>Latest rating: {data?.reviews_reviewAdded?.rating}</p>;
}
```

```
Client                     WunderGraph Server              Upstream API
  │  GET /operations/            │                              │
  │  OnReviewAdded (SSE) ───────►│                              │
  │                               │  subscribes to upstream ───►│
  │◄── event stream ──────────────│◄── pushes new review ───────│
  │   (rating:5, text:"Great!")   │                              │
```

WunderGraph handles the underlying transport (WebSocket to the upstream GraphQL API, or polling for REST APIs) and re-exposes it to the client as **Server-Sent Events** — no client-side WebSocket management needed.

---

### 10. **What is "live queries" in WunderGraph and how does it differ from subscriptions?**

**Answer:** A **live query** re-runs a normal `query` operation on an interval or on data change and streams updated results — useful for APIs that don't natively support subscriptions.

```tsx
const { data } = useQuery({
  operationName: "GetCountry",
  input: { code: "US" },
  liveQuery: true, // re-polls and streams updates automatically
});
```

| Feature       | Subscription                             | Live Query                                  |
| --------------- | ------------------------------------------- | ----------------------------------------------- |
| Requires upstream support | ✅ Upstream must support subscriptions        | ❌ Works on any query, even plain REST wrapped APIs |
| Mechanism        | Event-driven (push)                          | Polling under the hood, exposed as a stream       |
| Use case           | True real-time events (chat, notifications)    | "Good enough" freshness for dashboards, lists      |

---

## Composition & Security

### 11. **How does WunderGraph compose multiple APIs into a single graph?**

**Answer:** Every configured data source is introspected and merged into **one virtual schema**, namespaced to avoid collisions.

```ts
configureWunderGraphApplication({
  apis: [
    introspect.graphql({ apiNamespace: "countries", url: "..." }),
    introspect.openApi({ apiNamespace: "products", source: { ... } }),
    introspect.postgresql({ apiNamespace: "db", databaseURL: "..." }),
  ],
});
```

```
Unified schema (conceptually):

type Query {
  countries_countries: [Country!]!
  products_getProduct(id: ID!): Product
  db_findManyusers: [users!]!
}
```

You can then write **one operation** that queries across multiple sources:

```graphql
# operations/ProductWithCountry.graphql
query ProductWithCountry($productId: ID!, $countryCode: ID!) {
  products_getProduct(id: $productId) { name price }
  countries_country(code: $countryCode) { name currency }
}
```

```
Client's single HTTP call
        │
        ▼
┌────────────────────┐
│  WunderGraph Server │
└──────────┬──────────┘
      ┌────┴────┐
      ▼         ▼
 Products API  Countries API   ← fetched in parallel, merged into one response
```

---

### 12. **What are "Custom Resolvers" / TypeScript operations in WunderGraph?**

**Answer:** When declarative composition isn't enough (custom business logic, calling non-API code), WunderGraph lets you write **TypeScript operations** that run server-side alongside `.graphql` operations.

```ts
// operations/CalculateDiscount.ts
import { createOperation, z } from "../generated/wundergraph.factory";

export default createOperation.query({
  input: z.object({
    productId: z.string(),
    userTier: z.enum(["bronze", "silver", "gold"]),
  }),
  handler: async ({ input, operations }) => {
    const product = await operations.query({
      operationName: "GetProduct",
      input: { id: input.productId },
    });

    const discountRates = { bronze: 0.05, silver: 0.1, gold: 0.2 };
    const discount = product.data!.price * discountRates[input.userTier];

    return { finalPrice: product.data!.price - discount };
  },
});
```

```
Two operation types, same generated-client interface:

operations/GetCountry.graphql   → declarative, composed automatically
operations/CalculateDiscount.ts → imperative, calls OTHER operations + custom logic

Both show up identically as: wundergraph.query.CalculateDiscount({ input })
```

> This is the WunderGraph equivalent of an Apollo custom resolver — but it composes **other WunderGraph operations** as building blocks, rather than reaching into raw data sources directly.

---

### 13. **How does authentication work in WunderGraph?**

**Answer:** WunderGraph has **first-class auth** built into the server layer (not something you wire up per-resolver).

```ts
configureWunderGraphApplication({
  authentication: {
    cookieBased: {
      providers: [
        authProviders.github({
          id: "github",
          clientId: new EnvironmentVariable("GITHUB_CLIENT_ID"),
          clientSecret: new EnvironmentVariable("GITHUB_CLIENT_SECRET"),
        }),
      ],
    },
  },
});
```

```graphql
# operations/GetMyProfile.graphql
# @wg-authentication-required
query GetMyProfile {
  db_findFirstuser(where: { id: { equals: "$$user.id$$" } }) {
    name
    email
  }
}
```

```
Request flow:

Browser ──► OAuth login (GitHub) ──► WunderGraph sets secure httpOnly cookie
   │
   ▼
GET /operations/GetMyProfile
   │
   ▼
WunderGraph validates cookie session ──► injects $$user.id$$ into operation
   │
   ▼
Rejects with 401 if @wg-authentication-required and no valid session
```

| Feature                 | Benefit                                              |
| -------------------------- | ------------------------------------------------------- |
| `cookieBased` sessions      | No manual JWT handling on the client                     |
| `@wg-authentication-required` | Declarative per-operation auth, enforced by the server   |
| `$$user.id$$` injection      | Safe way to scope queries to the logged-in user            |

---

### 14. **How does WunderGraph prevent API abuse by default?**

**Answer:** Unlike a raw GraphQL endpoint (where anyone can send arbitrary, deeply nested queries), WunderGraph is **allowlist-only by design**.

```
Raw GraphQL server:                    WunderGraph:

Client sends ANY query string   ──►    Client can ONLY call operations
Server tries to execute it              that exist as compiled .graphql/.ts
                                         files in your repo

Risk: malicious deeply-nested          Risk surface: limited to operations
queries, introspection abuse,           YOU wrote and reviewed in code review
resource exhaustion attacks
```

| Default protection                | How                                                          |
| ------------------------------------ | ----------------------------------------------------------------- |
| No ad-hoc queries                    | Only pre-compiled operations are reachable via HTTP                 |
| Introspection disabled to clients    | Clients can't discover/query your full unified schema directly       |
| Per-operation auth                    | `@wg-authentication-required` directive                              |
| Rate limiting                          | Configurable per operation                                            |
| CORS allowlist                         | `cors.allowedOrigins` in config                                        |

> This is philosophically the same protection as **Apollo persisted queries + query allowlisting**, except in WunderGraph it's the default behavior, not an opt-in add-on.

---

## Deployment & Advanced Patterns

### 15. **What is the WunderGraph development workflow (`wunderctl`)?**

**Answer:**

```
1. wunderctl init                  → scaffolds a new project
2. Edit wundergraph.config.ts       → declare data sources
3. wunderctl generate               → introspects APIs, builds unified schema + types
4. Write operations/*.graphql       → your app's queries/mutations
5. wunderctl up (or `npm run start`)→ runs dev server with hot-reload
6. wunderctl generate (again)       → regenerates types whenever operations/config change
```

```
File watcher loop during development:

  Edit operations/GetBooks.graphql
            │
            ▼
   wunderctl detects change
            │
            ▼
   Regenerates: schema, TS types, client SDK
            │
            ▼
   Frontend hot-reloads with new types instantly
```

---

### 16. **How do you handle errors in a WunderGraph operation?**

**Answer:**

```ts
// operations/GetOrder.ts
import { createOperation, z } from "../generated/wundergraph.factory";
import { OperationError } from "@wundergraph/sdk/operations";

class OrderNotFoundError extends OperationError {
  statusCode = 404;
  code = "OrderNotFound" as const;
  message = "Order does not exist";
}

export default createOperation.query({
  input: z.object({ orderId: z.string() }),
  errors: [OrderNotFoundError],
  handler: async ({ input, operations }) => {
    const order = await operations.query({
      operationName: "FindOrder",
      input: { id: input.orderId },
    });
    if (!order.data) throw new OrderNotFoundError();
    return order.data;
  },
});
```

```tsx
// Client side — errors are typed too
const { data, error } = useQuery({ operationName: "GetOrder", input: { orderId } });

if (error?.code === "OrderNotFound") {
  // TypeScript knows this error code is possible for this operation
}
```

> Declaring `errors: [...]` on a TypeScript operation makes the possible error codes part of the **generated client's type signature** — the frontend gets compile-time awareness of what can go wrong, not just a generic `Error`.

---

### 17. **How does WunderGraph handle caching?**

**Answer:** Because query operations map to HTTP `GET`, WunderGraph can leverage standard **HTTP caching** semantics — no custom cache protocol needed.

```graphql
# operations/GetCountry.graphql
# @wg-cache(maxAge: 60, staleWhileRevalidate: 300)
query GetCountry($code: ID!) {
  countries_country(code: $code) { name }
}
```

```
Cache-Control: max-age=60, stale-while-revalidate=300

t=0s    Request → MISS → fetch upstream → cache
t=30s   Request → HIT (fresh) → served instantly from cache
t=90s   Request → STALE → serve stale copy, revalidate in background
t=400s  Request → MISS (expired) → fetch upstream again
```

| Layer                | Mechanism                                    |
| ----------------------- | ------------------------------------------------ |
| Operation-level          | `@wg-cache` directive per `.graphql` file           |
| CDN / edge                | Standard HTTP cache headers, works with any CDN      |
| Client                     | Generated hooks respect cache headers automatically   |

---

### 18. **How would you deploy a WunderGraph application to production?**

**Answer:**

```
                     ┌─────────────────────┐
   Client (React) ──►│   CDN / Edge Cache   │──► cached GET operations served instantly
                     └──────────┬───────────┘
                                │ cache miss
                                ▼
                     ┌─────────────────────┐
                     │  WunderGraph Server  │  ← Docker container / Node process
                     │  (wunderctl start)   │
                     └──────────┬───────────┘
                     ┌──────────┼──────────┐
                     ▼          ▼          ▼
                 GraphQL API  REST API   Database
```

```dockerfile
# Typical production build
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm install && npx wunderctl generate
CMD ["npx", "wunderctl", "start"]
```

| Consideration          | Guidance                                                      |
| ------------------------- | ------------------------------------------------------------------ |
| Build step                 | Run `wunderctl generate` at build time, not runtime                  |
| Environment variables       | Store upstream API URLs/secrets via `EnvironmentVariable(...)`         |
| Horizontal scaling            | WunderGraph server is stateless — scale replicas behind a load balancer |
| Edge caching                    | Pair with a CDN for `@wg-cache` operations for global low latency         |

---

### 19. **What are the trade-offs of using WunderGraph vs a hand-rolled BFF (Backend-for-Frontend)?**

**Answer:**

| Aspect                     | Hand-rolled BFF (e.g. Express + custom resolvers) | WunderGraph                                       |
| ----------------------------- | ------------------------------------------------------ | ------------------------------------------------------ |
| Setup speed                     | Slower — write auth, caching, types manually              | Faster — declarative config, generated everything          |
| Flexibility                       | ✅ Full control over every request                         | ⚠️ Some patterns must fit WunderGraph's operation model     |
| Type safety                         | Manual (or extra codegen tooling)                            | ✅ Built-in, automatic end-to-end                            |
| Security defaults                     | You must implement allowlisting yourself                        | ✅ Allowlist-by-default ([Q14](#14-how-does-wundergraph-prevent-api-abuse-by-default)) |
| Learning curve                          | Familiar if you already know Express/Fastify/etc.                 | New concepts: operations, `wunderctl`, config DSL              |
| Vendor/ecosystem lock-in                  | None — plain Node.js                                                | Tied to WunderGraph's tooling and conventions                    |

> **When to choose WunderGraph:** you have several existing APIs (GraphQL/REST/DB) to unify quickly with strong type safety and secure-by-default behavior, and you're comfortable working within its operation-based model.
>
> **When to choose a hand-rolled BFF:** you need very custom request handling that doesn't fit the declarative operation model, or want zero framework lock-in.

---

### 20. **How does WunderGraph's security model compare conceptually to Apollo Federation + persisted queries?**

**Answer:** Both aim to solve "expose GraphQL safely," but from different defaults:

```
Apollo (opt-in security):                 WunderGraph (secure by default):

Raw GraphQL endpoint exposed              No raw GraphQL endpoint exposed
        │                                          │
        ▼                                          ▼
You ADD: persisted queries,               You get: only compiled operations
  depth limiting, query allowlist            reachable, out of the box
  (extra libraries/config)                   (built into the platform)
        │                                          │
        ▼                                          ▼
Federation composes multiple              introspect.* composes multiple
  GraphQL subgraphs behind a router          APIs (GraphQL/REST/DB) behind
  — clients still speak GraphQL               a unified schema — clients call
                                               generated typed functions, not GraphQL
```

| Dimension                  | Apollo Federation                              | WunderGraph                                       |
| ----------------------------- | -------------------------------------------------- | -------------------------------------------------------- |
| What composes                    | Multiple **GraphQL** subgraphs                       | Multiple **any-protocol** APIs (GraphQL, REST, DB, gRPC)    |
| Client-facing protocol             | GraphQL (query language exposed)                       | Generated typed RPC-style functions (GraphQL hidden)          |
| Default exposure                     | Full schema queryable (unless you add restrictions)      | Only pre-registered operations queryable                        |
| Best fit                               | Large orgs standardizing on GraphQL internally             | Teams wanting a secure, typed BFF over heterogeneous APIs         |

> **Takeaway for interviews:** Federation solves *"how do many GraphQL teams share one graph,"* while WunderGraph solves *"how do I safely expose a typed API to my frontend without hand-writing a BFF or trusting clients with raw GraphQL."* They're not mutually exclusive — WunderGraph can even use a federated gateway as one of its composed data sources.
