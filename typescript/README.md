## 50 TypeScript Questions & Examples for Mid-to-Advanced Interviews

As a backend engineer preparing for interviews focused on TypeScript, I created this compact yet comprehensive guide featuring 50 well-structured questions and answers with code examples. These questions are curated for developers with hands-on experience who want to master advanced features, design patterns, type-level programming, and runtime behavior. Perfect for last-minute prep or future revision.

All code blocks have now been formatted in Visual Studio Code style for clarity.

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
const user = { name: 'Ankur', age: 30 };
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
  if (typeof value === 'string') {
    console.log(value.toUpperCase());
  } else {
    console.log(value.toFixed(2));
  }
}
```

### 9. Explain type guards.
```ts
function isString(value: any): value is string {
  return typeof value === 'string';
}
```

### 10. What is a discriminated union?
```ts
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'square'; side: number };

function area(shape: Shape) {
  if (shape.kind === 'circle') return Math.PI * shape.radius ** 2;
  return shape.side ** 2;
}
```

### 11. What is the `never` type?
```ts
function throwError(): never {
  throw new Error('Something went wrong');
}
```

### 12. What are utility types?
```ts
type User = { name: string; age: number };
type PartialUser = Partial<User>;
type RequiredUser = Required<User>;
type ReadonlyUser = Readonly<User>;
type PickName = Pick<User, 'name'>;
type OmitAge = Omit<User, 'age'>;
```

### 13. Difference between `readonly` and `const`?
```ts
const x = 10; // immutable variable

const obj: Readonly<{ name: string }> = { name: 'Ankur' };
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
    console.log('Moving');
  }
}

interface IAnimal {
  makeSound(): void;
}
```

### 16. What is `satisfies` operator in TS 4.9+?
```ts
const config = {
  theme: 'dark',
  debug: true,
} satisfies { theme: string; debug: boolean };
```

### 17. What are conditional types?
```ts
type Message<T> = T extends string ? 'Text' : 'Other';
```

### 18. What is the `infer` keyword used for?
```ts
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

### 19. What is type assertion vs type casting?
```ts
const value: any = 'Hello';
const length = (value as string).length;
```

### 20. Difference between enums and union types?
```ts
enum RoleEnum {
  Admin,
  User,
}

type RoleUnion = 'Admin' | 'User';
```

### 21. Explain mapped types.
```ts
type ReadOnly<T> = {
  readonly [P in keyof T]: T[P];
};
```

### 22. How do template literal types work?
```ts
type Status = 'success' | 'failure';
type Message = `Server responded with ${Status}`;
```

### 23. How to enforce exact object types?
```ts
function acceptExact(obj: { name: string }) {
  return obj;
}

acceptExact({ name: 'Ankur', extra: true }); // Error
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
declare module 'my-lib' {
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
declare module 'express' {
  interface Request {
    user?: string;
  }
}
```

### 31. Explain `as const`.
```ts
const config = { mode: 'dark' } as const;
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
take({ name: 'Ankur', age: 30 }); // Error
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
type Path<T> = T extends object ? {
  [K in keyof T]: `${K & string}` | `${K & string}.${Path<T[K]>}`
}[keyof T] : never;
```

### 38. How does type compatibility differ from assignability?

Compatibility is structural; assignability includes additional compile-time checks for literals, constants, etc.

### 39. What is a branded type?
```ts
type USD = string & { __brand: 'USD' };
```

### 40. What is module resolution?

TypeScript resolves modules using the Node.js module resolution algorithm and `tsconfig.json`.

### 41. Difference between `export =` and `export default`?
```ts
// CommonJS:
export = function greet() {}

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
type T = Exclude<'a' | 'b' | 'c', 'a'>; // 'b' | 'c'
```

### 47. Explain module isolation.
```ts
export const x = 1;
import { x } from './file1';
```

### 48. What is non-null assertion?
```ts
let name: string | undefined;
console.log(name!.length);
```

### 49. What is a phantom type?
```ts
type Tagged<T> = { __type?: T };
type UserId = string & Tagged<'UserId'>;
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
    handler: (payload: Events[K]) => void
  ): void {
    // implementation
  }
}
```

### 51. **What is the `satisfies` operator introduced in TS 4.9?**
```ts
const config = {
  debug: true,
  mode: 'dev',
} satisfies { debug: boolean; mode: string };
```

### 52. **What does `--noUncheckedIndexedAccess` do?**
```ts
let arr: string[] = [];
let val: string = arr[0]; // Error if flag is set, val may be undefined
```

### 53. **What is the benefit of `satisfies` over type assertion?**
```ts
const config = {
  path: '/api'
} satisfies { path: string; method?: 'GET' | 'POST' }; // Ensures no extra props
```

### 54. **What does `--exactOptionalPropertyTypes` do?**
```ts
type User = { name?: string }; // name is now truly optional and undefined must be explicitly passed
```

### 55. **How does TS 5.0 handle const enums in isolatedModules?**
```ts
// Must use preserveConstEnums to retain them in output
const enum Status { Success, Failure }
```

### 56. **What is the new `extends infer` use case in TS 4.9?**
```ts
type GetPromiseType<T> = T extends Promise<infer R> ? R : T;
```

### 57. **How does `in` operator narrowing improve post TS 4.9?**
```ts
function isDog(pet: any): pet is { bark: () => void } {
  return 'bark' in pet;
}
```

### 58. **How does TS support `Object.hasOwn()` now?**
```ts
if (Object.hasOwn(obj, 'key')) {
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
  if (typeof x === 'string') {
    x.toUpperCase(); // ok
  }
}
```

### 61. **What is `const` type parameter inference (TS 5.0)?**
```ts
const makePair = <T extends readonly unknown[]>(tuple: [...T]) => tuple;

const result = makePair([1, 'a'] as const); // [1, 'a']
```

### 62. **How does `ReturnType<typeof fn>` improve developer workflow?**
```ts
function fetchUser() {
  return { name: 'Ankur', age: 30 };
}
type User = ReturnType<typeof fetchUser>;
```

### 63. **What is `--verbatimModuleSyntax` introduced for?**
- Enforces that import/export matches ECMAScript module semantics exactly (no re-writing).

### 64. **What is a `using` declaration in TS 5.2+ (proposal)?**
```ts
using db = new Database();
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

let p: Person = { name: 'Ankur' };
console.log(p.name); // error if flag enabled
```

### 69. **How to use `Object.entries()` safely in TS 4.9+?**
```ts
const user = { name: 'Ankur', age: 30 };
Object.entries(user).forEach(([key, value]) => {
  console.log(key, value); // key is still string, but can narrow better
});
```

### 70. **What is `readonly` array vs tuple behavior in newer TS?**
```ts
const values: readonly [string, number] = ['age', 30];
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

