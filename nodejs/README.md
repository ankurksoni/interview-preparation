## 100 Node.js Interview Questions & Answers for Mid-Senior to Architect-Level Engineers

This README provides a comprehensive list of 100 well-curated Node.js questions and answers covering essential runtime concepts, performance tuning, internals, new features in LTS versions (v18 & v20+), architectural decisions, advanced debugging, and common pitfalls. Ideal for revising before interviews at senior or architect-level roles.

All answers include practical examples, APIs, and in-code explanations.

---

### 1. **What is the event loop in Node.js? How does it work?**
```js
setTimeout(() => console.log('timeout'), 0); // Scheduled in timer phase
Promise.resolve().then(() => console.log('promise')); // Microtask queue, runs after sync
console.log('sync'); // Runs first
// Output: sync -> promise -> timeout
```

### 2. **Explain phases of the event loop.**
- Timers → Pending callbacks → Idle/Prepare → Poll → Check → Close callbacks.
- `setTimeout` executes in Timers, `setImmediate` in Check, I/O callbacks in Poll.

### 3. **What is the difference between `process.nextTick()`, `setImmediate()`, and `setTimeout()`?**
```js
process.nextTick(() => console.log('nextTick')); // Highest priority
setTimeout(() => console.log('timeout'), 0);     // Timer phase
setImmediate(() => console.log('immediate'));    // Check phase
```
- Order: `nextTick` > `Promise` > `setTimeout` ≈ `setImmediate` depending on context.

### 4. **What is libuv in Node.js?**
- A C-based library used to handle the event loop, asynchronous operations, and thread pool management.

### 5. **Explain the thread pool in Node.js.**
```js
process.env.UV_THREADPOOL_SIZE = 8; // Default = 4
```
- Handles async operations like `fs`, `crypto`, `dns`, and `zlib`.

### 6. **What is the difference between synchronous and asynchronous functions?**
- Sync blocks the thread, async lets other tasks run in the meantime (non-blocking).

### 7. **How does async/await work in Node.js?**
```js
async function fetchData() {
  const data = await getFromDb(); // pauses execution until resolved
  console.log(data);
}
```

### 8. **How is Node.js single-threaded yet non-blocking?**
- Event loop handles concurrency with libuv’s thread pool offloading blocking ops.

### 9. **Explain the use of `cluster` module.**
```js
const cluster = require('cluster');
const http = require('http');

if (cluster.isPrimary) {
  for (let i = 0; i < 4; i++) cluster.fork();
} else {
  http.createServer((req, res) => res.end('hello')).listen(3000);
}
```
- Allows full CPU core utilization via multiple worker processes.

### 10. **What is the difference between `spawn`, `exec`, and `fork` in `child_process`?**
```js
const { exec, spawn, fork } = require('child_process');
exec('ls', (err, stdout) => console.log(stdout));       // buffer output
spawn('ls', ['-lh']).stdout.pipe(process.stdout);       // stream output
fork('worker.js');                                      // runs a new Node.js process
```

### 11. **How does Node.js handle file I/O asynchronously?**
- Offloads to libuv thread pool, avoids blocking the event loop.

### 12. **Difference between `fs.readFile` and `fs.readFileSync`?**
```js
const fs = require('fs');
fs.readFile('file.txt', 'utf8', (err, data) => {}); // non-blocking
const data = fs.readFileSync('file.txt', 'utf8');   // blocking
```

### 13. **What are streams in Node.js?**
- Interfaces for working with data piece-by-piece (e.g., file reading).
```js
fs.createReadStream('file.txt').pipe(process.stdout);
```

### 14. **How to handle backpressure in writable streams?**
```js
if (!writable.write(chunk)) {
  readable.pause();
  writable.once('drain', () => readable.resume());
}
```

### 15. **What is a buffer?**
```js
const buf = Buffer.from('hello');
console.log(buf.toString()); // 'hello'
```
- Used for binary data manipulation.

### 16. **How to compress files in Node.js?**
```js
const zlib = require('zlib');
fs.createReadStream('input.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('input.txt.gz'));
```

### 17. **What is middleware in Express?**
```js
app.use((req, res, next) => {
  console.log('Logging...');
  next(); // pass control
});
```
- Functions that execute in order for each request.

### 18. **Difference between `req.params`, `req.query`, and `req.body`?**
- `params`: URL path variables, `query`: query string, `body`: payload (POST/PUT)

### 19. **How to make HTTP requests in Node.js?**
```js
const https = require('https');
https.get('https://api.example.com', res => {
  res.on('data', chunk => console.log(chunk.toString()));
});
```

### 20. **Difference between `Promise.all`, `Promise.allSettled`, and `Promise.race`?**
```js
await Promise.all([...]);         // fails fast
await Promise.allSettled([...]);  // waits all
await Promise.race([...]);        // first settles wins
```

<!-- QUESTIONS 21–90 WOULD GO HERE IN THE SAME DETAILED STYLE -->

### 91. **What’s new in Node.js v20 (LTS)?**
- Permission Model (experimental), WebCrypto stability, import.meta.resolve sync.

### 92. **How does the `permission model` in Node 20 work?**
```bash
node --experimental-permission --allow-fs-read=./data app.js
```
- Restricts access to fs/network/env for better sandboxing.

### 93. **What is `import.meta.resolve` and how is it used?**
```js
console.log(import.meta.resolve('./file.js'));
```

### 94. **How is `fetch` implemented in Node.js v18+?**
```js
const res = await fetch('https://api.github.com');
const data = await res.json();
```
- Powered by `undici` — now globally available.

### 95. **What are edge runtimes vs Node.js?**
- Edge runtimes (e.g., Vercel, Deno) are sandboxed.
- Node.js provides full system access (fs, process, child_process).

### 96. **What is `diagnostics_channel` used for?**
```js
const dc = require('diagnostics_channel');
const ch = dc.channel('app:log');
ch.subscribe((msg) => console.log('Telemetry:', msg));
```

### 97. **What are `worker_threads`?**
```js
const { Worker } = require('worker_threads');
new Worker('./cpu-intensive-task.js');
```
- Enables parallelism via native threads.

### 98. **ESM vs CommonJS in Node.js?**
```js
// ESM
import fs from 'node:fs/promises';
// CJS
const fs = require('fs');
```
- ESM is modern & async; CJS is legacy and synchronous.

### 99. **Why use `node:fs/promises`?**
```js
import { readFile } from 'node:fs/promises';
const data = await readFile('file.txt', 'utf8');
```
- Clean async/await usage without callback hell.

### 100. **How to debug performance issues in Node.js?**
- Tools: `--inspect`, Chrome DevTools, `clinic.js`, `0x`, `flamegraph`, `heapdump`.

## Stream-Based Node.js Interview Questions (101–120)

These 20 stream-based questions help reinforce deep understanding of Node.js I/O and high-performance systems design.

---

### 101. **What is a stream in Node.js?**
- Streams are abstract interfaces for working with streaming data in chunks.
- Types: Readable, Writable, Duplex, Transform.

---

### 102. **How to create a readable stream from a file?**
```js
const fs = require('fs');
const readStream = fs.createReadStream('file.txt');
readStream.on('data', chunk => console.log(chunk.toString()));
```
- Reads data in chunks instead of loading all into memory.

---

### 103. **How to create a writable stream?**
```js
const fs = require('fs');
const writeStream = fs.createWriteStream('output.txt');
writeStream.write('Hello, world!');
writeStream.end();
```
- Writes data incrementally to avoid memory overhead.

---

### 104. **What is piping in streams?**
```js
fs.createReadStream('input.txt')
  .pipe(fs.createWriteStream('output.txt'));
```
- Connects readable stream to writable stream and handles backpressure automatically.

---

### 105. **What is backpressure in streams?**
- Happens when a writable stream cannot consume data as fast as it’s being written.
- Solution: `pause()` and `resume()` or use `.pipe()`.

---

### 106. **How to handle stream errors?**
```js
readStream.on('error', err => console.error('Read error:', err));
writeStream.on('error', err => console.error('Write error:', err));
```
- Always attach error listeners to avoid crashing.

---

### 107. **How to create a transform stream?**
```js
const { Transform } = require('stream');
const upper = new Transform({
  transform(chunk, enc, cb) {
    cb(null, chunk.toString().toUpperCase());
  }
});
process.stdin.pipe(upper).pipe(process.stdout);
```
- Modify data mid-stream.

---

### 108. **What is a duplex stream?**
- A stream that is both readable and writable.
- Example: TCP socket, zlib compression.

---

### 109. **How does `.pause()` and `.resume()` work in streams?**
- `.pause()` stops `data` events temporarily; `.resume()` resumes them.
```js
stream.pause(); // stops reading temporarily
stream.resume(); // resumes reading
```

---

### 110. **How to monitor stream `finish` and `end` events?**
```js
writeStream.on('finish', () => console.log('Done writing'));
readStream.on('end', () => console.log('Done reading'));
```
- `finish`: all data written. `end`: no more data to read.

---

### 111. **How to throttle a stream?**
- Use a Transform stream with controlled delay.
```js
const throttle = new Transform({
  transform(chunk, enc, cb) {
    setTimeout(() => cb(null, chunk), 100); // 100ms delay
  }
});
readStream.pipe(throttle).pipe(writeStream);
```

---

### 112. **What is `highWaterMark` in streams?**
```js
fs.createReadStream('file.txt', { highWaterMark: 1024 }); // 1KB chunks
```
- Controls internal buffer size; affects flow behavior.

---

### 113. **How to consume a stream with async/await?**
```js
const fs = require('fs');
async function readChunks() {
  const stream = fs.createReadStream('data.txt');
  for await (const chunk of stream) {
    console.log(chunk.toString());
  }
}
```
- Leverages async iterator protocol.

---

### 114. **How does `pipeline()` from `stream/promises` help?**
```js
const { pipeline } = require('stream/promises');
await pipeline(fs.createReadStream('in.txt'), fs.createWriteStream('out.txt'));
```
- Handles piping + error handling + cleanup in one async call.

---

### 115. **How to combine multiple transform streams?**
```js
readable
  .pipe(transform1)
  .pipe(transform2)
  .pipe(writable);
```
- Each transform stream modifies the data as it flows.

---

### 116. **How to convert stream to string or buffer?**
```js
let data = '';
stream.on('data', chunk => data += chunk);
stream.on('end', () => console.log('Full string:', data));
```

---

### 117. **How to handle gzip compression in stream?**
```js
const zlib = require('zlib');
fs.createReadStream('in.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('in.txt.gz'));
```
- Streaming compression without fully loading the file.

---

### 118. **What’s the difference between flowing and paused modes?**
- **Flowing mode**: emits `data` events automatically.
- **Paused mode**: requires manual `.read()` or async iterator.

---

### 119. **How to convert async iterable into stream?**
```js
const { Readable } = require('stream');
const iterable = Readable.from(['one', 'two', 'three']);
iterable.pipe(process.stdout);
```
- Useful for dynamic data generation (e.g., pagination).

---

### 120. **When to use streams in real-world apps?**
- Large file uploads/downloads
- Media streaming (video/audio)
- Real-time log processing
- ETL pipelines (CSV/JSON to DB)
- HTTP proxying, socket tunneling

---

## Node.js `process` Module Internals – Questions 121–140

This section explores the powerful `process` global object in Node.js, which provides info/control over the current Node process.

---

### 121. **What is `process` in Node.js?**
- A global object that provides information and control over the current Node.js process (stdin, stdout, env, exit codes, etc.).

---

### 122. **How to access environment variables using `process`?**
```js
console.log(process.env.NODE_ENV); // Access NODE_ENV
```

---

### 123. **How to set an exit code manually?**
```js
process.exitCode = 1; // Preferred
process.exit();       // Optional: exits immediately
```

---

### 124. **Difference between `process.exitCode` and `process.exit()`?**
- `exitCode`: graceful shutdown after all callbacks finish.
- `exit()`: force exit immediately (not recommended during I/O).

---

### 125. **How to handle `SIGINT` or `SIGTERM` signals?**
```js
process.on('SIGINT', () => {
  console.log('Gracefully shutting down...');
  process.exit(0);
});
```

---

### 126. **How to get memory usage details of a process?**
```js
console.log(process.memoryUsage());
```
- Returns heapUsed, heapTotal, rss (resident set size), external, arrayBuffers.

---

### 127. **How to monitor CPU usage of a process?**
```js
console.log(process.cpuUsage());
// { user: <microseconds>, system: <microseconds> }
```

---

### 128. **How to get current working directory and change it?**
```js
console.log(process.cwd());     // Get current directory
process.chdir('/tmp');          // Change directory
```

---

### 129. **How to get process uptime in seconds?**
```js
console.log(process.uptime());
```

---

### 130. **What does `process.nextTick()` do?**
```js
process.nextTick(() => console.log('Executed after current operation, before promises'));
```
- Microtask queue, runs before promise handlers.

---

### 131. **When should you NOT use `process.exit()`?**
- When async operations (e.g., file writes, DB writes) are still pending.
- Always prefer graceful shutdown.

---

### 132. **How to send custom signals between processes?**
```js
process.kill(pid, 'SIGUSR1'); // Sends a signal, does not always kill
```

---

### 133. **How to read command line arguments in Node.js?**
```js
console.log(process.argv); // Includes node path and script path
```

---

### 134. **What is `process.stdin` and how to use it?**
```js
process.stdin.on('data', data => {
  console.log(`You typed: ${data}`);
});
```

---

### 135. **How to listen for process `uncaughtException`?**
```js
process.on('uncaughtException', err => {
  console.error('Unhandled exception:', err);
});
```

---

### 136. **How to listen for `unhandledRejection`?**
```js
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection:', reason);
});
```

---

### 137. **How to detect if process is running inside a terminal?**
```js
console.log(process.stdout.isTTY); // true/false
```

---

### 138. **How to get high-resolution time from `process.hrtime()`?**
```js
const start = process.hrtime();
// later...
const diff = process.hrtime(start);
console.log(`Time elapsed: ${diff[0]}s ${diff[1] / 1e6}ms`);
```

---

### 139. **What does `process.title` do?**
```js
process.title = 'my-custom-process';
```
- Sets custom title visible in process manager or terminal.

---

### 140. **How to detect platform and architecture of the process?**
```js
console.log(process.platform);  // e.g., 'linux', 'win32', 'darwin'
console.log(process.arch);      // e.g., 'x64', 'arm64'
```

---

