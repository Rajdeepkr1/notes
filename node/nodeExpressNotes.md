# Node.js & Express — Deep Dive Roadmap

We'll go from fundamentals → internals → architecture → performance → testing → production → interview problems.

---

## 1. Node.js Fundamentals

**Definition:** Node.js is a JavaScript **runtime** built on Google's V8 engine that executes JavaScript outside the browser, giving it access to the filesystem, network, and OS — the foundation for using JS as a general-purpose server-side/scripting language.

**The `global` object. Definition:** Node's equivalent of the browser's `window` — an object whose properties are available anywhere without importing, holding things like `process`, `Buffer`, `setTimeout`, and (in CommonJS files) module-scoped helpers like `__dirname`.

**Modules — Definition:** a module is a single file whose code is scoped to itself by default; Node supports two module systems:

- **CommonJS (CJS)** — Node's original system, synchronous, using `require()`/`module.exports`:
```js
// math.js
function add(a, b) { return a + b; }
module.exports = { add };

// app.js
const { add } = require('./math');
```
- **ES Modules (ESM)** — the standard JS module system, using `import`/`export`, asynchronous by design, enabled via `"type": "module"` in `package.json` or a `.mjs` extension:
```js
// math.mjs
export function add(a, b) { return a + b; }

// app.mjs
import { add } from './math.mjs';
```

**`package.json` — Definition:** the manifest file describing a Node project — its name, version, dependencies, entry point, and scripts.

```json
{
  "name": "my-api",
  "version": "1.0.0",
  "type": "module",
  "scripts": { "start": "node src/index.js", "dev": "node --watch src/index.js" },
  "dependencies": { "express": "^4.19.0" }
}
```

**npm / npx / package managers — Definition:** `npm` is Node's default package manager (install, publish, run scripts); `npx` executes a package's binary without installing it globally/permanently; `pnpm` and `yarn` are alternative package managers optimizing install speed and disk usage (`pnpm` via a shared content-addressable store).

**`node_modules` & semver — Definition:** `node_modules` is the directory holding installed dependencies; **semantic versioning** (`MAJOR.MINOR.PATCH`) is the convention `package.json` version ranges rely on — `^1.2.3` allows minor/patch upgrades, `~1.2.3` allows only patch upgrades.

**Built-in globals:**

```js
console.log(__dirname);        // absolute path of the current file's directory (CJS only)
console.log(process.argv);     // CLI arguments
console.log(process.env.PORT); // environment variables
const buf = Buffer.from('hi'); // raw binary data
```

---

## 2. The Event Loop & Async Model

The core of "how Node actually works."

**Definition:** Node.js runs JavaScript on a **single thread** but achieves concurrency for I/O through an **event loop** — a runtime construct that continuously checks for completed asynchronous operations (timers, I/O, callbacks) and executes their associated callback functions, so a slow disk read or network call never blocks the thread from doing other work.

**Call stack — Definition:** the LIFO structure tracking which function is currently executing; synchronous code runs entirely on the call stack before the event loop can process anything else — a long-running synchronous function blocks everything.

**The event loop phases — Definition:** each iteration ("tick") of the event loop passes through a fixed set of phases, each with its own callback queue:

```
┌───────────────────────────┐
│           timers           │  → setTimeout/setInterval callbacks whose time has elapsed
├───────────────────────────┤
│      pending callbacks      │  → I/O callbacks deferred to the next loop iteration
├───────────────────────────┤
│           poll              │  → retrieve new I/O events; execute I/O callbacks
├───────────────────────────┤
│           check             │  → setImmediate() callbacks
├───────────────────────────┤
│       close callbacks       │  → e.g. socket.on('close', ...)
└───────────────────────────┘
```

**`process.nextTick()` vs microtasks vs macrotasks — Definition:**
- `process.nextTick()` callbacks run **immediately after the current operation**, before the event loop continues to any phase — highest priority, Node-specific (not a web standard).
- **Microtasks** (resolved Promise `.then()` callbacks, `queueMicrotask()`) run after `nextTick` callbacks, still before the event loop moves to the next phase.
- **Macrotasks** (`setTimeout`, `setInterval`, I/O callbacks) run in their respective event-loop phase, after all queued microtasks/nextTicks have drained.

```js
console.log('1');
setTimeout(() => console.log('2 (macrotask)'), 0);
Promise.resolve().then(() => console.log('3 (microtask)'));
process.nextTick(() => console.log('4 (nextTick)'));
console.log('5');
// output order: 1, 5, 4, 3, 2
```

**libuv & the thread pool — Definition:** libuv is the C library underlying Node that implements the event loop and provides a **thread pool** (default size 4) for operations that have no native async OS API — file I/O, DNS lookups, some `crypto` functions — so they don't block the main JS thread despite not being truly async at the OS level. Network I/O, by contrast, uses the OS's native async mechanisms directly (no thread pool needed).

**Blocking vs non-blocking I/O — Definition:** a **blocking** call halts the calling thread until it completes (e.g. `fs.readFileSync`); a **non-blocking** call returns immediately and delivers its result later via a callback/Promise (e.g. `fs.readFile`), letting the event loop keep processing other work in the meantime.

**Callbacks — Definition:** a function passed as an argument to be invoked once an asynchronous operation completes; Node's convention is **error-first**: `callback(err, result)`, where a non-null `err` signals failure.

```js
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) return console.error(err);
  console.log(data);
});
```

**Promises / `async`/`await`** — modern replacements for deeply nested callbacks ("callback hell"); see section 10 for deeper coverage.

**Event emitters — Definition:** `EventEmitter` is a core class implementing the publish/subscribe pattern — objects can `emit()` named events, and any number of listeners can `on()` (subscribe to) them; it's the base pattern behind streams, HTTP servers, and much of Node's async API surface.

```js
const { EventEmitter } = require('events');
const emitter = new EventEmitter();
emitter.on('greet', (name) => console.log(`Hello, ${name}`));
emitter.emit('greet', 'Ada');
```

---

## 3. Core Modules

**Definition:** Core modules are built into the Node runtime itself — no `npm install` required — providing the low-level APIs (filesystem, networking, streams) that higher-level frameworks like Express are built on top of.

**`fs` — Definition:** the filesystem module, available in three styles: synchronous (`readFileSync`, blocks the thread), classic async/callback (`readFile`), and Promise-based (`fs.promises` / `node:fs/promises`, works cleanly with `async`/`await`).

```js
import { readFile } from 'node:fs/promises';
const content = await readFile('data.txt', 'utf8');
```

**`path` — Definition:** utilities for working with file/directory paths in a cross-platform way (Windows `\` vs POSIX `/`).

```js
import path from 'node:path';
path.join(__dirname, 'data', 'file.txt');
path.resolve('./relative/path');
```

**`http`/`https` — Definition:** the low-level modules for creating HTTP(S) servers and clients — what Express is built on top of.

```js
import http from 'node:http';
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello');
});
server.listen(3000);
```

**`stream` — Definition:** the abstraction for working with data that arrives in chunks over time (files, network responses) rather than all at once in memory. Four types: **Readable** (source of data), **Writable** (destination), **Duplex** (both), **Transform** (a Duplex stream that modifies data as it passes through, e.g. compression).

```js
import fs from 'node:fs';
fs.createReadStream('big.txt').pipe(fs.createWriteStream('copy.txt')); // piping
```

**Backpressure — Definition:** the condition where a writable stream can't consume data as fast as a readable stream produces it; `.pipe()` handles this automatically by pausing the source when the destination's internal buffer is full, resuming once it drains — manual stream code must respect the boolean return value of `.write()` to avoid unbounded memory growth.

**`buffer` — Definition:** `Buffer` is Node's class for handling raw binary data directly (outside the browser's `ArrayBuffer`/`TypedArray` world Node also supports) — used when reading files, network payloads, or binary protocols.

**`crypto` — Definition:** provides cryptographic functionality — hashing, HMAC, cipher/decipher, and random byte generation — used for password hashing, token signing, and secure random values.

```js
import crypto from 'node:crypto';
const hash = crypto.createHash('sha256').update('data').digest('hex');
const token = crypto.randomBytes(32).toString('hex');
```

**`child_process` — Definition:** spawns and manages separate OS processes from Node — `spawn` (streams stdout/stdin, good for long-running/large output), `exec` (buffers full output, good for short commands), `fork` (spawns another Node process with a built-in IPC channel).

**`cluster` — Definition:** lets a Node application spawn multiple worker processes, each with its own event loop, sharing the same server port — the standard way to use all CPU cores, since a single Node process runs JS on one thread.

**`worker_threads` — Definition:** runs JavaScript in parallel **threads** within the same process (unlike `cluster`'s separate processes), sharing memory via `SharedArrayBuffer` — used to offload CPU-heavy synchronous work (image processing, heavy computation) without blocking the main event loop.

---

## 4. Express — Fundamentals

**Definition:** Express is a minimal, unopinionated web framework built on top of Node's `http` module, providing routing, middleware, and request/response helpers so you don't build an HTTP server's plumbing from scratch.

```js
import express from 'express';
const app = express();

app.get('/', (req, res) => res.send('Hello World'));
app.listen(3000, () => console.log('Listening on 3000'));
```

**Routing — Definition:** the mechanism for mapping an HTTP method + URL path to a handler function.

```js
app.get('/users/:id', (req, res) => {
  const { id } = req.params;       // route parameter
  const { verbose } = req.query;   // query parameter (?verbose=true)
  res.json({ id, verbose });
});

app.post('/users', (req, res) => { /* create */ });
app.put('/users/:id', (req, res) => { /* replace */ });
app.patch('/users/:id', (req, res) => { /* partial update */ });
app.delete('/users/:id', (req, res) => { /* delete */ });
```

**`express.Router()` — Definition:** a mini, mountable instance of Express's routing/middleware system, used to group related routes into their own module.

```js
// routes/users.js
const router = express.Router();
router.get('/', listUsers);
router.get('/:id', getUser);
export default router;

// app.js
app.use('/api/users', usersRouter); // mounted at /api/users
```

**Request & Response objects — Definition:** `req` represents the incoming HTTP request (params, query, body, headers); `res` is the outgoing response, with methods to control status, headers, and body.

**Middleware — Definition:** a function with the signature `(req, res, next)` that executes during the request/response cycle, able to run code, modify `req`/`res`, end the cycle, or call `next()` to pass control to the next middleware/handler in the chain.

```js
function logger(req, res, next) {
  console.log(`${req.method} ${req.url}`);
  next(); // MUST call next() or the request hangs
}
app.use(logger);
```

- **Application-level** — bound via `app.use()`/`app.get()` etc., run for matching requests across the whole app.
- **Router-level** — same idea, bound to a `Router` instance instead of the app.
- **Built-in** — `express.json()` (parses JSON bodies), `express.urlencoded()` (form bodies), `express.static()` (serves files from a directory).
- **Third-party** — `cors` (Cross-Origin headers), `helmet` (security headers), `morgan` (request logging).
- **Error-handling middleware** — a middleware with **four** parameters, `(err, req, res, next)`; Express recognizes it by arity and routes errors to it (see section 5).

**Middleware execution order** — middleware runs in the exact order it's registered; a request flows through each matching middleware in sequence until one sends a response or an error occurs.

---

## 5. Express — Deep Dive

**Middleware chaining internals — Definition:** internally, Express maintains an ordered array of middleware layers per app/router; on each request it walks the array, invoking each matching layer and waiting for it to call `next()` before advancing to the next one — effectively a manually-driven, linear pipeline.

**Route matching internals — Definition:** Express compiles each route path (which may include `:params`, wildcards, or regex) into an internal matching function (historically via the `path-to-regexp` library) checked against the incoming URL in registration order — the first matching route/middleware wins.

**Modular routers & mounting** — see section 4; mounting a router with `app.use('/api/users', router)` means every path inside that router is relative to `/api/users`, and Express strips that prefix before the router sees `req.url`.

**Request lifecycle:**

```
Incoming request
  → matching application-level middleware (in registration order)
  → matching router-level middleware (if mounted path matches)
  → matching route handler
  → res.send()/res.json()/etc. ends the cycle
  → (if an error is thrown/passed to next(err) at any point) → error-handling middleware
```

**Sending responses:**

```js
res.status(201).json({ id: 1 });     // status + JSON body
res.send('plain text or HTML');       // content-type inferred
res.sendFile(path.join(__dirname, 'file.pdf'));
res.redirect('/login');
res.render('profile', { user });      // with a templating engine configured
```

**File uploads (`multer`) — Definition:** `multer` is middleware for parsing `multipart/form-data` (the encoding used by HTML file-upload forms), which `express.json()`/`express.urlencoded()` do not handle.

```js
import multer from 'multer';
const upload = multer({ dest: 'uploads/' });
app.post('/upload', upload.single('avatar'), (req, res) => {
  res.json({ path: req.file.path });
});
```

**Templating engines (EJS, Pug)** — server-side HTML templating, configured via `app.set('view engine', 'ejs')`; largely superseded in modern stacks by a separate frontend (React/Vue) consuming a JSON API, but still common for simple server-rendered admin panels/emails.

### Error handling patterns

**Centralized error handler — Definition:** a single error-handling middleware, registered last (after all routes), that catches errors from anywhere in the app and formats a consistent error response.

```js
app.use((err, req, res, next) => {
  console.error(err);
  const status = err.statusCode || 500;
  res.status(status).json({ error: err.message || 'Internal Server Error' });
});
```

**Async error handling — Definition:** Express (pre-v5) does not automatically catch a rejected Promise thrown inside an `async` route handler — an unhandled rejection there crashes silently or hangs the request, so async handlers must either be wrapped or use `try`/`catch` with `next(err)`:

```js
// manual
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await db.findUser(req.params.id);
    res.json(user);
  } catch (err) { next(err); }
});

// wrapper to avoid repeating try/catch everywhere
const asyncHandler = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
app.get('/users/:id', asyncHandler(async (req, res) => {
  res.json(await db.findUser(req.params.id));
}));
```

(Express 5 fixes this at the framework level — async handler rejections are forwarded to `next()` automatically.)

**Custom error classes:**

```js
class ApiError extends Error {
  constructor(statusCode, message) { super(message); this.statusCode = statusCode; }
}
throw new ApiError(404, 'User not found');
```

---

## 6. Building REST APIs

**Definition:** REST (Representational State Transfer) is an architectural style for web APIs where **resources** (nouns, e.g. `/users`) are manipulated through standard HTTP methods (verbs: GET/POST/PUT/PATCH/DELETE), with each request being stateless (containing everything the server needs to process it).

**Resource-based routing:**

```
GET    /users        → list users
POST   /users        → create a user
GET    /users/:id    → get one user
PATCH  /users/:id    → partially update a user
DELETE /users/:id    → delete a user
```

**Status codes — Definition:** three-digit HTTP response codes signaling the outcome of a request: `2xx` success (`200` OK, `201` Created, `204` No Content), `4xx` client error (`400` Bad Request, `401` Unauthorized, `403` Forbidden, `404` Not Found, `409` Conflict, `422` Unprocessable Entity), `5xx` server error (`500` Internal Server Error).

**Request validation — Definition:** checking that an incoming request's body/params/query match an expected shape/type *before* business logic runs, rejecting malformed input early with a `400`.

```js
import { z } from 'zod';
const createUserSchema = z.object({ email: z.string().email(), age: z.number().int().positive() });

app.post('/users', (req, res, next) => {
  const result = createUserSchema.safeParse(req.body);
  if (!result.success) return res.status(400).json({ errors: result.error.issues });
  // result.data is now validated & typed
});
```

**Pagination, filtering, sorting:**

```
GET /users?page=2&limit=20&sort=-createdAt&status=active
```

**API versioning — Definition:** exposing multiple, non-breaking-compatible versions of an API simultaneously (`/api/v1/users` vs `/api/v2/users`, or a version header) so existing clients don't break when the API evolves.

**OpenAPI / Swagger — Definition:** a machine-readable specification format for describing a REST API's endpoints, parameters, and responses, from which interactive documentation and client SDKs can be generated.

---

## 7. Databases & ORMs

**Definition:** most Express APIs persist data in a database, accessed either through a low-level driver (raw queries) or an **ORM** (Object-Relational Mapper) / query builder that maps database rows to JS objects and generates SQL for you.

**SQL vs NoSQL** — SQL databases (PostgreSQL, MySQL) enforce a fixed schema and relational integrity via foreign keys/joins; NoSQL databases (MongoDB) store flexible, often denormalized documents — the choice depends on how structured and relational the data naturally is.

```js
// raw driver — pg
import { Pool } from 'pg';
const pool = new Pool();
const { rows } = await pool.query('SELECT * FROM users WHERE id = $1', [id]); // parameterized — prevents SQL injection

// MongoDB native driver
const users = await db.collection('users').find({ status: 'active' }).toArray();
```

**Prisma — Definition:** a modern, type-safe ORM that generates a fully-typed database client from a declarative schema file, giving compile-time safety for queries.

```prisma
// schema.prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
}
```

```ts
const user = await prisma.user.create({ data: { email: 'a@b.com' } }); // fully typed
```

**TypeORM / Sequelize / Knex** — TypeORM and Sequelize are class/model-based ORMs (Active Record or Data Mapper style); Knex is a lower-level SQL query builder without a full ORM's model layer — a middle ground between raw SQL and a full ORM.

**Connection pooling — Definition:** reusing a fixed set of open database connections across requests instead of opening/closing a new connection per query — opening a DB connection is relatively expensive, so a pool amortizes that cost.

**Migrations — Definition:** version-controlled, incremental scripts that evolve a database schema over time (add a column, create a table), so schema changes can be applied consistently across environments and rolled back if needed.

**Transactions — Definition:** a group of database operations executed as a single atomic unit — either all succeed and are committed, or any failure rolls back every operation in the group, preserving data consistency.

```ts
await prisma.$transaction(async (tx) => {
  await tx.account.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } });
  await tx.account.update({ where: { id: 2 }, data: { balance: { increment: 100 } } });
});
```

---

## 8. Authentication & Authorization

**Definition:** **Authentication** verifies who a user is (login); **authorization** determines what an authenticated user is allowed to do (permissions).

**Session-based auth — Definition:** after login, the server creates a session record (often in a store like Redis) and sends the client a session ID in a cookie; subsequent requests include that cookie, and the server looks up the session to identify the user — state lives on the server.

**Token-based auth (JWT) — Definition:** after login, the server issues a signed **JSON Web Token** containing the user's claims; the client sends it (typically in an `Authorization: Bearer <token>` header) on every request, and the server verifies its signature without needing to store session state — state lives in the token itself.

```js
import jwt from 'jsonwebtoken';
const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET, { expiresIn: '15m' });

// middleware
function requireAuth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch { res.status(401).json({ error: 'Invalid token' }); }
}
```

**Password hashing — Definition:** transforming a plaintext password into a one-way, salted hash before storing it, so the original password can never be recovered even if the database leaks; `bcrypt`/`argon2` are purpose-built, deliberately slow hashing algorithms (resisting brute-force) — never use a fast general-purpose hash like SHA-256 for passwords.

```js
import bcrypt from 'bcrypt';
const hash = await bcrypt.hash(password, 10);       // 10 = cost factor
const isValid = await bcrypt.compare(password, hash);
```

**OAuth2 / social login — Definition:** a delegated-authorization protocol letting a user grant an application limited access to their account on another service (Google, GitHub) without sharing their password with that application; `passport.js` is the common middleware-based abstraction for implementing OAuth2 (and other) strategies in Express.

**Refresh tokens — Definition:** a long-lived token, stored securely, used to obtain a new short-lived access token without forcing the user to log in again — limits the exposure window of a stolen access token while avoiding constant re-authentication.

**RBAC (Role-Based Access Control) — Definition:** an authorization model where permissions are attached to **roles** (admin, editor, viewer), and users are granted access by being assigned one or more roles, rather than permissions being checked per-user individually.

```js
function requireRole(role) {
  return (req, res, next) => req.user.role === role ? next() : res.status(403).end();
}
app.delete('/users/:id', requireAuth, requireRole('admin'), deleteUser);
```

---

## 9. Security

**Definition:** API security is the set of practices that prevent a Node/Express backend from being exploited via malicious input, forged requests, or leaked configuration.

**Injection (SQL/NoSQL) — Definition:** an attack where untrusted input is concatenated directly into a query string and executed as code rather than data — prevented by always using **parameterized queries** (`$1`, `?` placeholders) or an ORM, never string-concatenating user input into a query.

```js
// ❌ vulnerable
db.query(`SELECT * FROM users WHERE email = '${email}'`);
// ✅ parameterized
db.query('SELECT * FROM users WHERE email = $1', [email]);
```

**XSS / CSRF** — same concepts covered in the Angular/React notes; an Express API's role is mainly to *not reflect unsanitized input back as HTML* and to support the frontend's CSRF defenses (e.g. validating a CSRF token header on state-changing requests when using cookie-based auth).

**`helmet` — Definition:** middleware that sets a collection of security-related HTTP response headers (CSP, `X-Content-Type-Options`, `X-Frame-Options`, etc.) with sane defaults in one call.

```js
import helmet from 'helmet';
app.use(helmet());
```

**Rate limiting — Definition:** capping how many requests a client (by IP, user, or API key) can make in a given time window, to prevent brute-force attacks and abuse.

```js
import rateLimit from 'express-rate-limit';
app.use(rateLimit({ windowMs: 15 * 60 * 1000, max: 100 })); // 100 requests / 15 min
```

**Input sanitization/validation** — see section 6 (Zod/Joi); validate shape *and* strip/escape dangerous characters where the input will later be rendered or interpreted (HTML, shell commands, file paths).

**CORS configuration — Definition:** Cross-Origin Resource Sharing headers that tell browsers which origins are allowed to make requests to this API from client-side JS; configured via the `cors` middleware, restricted to known frontend origins in production rather than left wide open.

```js
import cors from 'cors';
app.use(cors({ origin: 'https://app.example.com', credentials: true }));
```

**Secrets management** — API keys, DB credentials, and signing secrets belong in environment variables (or a dedicated secrets manager), never committed to source control; `.env` files should be git-ignored.

---

## 10. Async Patterns & Error Handling

**Promise chaining vs `async`/`await` — Definition:** a Promise represents a value that will be available (or fail) in the future; `.then()` chaining composes async operations sequentially, while `async`/`await` is syntax sugar over Promises that lets asynchronous code read like synchronous code.

```js
// chaining
fetchUser(id).then(user => fetchOrders(user.id)).then(orders => console.log(orders)).catch(console.error);

// async/await — equivalent, more readable
try {
  const user = await fetchUser(id);
  const orders = await fetchOrders(user.id);
} catch (err) { console.error(err); }
```

**`Promise` combinators — Definition:**

```js
await Promise.all([p1, p2]);        // rejects fast if ANY promise rejects; resolves with all values
await Promise.allSettled([p1, p2]); // never rejects; resolves with {status, value|reason} for each
await Promise.race([p1, p2]);       // settles as soon as the FIRST promise settles (resolve or reject)
await Promise.any([p1, p2]);        // resolves as soon as the FIRST promise resolves; rejects only if ALL reject
```

**Unhandled promise rejections — Definition:** a rejected Promise with no `.catch()`/`try-catch` attached; by default Node logs a warning and, in recent versions, **terminates the process** — always handle rejections, and register a global fallback:

```js
process.on('unhandledRejection', (reason) => { console.error(reason); /* log & exit gracefully */ });
process.on('uncaughtException', (err) => { console.error(err); process.exit(1); }); // last resort only
```

**Retry & backoff — Definition:** automatically re-attempting a failed operation (typically a network call), with increasing delay between attempts (**exponential backoff**), to ride out transient failures without hammering a struggling downstream service.

```js
async function withRetry(fn, retries = 3, delay = 500) {
  for (let i = 0; i < retries; i++) {
    try { return await fn(); }
    catch (err) { if (i === retries - 1) throw err; await new Promise(r => setTimeout(r, delay * 2 ** i)); }
  }
}
```

**Circuit breakers — Definition:** a pattern that stops calling a downstream service once it's failing repeatedly (the circuit "opens"), returning errors immediately for a cooldown period instead of piling up slow, doomed requests — prevents cascading failure across services.

---

## 11. Streams & Performance

**When to use streams** — any time data is too large to comfortably hold in memory at once, or you want to start processing before all the data has arrived: large file uploads/downloads, video, large CSV exports, proxying an HTTP response.

**Compression (`zlib`) — Definition:** the built-in module for gzip/deflate/brotli compression, commonly applied to HTTP responses via the `compression` middleware to reduce payload size over the network.

```js
import compression from 'compression';
app.use(compression());
```

**Caching strategies — Definition:** storing the result of an expensive operation (a DB query, a computed response) so subsequent identical requests can be served faster. **In-memory** caching (a plain `Map`, or `node-cache`) is simplest but doesn't share state across multiple server instances; **Redis** is an external, shared cache usable across all instances of a horizontally-scaled app.

```js
import { createClient } from 'redis';
const redis = createClient();
async function getUser(id) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);
  const user = await db.findUser(id);
  await redis.set(`user:${id}`, JSON.stringify(user), { EX: 300 }); // 5 min TTL
  return user;
}
```

**Clustering for multi-core usage** — see section 3 (`cluster`); a single Node process only uses one CPU core for JS execution, so a multi-core machine needs multiple worker processes (via `cluster` or a process manager like PM2 `-i max`) to use its full capacity.

**Worker threads for CPU-bound work** — see section 3; offloads synchronous, CPU-heavy code (image resizing, complex calculations) to a separate thread so it doesn't block the event loop from serving other requests.

**Load testing — Definition:** simulating many concurrent users/requests against an API to measure throughput, latency, and breaking points before real users find them — tools: `autocannon` (simple, Node-native), `k6` (scriptable, more advanced scenarios).

**Profiling — Definition:** measuring where a running application spends its time/memory to find bottlenecks, rather than guessing. `node --prof` produces a V8 CPU profile; `clinic.js` provides higher-level, visual profiling (`clinic doctor`, `clinic flame`) for diagnosing event-loop blocking, memory leaks, and I/O bottlenecks.

**Memory leaks & GC basics — Definition:** a memory leak occurs when objects are unintentionally kept reachable (e.g. an ever-growing array, listeners never removed, closures retaining large objects) so the garbage collector can never reclaim them, causing memory usage to climb until the process crashes.

---

## 12. Node.js Internals

This is where we go beyond normal tutorials.

**Definition:** Node internals covers how a layer of C++ bindings and the libuv library connect the V8 JavaScript engine to real operating-system I/O — the machinery underneath every `fs.readFile` or `http.createServer` call.

Understand:

```
JS Code
  ↓
V8 (parses, compiles, executes JS)
  ↓
Node.js APIs (fs, http, etc. — C++ bindings)
  ↓
libuv (event loop, thread pool, async I/O)
  ↓
OS kernel (actual I/O)
```

**V8 engine — Definition:** the open-source JavaScript (and WebAssembly) engine, developed by Google, that Node embeds to parse, compile (via a JIT compiler — first an interpreter, `Ignition`, then optimizing compiler `TurboFan` for hot code paths), and execute JavaScript. **Hidden classes** are an internal V8 optimization that gives objects with the same shape (same properties, added in the same order) a shared internal structure, making property access much faster than a naive dictionary lookup — a reason "shape-consistent" objects are faster in practice.

**libuv architecture — Definition:** the cross-platform C library providing Node's event loop, asynchronous file/network I/O, and the thread pool — it abstracts OS-specific async I/O mechanisms (epoll on Linux, kqueue on macOS, IOCP on Windows) behind one consistent API.

**The thread pool — Definition:** libuv's default pool of 4 threads (configurable via `UV_THREADPOOL_SIZE`) used for operations without a native async OS interface — most `fs` operations, DNS lookups (`dns.lookup`), and some `crypto` functions — so these don't block the main thread even though the underlying OS call is synchronous.

**Microtask vs macrotask queue ordering** — see section 2; the precise ordering (`nextTick` > microtasks > phase-specific macrotasks) is a very common interview question and source of subtle bugs when mixing `setTimeout`, Promises, and `process.nextTick`.

**Garbage collection — Definition:** V8's automatic memory management, primarily **generational**: new objects are allocated in a small "young generation" heap, collected frequently and cheaply (most objects die young — the "generational hypothesis"); objects that survive multiple collections are promoted to the "old generation," collected less often using a **mark-sweep(-compact)** algorithm that finds unreachable objects and reclaims their memory.

**Node's module resolution algorithm — Definition:** the deterministic set of rules Node follows to resolve a `require()`/`import` specifier to an actual file — checking core modules first, then relative/absolute paths, then walking up `node_modules` directories from the requiring file's location to the filesystem root.

**ESM vs CJS interop** — CommonJS modules can generally be `import`ed from ESM (as a default export), but ESM modules cannot be `require()`d synchronously from CJS (since ESM loading is inherently asynchronous) — a common source of dependency compatibility friction when migrating a project to `"type": "module"`.

---

## 13. Testing

**Definition:** testing a Node/Express backend covers **unit** tests (an isolated function/module), **integration** tests (a route handler exercised against a real or in-memory dependency), and **E2E** tests (the full API, over HTTP, against a real or test database).

**Test runners — Definition:** Jest and Vitest are full-featured test frameworks (`describe`/`it`/`expect`, mocking, coverage); Node also ships a built-in test runner (`node:test`) requiring no dependency for basic unit tests.

**Testing Express routes (`supertest`) — Definition:** a library that wraps an Express `app` and issues real HTTP requests against it in-process (no need to actually bind a port), asserting on status codes and response bodies.

```js
import request from 'supertest';
import app from '../app.js';

test('GET /users/:id returns a user', async () => {
  const res = await request(app).get('/users/1');
  expect(res.status).toBe(200);
  expect(res.body).toMatchObject({ id: 1 });
});
```

**Mocking — Definition:** substituting a fake implementation for a real dependency (a module, the database) in a test, so the test is isolated from that dependency's real, potentially slow or non-deterministic behavior.

```js
jest.mock('../services/emailService', () => ({ sendWelcomeEmail: jest.fn() }));
```

**Test databases / test containers — Definition:** running tests against a real database instance dedicated to testing — either a separate always-on test DB, or one spun up on demand in a Docker container (via `testcontainers`) — giving higher confidence than mocking the database entirely, at the cost of slower tests.

**Integration & E2E testing** — integration tests exercise a route handler with its real service/DB layer (but perhaps mocked external APIs); E2E tests hit the fully deployed (or locally running) API over real HTTP, verifying the whole system end-to-end.

---

## 14. Architecture

**Definition:** backend architecture, in this context, is how a Node/Express codebase's folders and layers are organized so business logic stays testable and decoupled from the HTTP framework itself.

```
src/
 ├── config/          # env, db connection, constants
 ├── routes/          # route definitions
 ├── controllers/      # request handlers
 ├── services/         # business logic
 ├── models/           # ORM models / schemas
 ├── middlewares/
 ├── validators/
 ├── utils/
 └── app.ts / server.ts
```

**Layered architecture — Definition:** organizing code into distinct layers with a one-directional dependency flow — **routes** (URL → controller mapping) → **controllers** (parse the HTTP request, call a service, shape the HTTP response) → **services** (business logic, framework-agnostic) → **models** (data access) — so business logic doesn't depend on Express-specific `req`/`res` objects and can be tested/reused independently.

```js
// controller — thin, HTTP-aware
export async function createUser(req, res, next) {
  try {
    const user = await userService.create(req.body);
    res.status(201).json(user);
  } catch (err) { next(err); }
}

// service — business logic, no req/res
export async function create(data) {
  const existing = await userRepository.findByEmail(data.email);
  if (existing) throw new ApiError(409, 'Email already in use');
  return userRepository.insert({ ...data, password: await hashPassword(data.password) });
}
```

**MVC in Express** — Express doesn't enforce MVC, but the controller/service/model split above maps loosely onto it (routes+controllers ≈ the "C", models ≈ the "M"; there's no "V" in a JSON API).

**Dependency injection in Node — Definition:** supplying a module's dependencies (a DB client, a config object) from the outside rather than importing/instantiating them directly inside the module — in Node this is often done manually (passing dependencies as constructor/function arguments) rather than via a heavyweight DI container, since ESM/CJS module caching already gives natural singletons for free in many cases.

**Microservices vs monolith — Definition:** a **monolith** is a single deployable application containing all business logic; a **microservices** architecture splits the system into multiple independently deployable services communicating over the network (HTTP, message queues) — trades operational complexity for independent scaling/deployment of each piece, generally only worth it past a certain team/system size.

**Message queues (RabbitMQ, Kafka) — Definition:** infrastructure for asynchronous, decoupled communication between services — a producer publishes a message without waiting for a consumer to process it, enabling load leveling, retries, and multiple independent consumers of the same event.

**Event-driven architecture — Definition:** a design where services communicate primarily by emitting and reacting to events (e.g. "OrderPlaced") rather than direct synchronous calls, reducing tight coupling between producers and consumers of that information.

---

## 15. TypeScript with Node/Express

**Definition:** using TypeScript with Express means typing route handlers, middleware, and configuration so mistakes (wrong body shape, missing env var) are caught at compile time.

```ts
import { Request, Response, NextFunction } from 'express';

interface CreateUserBody { email: string; password: string; }

app.post('/users', (req: Request<{}, {}, CreateUserBody>, res: Response) => {
  const { email, password } = req.body; // typed
});

function requireAuth(req: Request, res: Response, next: NextFunction) {
  // ...
  next();
}
```

**Typing environment variables:**

```ts
// env.ts — validate once at startup, fail fast if misconfigured
import { z } from 'zod';
const envSchema = z.object({ PORT: z.coerce.number(), DATABASE_URL: z.string().url() });
export const env = envSchema.parse(process.env);
```

**Typed request validation** — combine section 6's Zod validation with TypeScript so `z.infer<typeof schema>` becomes the request body's type, keeping the runtime check and the compile-time type in sync from one source of truth.

**Dev tooling** — `ts-node`/`tsx` run TypeScript directly in development without a separate build step; production deployments typically run `tsc` (or esbuild/swc) ahead of time and execute the compiled plain-JS output for speed and to avoid shipping the TS toolchain to production.

---

## 16. Production Engineering

**Definition:** production engineering covers the practices that keep a Node/Express service reliable, observable, and safely deployable once real traffic depends on it.

**Environment configuration (`dotenv`)** — loads variables from a local `.env` file into `process.env` during development; production environments set real environment variables directly (via the hosting platform), not a checked-in `.env` file.

**Process managers (PM2) — Definition:** a production process manager for Node that keeps an app running (auto-restart on crash), manages logs, and can run an app in `cluster` mode across multiple CPU cores with one command.

```bash
pm2 start server.js -i max   # cluster mode across all CPU cores
pm2 logs
pm2 restart server
```

**Logging (Winston, Pino) — Definition:** structured logging libraries that output leveled (`info`/`warn`/`error`), timestamped, often JSON-formatted log lines — far more useful in production than `console.log`, since structured logs can be filtered/queried by a log aggregation tool.

```js
import pino from 'pino';
const logger = pino();
logger.info({ userId: 1 }, 'user logged in');
```

**Health checks — Definition:** a lightweight endpoint (`GET /health`) that reports whether the service (and its critical dependencies, like the database) is up, used by load balancers/orchestrators to decide whether to route traffic to this instance.

**Graceful shutdown — Definition:** on receiving a termination signal (`SIGTERM`, sent by most deploy/orchestration tools before killing a process), the app stops accepting new connections, finishes in-flight requests, closes DB connections cleanly, then exits — instead of dying mid-request.

```js
process.on('SIGTERM', async () => {
  server.close(() => console.log('HTTP server closed'));
  await db.disconnect();
  process.exit(0);
});
```

**Docker for Node apps** — a multi-stage build installs dependencies and (if TS) compiles in a build stage, then copies only the production output + `node_modules` into a slim final image, minimizing image size and attack surface.

**CI/CD, reverse proxies, horizontal scaling** — same concepts as the Angular/React production notes: an automated pipeline builds/tests/deploys the app; Nginx (or a cloud load balancer) typically sits in front of Node to handle TLS termination, static file serving, and distributing requests across multiple Node instances/cores for horizontal scaling.

**Monitoring & APM (Datadog, New Relic) — Definition:** Application Performance Monitoring tools that automatically instrument a Node app to track request latency, throughput, error rates, and slow database queries in production, beyond what basic logging captures.

**Error tracking (Sentry)** — captures unhandled exceptions/rejections with full stack traces and request context, alerting the team and aggregating recurring issues, rather than errors only existing as log lines someone has to go looking for.

**Zero-downtime deploys — Definition:** deploying a new version without any window where the service is unreachable — typically via rolling deploys (start new instances, wait until healthy, then terminate old ones) rather than stopping the old version before starting the new one.

---

## 17. Real-Time & Advanced Topics

**WebSockets — Definition:** a protocol providing a persistent, full-duplex connection between client and server over a single TCP connection, allowing either side to push messages at any time — unlike HTTP's request/response model.

**Socket.IO — Definition:** a library built on top of WebSockets (with automatic fallback to HTTP long-polling) that adds higher-level features — rooms, broadcasting, automatic reconnection — on top of the raw WebSocket protocol.

```js
import { Server } from 'socket.io';
const io = new Server(httpServer);
io.on('connection', (socket) => {
  socket.join('room1');
  socket.on('message', (msg) => io.to('room1').emit('message', msg));
});
```

**Server-Sent Events (SSE) — Definition:** a simpler, one-way alternative to WebSockets where the server streams events to the client over a single long-lived HTTP response — sufficient when only server-to-client push is needed (live notifications, progress updates), with the browser's native `EventSource` API on the client side.

**GraphQL with Node (Apollo Server) — Definition:** an alternative to REST where the client sends a single query describing exactly the data shape it needs, resolved by **resolver** functions on the server — reduces over/under-fetching compared to fixed REST endpoints, at the cost of more complex server-side setup (schema, resolvers, query complexity limits).

**Cron jobs (`node-cron`) — Definition:** scheduling a function to run on a recurring time-based schedule (cron syntax) within a running Node process — useful for periodic cleanup/report tasks, though a dedicated scheduler is preferable for anything business-critical (a single Node instance restarting can silently skip a scheduled run).

```js
import cron from 'node-cron';
cron.schedule('0 * * * *', () => cleanupExpiredSessions()); // every hour
```

**Background job queues (BullMQ) — Definition:** a Redis-backed job queue letting slow or unreliable work (sending emails, processing uploads, generating reports) be pushed onto a queue and processed asynchronously by separate worker processes — with built-in retries, delays, and concurrency control — instead of doing that work inline within an HTTP request/response cycle.

```js
import { Queue, Worker } from 'bullmq';
const emailQueue = new Queue('emails');
await emailQueue.add('welcome', { userId: 1 });

new Worker('emails', async (job) => { await sendEmail(job.data.userId); });
```

**API gateways — Definition:** a single entry point that sits in front of multiple backend services (or route groups), handling cross-cutting concerns — auth, rate limiting, routing, request/response transformation — centrally, rather than duplicating that logic in every service.
