# TypeScript Interview Questions & Answers

A comprehensive guide featuring well-structured questions and answers with code examples for mid-to-advanced TypeScript interviews. Covers type-level programming, design patterns, utility types, and modern TypeScript 5.x features.

---

### 1. What is the difference between `interface` and `type`?

```ts
interface User {
  name: string;
}

type Admin = {
  role: string;
};
```

### 2. Can interfaces be merged?

```ts
interface Box {
  size: number;
}

interface Box {
  label: string;
}
```

### 3. How does structural typing work in TypeScript?

```ts
interface Point {
  x: number;
  y: number;
}

function logPoint(p: Point) {
  console.log(p.x, p.y);
}

logPoint({ x: 1, y: 2, z: 3 });
```

### 4. Explain `keyof`, `typeof`, and `in`.

```ts
const user = { name: "Ankur", age: 30 };
type Keys = keyof typeof user;

type Mapped = {
  [K in Keys]: string;
};
```

### 5. What is type inference?

```ts
let age = 30; // inferred as number
```

### 6. What is declaration merging?

```ts
interface Window {
  title: string;
}

interface Window {
  width: number;
}
```

### 7. What is the difference between `unknown` and `any`?

```ts
let a: unknown = 10;
a.toFixed(); // Error

let b: any = 10;
b.toFixed(); // OK
```

### 8. How does type narrowing work?

```ts
function log(value: string | number) {
  if (typeof value === "string") {
    console.log(value.toUpperCase());
  } else {
    console.log(value.toFixed(2));
  }
}
```

### 9. Explain type guards.

```ts
function isString(value: any): value is string {
  return typeof value === "string";
}
```

### 10. What is a discriminated union?

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number };

function area(shape: Shape) {
  if (shape.kind === "circle") return Math.PI * shape.radius ** 2;
  return shape.side ** 2;
}
```

### 11. What is the `never` type?

```ts
function throwError(): never {
  throw new Error("Something went wrong");
}
```

### 12. What are utility types?

```ts
type User = { name: string; age: number };
type PartialUser = Partial<User>;
type RequiredUser = Required<User>;
type ReadonlyUser = Readonly<User>;
type PickName = Pick<User, "name">;
type OmitAge = Omit<User, "age">;
```

### 13. Difference between `readonly` and `const`?

```ts
const x = 10; // immutable variable

const obj: Readonly<{ name: string }> = { name: "Ankur" };
// obj.name = 'John'; // Error
```

### 14. How to create a deep readonly type?

```ts
type DeepReadonly<T> = {
  readonly [K in keyof T]: DeepReadonly<T[K]>;
};
```

### 15. Difference between `interface` and `abstract class`?

```ts
abstract class Animal {
  abstract makeSound(): void;
  move() {
    console.log("Moving");
  }
}

interface IAnimal {
  makeSound(): void;
}
```

### 16. What is `satisfies` operator in TS 4.9+?

```ts
const config = {
  theme: "dark",
  debug: true,
} satisfies { theme: string; debug: boolean };
```

### 17. What are conditional types?

```ts
type Message<T> = T extends string ? "Text" : "Other";
```

### 18. What is the `infer` keyword used for?

```ts
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

### 19. What is type assertion vs type casting?

```ts
const value: any = "Hello";
const length = (value as string).length;
```

### 20. Difference between enums and union types?

```ts
enum RoleEnum {
  Admin,
  User,
}

type RoleUnion = "Admin" | "User";
```

### 21. Explain mapped types.

```ts
type ReadOnly<T> = {
  readonly [P in keyof T]: T[P];
};
```

### 22. How do template literal types work?

```ts
type Status = "success" | "failure";
type Message = `Server responded with ${Status}`;
```

### 23. How to enforce exact object types?

```ts
function acceptExact(obj: { name: string }) {
  return obj;
}

acceptExact({ name: "Ankur", extra: true }); // Error
```

### 24. What is a tuple in TypeScript?

```ts
let point: [number, number] = [10, 20];
```

### 25. What is a hybrid type?

```ts
interface Counter {
  (): number;
  count: number;
}
```

### 26. Explain `this` parameter typing.

```ts
function show(this: HTMLInputElement) {
  console.log(this.value);
}
```

### 27. What is a declaration file?

```ts
declare module "my-lib" {
  export function greet(name: string): string;
}
```

### 28. Difference between global, module, and file scope?

```ts
export const x = 5; // now this file is a module
```

### 29. How to avoid name collisions in declaration files?

```ts
declare namespace MyLib {
  export interface Config {
    key: string;
  }
}
```

### 30. What is module augmentation?

```ts
declare module "express" {
  interface Request {
    user?: string;
  }
}
```

### 31. Explain `as const`.

```ts
const config = { mode: "dark" } as const;
```

### 32. How do recursive types work?

```ts
type Tree<T> = {
  value: T;
  children?: Tree<T>[];
};
```

### 33. What is excess property checking?

```ts
function take(user: { name: string }) {}
take({ name: "Ankur", age: 30 }); // Error
```

### 34. Explain `noUncheckedIndexedAccess` flag.

```ts
let arr: string[] = [];
let name: string = arr[0]; // Error if flag is on
```

### 35. How to narrow types for arrays of discriminated unions?

```ts
const shapes: Shape[] = [...];
const circles = shapes.filter(
  (s): s is { kind: 'circle'; radius: number } => s.kind === 'circle'
);
```

### 36. What are ambient declarations?

```ts
declare const VERSION: string;
```

### 37. What are template recursive types?

```ts
type Path<T> = T extends object
  ? {
      [K in keyof T]: `${K & string}` | `${K & string}.${Path<T[K]>}`;
    }[keyof T]
  : never;
```

### 38. How does type compatibility differ from assignability?

Compatibility is structural; assignability includes additional compile-time checks for literals, constants, etc.

### 39. What is a branded type?

```ts
type USD = string & { __brand: "USD" };
```

### 40. What is module resolution?

TypeScript resolves modules using the Node.js module resolution algorithm and `tsconfig.json`.

### 41. Difference between `export =` and `export default`?

```ts
// CommonJS:
export = function greet() {};

// ESModule:
export default function greet() {}
```

### 42. How to use generics with constraints?

```ts
function logLength<T extends { length: number }>(input: T) {
  console.log(input.length);
}
```

### 43. What is a function overload in TypeScript?

```ts
function combine(a: string, b: string): string;
function combine(a: number, b: number): number;
function combine(a: any, b: any): any {
  return a + b;
}
```

### 44. Difference between `void` and `undefined`?

```ts
function doSomething(): void {}
let x: undefined = undefined;
```

### 45. What is declaration bundling?

```bash
tsc --declaration --emitDeclarationOnly --outFile index.d.ts
```

### 46. How does `Exclude` work?

```ts
type T = Exclude<"a" | "b" | "c", "a">; // 'b' | 'c'
```

### 47. Explain module isolation.

```ts
export const x = 1;
import { x } from "./file1";
```

### 48. What is non-null assertion?

```ts
let name: string | undefined;
console.log(name!.length);
```

### 49. What is a phantom type?

```ts
type Tagged<T> = { __type?: T };
type UserId = string & Tagged<"UserId">;
```

### 50. How to build a type-safe event system?

```ts
type Events = {
  login: { userId: string };
  logout: void;
};

class EventBus {
  on<K extends keyof Events>(
    event: K,
    handler: (payload: Events[K]) => void,
  ): void {
    // implementation
  }
}
```

### 51. **What are `const` type parameters in TS 5.0?**

```ts
function createPair<const T extends readonly [unknown, unknown]>(pair: T): T {
  return pair;
}

const result = createPair([1, "hello"]); // Type: readonly [1, "hello"]
// Without `const`, type would be (string | number)[]
```

### 52. **What is the `using` declaration in TS 5.2+?**

```ts
// Explicit Resource Management (ECMAScript proposal)
// The `using` keyword auto-disposes resources when they go out of scope
class DatabaseConnection {
  [Symbol.dispose]() {
    console.log("Connection closed");
  }
}

function doWork() {
  using db = new DatabaseConnection();
  // db is automatically disposed when scope exits
}
```

### 53. **What is the `NoInfer<T>` utility type in TS 5.4?**

```ts
// Prevents TypeScript from inferring a type parameter from a specific position
function createStreetLight<T extends string>(
  colors: T[],
  defaultColor: NoInfer<T>,
) {
  // defaultColor must be one of the colors, but won't widen the type
}

createStreetLight(["red", "yellow", "green"], "red"); // OK
createStreetLight(["red", "yellow", "green"], "blue"); // Error!
```

### 54. **What does `--exactOptionalPropertyTypes` do?**

```ts
type User = { name?: string }; // name is now truly optional and undefined must be explicitly passed
```

### 55. **How does TS 5.0 handle const enums in isolatedModules?**

```ts
// Must use preserveConstEnums to retain them in output
const enum Status {
  Success,
  Failure,
}
```

### 56. **What is the new `extends infer` use case in TS 4.9?**

```ts
type GetPromiseType<T> = T extends Promise<infer R> ? R : T;
```

### 57. **How does `in` operator narrowing improve post TS 4.9?**

```ts
function isDog(pet: any): pet is { bark: () => void } {
  return "bark" in pet;
}
```

### 58. **How does TS support `Object.hasOwn()` now?**

```ts
if (Object.hasOwn(obj, "key")) {
  console.log(obj.key); // key is now narrowed
}
```

### 59. **What are named tuple elements (TS 4.0+)?**

```ts
type Point = [x: number, y: number];
```

### 60. **What is `unknown` literal narrowing?**

```ts
function fn(x: unknown) {
  if (typeof x === "string") {
    x.toUpperCase(); // ok
  }
}
```

### 61. **What is `const` type parameter inference (TS 5.0)?**

```ts
const makePair = <T extends readonly unknown[]>(tuple: [...T]) => tuple;

const result = makePair([1, "a"] as const); // [1, 'a']
```

### 62. **How does `ReturnType<typeof fn>` improve developer workflow?**

```ts
function fetchUser() {
  return { name: "Ankur", age: 30 };
}
type User = ReturnType<typeof fetchUser>;
```

### 63. **What is `--verbatimModuleSyntax` introduced for?**

- Enforces that import/export matches ECMAScript module semantics exactly (no re-writing).

### 64. **What is the `using` declaration in TS 5.2+ (Explicit Resource Management)?**

```ts
// Implements the TC39 Explicit Resource Management proposal
// Resources with [Symbol.dispose] are auto-cleaned at end of scope
async function readFile() {
  await using file = await openFile("data.txt");
  // file is automatically closed when scope exits
}
```

### 65. **What is `infer extends` pattern in conditional types?**

```ts
type Flatten<T> = T extends (infer R)[] ? R : T;
```

### 66. **How do enums improve constant expression inference in TS 4.9+?**

```ts
const enum Colors {
  Red = 1,
  Green = 2,
}
```

### 67. **How does `extends` narrow to `never` improve inference?**

```ts
type IsString<T> = T extends string ? true : false;
```

### 68. **How does `--noPropertyAccessFromIndexSignature` affect safety?**

```ts
interface Person {
  [key: string]: string;
  name: string;
}

let p: Person = { name: "Ankur" };
console.log(p.name); // error if flag enabled
```

### 69. **How to use `Object.entries()` safely in TS 4.9+?**

```ts
const user = { name: "Ankur", age: 30 };
Object.entries(user).forEach(([key, value]) => {
  console.log(key, value); // key is still string, but can narrow better
});
```

### 70. **What is `readonly` array vs tuple behavior in newer TS?**

```ts
const values: readonly [string, number] = ["age", 30];
```

---

## 🔧 `tsconfig.json` and `tsc` Command-Specific (71–80)

### 71. **What does `tsc --init` generate?**

- A default `tsconfig.json` file with commented-out configurations.

### 72. **What is `include`, `exclude`, and `files` in tsconfig?**

```json
{
  "include": ["src/**/*"],
  "exclude": ["node_modules"],
  "files": ["index.ts"]
}
```

### 73. **What does `composite` option enable?**

```json
{
  "compilerOptions": {
    "composite": true
  }
}
```

- Enables project references and incremental builds.

### 74. **Difference between `target`, `module`, and `lib`?**

- `target`: output JS version
- `module`: module system (e.g., CommonJS, ESNext)
- `lib`: library type declarations to include

### 75. **How does `paths` and `baseUrl` work together?**

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

### 76. **What is `tsc --watch` used for?**

- Recompiles code on file change for fast feedback.

### 77. **What does `declaration: true` do?**

```json
{
  "compilerOptions": {
    "declaration": true
  }
}
```

- Generates `.d.ts` files for all source files.

### 78. **What is `tsconfig.build.json` used for?**

- Used as a separate config during build steps (e.g., `exclude: ["tests"]`).

### 79. **How does `tsc --build` (`tsbuild`) differ from `tsc`?**

- Used to compile **project references** or mono-repo builds.

### 80. **How to enable strictest type checking in tsconfig?**

```json
{
  "compilerOptions": {
    "strict": true
  }
}
```

---

🧠 This completes the extended series of **80 TypeScript interview Q&A**, now covering modern TS (4.9+), config tips, and advanced typing patterns.

---

## Scalability, Resiliency, Error Handling & Performance Patterns in TypeScript

### 81. **How do you build type-safe error handling in TypeScript?**

**Answer:** Use **discriminated unions** instead of thrown exceptions for expected errors. This forces callers to handle every case.

```ts
// Result type — no exceptions needed
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

type ValidationError = { code: "VALIDATION"; field: string; message: string };
type NotFoundError = { code: "NOT_FOUND"; resource: string };
type AppError = ValidationError | NotFoundError;

async function getUser(id: string): Promise<Result<User, AppError>> {
  if (!id.match(/^[a-f0-9-]{36}$/)) {
    return {
      success: false,
      error: { code: "VALIDATION", field: "id", message: "Invalid UUID" },
    };
  }
  const user = await db.findUser(id);
  if (!user) {
    return {
      success: false,
      error: { code: "NOT_FOUND", resource: `User:${id}` },
    };
  }
  return { success: true, data: user };
}

// Caller MUST handle both cases — compiler enforces it
const result = await getUser("abc");
if (!result.success) {
  switch (result.error.code) {
    case "VALIDATION":
      return res.status(400).json(result.error);
    case "NOT_FOUND":
      return res.status(404).json(result.error);
    // No default needed — TS ensures exhaustive check
  }
}
console.log(result.data.name); // TS knows data exists here
```

---

### 82. **How do you type retry logic and circuit breakers in TypeScript?**

**Answer:** Use generics to make resilience patterns type-safe and reusable.

```ts
interface RetryOptions {
  maxRetries: number;
  baseDelayMs: number;
  maxDelayMs: number;
}

async function withRetry<T>(
  fn: () => Promise<T>,
  opts: RetryOptions = { maxRetries: 3, baseDelayMs: 1000, maxDelayMs: 10000 },
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));
      if (attempt === opts.maxRetries) break;

      const delay = Math.min(opts.baseDelayMs * 2 ** attempt, opts.maxDelayMs);
      await new Promise((r) => setTimeout(r, delay));
    }
  }
  throw lastError;
}

// Type-safe — return type inferred from fn
const user = await withRetry(() => fetchUser("123")); // user: User
const orders = await withRetry(() => fetchOrders("123"), {
  maxRetries: 5,
  baseDelayMs: 500,
  maxDelayMs: 8000,
});
```

---

### 83. **How do you type event-driven / distributed message handlers?**

**Answer:** Use a **message map** pattern to ensure producers and consumers agree on message shapes.

```ts
// Shared contract between services
interface EventMap {
  "order.created": { orderId: string; userId: string; total: number };
  "order.shipped": { orderId: string; trackingId: string; carrier: string };
  "user.registered": { userId: string; email: string };
}

type EventName = keyof EventMap;

// Type-safe publisher
async function publish<E extends EventName>(
  event: E,
  payload: EventMap[E],
): Promise<void> {
  await messageBus.send({ type: event, data: payload, timestamp: Date.now() });
}

// Type-safe subscriber
function subscribe<E extends EventName>(
  event: E,
  handler: (data: EventMap[E]) => Promise<void>,
): void {
  messageBus.on(event, handler);
}

// ✅ Compiler enforces correct payload
publish("order.created", { orderId: "1", userId: "u1", total: 99.99 });
subscribe("order.shipped", async (data) => {
  console.log(data.trackingId); // TS knows this exists
});

// ❌ Compiler catches mismatches
publish("order.created", { orderId: "1", trackingId: "T1" }); // Error: missing userId, total
```

---

### 84. **How do you type configuration for scalable applications?**

**Answer:** Use `as const` + mapped types to make config type-safe with environment validation at startup.

```ts
const envSchema = {
  PORT: { type: "number", default: 3000 },
  DATABASE_URL: { type: "string", required: true },
  REDIS_URL: { type: "string", required: true },
  MAX_CONNECTIONS: { type: "number", default: 10 },
  LOG_LEVEL: {
    type: "enum",
    values: ["debug", "info", "warn", "error"] as const,
    default: "info" as const,
  },
} as const;

type EnvSchema = typeof envSchema;

// Derive the config type from the schema
type ConfigType = {
  [K in keyof EnvSchema]: EnvSchema[K] extends { type: "number" }
    ? number
    : EnvSchema[K] extends { type: "enum"; values: readonly (infer V)[] }
      ? V
      : string;
};
// Result: { PORT: number; DATABASE_URL: string; REDIS_URL: string; MAX_CONNECTIONS: number; LOG_LEVEL: "debug"|"info"|"warn"|"error" }

function loadConfig(): ConfigType {
  // validate at startup — fail fast if missing required vars
  for (const [key, schema] of Object.entries(envSchema)) {
    if ("required" in schema && schema.required && !process.env[key]) {
      throw new Error(`Missing required env var: ${key}`);
    }
  }
  return {
    /* ... parsed values ... */
  } as ConfigType;
}

const config = loadConfig(); // crashes at startup if DB/Redis URL missing
```

---

### 85. **How do you type health check endpoints for availability monitoring?**

**Answer:**

```ts
type HealthStatus = "healthy" | "degraded" | "unhealthy";

interface DependencyHealth {
  name: string;
  status: HealthStatus;
  latencyMs: number;
  message?: string;
}

interface HealthCheckResponse {
  status: HealthStatus;
  uptime: number;
  timestamp: string;
  dependencies: DependencyHealth[];
}

async function checkDependency(
  name: string,
  checkFn: () => Promise<void>,
): Promise<DependencyHealth> {
  const start = Date.now();
  try {
    await checkFn();
    return { name, status: "healthy", latencyMs: Date.now() - start };
  } catch (error) {
    return {
      name,
      status: "unhealthy",
      latencyMs: Date.now() - start,
      message: error instanceof Error ? error.message : "Unknown error",
    };
  }
}

async function healthCheck(): Promise<HealthCheckResponse> {
  const deps = await Promise.all([
    checkDependency("postgres", () => db.query("SELECT 1")),
    checkDependency("redis", () => redis.ping()),
    checkDependency("external-api", () =>
      fetch("https://api.example.com/health").then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
      }),
    ),
  ]);

  const overall: HealthStatus = deps.every((d) => d.status === "healthy")
    ? "healthy"
    : deps.some((d) => d.status === "unhealthy")
      ? "unhealthy"
      : "degraded";

  return {
    status: overall,
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
    dependencies: deps,
  };
}
```

---

### 86. **How do you type rate limiters and throttling for performance?**

**Answer:** Use generics to build a type-safe rate limiter that works with any async function.

```ts
interface RateLimiterOptions {
  maxRequests: number;
  windowMs: number;
}

class RateLimiter {
  private timestamps: number[] = [];

  constructor(private opts: RateLimiterOptions) {}

  canProceed(): boolean {
    const now = Date.now();
    this.timestamps = this.timestamps.filter(
      (t) => now - t < this.opts.windowMs,
    );
    if (this.timestamps.length >= this.opts.maxRequests) return false;
    this.timestamps.push(now);
    return true;
  }
}

function withRateLimit<TArgs extends unknown[], TReturn>(
  fn: (...args: TArgs) => Promise<TReturn>,
  limiter: RateLimiter,
): (...args: TArgs) => Promise<TReturn> {
  return async (...args: TArgs) => {
    if (!limiter.canProceed()) {
      throw new Error("Rate limit exceeded");
    }
    return fn(...args);
  };
}

// Usage — fully typed
const limiter = new RateLimiter({ maxRequests: 100, windowMs: 60_000 });
const limitedFetchUser = withRateLimit(fetchUser, limiter); // (id: string) => Promise<User>
```

---

### 87. **What is the `using` keyword (Explicit Resource Management) and how does it help with resource cleanup?**

**Answer:** TC39 Stage 3+ (TypeScript 5.2+). Guarantees resources like DB connections and file handles are cleaned up — critical for scalable apps.

```ts
class DatabaseConnection implements Disposable {
  constructor(private pool: Pool) {}
  private conn?: PoolClient;

  async connect() {
    this.conn = await this.pool.connect();
    return this;
  }

  async query(sql: string, params?: unknown[]) {
    return this.conn!.query(sql, params);
  }

  [Symbol.dispose]() {
    this.conn?.release(); // always returns connection to pool
    console.log("Connection released back to pool");
  }
}

// Connection is ALWAYS released, even if query throws
async function getUser(id: string) {
  using conn = await new DatabaseConnection(pool).connect();
  const result = await conn.query("SELECT * FROM users WHERE id = $1", [id]);
  return result.rows[0];
  // conn[Symbol.dispose]() called automatically here
}
```

> **Why this matters for scalability:** Connection leaks are the #1 cause of pool exhaustion under load. `using` makes leaks impossible.

---

### 88. **How do you type a distributed lock in TypeScript?**

**Answer:**

```ts
interface DistributedLock {
  acquire(key: string, ttlMs: number): Promise<string | null>; // returns lock token or null
  release(key: string, token: string): Promise<boolean>;
}

// Higher-order function: type-safe critical section
async function withLock<T>(
  lock: DistributedLock,
  key: string,
  ttlMs: number,
  fn: () => Promise<T>,
): Promise<T> {
  const token = await lock.acquire(key, ttlMs);
  if (!token) {
    throw new Error(`Failed to acquire lock: ${key}`);
  }
  try {
    return await fn();
  } finally {
    await lock.release(key, token);
  }
}

// Usage
const result = await withLock(redisLock, `order:${orderId}`, 5000, async () => {
  const order = await getOrder(orderId);
  order.status = "processing";
  await saveOrder(order);
  return order; // Return type inferred as Order
});
```

---

### 89. **How do you model distributed system states with TypeScript?**

**Answer:** Use discriminated unions to model state machines — the compiler prevents invalid state transitions.

```ts
type OrderState =
  | { status: "pending"; createdAt: Date }
  | { status: "confirmed"; confirmedAt: Date; paymentId: string }
  | { status: "shipped"; shippedAt: Date; trackingNumber: string }
  | { status: "delivered"; deliveredAt: Date }
  | { status: "cancelled"; cancelledAt: Date; reason: string };

// Only valid transitions allowed
function shipOrder(
  order: Extract<OrderState, { status: "confirmed" }>,
): Extract<OrderState, { status: "shipped" }> {
  return {
    status: "shipped",
    shippedAt: new Date(),
    trackingNumber: generateTrackingNumber(),
  };
}

// ❌ Compiler error — can't ship a pending order
const pending: Extract<OrderState, { status: "pending" }> = {
  status: "pending",
  createdAt: new Date(),
};
// shipOrder(pending); // Error: status 'pending' not assignable to 'confirmed'
```

---

### 90. **How do you type parallel batch processing for performance?**

**Answer:** Use generics to build a type-safe batch processor with configurable concurrency.

```ts
async function processBatch<T, R>(
  items: T[],
  processor: (item: T) => Promise<R>,
  concurrency: number = 5,
): Promise<R[]> {
  const results: R[] = [];
  const executing = new Set<Promise<void>>();

  for (const [index, item] of items.entries()) {
    const task = processor(item).then((result) => {
      results[index] = result;
    });
    const wrapped = task.then(() => executing.delete(wrapped));
    executing.add(wrapped);

    if (executing.size >= concurrency) {
      await Promise.race(executing);
    }
  }

  await Promise.all(executing);
  return results;
}

// Fully typed — processes User[] → ProcessedUser[]
const users = await getUsers();
const enriched = await processBatch(
  users,
  async (user) => ({
    ...user,
    orders: await fetchOrders(user.id),
    score: await calculateScore(user.id),
  }),
  10, // max 10 concurrent
);
```
