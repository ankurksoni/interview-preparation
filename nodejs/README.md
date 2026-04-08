# Node.js Interview Questions & Answers

A comprehensive set of Q&A for Node.js interviews, ranging from medium to high difficulty, with a code sample for each. Covers core Node.js concepts, asynchronous programming, performance optimization, security, and advanced topics like streams, clustering, and microservices.

## 1. What is the Event Loop in Node.js, and how does it work?

**Answer**: The Event Loop is the core mechanism in Node.js that handles asynchronous operations. It processes events in phases: timers, pending callbacks, idle/prepare, poll, check, and close. It ensures non-blocking I/O by delegating tasks to the libuv thread pool and processing callbacks when they’re ready.

**Code Sample**:

```javascript
setTimeout(() => console.log("Timer"), 0);
setImmediate(() => console.log("Immediate"));
Promise.resolve().then(() => console.log("Promise"));
console.log("Sync");
// Output: Sync -> Promise -> Immediate -> Timer
```

---

## 2. How does Node.js handle asynchronous operations differently from synchronous ones?

**Answer**: Node.js uses callbacks, Promises, or async/await for asynchronous operations, leveraging the Event Loop to avoid blocking the main thread. Synchronous operations execute immediately, blocking further execution until complete.

**Code Sample**:

```javascript
const fs = require("fs").promises;
async function readFileAsync() {
  try {
    const data = await fs.readFile("file.txt", "utf8");
    console.log(data);
  } catch (err) {
    console.error(err);
  }
}
readFileAsync();
```

---

## 3. What are Streams in Node.js, and when should you use them?

**Answer**: Streams are objects that allow reading or writing data in chunks, useful for handling large datasets or real-time data. They come in four types: Readable, Writable, Duplex, and Transform. Use streams for memory efficiency with large files or data processing.

**Code Sample**:

```javascript
const fs = require("fs");
const readStream = fs.createReadStream("largeFile.txt");
readStream.on("data", (chunk) => console.log(`Received ${chunk.length} bytes`));
readStream.on("end", () => console.log("Finished reading"));
```

---

## 4. How can you handle errors in asynchronous code in Node.js?

**Answer**: Use try-catch with async/await, `.catch()` with Promises, or error-first callbacks. For unhandled Promise rejections, use `process.on('unhandledRejection')`.

**Code Sample**:

```javascript
process.on("unhandledRejection", (err) => console.error("Unhandled:", err));
async function fetchData() {
  try {
    await Promise.reject(new Error("Failed"));
  } catch (err) {
    console.error(err.message);
  }
}
fetchData();
```

---

## 5. What is the difference between `require` and `import` in Node.js?

**Answer**: `require` is CommonJS (CJS), synchronous, and the traditional Node.js module system. `import` is ES Modules (ESM), supports static analysis and tree-shaking, and is the modern standard. ESM requires `"type": "module"` in `package.json` or `.mjs` file extension. Node.js fully supports ESM since v16+.

**Code Sample**:

```javascript
// CommonJS (legacy, still widely used)
const fs = require("fs");
const { readFile } = require("fs/promises");

// ES Modules (modern, recommended for new projects)
import fs from "node:fs";
import { readFile } from "node:fs/promises";
```

> **Note:** The `node:` protocol prefix (e.g., `node:fs`) is recommended for built-in modules to clearly distinguish them from npm packages.

---

## 6. How does Node.js handle child processes, and what are the use cases?

**Answer**: Node.js uses the `child_process` module to spawn, fork, or execute child processes. Use cases include running CPU-intensive tasks, executing shell commands, or parallelizing work.

**Code Sample**:

```javascript
const { spawn } = require("child_process");
const ls = spawn("ls", ["-lh"]);
ls.stdout.on("data", (data) => console.log(`Output: ${data}`));
ls.on("error", (err) => console.error(err));
```

---

## 7. What is middleware in Express, and how do you create custom middleware?

**Answer**: Middleware in Express is a function that processes requests in the request-response cycle. It can modify requests, responses, or terminate the cycle. Custom middleware is created as a function with `req`, `res`, and `next` parameters.

**Code Sample**:

```javascript
const express = require("express");
const app = express();
const logger = (req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
};
app.use(logger);
app.get("/", (req, res) => res.send("Hello"));
app.listen(3000);
```

---

## 8. How can you improve the performance of a Node.js application?

**Answer**: Use clustering, caching (e.g., Redis), load balancing, asynchronous code, streams for large data, and optimize database queries. Profile with tools like `clinic.js` or `--inspect` to identify bottlenecks.

**Code Sample**:

```javascript
import cluster from "node:cluster";
import http from "node:http";
import { availableParallelism } from "node:os";

const numCPUs = availableParallelism();

if (cluster.isPrimary) {
  for (let i = 0; i < numCPUs; i++) cluster.fork();
  cluster.on("exit", (worker) => {
    console.log(`Worker ${worker.process.pid} exited. Restarting...`);
    cluster.fork();
  });
} else {
  http.createServer((req, res) => res.end("Hello")).listen(3000);
}
```

> **Note:** `cluster.isMaster` is deprecated since Node.js v16. Use `cluster.isPrimary` instead. Similarly, use `os.availableParallelism()` (Node.js v19.4+) over `os.cpus().length`.

---

## 9. What is the purpose of the `package.json` file in a Node.js project?

**Answer**: `package.json` defines project metadata, dependencies, scripts, and configuration. It enables reproducible builds and dependency management with npm, Yarn, or pnpm.

**Code Sample**:

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module",
  "engines": {
    "node": ">=20.0.0"
  },
  "dependencies": {
    "express": "^5.1.0"
  },
  "scripts": {
    "start": "node index.js",
    "dev": "node --watch index.js"
  }
}
```

> **Note:** Node.js v18.11+ supports `--watch` mode natively, reducing the need for tools like `nodemon` during development.

---

## 10. How do you handle environment variables in Node.js?

**Answer**: Use `process.env` to access environment variables. Store them in a `.env` file and load with the `dotenv` package for security and portability.

**Code Sample**:

```javascript
require("dotenv").config();
console.log(process.env.DB_HOST);
```

---

## 11. What is the difference between `setTimeout`, `setImmediate`, and `process.nextTick`?

**Answer**: `setTimeout` schedules a callback after a delay, `setImmediate` runs after the poll phase, and `process.nextTick` runs before the next Event Loop iteration. `nextTick` has the highest priority.

**Code Sample**:

```javascript
setTimeout(() => console.log("Timeout"), 0);
setImmediate(() => console.log("Immediate"));
process.nextTick(() => console.log("NextTick"));
// Output: NextTick -> Immediate -> Timeout
```

---

## 12. How can you secure a Node.js application?

**Answer**: Use helmets for HTTP headers, sanitize inputs, use parameterized queries for SQL, implement JWT for authentication, and rate-limit APIs. Regularly update dependencies.

**Code Sample**:

```javascript
const express = require("express");
const helmet = require("helmet");
const app = express();
app.use(helmet());
app.get("/", (req, res) => res.send("Secure"));
app.listen(3000);
```

---

## 13. What is the `cluster` module in Node.js, and how does it work?

**Answer**: The `cluster` module allows Node.js to utilize multiple CPU cores by forking worker processes. The primary process distributes incoming connections to workers.

**Code Sample**:

```javascript
import cluster from "node:cluster";
import http from "node:http";

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);
  cluster.fork();
  cluster.fork();
} else {
  http
    .createServer((req, res) => res.end(`Worker ${process.pid}`))
    .listen(3000);
  console.log(`Worker ${process.pid} started`);
}
```

> **Note:** `cluster.isMaster` and the `'master'` event are deprecated. Use `cluster.isPrimary` and `'primary'` instead.

---

## 14. How do you handle file uploads in a Node.js application?

**Answer**: Use middleware like `multer` to handle multipart/form-data for file uploads. Store files on disk or in cloud storage like S3.

**Code Sample**:

```javascript
const express = require("express");
const multer = require("multer");
const upload = multer({ dest: "uploads/" });
const app = express();
app.post("/upload", upload.single("file"), (req, res) => res.send("Uploaded"));
app.listen(3000);
```

---

## 15. What are Promises in Node.js, and how do they differ from callbacks?

**Answer**: Promises represent the eventual completion of an asynchronous operation, offering `.then()` and `.catch()` for handling. They avoid callback hell and provide cleaner error handling compared to callbacks.

**Code Sample**:

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => resolve("Done"), 1000);
});
promise
  .then((result) => console.log(result))
  .catch((err) => console.error(err));
```

---

## 16. How does Node.js handle DNS resolution?

**Answer**: Node.js uses the `dns` module for DNS resolution, leveraging libuv for asynchronous lookups. It supports methods like `dns.lookup` and `dns.resolve`.

**Code Sample**:

```javascript
const dns = require("dns");
dns.lookup("example.com", (err, address) => {
  if (err) console.error(err);
  console.log(`Address: ${address}`);
});
```

---

## 17. What is the purpose of the `buffer` module in Node.js?

**Answer**: The `buffer` module handles binary data, useful for streams, file operations, or network protocols. Buffers are fixed-size chunks of memory outside the V8 heap.

**Code Sample**:

```javascript
const buf = Buffer.from("Hello");
console.log(buf.toString("hex")); // Output: 48656c6c6f
```

---

## 18. How can you implement rate limiting in a Node.js application?

**Answer**: Use middleware like `express-rate-limit` to restrict the number of requests per IP over a time window, preventing abuse or DDoS attacks.

**Code Sample**:

```javascript
const express = require("express");
const rateLimit = require("express-rate-limit");
const app = express();
app.use(rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }));
app.get("/", (req, res) => res.send("Hello"));
app.listen(3000);
```

---

## 19. What is the difference between `fork` and `spawn` in the `child_process` module?

**Answer**: `fork` creates a new Node.js process with an IPC channel, ideal for running JavaScript files. `spawn` launches any command-line process, offering more flexibility but no built-in IPC.

**Code Sample**:

```javascript
const { fork } = require("child_process");
const child = fork("child.js");
child.send("Hello");
child.on("message", (msg) => console.log(`Child: ${msg}`));
```

---

## 20. How do you implement WebSockets in Node.js?

**Answer**: Use the `ws` library or `Socket.IO` for real-time bidirectional communication. WebSockets are ideal for chat apps, live updates, or gaming.

**Code Sample**:

```javascript
const WebSocket = require("ws");
const wss = new WebSocket.Server({ port: 8080 });
wss.on("connection", (ws) => {
  ws.on("message", (msg) => ws.send(`Echo: ${msg}`));
});
```

---

## 21. What is the `async_hooks` module, and when is it used?

**Answer**: The `async_hooks` module tracks asynchronous resources, useful for debugging or monitoring async operations. It’s advanced and typically used in diagnostics tools.

**Code Sample**:

```javascript
const asyncHooks = require("async_hooks");
const hook = asyncHooks.createHook({
  init(asyncId, type) {
    console.log(`Init ${type}: ${asyncId}`);
  },
});
hook.enable();
setTimeout(() => {}, 1000);
```

---

## 22. How do you handle large JSON payloads in Node.js?

**Answer**: Use streams to process large JSON data incrementally with libraries like `JSONStream`. Avoid loading the entire payload into memory.

**Code Sample**:

```javascript
const JSONStream = require("JSONStream");
const fs = require("fs");
const stream = fs.createReadStream("large.json").pipe(JSONStream.parse("*"));
stream.on("data", (data) => console.log(data));
```

---

## 23. What is the difference between `http` and `https` modules in Node.js?

**Answer**: The `http` module handles HTTP requests, while `https` handles secure HTTPS requests using TLS/SSL. `https` requires certificates for encryption.

**Code Sample**:

```javascript
const https = require("https");
const fs = require("fs");
const options = {
  key: fs.readFileSync("key.pem"),
  cert: fs.readFileSync("cert.pem"),
};
https.createServer(options, (req, res) => res.end("Secure")).listen(443);
```

---

## 24. How can you optimize database queries in a Node.js application?

**Answer**: Use connection pooling, indexes, parameterized queries, and caching. Libraries like `knex` or ORMs like `Sequelize` help optimize queries.

**Code Sample**:

```javascript
const { Pool } = require("pg");
const pool = new Pool({ user: "user", database: "db" });
pool.query("SELECT * FROM users WHERE id = $1", [1], (err, res) => {
  console.log(res.rows);
});
```

---

## 25. What is the purpose of the `os` module in Node.js?

**Answer**: The `os` module provides system-related information like CPU count, memory, or platform. It’s useful for system monitoring or clustering.

**Code Sample**:

```javascript
const os = require("os");
console.log(`CPUs: ${os.cpus().length}, Free Memory: ${os.freemem()}`);
```

---

## 26. How do you implement authentication in a Node.js application?

**Answer**: Use JWT or OAuth for token-based authentication. Libraries like `passport` simplify integrating strategies like local, Google, or GitHub auth.

**Code Sample**:

```javascript
const jwt = require("jsonwebtoken");
const token = jwt.sign({ userId: 1 }, "secret", { expiresIn: "1h" });
console.log(token);
```

---

## 27. What are microservices, and how can Node.js support them?

**Answer**: Microservices are independent, modular services communicating via APIs. Node.js supports them with lightweight frameworks like Express and tools like Docker or Kafka.

**Code Sample**:

```javascript
const express = require("express");
const app = express();
app.get("/service", (req, res) => res.json({ status: "Up" }));
app.listen(3001);
```

---

## 28. How do you handle memory leaks in Node.js?

**Answer**: Use tools like `heapdump` or `--inspect` to profile memory. Avoid global variables, unclosed streams, or unhandled Promises. Monitor with `process.memoryUsage()`.

**Code Sample**:

```javascript
console.log(process.memoryUsage());
setInterval(() => {
  console.log(process.memoryUsage().heapUsed / 1024 / 1024, "MB");
}, 1000);
```

---

## 29. What is the `crypto` module in Node.js, and how is it used?

**Answer**: The `crypto` module provides cryptographic functions like hashing, encryption, or signing. It’s used for secure data handling or password hashing.

**Code Sample**:

```javascript
const crypto = require("crypto");
const hash = crypto.createHash("sha256").update("password").digest("hex");
console.log(hash);
```

---

## 30. How do you implement caching in a Node.js application?

**Answer**: Use in-memory caching with `node-cache` or distributed caching with Redis. Cache frequently accessed data to reduce database load.

**Code Sample**:

```javascript
const NodeCache = require("node-cache");
const cache = new NodeCache({ stdTTL: 100 });
cache.set("key", "value");
console.log(cache.get("key"));
```

---

## 31. What is the difference between `EventEmitter` and `process` events?

**Answer**: `EventEmitter` is a class for custom event handling, while `process` events handle system-level events like `exit` or `uncaughtException`.

**Code Sample**:

```javascript
const EventEmitter = require("events");
const emitter = new EventEmitter();
emitter.on("event", () => console.log("Triggered"));
emitter.emit("event");
```

---

## 32. How do you handle CORS in a Node.js application?

**Answer**: Use the `cors` middleware in Express to enable Cross-Origin Resource Sharing, allowing controlled access to APIs from different domains.

**Code Sample**:

```javascript
const express = require("express");
const cors = require("cors");
const app = express();
app.use(cors());
app.get("/", (req, res) => res.send("CORS enabled"));
app.listen(3000);
```

---

## 33. What is the `worker_threads` module, and when is it used?

**Answer**: The `worker_threads` module enables multi-threading in Node.js for CPU-intensive tasks, unlike the single-threaded Event Loop.

**Code Sample**:

```javascript
const { Worker, isMainThread } = require("worker_threads");
if (isMainThread) {
  new Worker(__filename);
} else {
  console.log("Worker running");
}
```

---

## 34. How do you implement a REST API in Node.js?

**Answer**: Use Express to define routes, handle HTTP methods, and process requests/responses. Follow REST principles like statelessness and resource-based URLs.

**Code Sample**:

```javascript
const express = require("express");
const app = express();
app.use(express.json());
app.get("/users/:id", (req, res) => res.json({ id: req.params.id }));
app.listen(3000);
```

---

## 35. What is the purpose of the `zlib` module in Node.js?

**Answer**: The `zlib` module provides compression and decompression, useful for reducing data size in HTTP responses or file operations.

**Code Sample**:

```javascript
const zlib = require("zlib");
const input = "Hello";
zlib.gzip(input, (err, compressed) =>
  console.log(compressed.toString("base64")),
);
```

---

## 36. How do you handle versioning in a Node.js API?

**Answer**: Use URL versioning (e.g., `/v1/resource`), headers, or query parameters. URL versioning is the most common for simplicity.

**Code Sample**:

```javascript
const express = require("express");
const app = express();
app.get("/v1/users", (req, res) => res.send("Version 1"));
app.listen(3000);
```

---

## 37. What is the `Transform` stream, and how is it used?

**Answer**: A `Transform` stream modifies data as it passes through, like compressing or parsing. It’s both readable and writable.

**Code Sample**:

```javascript
const { Transform } = require("stream");
const upperCaseTr = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  },
});
process.stdin.pipe(upperCaseTr).pipe(process.stdout);
```

---

## 38. How do you debug a Node.js application in production?

**Answer**: Use logging with `winston`, monitor with Prometheus, or debug remotely with `--inspect`. Avoid `console.log` in production for performance.

**Code Sample**:

```javascript
const winston = require("winston");
const logger = winston.createLogger({
  transports: [new winston.transports.File({ filename: "app.log" })],
});
logger.info("App started");
```

---

## 39. How does Node.js compare to Deno and Bun?

**Answer**: All three are JavaScript/TypeScript server runtimes.

- **Node.js**: The most mature runtime with the largest ecosystem (npm). Supports both CJS and ESM. Uses V8 engine.
- **Deno**: Created by Node.js's original author. Built-in TypeScript support, security-first (permissions model), uses ESM by default. Deno 2.0+ is fully backwards-compatible with npm packages.
- **Bun**: Fastest runtime (uses JavaScriptCore instead of V8). Built-in bundler, transpiler, test runner, and package manager. Aims for Node.js API compatibility.

**Code Sample** (Node.js with ESM):

```javascript
import { createServer } from "node:http";
createServer((req, res) => res.end("Hello")).listen(3000);
```

---

## 40. How do you handle graceful shutdown in Node.js?

**Answer**: Listen for `SIGTERM` or `SIGINT`, close connections, and exit cleanly to prevent data loss or corruption.

**Code Sample**:

```javascript
const http = require("http");
const server = http.createServer((req, res) => res.end("Hello"));
server.listen(3000);
process.on("SIGTERM", () => {
  server.close(() => process.exit(0));
});
```

---

## 41. What is GraphQL, and how can it be implemented in Node.js?

**Answer**: GraphQL is a query language for APIs, allowing clients to request specific data. Use `graphql-yoga` or `@apollo/server` in Node.js for implementation.

**Code Sample**:

```javascript
import { createSchema, createYoga } from "graphql-yoga";
import { createServer } from "node:http";

const yoga = createYoga({
  schema: createSchema({
    typeDefs: `
      type Query {
        hello: String
      }
    `,
    resolvers: {
      Query: {
        hello: () => "Hello from GraphQL!",
      },
    },
  }),
});

createServer(yoga).listen(3000, () => {
  console.log("GraphQL server running at http://localhost:3000/graphql");
});
```

---

## 42. How do you implement pagination in a Node.js API?

**Answer**: Use query parameters like `page` and `offset` or cursor-based pagination for scalability. Return metadata with total items.

**Code Sample**:

```javascript
const express = require("express");
const app = express();
app.get("/items", (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = 10;
  const offset = (page - 1) * limit;
  res.json({ items: [], total: 100, page, limit });
});
app.listen(3000);
```

---

## 43. What is the `vm` module in Node.js, and when is it used?

**Answer**: The `vm` module runs JavaScript in a sandboxed context, useful for executing untrusted code or testing scripts safely.

**Code Sample**:

```javascript
const vm = require("vm");
const script = new vm.Script("result = 1 + 1");
const context = { result: 0 };
script.runInContext(vm.createContext(context));
console.log(context.result); // Output: 2
```

---

## 44. How do you handle long-polling in Node.js?

**Answer**: Use timeouts to keep HTTP connections open until data is available or the request times out. WebSockets are preferred for real-time.

**Code Sample**:

```javascript
const express = require("express");
const app = express();
app.get("/poll", (req, res) => {
  setTimeout(() => res.send("Data"), 5000);
});
app.listen(3000);
```

---

## 45. What is the purpose of the `path` module in Node.js?

**Answer**: The `path` module handles file paths, ensuring cross-platform compatibility for operations like joining or resolving paths.

**Code Sample**:

```javascript
const path = require("path");
console.log(path.join(__dirname, "file.txt"));
```

---

## 46. How do you implement a message queue in Node.js?

**Answer**: Use libraries like `bull` (Redis-based) or `kafkajs` for message queues to handle background jobs or inter-service communication.

**Code Sample**:

```javascript
const Queue = require("bull");
const queue = new Queue("my-queue");
queue.add({ task: "process" });
queue.process((job) => console.log(job.data));
```

---

## 47. What is the difference between `global` and `process` in Node.js?

**Answer**: `global` is the global namespace for variables, while `process` is an object providing information about the current Node.js process.

**Code Sample**:

```javascript
global.myVar = "test";
console.log(process.argv, global.myVar);
```

---

## 48. How do you handle file compression in Node.js?

**Answer**: Use the `zlib` module or libraries like `archiver` to compress files or streams, often for backups or downloads.

**Code Sample**:

```javascript
const fs = require("fs");
const zlib = require("zlib");
fs.createReadStream("file.txt")
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream("file.txt.gz"));
```

---

## 49. What is server-sent events (SSE) in Node.js, and how is it implemented?

**Answer**: SSE enables servers to send events to clients HTTP connections over, useful for real-time updates. Set `Content-Type: text/event-stream`.

**Code Sample**:

```javascript
const express = require("http");
const http = http.createServer((req, res) => {
  if (req.url === "/events") {
    res.writeHead(200, {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      Connection: "keep-alive",
    });
    setInterval(() => res.write(`data: ${new Date().toISOString()}\n/n`), 1000);
  } else {
    res.writeHead(404);
    res.end();
  }
});
http.listen(3000);
```

---

## 50. How do you profile a Node.js application for CPU usage?

**Answer**: Use `--prof` flag, analyze with `--prof-process`, or tools like `clinic.js`. Monitor CPU with `process.cpuUsage()` for real-time insights.

**Code Sample**:

```javascript
const startUsage = process.cpuUsage();

setInterval(() => {
  const usage = process.cpuUsage(startUsage);
  console.log(
    `User: ${(usage.user / 1_000_000).toFixed(2)}s, System: ${(usage.system / 1_000_000).toFixed(2)}s`,
  );
}, 1000);
```

## 51. What is the difference between `dependencies` and `devDependencies` in `package.json`?

**Answer**: `dependencies` are required for the application to run in production, while `devDependencies` are only needed for development or testing (e.g., testing frameworks, linters). `npm install --production` skips `devDependencies`.

**Code Sample**:

```json
{
  "dependencies": {
    "express": "^4.17.1"
  },
  "devDependencies": {
    "jest": "^29.0.0"
  }
}
```

---

## 52. How does the `scripts` field in `package.json` work, and how can you run custom scripts?

**Answer**: The `scripts` field defines commands executable via `npm run <script-name>`. Predefined scripts like `start` or `test` can be run directly (e.g., `npm start`). Custom scripts are run with `npm run <name>`.

**Code Sample**:

```json
{
  "scripts": {
    "start": "node index.js",
    "build": "webpack --config webpack.config.js"
  }
}
```

---

## 53. What is the `engines` field in `package.json`, and why is it useful?

**Answer**: The `engines` field specifies the Node.js or npm versions compatible with the project. It helps ensure consistency across environments and prevents runtime issues due to version mismatches.

**Code Sample**:

```json
{
  "engines": {
    "node": ">=14.0.0",
    "npm": ">=6.0.0"
  }
}
```

---

## 54. How does the `type` field in `package.json` affect module resolution?

**Answer**: The `type` field, set to `"module"` or `"commonjs"`, determines whether `.js` files are treated as ES Modules (ESM) or CommonJS. If `"module"`, `import`/`export` syntax is used; otherwise, `require`/`module.exports` is assumed.

**Code Sample**:

```json
{
  "type": "module",
  "main": "index.js"
}
```

---

## 55. What is the purpose of the `peerDependencies` field in `package.json`?

**Answer**: `peerDependencies` specify dependencies that the consuming project must provide, typically for plugins or libraries that depend on a shared framework (e.g., React). It avoids version conflicts.

**Code Sample**:

```json
{
  "peerDependencies": {
    "react": "^18.0.0"
  }
}
```

---

## 56. How can you use `package.json` to manage different environments (e.g., development vs. production)?

**Answer**: Use `scripts` with environment-specific commands or `NODE_ENV` to toggle behavior. Tools like `dotenv` can load environment variables from `.env` files referenced in `package.json` scripts.

**Code Sample**:

```json
{
  "scripts": {
    "start:dev": "NODE_ENV=development node index.js",
    "start:prod": "NODE_ENV=production node index.js"
  }
}
```

---

## 57. What is the `workspaces` field in `package.json`, and how is it used in monorepos?

**Answer**: The `workspaces` field enables managing multiple packages in a monorepo by linking them under a single `node_modules`. It simplifies dependency management and script execution across packages.

**Code Sample**:

```json
{
  "workspaces": ["packages/*"]
}
```

---

## 58. How does the `resolutions` field in `package.json` help resolve dependency conflicts?

**Answer**: The `resolutions` field (used with Yarn) forces a specific version of a dependency to resolve conflicts when multiple packages require different versions of the same dependency.

**Code Sample**:

```json
{
  "resolutions": {
    "lodash": "4.17.21"
  }
}
```

---

## 59. What is the `bin` field in `package.json`, and how does it create executable scripts?

**Answer**: The `bin` field maps command names to executable scripts, making them available globally or locally when the package is installed. It’s used for CLI tools.

**Code Sample**:

```json
{
  "bin": {
    "mycli": "./bin/cli.js"
  }
}
```

---

## 60. How can you use `package.json` to enforce dependency constraints with `dependency-check` or similar tools?

**Answer**: Tools like `dependency-check` analyze `package.json` to ensure all required dependencies are listed and unused ones are removed. The `dependencies` and `devDependencies` fields are validated against imports in the codebase.

**Code Sample**:

```json
{
  "scripts": {
    "check-deps": "dependency-check . --unused --no-dev"
  },
  "dependencies": {
    "express": "^4.17.1"
  }
}
```
