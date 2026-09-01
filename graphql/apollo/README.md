# Apollo GraphQL Interview Questions & Answers

A **novice-to-medium** collection of Apollo GraphQL Q&A — Apollo Server, Apollo Client, and Apollo Federation — with diagrams and practical examples, growing gradually from fundamentals to more complex patterns.

---

## Fundamentals

### 1. **What is GraphQL and how is Apollo related to it?**

**Answer:** GraphQL is a **query language for APIs** and a runtime for executing those queries against your data. Unlike REST (many endpoints, fixed shapes), GraphQL exposes **one endpoint** and lets clients ask for exactly the fields they need.

Apollo is a company/ecosystem that builds the most popular GraphQL tooling on top of the GraphQL spec:

```
        GraphQL (the spec/language)
                 │
   ┌─────────────┼─────────────┐
   ▼             ▼             ▼
Apollo Server  Apollo Client  Apollo Federation
(runs a         (fetches/     (composes many
 GraphQL API)    caches on     GraphQL services
                 the client)   into one graph)
```

| Piece                | Runs where | Purpose                                  |
| --------------------- | ---------- | ----------------------------------------- |
| **Apollo Server**     | Backend    | Executes queries, resolves data           |
| **Apollo Client**     | Frontend   | Sends queries, caches responses           |
| **Apollo Federation** | Backend    | Combines multiple GraphQL APIs into one   |
| **Apollo Studio**     | Cloud      | Schema registry, metrics, observability   |

---

### 2. **What are the core building blocks of a GraphQL schema?**

**Answer:** A schema is written in the **Schema Definition Language (SDL)** and defines the shape of your API.

```graphql
type Book {
  id: ID!
  title: String!
  author: Author!
  publishedYear: Int
}

type Author {
  id: ID!
  name: String!
  books: [Book!]!
}

type Query {
  books: [Book!]!
  book(id: ID!): Book
}

type Mutation {
  addBook(title: String!, authorId: ID!): Book!
}
```

| Concept       | Meaning                                              |
| ------------- | ----------------------------------------------------- |
| `type`        | An object shape with fields                           |
| `Query`       | Entry points for reading data                         |
| `Mutation`    | Entry points for writing/changing data                |
| `Subscription`| Entry points for real-time updates                    |
| `!`           | Field is **non-nullable**                             |
| `[Book!]!`    | A non-null list of non-null `Book`s                   |
| Scalars       | Built-in types: `ID`, `String`, `Int`, `Float`, `Boolean` |

---

### 3. **How do you set up a basic Apollo Server?**

**Answer:**

```ts
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";

const typeDefs = `#graphql
  type Book {
    title: String!
    author: String!
  }

  type Query {
    books: [Book!]!
  }
`;

const books = [
  { title: "The Hobbit", author: "J.R.R. Tolkien" },
  { title: "Dune", author: "Frank Herbert" },
];

const resolvers = {
  Query: {
    books: () => books,
  },
};

const server = new ApolloServer({ typeDefs, resolvers });

const { url } = await startStandaloneServer(server, {
  listen: { port: 4000 },
});

console.log(`🚀 Server ready at ${url}`);
```

```
Client Request                  Apollo Server
───────────────                 ─────────────
query {           ───POST──►    1. Parse query against typeDefs
  books {                       2. Validate (does `title` exist on Book?)
    title                       3. Execute resolvers
  }                              4. Return JSON matching query shape
}                  ◄───JSON────
```

---

### 4. **What are resolvers and how does resolution work?**

**Answer:** A **resolver** is a function that returns the data for a single field. GraphQL execution walks the query tree and calls one resolver per field.

```ts
const resolvers = {
  Query: {
    book: (parent, args, context, info) => {
      return books.find((b) => b.id === args.id);
    },
  },
  Book: {
    // Called once per Book returned, to resolve the nested `author` field
    author: (parent, args, context) => {
      return authors.find((a) => a.id === parent.authorId);
    },
  },
};
```

The 4 resolver arguments:

| Argument  | Meaning                                              |
| --------- | ----------------------------------------------------- |
| `parent`  | The result of the parent field's resolver              |
| `args`    | Arguments passed in the query, e.g. `{ id: "1" }`      |
| `context` | Shared object per-request (auth, DB connections)       |
| `info`    | Metadata about the query (rarely used directly)        |

```
Query: book(id: "1") { title author { name } }

Resolution order:
  Query.book(parent=undefined, args={id:"1"})  → returns book object
       │
       ▼
  Book.title    → resolved from parent.title (default resolver)
  Book.author(parent=book)  → returns author object
       │
       ▼
  Author.name   → resolved from parent.name (default resolver)
```

> **Default resolver:** If you don't define a resolver for a field, Apollo uses a default one that reads `parent[fieldName]`.

---

### 5. **What is the `context` object used for in Apollo Server?**

**Answer:** `context` is built **once per request** and passed to every resolver — the standard place for auth, data loaders, and DB clients.

```ts
const server = new ApolloServer({ typeDefs, resolvers });

const { url } = await startStandaloneServer(server, {
  context: async ({ req }) => {
    const token = req.headers.authorization || "";
    const user = await getUserFromToken(token);
    return { user, db: dbClient };
  },
});

// Inside a resolver:
const resolvers = {
  Query: {
    myOrders: (parent, args, context) => {
      if (!context.user) throw new Error("Unauthenticated");
      return context.db.orders.findByUser(context.user.id);
    },
  },
};
```

---

## Queries, Mutations & Client Basics

### 6. **How do queries, mutations, and subscriptions differ?**

**Answer:**

| Operation        | Purpose              | Analogy (REST)   | Execution              |
| ----------------- | --------------------- | ----------------- | ------------------------ |
| `query`           | Read data              | `GET`              | Fields resolved in parallel |
| `mutation`        | Write/change data      | `POST/PUT/DELETE`  | Fields resolved **serially** (top-level) |
| `subscription`    | Real-time stream       | WebSocket/SSE      | Long-lived connection, pushes events |

```graphql
mutation AddBook {
  addBook(title: "Dune", authorId: "2") {
    id
    title
  }
}

subscription OnBookAdded {
  bookAdded {
    id
    title
  }
}
```

> **Why mutations run serially:** if you send two mutations in one request, GraphQL guarantees the first fully completes before the second starts — avoiding race conditions like double-charging a wallet.

---

### 7. **How do you set up Apollo Client in a React app?**

**Answer:**

```tsx
import {
  ApolloClient,
  InMemoryCache,
  ApolloProvider,
  gql,
  useQuery,
} from "@apollo/client";

const client = new ApolloClient({
  uri: "http://localhost:4000/graphql",
  cache: new InMemoryCache(),
});

function App() {
  return (
    <ApolloProvider client={client}>
      <BookList />
    </ApolloProvider>
  );
}

const GET_BOOKS = gql`
  query GetBooks {
    books {
      id
      title
      author
    }
  }
`;

function BookList() {
  const { loading, error, data } = useQuery(GET_BOOKS);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <ul>
      {data.books.map((book: any) => (
        <li key={book.id}>{book.title}</li>
      ))}
    </ul>
  );
}
```

```
useQuery lifecycle:
  mount ──► loading:true ──► fetch ──► cache write ──► loading:false, data set
                                              │
                              (re-render triggered automatically)
```

---

### 8. **How do you run a mutation from Apollo Client?**

**Answer:**

```tsx
import { gql, useMutation } from "@apollo/client";

const ADD_BOOK = gql`
  mutation AddBook($title: String!, $authorId: ID!) {
    addBook(title: $title, authorId: $authorId) {
      id
      title
    }
  }
`;

function AddBookForm() {
  const [addBook, { loading, error }] = useMutation(ADD_BOOK, {
    refetchQueries: ["GetBooks"], // Re-run named queries after mutation
  });

  const handleSubmit = (title: string, authorId: string) => {
    addBook({ variables: { title, authorId } });
  };

  return <button onClick={() => handleSubmit("Dune", "2")}>Add</button>;
}
```

| Option           | Purpose                                             |
| ----------------- | ----------------------------------------------------- |
| `refetchQueries`  | Re-fetch specific queries after mutation completes     |
| `optimisticResponse` | Show a fake result immediately, before server responds |
| `update`          | Manually patch the cache with the mutation result      |

---

### 9. **What are variables and why use them instead of string interpolation?**

**Answer:** Variables let you parameterize a query **without** building query strings manually — enabling caching, validation, and safe reuse.

```graphql
query GetBook($id: ID!) {
  book(id: $id) {
    title
    author
  }
}
```

```json
// Sent alongside the query
{ "id": "42" }
```

```
❌ Bad (string interpolation):
`query { book(id: "${userInput}") { title } }`
→ Different string per id → cache can't recognize it's the "same" query
→ Vulnerable if userInput isn't sanitized

✅ Good (variables):
Same query document + different variables
→ Apollo can cache/normalize based on the query's shape
```

---

### 10. **What are fragments and why are they useful?**

**Answer:** A **fragment** is a reusable set of fields you can share across multiple queries/mutations.

```graphql
fragment BookFields on Book {
  id
  title
  publishedYear
}

query GetBooks {
  books {
    ...BookFields
    author {
      name
    }
  }
}

query GetBook($id: ID!) {
  book(id: $id) {
    ...BookFields
  }
}
```

> **Benefit:** Change the fragment once, and every query using it updates — reduces duplication and keeps UI components' data needs colocated (a common pattern: each React component defines its own fragment).

---

## Apollo Client Caching

### 11. **How does Apollo Client's normalized cache work?**

**Answer:** Apollo Client's `InMemoryCache` **normalizes** data — every object with an `id`/`__typename` is stored once in a flat lookup table, not nested inside each query result.

```
Query result (nested):                   Normalized cache (flat):
{                                         {
  book: {                                   "Book:1": {
    id: "1",                                  id: "1",
    title: "Dune",                            title: "Dune",
    author: {                                 author: { __ref: "Author:2" }
      id: "2",                              },
      name: "Frank Herbert"                 "Author:2": {
    }                                          id: "2",
  }                                            name: "Frank Herbert"
}                                           }
                                          }
```

```
Cache key = __typename + id  (e.g. "Book:1")

Benefit: If ANY query updates Author:2's name,
every component displaying that author re-renders
with the new value — automatically, no manual sync.
```

```ts
const cache = new InMemoryCache({
  typePolicies: {
    Book: {
      keyFields: ["id"], // customize how cache keys are built
    },
  },
});
```

---

### 12. **What are the different `fetchPolicy` options in Apollo Client?**

**Answer:**

| Policy                | Behavior                                                    |
| ---------------------- | -------------------------------------------------------------- |
| `cache-first` (default)| Use cache if present; otherwise hit network                    |
| `cache-and-network`    | Return cache immediately, **also** fetch from network and update |
| `network-only`         | Always hit network, but still update the cache                 |
| `cache-only`           | Never hit network — error if not in cache                      |
| `no-cache`             | Always hit network, don't write to cache                       |

```tsx
const { data } = useQuery(GET_BOOKS, {
  fetchPolicy: "cache-and-network",
});
```

```
cache-first:                      cache-and-network:
┌────────┐                        ┌────────┐
│ cache? │──yes──► return         │ return cache (stale-ok)
└───┬────┘                        └───┬────┘
    no                                │
    ▼                                 ▼
 network ──► cache ──► return    network ──► cache ──► re-render with fresh data
```

---

### 13. **How does Apollo Client update the cache after a mutation?**

**Answer:** Three strategies, from simplest to most manual:

```tsx
// 1. Automatic — if the mutation returns fields matching a cached object
//    with the same id/__typename, the cache updates automatically.
const [updateBook] = useMutation(gql`
  mutation UpdateBook($id: ID!, $title: String!) {
    updateBook(id: $id, title: $title) {
      id      # Apollo matches "Book:1" in cache and merges the new title
      title
    }
  }
`);

// 2. update() — manually write to cache (e.g. for list additions)
const [addBook] = useMutation(ADD_BOOK, {
  update(cache, { data: { addBook } }) {
    cache.modify({
      fields: {
        books(existingBooks = []) {
          const newBookRef = cache.writeFragment({
            data: addBook,
            fragment: gql`fragment NewBook on Book { id title }`,
          });
          return [...existingBooks, newBookRef];
        },
      },
    });
  },
});

// 3. refetchQueries — brute force, simplest, extra network round trip
useMutation(ADD_BOOK, { refetchQueries: ["GetBooks"] });
```

---

### 14. **What is optimistic UI in Apollo Client?**

**Answer:** `optimisticResponse` lets the UI update **instantly**, assuming the mutation will succeed, then reconciles with the real server response (or rolls back on error).

```tsx
const [likePost] = useMutation(LIKE_POST, {
  optimisticResponse: {
    likePost: {
      __typename: "Post",
      id: postId,
      likes: currentLikes + 1, // guessed result
    },
  },
});
```

```
Timeline:

t=0ms   User clicks "like"
t=1ms   Cache updated with optimisticResponse → UI shows likes+1 instantly
t=250ms Server responds with real data
t=251ms Cache reconciled with real response (or rolled back if mutation failed)

User perceives: instant feedback, no spinner
```

---

## Directives, Errors & Schema Design

### 15. **What are GraphQL directives (`@include`, `@skip`, custom)?**

**Answer:** Directives conditionally alter query execution or add schema behavior.

```graphql
query GetProfile($withEmail: Boolean!) {
  user {
    name
    email @include(if: $withEmail)   # only fetched if withEmail=true
    phone @skip(if: $withEmail)      # skipped if withEmail=true
  }
}
```

```graphql
# Custom directive (defined server-side)
directive @auth(requires: Role = USER) on FIELD_DEFINITION

type Mutation {
  deleteUser(id: ID!): Boolean @auth(requires: ADMIN)
}
```

| Built-in directive | Purpose                          |
| ------------------- | ----------------------------------- |
| `@include(if:)`     | Include field conditionally         |
| `@skip(if:)`        | Skip field conditionally            |
| `@deprecated`       | Mark a field/enum value deprecated  |

---

### 16. **How does error handling work in GraphQL/Apollo?**

**Answer:** GraphQL always returns HTTP 200 (usually) — errors live in an `errors` array **alongside** `data`, since a query can partially succeed.

```json
{
  "data": {
    "book": { "title": "Dune" },
    "author": null
  },
  "errors": [
    {
      "message": "Author not found",
      "path": ["author"],
      "extensions": { "code": "NOT_FOUND" }
    }
  ]
}
```

```ts
import { GraphQLError } from "graphql";

const resolvers = {
  Query: {
    author: (_, { id }) => {
      const author = authors.find((a) => a.id === id);
      if (!author) {
        throw new GraphQLError("Author not found", {
          extensions: { code: "NOT_FOUND", http: { status: 404 } },
        });
      }
      return author;
    },
  },
};
```

```tsx
// Client side
const { data, error } = useQuery(GET_AUTHOR);
if (error) {
  error.graphQLErrors.forEach((e) => console.log(e.extensions.code));
}
```

---

### 17. **What is N+1 query problem and how does DataLoader solve it?**

**Answer:** Without batching, resolving `author` for **each** book triggers a **separate** DB query — N books = N queries (+1 for the books query itself).

```
❌ N+1 problem:
Query: books { title author { name } }

books()        → 1 query: SELECT * FROM books
author(book1)  → 1 query: SELECT * FROM authors WHERE id=1
author(book2)  → 1 query: SELECT * FROM authors WHERE id=2
author(book3)  → 1 query: SELECT * FROM authors WHERE id=1  (duplicate!)
                  = 4 queries total for 3 books
```

```ts
import DataLoader from "dataloader";

const authorLoader = new DataLoader(async (ids: readonly string[]) => {
  const authors = await db.authors.findByIds(ids);
  // Must return results in the SAME order as `ids`
  return ids.map((id) => authors.find((a) => a.id === id));
});

const resolvers = {
  Book: {
    author: (book, args, context) => context.authorLoader.load(book.authorId),
  },
};
```

```
✅ With DataLoader (batches + caches within one request):

author(book1) ─┐
author(book2) ─┼─► DataLoader batches into ─► 1 query: WHERE id IN (1,2)
author(book3) ─┘     (dedupes id=1 automatically)
                  = 2 queries total for 3 books
```

> Create a **new DataLoader instance per request** (in `context`) — never share across requests, or you'll leak cached data between users.

---

### 18. **How do you paginate a GraphQL API (cursor-based / Relay-style)?**

**Answer:**

```graphql
type BookConnection {
  edges: [BookEdge!]!
  pageInfo: PageInfo!
}

type BookEdge {
  node: Book!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}

type Query {
  books(first: Int!, after: String): BookConnection!
}
```

```graphql
query {
  books(first: 10, after: "cursor-abc") {
    edges {
      node { title }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

```
Page 1                          Page 2
┌─────────────┐                 ┌─────────────┐
│ book1..10   │  endCursor=X    │ book11..20  │
│ hasNext:true│──after: X─────► │ hasNext:true│
└─────────────┘                 └─────────────┘
```

```tsx
// Apollo Client's fetchMore for infinite scroll
const { data, fetchMore } = useQuery(GET_BOOKS, {
  variables: { first: 10 },
});

fetchMore({
  variables: { after: data.books.pageInfo.endCursor },
});
```

> **Why cursors over offset/limit:** cursors are stable even if items are inserted/deleted between page loads; `OFFSET 10000` in SQL also gets slow at scale.

---

### 19. **What is schema stitching / Apollo Federation, and why use it?**

**Answer:** As an app grows, one monolithic GraphQL schema becomes unwieldy. **Federation** lets multiple teams own separate GraphQL services ("subgraphs") that compose into a single unified graph via a **gateway (router)**.

```
                Client
                  │
                  ▼
          ┌───────────────┐
          │ Apollo Router  │  ← single GraphQL endpoint clients call
          │  (Gateway)     │
          └───────┬───────┘
        ┌──────────┼──────────┐
        ▼          ▼          ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Users   │ │ Products│ │ Reviews │   ← independent subgraphs,
   │ subgraph│ │ subgraph│ │ subgraph│      owned by different teams
   └─────────┘ └─────────┘ └─────────┘
```

```graphql
# Products subgraph
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
}

# Reviews subgraph — extends Product with a new field
extend type Product @key(fields: "id") {
  id: ID! @external
  reviews: [Review!]!
}
```

```graphql
# Client sends ONE query, router fans it out to both subgraphs
query {
  product(id: "1") {
    name       # from Products subgraph
    reviews {  # from Reviews subgraph — stitched together automatically
      rating
    }
  }
}
```

| Benefit                    | Why it matters                                    |
| ---------------------------- | ---------------------------------------------------- |
| Team autonomy                 | Each team owns/deploys its subgraph independently    |
| Single client-facing API      | Frontend still queries one unified graph              |
| `@key` directive               | Tells the router how to join entities across subgraphs |

---

### 20. **What are best practices for designing a production Apollo GraphQL API?**

**Answer:**

| Practice                          | Why                                                          |
| ----------------------------------- | ---------------------------------------------------------------- |
| Use DataLoader for all relations     | Avoid N+1 queries ([Q17](#17-what-is-n1-query-problem-and-how-does-dataloader-solve-it)) |
| Set query complexity/depth limits    | Prevent malicious deeply-nested queries from DOSing the server    |
| Persisted queries                    | Client sends a hash, not full query text — reduces payload & allowlists queries |
| Version fields with `@deprecated`, not new endpoints | GraphQL evolves in-place; avoid `/v2` sprawl        |
| Use `context` for auth, never resolver args | Keep auth centralized and consistent                       |
| Paginate all lists (cursor-based)    | Avoid unbounded result sets ([Q18](#18-how-do-you-paginate-a-graphql-api-cursor-based--relay-style)) |
| Structured error codes (`extensions.code`) | Let clients branch on error type reliably ([Q16](#16-how-does-error-handling-work-in-graphqlapollo)) |
| Monitor with Apollo Studio / tracing  | Catch slow resolvers and schema usage in production               |

```ts
// Example: limiting query depth to prevent abuse
import depthLimit from "graphql-depth-limit";

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [depthLimit(7)],
});
```

```graphql
# Persisted query — client sends only a hash, server has the full query registered
# POST /graphql
# { "extensions": { "persistedQuery": { "sha256Hash": "abc123..." } } }
```

> **Golden rule:** GraphQL's flexibility is also its risk surface — a client can request unbounded nested data. Always pair GraphQL with depth limiting, complexity analysis, and pagination in production.
