Here is the **first section** of your complete `README.md` file for JavaScript interview questions, fully structured with 10 detailed questions and answers under:

---

### ✅ **1. Closures and Lexical Scoping**

````markdown
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
````

### Q2: How do closures help in data encapsulation?

Closures allow us to hide data and expose only selected methods.

```js
function createCounter() {
  let count = 0;
  return {
    increment: () => ++count,
    get: () => count
  };
}
const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.get());       // 1
```

### Q3: Can closures lead to memory leaks?

Yes, improper use of closures (like holding references to DOM nodes) can cause memory leaks if not handled correctly.

### Q4: How do closures behave inside a loop?

Without using `let`, all closures will share the same outer variable.

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // logs 3, 3, 3
}

for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // logs 0, 1, 2
}
```

### Q5: What is lexical scoping?

Lexical scoping means that the scope of a variable is defined by its position in the source code.

```js
function outer() {
  let x = 10;
  function inner() {
    console.log(x); // 10
  }
  return inner;
}
```

### Q6: What are common use cases of closures?

* Data hiding
* Function factories
* Currying
* Memoization

### Q7: Can a closure access updated variables?

Yes, closures access the reference to the variable, not a snapshot.

### Q8: How to break a closure reference?

Assign `null` or `undefined` to the closure to break reference and aid garbage collection.

### Q9: Explain closure with `setTimeout`.

```js
function timer() {
  for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 1000);
  }
}
timer(); // Logs 0, 1, 2 after 1s
```

### Q10: Is closure a runtime or compile-time feature?

Closure is a runtime feature based on lexical environment snapshots created during execution.

```

---

Shall I now continue with the **next section**: `2. Event Loop, Microtasks vs Macrotasks`? It will also contain 10 detailed questions with code and explanations.
```
