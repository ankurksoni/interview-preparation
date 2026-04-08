# JavaScript Interview Questions & Answers

A comprehensive guide covering essential JavaScript concepts for mid-to-advanced level interviews. Each section includes questions with detailed explanations and practical code examples.

---

## Table of Contents

1. [Closures and Lexical Scoping](#1-closures-and-lexical-scoping)
2. [Event Loop, Microtasks vs Macrotasks](#2-event-loop-microtasks-vs-macrotasks)
3. [Prototypes and Inheritance](#3-prototypes-and-inheritance)
4. [Promises and async/await](#4-promises-and-asyncawait)
5. [this Keyword and Binding](#5-this-keyword-and-binding)
6. [ES2020+ Modern Features](#6-es2020-modern-features)
7. [Scope, Hoisting, and Temporal Dead Zone](#7-scope-hoisting-and-temporal-dead-zone)
8. [Objects, Arrays, and Iteration](#8-objects-arrays-and-iteration)
9. [Error Handling](#9-error-handling)
10. [Memory Management and Performance](#10-memory-management-and-performance)

---

## 1. Closures and Lexical Scoping

### Q1: What is a closure in JavaScript?

A closure is a function that retains access to variables from its lexical scope even when the function is executed outside of that scope.

```js
function outer() {
  let counter = 0;
  return function inner() {
    counter++;
    return counter;
  };
}

const fn = outer();
console.log(fn()); // 1
console.log(fn()); // 2
```

### Q2: How do closures help in data encapsulation?

Closures allow us to hide data and expose only selected methods.

```js
function createCounter() {
  let count = 0;
  return {
    increment: () => ++count,
    decrement: () => --count,
    get: () => count,
  };
}
const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.get()); // 1
// count is not accessible from outside
```

### Q3: Can closures lead to memory leaks?

Yes. If a closure retains references to large objects (like DOM nodes or big arrays), the garbage collector cannot free them as long as the closure exists.

```js
function leaky() {
  const bigArray = new Array(1_000_000).fill("data");
  return function () {
    // bigArray is retained even if not used directly
    console.log("still holding reference");
  };
}
// Fix: set references to null when no longer needed
```

### Q4: How do closures behave inside a loop?

Without block scoping (`let`), all closures share the same variable reference.

```js
// Problem: var is function-scoped
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 3, 3, 3
}

// Fix: let is block-scoped, creates a new binding per iteration
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 0, 1, 2
}
```

### Q5: What is lexical scoping?

Lexical scoping means that a function's scope is determined by where it's defined in the source code, not where it's called.

```js
const x = "global";
function outer() {
  const x = "outer";
  function inner() {
    console.log(x); // 'outer' — lexically determined
  }
  return inner;
}
outer()();
```

### Q6: Explain currying using closures.

Currying transforms a function with multiple arguments into a series of functions each taking a single argument.

```js
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn(...args);
    return (...next) => curried(...args, ...next);
  };
}

const add = (a, b, c) => a + b + c;
const curriedAdd = curry(add);
console.log(curriedAdd(1)(2)(3)); // 6
console.log(curriedAdd(1, 2)(3)); // 6
```

### Q7: How does memoization use closures?

```js
function memoize(fn) {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

const factorial = memoize((n) => (n <= 1 ? 1 : n * factorial(n - 1)));
console.log(factorial(5)); // 120
```

---

## 2. Event Loop, Microtasks vs Macrotasks

### Q8: What is the JavaScript Event Loop?

The Event Loop is the mechanism that allows JavaScript to perform non-blocking operations despite being single-threaded. It continuously checks the call stack and task queues.

**Execution order:** Call Stack → Microtask Queue → Macrotask Queue

```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");
// Output: 1, 4, 3, 2
```

### Q9: What is the difference between microtasks and macrotasks?

| Microtasks                   | Macrotasks                  |
| ---------------------------- | --------------------------- |
| `Promise.then/catch/finally` | `setTimeout`, `setInterval` |
| `queueMicrotask()`           | `requestAnimationFrame`     |
| `MutationObserver`           | I/O operations              |

Microtasks are processed **after** the current task completes but **before** the next macrotask.

```js
setTimeout(() => console.log("macro 1"), 0);
setTimeout(() => console.log("macro 2"), 0);

Promise.resolve()
  .then(() => console.log("micro 1"))
  .then(() => console.log("micro 2"));

// Output: micro 1, micro 2, macro 1, macro 2
```

### Q10: What does `queueMicrotask` do?

It schedules a function to run as a microtask, before the next macrotask.

```js
console.log("start");
queueMicrotask(() => console.log("microtask"));
console.log("end");
// Output: start, end, microtask
```

### Q11: Can microtasks starve macrotasks?

Yes. If microtasks continuously schedule more microtasks, macrotasks (including UI rendering) will be blocked.

```js
// ❌ This blocks the event loop
function block() {
  queueMicrotask(block);
}
block(); // Never yields to macrotasks
```

### Q12: Explain `requestAnimationFrame` vs `setTimeout`.

`requestAnimationFrame` runs before the browser's next repaint (~60fps), making it ideal for animations. `setTimeout` runs in the macrotask queue and is not synchronized with the display refresh rate.

```js
// Smooth animation
function animate() {
  element.style.left = `${pos++}px`;
  if (pos < 300) requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

### Q13: Predict the output of this event loop puzzle.

```js
async function async1() {
  console.log("async1 start");
  await async2();
  console.log("async1 end");
}
async function async2() {
  console.log("async2");
}
console.log("script start");
setTimeout(() => console.log("setTimeout"), 0);
async1();
new Promise((resolve) => {
  console.log("promise1");
  resolve();
}).then(() => console.log("promise2"));
console.log("script end");

// Output:
// script start
// async1 start
// async2
// promise1
// script end
// async1 end
// promise2
// setTimeout
```

---

## 3. Prototypes and Inheritance

### Q14: What is prototypal inheritance?

Every JavaScript object has an internal `[[Prototype]]` link to another object. When a property is accessed, the engine walks up the prototype chain.

```js
const animal = {
  speak() {
    return `${this.name} makes a sound`;
  },
};

const dog = Object.create(animal);
dog.name = "Rex";
console.log(dog.speak()); // "Rex makes a sound"
console.log(Object.getPrototypeOf(dog) === animal); // true
```

### Q15: How do ES6 classes relate to prototypes?

ES6 classes are syntactic sugar over prototypal inheritance. Under the hood, they still use prototypes.

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  bark() {
    return `${this.name} barks`;
  }
}

const d = new Dog("Rex");
console.log(d.speak()); // "Rex makes a sound"
console.log(d instanceof Animal); // true
```

### Q16: What is the difference between `__proto__` and `prototype`?

- `prototype` is a property on constructor functions, used to set up the prototype chain for instances.
- `__proto__` (deprecated; use `Object.getPrototypeOf()`) is the actual internal `[[Prototype]]` link of an object instance.

```js
function Person(name) {
  this.name = name;
}
const p = new Person("Alice");
console.log(Object.getPrototypeOf(p) === Person.prototype); // true
```

### Q17: How does `Object.create()` work?

It creates a new object with the specified prototype.

```js
const base = {
  greet() {
    return "hello";
  },
};
const child = Object.create(base, {
  name: { value: "World", writable: true },
});
console.log(child.greet()); // "hello"
console.log(child.hasOwnProperty("greet")); // false
console.log(child.hasOwnProperty("name")); // true
```

### Q18: What are static methods and how do they differ from instance methods?

Static methods belong to the class itself, not to instances.

```js
class MathHelper {
  static add(a, b) {
    return a + b;
  }
}
console.log(MathHelper.add(2, 3)); // 5

const m = new MathHelper();
// m.add(2, 3) → TypeError: m.add is not a function
```

---

## 4. Promises and async/await

### Q19: What are the three states of a Promise?

- **Pending** – initial state
- **Fulfilled** – operation succeeded (resolved with a value)
- **Rejected** – operation failed (rejected with a reason)

A Promise can only transition from Pending → Fulfilled or Pending → Rejected (never reversed).

### Q20: Explain `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`.

```js
const p1 = Promise.resolve(1);
const p2 = Promise.reject("error");
const p3 = Promise.resolve(3);

// Promise.all — rejects if ANY promise rejects
Promise.all([p1, p3]).then(console.log); // [1, 3]

// Promise.allSettled — waits for all, never rejects
Promise.allSettled([p1, p2, p3]).then(console.log);
// [{status:'fulfilled',value:1}, {status:'rejected',reason:'error'}, {status:'fulfilled',value:3}]

// Promise.race — resolves/rejects with first settled promise
Promise.race([p1, p2]).then(console.log); // 1

// Promise.any — resolves with first fulfilled; rejects only if ALL reject
Promise.any([p2, p3]).then(console.log); // 3
```

### Q21: What is the difference between `async/await` and `.then()` chaining?

Both handle Promises. `async/await` provides synchronous-looking code that's easier to read and debug, especially with try/catch.

```js
// .then() chaining
fetchUser()
  .then((user) => fetchPosts(user.id))
  .then((posts) => console.log(posts))
  .catch((err) => console.error(err));

// async/await — same logic, cleaner syntax
async function loadPosts() {
  try {
    const user = await fetchUser();
    const posts = await fetchPosts(user.id);
    console.log(posts);
  } catch (err) {
    console.error(err);
  }
}
```

### Q22: How do you run async operations in parallel?

Use `Promise.all` or `Promise.allSettled` instead of sequential `await`.

```js
// ❌ Sequential — slow
const users = await fetchUsers();
const orders = await fetchOrders();

// ✅ Parallel — fast
const [users, orders] = await Promise.all([fetchUsers(), fetchOrders()]);
```

### Q23: How do you implement a timeout for a Promise?

```js
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), ms),
  );
  return Promise.race([promise, timeout]);
}

// Usage
await withTimeout(fetch("/api/data"), 5000);
```

### Q24: What are top-level `await` and when can you use it?

Top-level `await` allows using `await` outside of an `async` function, but only in ES modules (files with `"type": "module"` or `.mjs` extension).

```js
// In an ES module
const response = await fetch("https://api.example.com/data");
const data = await response.json();
export default data;
```

---

## 5. this Keyword and Binding

### Q25: How does `this` work in different contexts?

| Context                   | `this` refers to         |
| ------------------------- | ------------------------ |
| Global scope (non-strict) | `globalThis` / `window`  |
| Global scope (strict)     | `undefined`              |
| Object method             | The object               |
| Arrow function            | Enclosing lexical `this` |
| Constructor (`new`)       | The new instance         |
| `call`/`apply`/`bind`     | Explicitly specified     |

### Q26: Compare arrow functions vs regular functions for `this`.

```js
const obj = {
  name: "Alice",
  // Regular function: this = obj
  greet() {
    console.log(this.name); // 'Alice'
  },
  // Arrow function: this = enclosing scope (NOT obj)
  greetArrow: () => {
    console.log(this.name); // undefined (inherits outer this)
  },
  // Common pattern: arrow inside method preserves this
  delayedGreet() {
    setTimeout(() => {
      console.log(this.name); // 'Alice' ✅
    }, 100);
  },
};
```

### Q27: What do `call`, `apply`, and `bind` do?

```js
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const user = { name: "Alice" };

// call: invokes immediately, args passed individually
greet.call(user, "Hello", "!"); // "Hello, Alice!"

// apply: invokes immediately, args as array
greet.apply(user, ["Hi", "."]); // "Hi, Alice."

// bind: returns a new function with this bound
const bound = greet.bind(user, "Hey");
bound("?"); // "Hey, Alice?"
```

### Q28: What is `this` inside a class?

```js
class Timer {
  count = 0;

  // Method shorthand: this depends on call site
  start() {
    // Arrow function preserves this from the class instance
    setInterval(() => {
      this.count++;
      console.log(this.count);
    }, 1000);
  }
}

const t = new Timer();
t.start(); // Works correctly — this refers to the Timer instance
```

---

## 6. ES2020+ Modern Features

### Q29: What is optional chaining (`?.`) and nullish coalescing (`??`)?

```js
const user = { address: { city: "NYC" } };

// Optional chaining — returns undefined instead of throwing
console.log(user.address?.city); // 'NYC'
console.log(user.address?.zip); // undefined
console.log(user.phone?.number); // undefined (no error)

// Nullish coalescing — fallback only for null/undefined (not 0, '', false)
const port = 0;
console.log(port || 3000); // 3000 — wrong! 0 is falsy
console.log(port ?? 3000); // 0    — correct!
```

### Q30: What are `WeakRef` and `FinalizationRegistry`?

`WeakRef` holds a weak reference to an object (doesn't prevent GC). `FinalizationRegistry` lets you run cleanup code after an object is garbage collected.

```js
let obj = { data: "important" };
const weakRef = new WeakRef(obj);

console.log(weakRef.deref()); // { data: 'important' }

obj = null; // eligible for GC now
// After GC: weakRef.deref() returns undefined

const registry = new FinalizationRegistry((value) => {
  console.log(`${value} was garbage collected`);
});
registry.register(obj, "myObject");
```

### Q31: What is `structuredClone` and how does it differ from spread/`JSON.parse`?

```js
const original = {
  date: new Date(),
  set: new Set([1, 2, 3]),
  nested: { a: 1 },
};

// ❌ JSON roundtrip — loses Date, Set, Map, undefined, etc.
const jsonCopy = JSON.parse(JSON.stringify(original));
console.log(jsonCopy.date instanceof Date); // false
console.log(jsonCopy.set); // {} (lost!)

// ❌ Spread — shallow copy only
const spreadCopy = { ...original };
spreadCopy.nested.a = 99;
console.log(original.nested.a); // 99 (mutated!)

// ✅ structuredClone — deep copy, preserves types
const clone = structuredClone(original);
clone.nested.a = 99;
console.log(original.nested.a); // 1 (safe)
console.log(clone.date instanceof Date); // true
console.log(clone.set instanceof Set); // true
```

### Q32: What are private class fields (`#`)?

```js
class BankAccount {
  #balance = 0; // Truly private

  deposit(amount) {
    if (amount > 0) this.#balance += amount;
  }

  get balance() {
    return this.#balance;
  }
}

const account = new BankAccount();
account.deposit(100);
console.log(account.balance); // 100
// account.#balance; → SyntaxError: Private field
```

### Q33: What are `Array.at()`, `Object.groupBy()`, and other modern array methods?

```js
const arr = [10, 20, 30, 40, 50];

// at() — supports negative indexing
console.log(arr.at(-1)); // 50
console.log(arr.at(-2)); // 40

// Object.groupBy() (ES2024)
const people = [
  { name: "Alice", age: 25 },
  { name: "Bob", age: 30 },
  { name: "Charlie", age: 25 },
];
const grouped = Object.groupBy(people, (p) => p.age);
// { 25: [{name:'Alice',...}, {name:'Charlie',...}], 30: [{name:'Bob',...}] }

// toSorted(), toReversed(), toSpliced() — non-mutating versions
const sorted = arr.toSorted((a, b) => b - a); // [50, 40, 30, 20, 10]
console.log(arr); // [10, 20, 30, 40, 50] — unchanged
```

### Q34: What are iterators and generators?

```js
// Generator function
function* range(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}

for (const n of range(1, 5)) {
  console.log(n); // 1, 2, 3, 4, 5
}

// Async generator
async function* fetchPages(url) {
  let page = 1;
  while (true) {
    const res = await fetch(`${url}?page=${page}`);
    const data = await res.json();
    if (data.length === 0) return;
    yield data;
    page++;
  }
}
```

### Q35: What is the `Proxy` object?

`Proxy` lets you intercept and customize fundamental operations on objects.

```js
const handler = {
  get(target, prop) {
    return prop in target ? target[prop] : `Property '${prop}' not found`;
  },
  set(target, prop, value) {
    if (prop === "age" && typeof value !== "number") {
      throw new TypeError("Age must be a number");
    }
    target[prop] = value;
    return true;
  },
};

const user = new Proxy({}, handler);
user.name = "Alice";
user.age = 30;
console.log(user.name); // 'Alice'
console.log(user.unknown); // "Property 'unknown' not found"
// user.age = 'thirty'; → TypeError: Age must be a number
```

---

## 7. Scope, Hoisting, and Temporal Dead Zone

### Q36: What are the different types of scope in JavaScript?

- **Global scope** — accessible everywhere
- **Function scope** — `var` is scoped to the nearest function
- **Block scope** — `let` and `const` are scoped to the nearest `{}`
- **Module scope** — top-level variables in ES modules are module-scoped

### Q37: What is hoisting?

Variable and function declarations are moved to the top of their scope during compilation. But only declarations are hoisted, not initializations.

```js
console.log(a); // undefined (var is hoisted, initialized as undefined)
console.log(b); // ReferenceError: Cannot access 'b' before initialization
var a = 1;
let b = 2;

// Function declarations are fully hoisted
greet(); // "Hello" — works!
function greet() {
  console.log("Hello");
}

// Function expressions are NOT hoisted
hello(); // TypeError: hello is not a function
var hello = function () {
  console.log("Hello");
};
```

### Q38: What is the Temporal Dead Zone (TDZ)?

The TDZ is the period between entering a block scope and the actual `let`/`const` declaration. Accessing the variable during this period throws a `ReferenceError`.

```js
{
  // TDZ for `x` starts here
  console.log(x); // ReferenceError
  let x = 10; // TDZ ends here
}
```

### Q39: Explain `var` vs `let` vs `const`.

| Feature        | `var`                | `let`     | `const`   |
| -------------- | -------------------- | --------- | --------- |
| Scope          | Function             | Block     | Block     |
| Hoisting       | Yes (as `undefined`) | Yes (TDZ) | Yes (TDZ) |
| Re-declaration | Yes                  | No        | No        |
| Re-assignment  | Yes                  | Yes       | No        |

```js
const obj = { a: 1 };
obj.a = 2; // ✅ OK — mutating the object is allowed
// obj = {};     // ❌ TypeError — reassignment is not
```

---

## 8. Objects, Arrays, and Iteration

### Q40: What is the difference between shallow copy and deep copy?

```js
const original = { a: 1, nested: { b: 2 } };

// Shallow copy — nested objects are still shared
const shallow = { ...original };
shallow.nested.b = 99;
console.log(original.nested.b); // 99 (mutated!)

// Deep copy with structuredClone
const deep = structuredClone(original);
deep.nested.b = 42;
console.log(original.nested.b); // 99 (unchanged from the earlier mutation)
```

### Q41: What is destructuring and how does it work with defaults and renaming?

```js
// Object destructuring with defaults and renaming
const { name: userName = "Anonymous", age = 0 } = { name: "Alice" };
console.log(userName); // 'Alice'
console.log(age); // 0

// Nested destructuring
const { address: { city } = {} } = { address: { city: "NYC" } };
console.log(city); // 'NYC'

// Array destructuring with skip
const [first, , third] = [1, 2, 3];
console.log(first, third); // 1, 3
```

### Q42: What is the difference between `Map` and plain objects?

| Feature     | Object                                  | Map                            |
| ----------- | --------------------------------------- | ------------------------------ |
| Key types   | Strings/Symbols                         | Any type                       |
| Order       | Not guaranteed (mostly insertion order) | Insertion order guaranteed     |
| Size        | `Object.keys(obj).length`               | `map.size`                     |
| Iteration   | `for...in`, `Object.entries()`          | `for...of`, `.forEach()`       |
| Performance | Ok for small data                       | Better for frequent add/delete |

```js
const map = new Map();
map.set({ id: 1 }, "user1"); // Objects as keys
map.set(42, "answer");
console.log(map.size); // 2
```

### Q43: What are `WeakMap` and `WeakSet`?

They hold weak references to objects, allowing garbage collection when no other references exist. Keys must be objects.

```js
const weakMap = new WeakMap();
let obj = { data: "secret" };
weakMap.set(obj, "metadata");

console.log(weakMap.get(obj)); // 'metadata'
obj = null; // obj can now be garbage collected
// weakMap entry is also cleaned up automatically
```

### Q44: What are `Symbol` and `Symbol.for()`?

Symbols are unique, immutable primitives used as property keys to avoid naming collisions.

```js
const s1 = Symbol("id");
const s2 = Symbol("id");
console.log(s1 === s2); // false — always unique

// Symbol.for() creates global/shared symbols
const s3 = Symbol.for("shared");
const s4 = Symbol.for("shared");
console.log(s3 === s4); // true — same reference
```

---

## 9. Error Handling

### Q45: How does `try/catch/finally` work?

```js
function divide(a, b) {
  try {
    if (b === 0) throw new Error("Division by zero");
    return a / b;
  } catch (err) {
    console.error(err.message);
    return null;
  } finally {
    // Always runs, even after return or throw
    console.log("cleanup done");
  }
}
```

### Q46: What are custom error classes?

```js
class ValidationError extends Error {
  constructor(field, message) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}

function validate(input) {
  if (!input.email) {
    throw new ValidationError("email", "Email is required");
  }
}

try {
  validate({});
} catch (err) {
  if (err instanceof ValidationError) {
    console.log(`${err.field}: ${err.message}`);
  }
}
```

### Q47: What is `AggregateError`?

Thrown by `Promise.any()` when all promises reject. Contains an `errors` array.

```js
Promise.any([Promise.reject("err1"), Promise.reject("err2")]).catch((err) => {
  console.log(err instanceof AggregateError); // true
  console.log(err.errors); // ['err1', 'err2']
});
```

### Q48: How do you handle errors in async functions?

```js
// Option 1: try/catch inside async
async function fetchData() {
  try {
    const res = await fetch("/api");
    return await res.json();
  } catch (err) {
    console.error("Fetch failed:", err);
  }
}

// Option 2: Catch at call site
fetchData().catch((err) => console.error(err));

// Option 3: Utility wrapper
async function tryCatch(promise) {
  try {
    const data = await promise;
    return [data, null];
  } catch (err) {
    return [null, err];
  }
}

const [data, err] = await tryCatch(fetch("/api"));
```

---

## 10. Memory Management and Performance

### Q49: How does garbage collection work in JavaScript?

JavaScript uses **mark-and-sweep** garbage collection. The GC starts from roots (global object, current call stack) and marks all reachable objects. Unmarked objects are freed.

Common memory leak sources:

- Forgotten event listeners
- Closures retaining large scopes
- Detached DOM nodes
- Global variables
- Uncleared timers/intervals

### Q50: What is debouncing vs throttling?

```js
// Debounce: wait until the user STOPS doing something
function debounce(fn, delay) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Throttle: execute at most once per interval
function throttle(fn, interval) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}

// Usage
window.addEventListener("scroll", throttle(handleScroll, 200));
searchInput.addEventListener("input", debounce(search, 300));
```

### Q51: What is the difference between `==` and `===`?

`==` performs type coercion before comparison. `===` checks both value and type (no coercion). Always prefer `===`.

```js
console.log(0 == ""); // true  (both coerced)
console.log(0 === ""); // false (different types)
console.log(null == undefined); // true
console.log(null === undefined); // false
console.log(NaN === NaN); // false (use Number.isNaN())
```

### Q52: What are `AbortController` and `AbortSignal`?

Used to cancel asynchronous operations like `fetch`, event listeners, or any custom async work.

```js
const controller = new AbortController();

fetch("/api/data", { signal: controller.signal })
  .then((res) => res.json())
  .catch((err) => {
    if (err.name === "AbortError") console.log("Fetch cancelled");
  });

// Cancel the request after 3 seconds
setTimeout(() => controller.abort(), 3000);

// Also works with addEventListener
element.addEventListener("click", handler, { signal: controller.signal });
controller.abort(); // Removes the event listener
```

---

## 📌 Summary

| Topic          | Key Takeaway                                                          |
| -------------- | --------------------------------------------------------------------- |
| Closures       | Functions retain their lexical scope                                  |
| Event Loop     | Microtasks before macrotasks, always                                  |
| Prototypes     | Classes are sugar over prototype chains                               |
| Promises       | Use `Promise.all` for parallelism, `async/await` for readability      |
| `this`         | Arrow functions inherit `this`; regular functions depend on call site |
| Modern JS      | Use `?.`, `??`, `structuredClone`, `#private`, `Object.groupBy`       |
| Scope          | Prefer `const` > `let` > never `var`                                  |
| Error Handling | Custom errors + `try/catch` in async code                             |
| Performance    | Debounce input, throttle scroll, watch for memory leaks               |

---

## 11. Scalability, Resiliency & Distributed Patterns

### Q53. **How do Web Workers help with scalability in JavaScript?**

**Answer:** Web Workers run code on a **separate OS thread**, keeping the main thread responsive. They're ideal for CPU-heavy tasks like parsing, image processing, or crypto.

```js
// main.js
const worker = new Worker("worker.js");
worker.postMessage({ data: largeArray });

worker.onmessage = (e) => {
  console.log("Processed result:", e.data);
};

worker.onerror = (e) => {
  console.error("Worker crashed:", e.message);
};

// worker.js
self.onmessage = (e) => {
  const result = e.data.data.map((x) => x * 2); // heavy computation
  self.postMessage(result);
};
```

| Feature           | Main Thread    | Web Worker |
| ----------------- | -------------- | ---------- |
| DOM access        | ✅             | ❌         |
| `fetch` / network | ✅             | ✅         |
| `setTimeout`      | ✅             | ✅         |
| SharedArrayBuffer | ✅             | ✅         |
| CPU-heavy safe    | ❌ (blocks UI) | ✅         |

---

### Q54. **What is the retry pattern with exponential backoff in JavaScript?**

**Answer:** When calling unreliable APIs or services, retry with increasing delays to avoid overwhelming the server and improve resilience.

```js
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      if (!response.ok && response.status >= 500) {
        throw new Error(`Server error: ${response.status}`);
      }
      return response;
    } catch (error) {
      if (attempt === maxRetries) throw error;

      const delay = Math.min(1000 * 2 ** attempt, 10000); // 1s, 2s, 4s... cap at 10s
      const jitter = delay * (0.5 + Math.random() * 0.5); // add randomness
      console.warn(
        `Attempt ${attempt + 1} failed. Retrying in ${Math.round(jitter)}ms...`,
      );
      await new Promise((r) => setTimeout(r, jitter));
    }
  }
}
```

> **Why jitter?** Without jitter, thousands of clients retry at the exact same moment (thundering herd). Jitter spreads retries across time.

---

### Q55. **How do you implement a circuit breaker in JavaScript?**

**Answer:** A circuit breaker prevents cascading failures by stopping calls to a failing service after a threshold, allowing it time to recover.

```js
class CircuitBreaker {
  constructor(fn, { threshold = 5, cooldown = 30000 } = {}) {
    this.fn = fn;
    this.threshold = threshold;
    this.cooldown = cooldown;
    this.failures = 0;
    this.state = "CLOSED"; // CLOSED → OPEN → HALF_OPEN
    this.nextAttempt = 0;
  }

  async call(...args) {
    if (this.state === "OPEN") {
      if (Date.now() < this.nextAttempt) {
        throw new Error("Circuit is OPEN — service unavailable");
      }
      this.state = "HALF_OPEN";
    }

    try {
      const result = await this.fn(...args);
      this.#onSuccess();
      return result;
    } catch (error) {
      this.#onFailure();
      throw error;
    }
  }

  #onSuccess() {
    this.failures = 0;
    this.state = "CLOSED";
  }

  #onFailure() {
    this.failures++;
    if (this.failures >= this.threshold) {
      this.state = "OPEN";
      this.nextAttempt = Date.now() + this.cooldown;
    }
  }
}

// Usage
const breaker = new CircuitBreaker(fetch, { threshold: 3, cooldown: 15000 });
try {
  const res = await breaker.call("https://api.example.com/data");
} catch (e) {
  console.error(e.message); // "Circuit is OPEN" after 3 failures
}
```

| State     | Behavior                                                |
| --------- | ------------------------------------------------------- |
| CLOSED    | Normal operation, calls pass through                    |
| OPEN      | All calls rejected immediately (fail fast)              |
| HALF_OPEN | One test call allowed; success → CLOSED, failure → OPEN |

---

### Q56. **How does `AbortController` improve resilience in HTTP calls?**

**Answer:** It lets you set **timeouts** and **cancel** in-flight requests, preventing hung connections from consuming resources.

```js
async function fetchWithTimeout(url, timeoutMs = 5000) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return await response.json();
  } catch (error) {
    if (error.name === "AbortError") {
      throw new Error(`Request to ${url} timed out after ${timeoutMs}ms`);
    }
    throw error;
  } finally {
    clearTimeout(timeoutId);
  }
}

// Cancel multiple parallel requests when one finishes
async function raceRequests(urls) {
  const controller = new AbortController();
  try {
    return await Promise.any(
      urls.map((url) =>
        fetch(url, { signal: controller.signal }).then((r) => r.json()),
      ),
    );
  } finally {
    controller.abort(); // cancel remaining requests
  }
}
```

---

### Q57. **What is the Bulkhead pattern and how does it apply in JavaScript?**

**Answer:** The Bulkhead pattern isolates different parts of your application so a failure in one doesn't sink everything — like compartments in a ship.

```js
class Bulkhead {
  constructor(maxConcurrent) {
    this.max = maxConcurrent;
    this.running = 0;
    this.queue = [];
  }

  async execute(fn) {
    if (this.running >= this.max) {
      await new Promise((resolve) => this.queue.push(resolve));
    }
    this.running++;
    try {
      return await fn();
    } finally {
      this.running--;
      if (this.queue.length > 0) {
        this.queue.shift()(); // release next from queue
      }
    }
  }
}

// Isolate API calls: max 5 concurrent to payment service
const paymentPool = new Bulkhead(5);
// Max 10 concurrent to inventory service
const inventoryPool = new Bulkhead(10);

// Payment service being slow won't block inventory calls
await Promise.all([
  paymentPool.execute(() => fetch("/api/payment")),
  inventoryPool.execute(() => fetch("/api/inventory")),
]);
```

---

### Q58. **How do you handle errors in `Promise.allSettled` vs `Promise.all` for distributed calls?**

**Answer:** When calling multiple independent services, `Promise.allSettled` gives you **partial success** instead of failing everything.

```js
// ❌ Promise.all — one failure kills ALL results
try {
  const [user, orders, recommendations] = await Promise.all([
    fetchUser(id),
    fetchOrders(id),
    fetchRecommendations(id), // if this fails, you lose user + orders too
  ]);
} catch (e) {
  // total failure — even though user and orders succeeded
}

// ✅ Promise.allSettled — graceful degradation
const results = await Promise.allSettled([
  fetchUser(id),
  fetchOrders(id),
  fetchRecommendations(id),
]);

const [user, orders, recommendations] = results.map((r) =>
  r.status === "fulfilled" ? r.value : null,
);

// Page works with user + orders even if recommendations failed
renderPage({ user, orders, recommendations }); // recommendations = null → show fallback
```

> **Rule of thumb:** Use `Promise.all` for tightly coupled operations (all-or-nothing). Use `Promise.allSettled` for best-effort / graceful degradation.

---

### Q59. **What is the BroadcastChannel API and how does it help with distributed browser tabs?**

**Answer:** `BroadcastChannel` lets multiple tabs/windows of the **same origin** communicate — useful for syncing auth state, caches, or real-time updates.

```js
// Tab 1: send logout signal
const channel = new BroadcastChannel("auth");
channel.postMessage({ type: "LOGOUT" });

// Tab 2: listen and react
const channel = new BroadcastChannel("auth");
channel.onmessage = (event) => {
  if (event.data.type === "LOGOUT") {
    clearSession();
    window.location.href = "/login";
  }
};
```

| Use case                  | Solution                     |
| ------------------------- | ---------------------------- |
| Sync auth across tabs     | BroadcastChannel             |
| Shared cache invalidation | BroadcastChannel + IndexedDB |
| Cross-tab leader election | Locks API + BroadcastChannel |

---

### Q60. **What are performance bottlenecks in JavaScript and how do you diagnose them?**

**Answer:**

| Bottleneck                     | Symptom                 | Diagnosis Tool                       | Fix                                                 |
| ------------------------------ | ----------------------- | ------------------------------------ | --------------------------------------------------- |
| Long task blocking main thread | UI janky, input lag     | `PerformanceObserver` for long tasks | Break into chunks with `scheduler.yield()`          |
| Memory leak                    | Growing heap, GC pauses | Chrome DevTools → Memory tab         | Clean up closures, WeakRef, WeakMap                 |
| Layout thrashing               | Slow DOM updates        | Performance panel → Layout events    | Batch DOM reads/writes, use `requestAnimationFrame` |
| Large bundle                   | Slow page load          | Lighthouse, webpack-bundle-analyzer  | Code splitting, tree shaking, dynamic `import()`    |
| Excessive re-renders (React)   | Slow UI updates         | React DevTools Profiler              | `useMemo`, `React.memo`, virtualization             |

```js
// Detect long tasks (>50ms) automatically
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn(`Long task detected: ${entry.duration}ms`, entry);
  }
});
observer.observe({ entryTypes: ["longtask"] });

// Break up heavy work to stay responsive
async function processLargeArray(items) {
  const CHUNK = 100;
  for (let i = 0; i < items.length; i += CHUNK) {
    const chunk = items.slice(i, i + CHUNK);
    processChunk(chunk);
    await scheduler.yield(); // give main thread back to browser
  }
}
```
