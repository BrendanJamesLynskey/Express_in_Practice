# Express in Practice

Full slide content in Markdown format. Each slide is separated by `---`.

Companion to [Introduction to Express.js](https://brendanjameslynskey.github.io/Introduction_to_Express_js/) — production patterns past "Hello World".

---

## Slide 01 — Title

# Express in Practice

**Production patterns for the Node.js web framework**

Express 5 · async/await · OpenAPI · helmet · CSP · CSRF

> The companion deck to *Introduction to Express.js* — everything past "Hello World": project structure at scale, async error discipline, OpenAPI-first design, security hardening, observability, performance and graceful shutdown.

Route · Validate · Authorise · Handle · Stream · Observe

---

## Slide 02 — Topics

### Foundations & structure
- What changed in Express 5 — and what didn't
- Async error handling — the one rule that prevents most outages
- Project structure at scale — layered, feature-based, contract-first
- Validation at the boundary — Zod, Joi, TypeBox

### Behaviour
- Authentication patterns — sessions, JWT, refresh tokens
- Authorisation patterns — route guards, RBAC, policy middleware
- Streaming — SSE, range requests, gzip, downloads
- File uploads — multer, busboy, direct-to-S3
- Rate limiting & abuse protection
- Caching — ETag, Cache-Control, Redis cache-aside

### Integration
- Database integration patterns — pool, request-scoped txn
- Background jobs & decoupling — BullMQ
- OpenAPI & contract-first design
- Testing — supertest, integration, contract tests

### Production
- Observability — pino, OpenTelemetry, request IDs
- Security hardening — helmet, CSP, CSRF, CORS
- Performance — clustering, PM2, keep-alive, compression
- Graceful shutdown — SIGTERM, drain, readiness
- Reverse proxy & TLS termination
- Express vs Fastify, Koa, Hono, NestJS
- Express 4 → 5 migration playbook

---

## Slide 03 — Express 5 — What Changed

Express 5 (stable in 2024) is the first major release in nine years. It modernises async handling and tightens routing — **small surface, real consequences**.

### The big wins
- **Native promise support** — rejected promises returned from handlers reach the error pipeline; no more `express-async-errors`
- **path-to-regexp v8** — `:param` still works; ReDoS-prone wildcards rejected by default
- **Body parsers built in** — `express.json()`, `express.urlencoded()`, `express.text()` — no separate `body-parser`
- Drop of legacy Node engines (Node ≥ 18)
- Stricter `req.query` parsing — opt-in extended syntax

### The async upgrade

```javascript
// Express 4 — needed try/catch or express-async-errors
app.get('/u/:id', async (req, res, next) => {
  try {
    res.json(await getUser(req.params.id));
  } catch (err) { next(err); }
});

// Express 5 — promise rejections forwarded automatically
app.get('/u/:id', async (req, res) => {
  res.json(await getUser(req.params.id));
});
```

### Breaking changes that bite
- `app.del()` removed — use `app.delete()`
- Regex paths like `'/foo*'` now require explicit syntax (`'/foo{*splat}'` or a `RegExp` literal)
- `req.param()` removed — use `req.params` / `req.query` / `req.body`
- Trust proxy default still `false` — opt in behind a load balancer
- `res.send(status)` overload removed — use `res.status(n).send()`
- `res.redirect(url, status)` → `res.redirect(status, url)`

---

## Slide 04 — Async Error Handling, Properly

The single biggest source of "the server fell over" is unhandled rejections that escape the request lifecycle. The fix is a **disciplined error pipeline** — one place where every error becomes an HTTP response.

### One central handler

```javascript
class HttpError extends Error {
  constructor(status, code, message, details) {
    super(message);
    this.status  = status;
    this.code    = code;
    this.details = details;
  }
}
const NotFound = (m='Not found') => new HttpError(404, 'not_found', m);
const Conflict = (m, d) => new HttpError(409, 'conflict', m, d);

// errorMiddleware — mounted last
module.exports = (err, req, res, _next) => {
  const status = err.status ?? 500;
  if (status >= 500) req.log.error({ err }, 'unhandled');
  res.status(status).json({
    error: { code: err.code ?? 'internal',
             message: status >= 500 ? 'Internal error' : err.message,
             details: err.details, request_id: req.id }
  });
};
```

### Throwing as control flow

```javascript
app.get('/users/:id', async (req, res) => {
  const user = await users.byId(req.params.id);
  if (!user) throw NotFound('user not found');
  res.json(user);
});

// translate library errors at one place — before the json handler
app.use((err, req, res, next) => {
  if (err instanceof ZodError)
    return next(new HttpError(400, 'validation_failed',
      'Invalid request', err.flatten()));
  if (err.code === '23505')
    return next(new HttpError(409, 'duplicate', 'Already exists'));
  next(err);
});
```

### Process-level safety net

```javascript
process.on('unhandledRejection', (err) => {
  log.fatal({ err }, 'unhandledRejection');
  shutdown(1);
});
process.on('uncaughtException', (err) => {
  log.fatal({ err }, 'uncaughtException');
  shutdown(1);
});
```

### Rules
- One `HttpError` shape, one error middleware
- Throw, don't return error envelopes — let the pipeline do it
- 5xx logs an error; 4xx does not (it's user error)
- Process-level handlers *exit*, they don't recover

---

## Slide 05 — Project Structure at Scale

The intro deck shows `app.js + routes/ + controllers/`. That works to ~5 kLOC. Past that you'll want one of two shapes: **layered** (technology cuts) or **feature-based** (domain cuts).

### Layered (small/medium apps)

```text
src/
├── app.js              # composition root
├── server.js           # boots app + db + queues
├── routes/             # express routers
│   ├── users.js
│   └── orders.js
├── controllers/        # http <-> service glue
├── services/           # domain logic
├── repositories/       # db access (knex/prisma)
├── schemas/            # Zod request/response
├── middleware/         # auth, rate limit, errors
├── lib/                # cross-cutting helpers
└── config/
```

### Feature-based (medium/large)

```text
src/
├── app.js
├── server.js
├── core/               # cross-cutting: logger, db, queue
├── modules/
│   ├── users/
│   │   ├── users.routes.js
│   │   ├── users.controller.js
│   │   ├── users.service.js
│   │   ├── users.repo.js
│   │   ├── users.schemas.js
│   │   └── users.test.js
│   ├── orders/
│   └── billing/
└── shared/
```

### The composition root

```javascript
function createApp({ db, queue, logger }) {
  const app = express();
  app.set('trust proxy', 1);
  app.use(requestId());
  app.use(httpLogger(logger));
  app.use(express.json({ limit: '128kb' }));
  app.use(rateLimit());

  app.use('/v1/users',  usersRouter({ db, queue }));
  app.use('/v1/orders', ordersRouter({ db, queue }));

  app.get('/healthz',  (_q,r) => r.json({ ok: true }));
  app.get('/readyz',   ready({ db, queue }));

  app.use(notFound);
  app.use(errorHandler);
  return app;
}
module.exports = { createApp };
```

### Why this shape
- No `require('./db')` in service code — everything injected
- Tests build their own `app` with mocks — no global teardown
- Routers mount under `/v1` from day one — versioning is free later
- Middleware order in *one* file — reading the request flow takes 10 seconds

---

## Slide 06 — Validation at the Boundary

Validate **once, at the edge**, with a single tool. The output of validation is your *only* input to handlers — raw `req.body` never touches the service layer.

### Zod — small, TS-first

```typescript
import { z } from 'zod';

const UserCreate = z.object({
  email: z.string().email(),
  name:  z.string().min(1).max(120),
  role:  z.enum(['user','admin']).default('user'),
});
type UserCreate = z.infer<typeof UserCreate>;

const validate = (schema, source = 'body') =>
  (req, _res, next) => {
    const r = schema.safeParse(req[source]);
    if (!r.success) return next(
      new HttpError(400, 'validation_failed', 'Invalid request', r.error.flatten()));
    req[source] = r.data;
    next();
  };

usersRouter.post('/', validate(UserCreate), createUser);
```

### Validate every input source
- `body` — JSON or form payload
- `params` — path parameters — *always* validate UUIDs / IDs
- `query` — coerce strings to numbers / booleans / dates
- `headers` — for API keys, content-type, idempotency keys

### Coercion + defaults

```typescript
const UserList = z.object({
  page: z.coerce.number().int().min(1).default(1),
  size: z.coerce.number().int().min(1).max(100).default(20),
  q:    z.string().trim().optional(),
});

usersRouter.get('/', validate(UserList, 'query'), async (req, res) => {
  res.json(await users.list(req.query));
});
```

### Tools compared

| Tool | Strength | Pick when |
|------|----------|-----------|
| Zod | TS inference | TypeScript codebase |
| Joi | Mature, expressive | JS / older codebase |
| TypeBox | JSON Schema native | Want OpenAPI from schema |
| express-validator | Built for Express | Per-field chained API |

> Don't trust `req.body` directly, don't validate inside the controller, don't have two validation libraries in one repo.

---

## Slide 07 — Authentication Patterns

### Server-side sessions
- Cookie holds an opaque session ID
- Session state in Redis (never in-memory)
- Cookie flags: `HttpOnly · Secure · SameSite=Lax`
- Rotate on privilege change; absolute + idle TTLs
- Best for browser apps you control

```javascript
app.use(session({
  store: new RedisStore({ client: redis }),
  secret: process.env.SESSION_SECRET,
  cookie: { httpOnly: true, secure: true,
            sameSite: 'lax', maxAge: 30*60*1000 },
  rolling: true, resave: false, saveUninitialized: false,
}));
```

### JWT access + refresh
- Short-lived **access JWT** (5–15 min) signed by your IdP
- Long-lived **refresh token** stored as a *HttpOnly cookie*
- Validate `iss / aud / exp / nbf` — never just decode
- Refresh-token rotation with reuse detection
- Best for APIs serving multiple clients

```javascript
const { payload } = await jwtVerify(token, jwks, {
  issuer: ISS, audience: AUD,
  algorithms: ['RS256','ES256']
});
req.user = { sub: payload.sub, scope: payload.scope };
```

### API keys / mTLS
- Keys: hash at rest (Argon2id), never log, scope per key
- Bind to caller IP / origin where realistic
- mTLS for service-to-service — verify SAN against an allow-list
- Use a header like `X-Api-Key`; rotate on a schedule
- Best for B2B and east-west calls

```javascript
async function apiKey(req, _res, next) {
  const k = req.get('x-api-key');
  if (!k) throw Unauthorized();
  const row = await keys.byHash(sha256(k));
  if (!row || row.revoked) throw Unauthorized();
  req.caller = { keyId: row.id, scopes: row.scopes };
  next();
}
```

> Cross-cutting: always rate-limit the login / refresh endpoints separately; log auth failures with *userless* metadata; never leak whether the email exists in the error message.

---

## Slide 08 — Authorisation Patterns

Authentication answers *who*; authorisation answers *may they*. Encode it as **middleware that throws** — the same pipeline that handles validation handles AuthZ failures.

### Route guards by scope

```javascript
const requireScope = (...scopes) => (req, _res, next) => {
  if (!req.user) throw Unauthorized();
  const have = new Set((req.user.scope ?? '').split(' '));
  if (!scopes.some(s => have.has(s))) throw Forbidden();
  next();
};

usersRouter.get ('/', requireScope('users:read'), list);
usersRouter.post('/', requireScope('users:write'), create);

const requireSelfOrAdmin = (req, _res, next) => {
  if (req.user.role === 'admin') return next();
  if (req.user.sub === req.params.id) return next();
  throw Forbidden();
};
usersRouter.get('/:id', requireSelfOrAdmin, byId);
```

### Resource-level checks

```javascript
async function ownPost(req, _res, next) {
  const post = await posts.byId(req.params.id);
  if (!post) throw NotFound();
  if (post.author_id !== req.user.sub && req.user.role !== 'admin')
    throw Forbidden();
  req.post = post;
  next();
}

postsRouter.put('/:id', ownPost, update);
postsRouter.delete('/:id', ownPost, destroy);
```

### Policy middleware (OPA / Cedar)

```javascript
async function authorise(req, _res, next) {
  const decision = await policy.evaluate({
    principal: req.user,
    action:    `${req.method} ${req.route.path}`,
    resource:  req.post ?? req.params,
    context:   { ip: req.ip, time: Date.now() },
  });
  if (!decision.allow) throw Forbidden(decision.reasons);
  next();
}
```

### Patterns to favour
- **Deny by default** — mount AuthZ *before* the handler, not as an "if" inside
- Return **404, not 403** when even the existence of a resource is sensitive (multi-tenant)
- Cache decisions per request, not per process
- Audit-log every decision > `info`; include `request_id` + principal + action
- Test the deny path — the false-positive is the bug that costs money

### Footguns
- Trusting `req.user.role` from the body / query
- String-comparing scopes case-insensitively
- Mixing AuthN and AuthZ in one middleware (hard to test)

---

## Slide 09 — Streaming Responses

Most APIs return JSON in one `res.json()`. Some don't — **downloads**, **SSE**, **progress**, **large reports** — benefit from streaming. Get it wrong and you OOM the process.

### File downloads — pipe, don't buffer

```javascript
// BAD — reads whole file into memory
app.get('/r/:id', async (req, res) => {
  const buf = await fs.readFile(path);
  res.type('application/pdf').send(buf);
});

// GOOD — pipeline with cleanup on abort
const { pipeline } = require('node:stream/promises');
app.get('/r/:id', async (req, res) => {
  res.type('application/pdf');
  res.setHeader('Content-Disposition',
                `attachment; filename="r-${req.params.id}.pdf"`);
  await pipeline(fs.createReadStream(path), res);
});
```

### SSE — progress / live updates

```javascript
app.get('/jobs/:id/stream', async (req, res) => {
  res.set({
    'Content-Type':      'text/event-stream',
    'Cache-Control':     'no-cache, no-transform',
    'Connection':        'keep-alive',
    'X-Accel-Buffering': 'no'
  });
  res.flushHeaders();

  const send = (e, d) => res.write(`event: ${e}\ndata: ${JSON.stringify(d)}\n\n`);
  const sub = events.subscribe(req.params.id, send);
  req.on('close', () => sub.unsubscribe());
});
```

### Streaming JSON arrays

```javascript
app.get('/users.ndjson', async (req, res) => {
  res.type('application/x-ndjson');
  for await (const row of knex('users').stream()) {
    if (!res.write(JSON.stringify(row) + '\n'))
      await once(res, 'drain');
  }
  res.end();
});
```

### Pitfalls
- `compression()` in front of SSE — chunked output gets buffered → *silent* client
- Missing `req.on('close')` — orphaned subscribers leak memory
- Awaiting a giant query before streaming — defeats the point
- Forgetting `X-Accel-Buffering: no` behind nginx

---

## Slide 10 — File Uploads

Three real choices — **memory**, **disk** or **direct-to-S3**. Pick by file size, retention, and whether your app instance should ever own the bytes.

### Multer — small files

```javascript
const multer = require('multer');
const upload = multer({
  storage: multer.memoryStorage(),
  limits:  { fileSize: 5 * 1024 * 1024, files: 1 },
  fileFilter: (_req, file, cb) => {
    if (!/^image\/(png|jpe?g|webp)$/.test(file.mimetype))
      return cb(new HttpError(415, 'unsupported_media', 'image png/jpg/webp only'));
    cb(null, true);
  }
});

app.post('/avatar', requireAuth, upload.single('image'), async (req, res) => {
  const url = await s3.put(req.file.buffer, key,
                           { ContentType: req.file.mimetype });
  res.json({ url });
});
```

### Direct-to-S3 (recommended)

```javascript
// 1) sign a one-time PUT URL on the server
app.post('/uploads/sign', requireAuth, async (req, res) => {
  const { contentType, contentLength } = SignReq.parse(req.body);
  if (contentLength > 50 * 1024 * 1024)
    throw HttpError(413, 'too_large', 'max 50MB');
  const key = `u/${req.user.sub}/${ulid()}`;
  const url = await getSignedUrl(s3, new PutObjectCommand({
    Bucket: BUCKET, Key: key, ContentType: contentType
  }), { expiresIn: 60 });
  res.json({ url, key });
});

// 2) browser uploads to S3, then POSTs the key back
app.post('/photos', requireAuth, async (req, res) => {
  const { key } = AttachReq.parse(req.body);
  await photos.create({ user: req.user.sub, key });
  res.status(201).end();
});
```

App never sees the bytes → no RAM spike, no egress cost, fewer middleware concerns. The same pattern works for GCS / Azure Blob.

### Always
- Cap `fileSize` *and* total request size at the proxy
- Validate MIME from sniffing the buffer, not just the header
- Strip / rename filenames; never echo raw user filenames into headers
- Scan with ClamAV (or async post-upload) before serving back

---

## Slide 11 — Rate Limiting & Abuse Protection

Three layers, each catching a different failure: **global** (DoS), **per-user** (fairness), **per-endpoint** (login brute force). Implement all three; they don't replace each other.

### express-rate-limit + Redis

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis').default;
const store = new RedisStore({ sendCommand: (...args) => redis.call(...args) });

// 1) global — per IP
const globalLimiter = rateLimit({
  store, windowMs: 60_000, max: 600,
  standardHeaders: 'draft-7', legacyHeaders: false,
  keyGenerator: (req) => req.ip,
});
app.use(globalLimiter);

// 2) per-user — after auth
const userLimiter = rateLimit({
  store, windowMs: 60_000, max: 1200,
  keyGenerator: (req) => req.user?.sub ?? req.ip,
});
app.use('/v1', requireAuth, userLimiter);
```

### Per-endpoint — the harshest one

```javascript
const loginLimiter = rateLimit({
  store, windowMs: 15 * 60_000, max: 5,
  keyGenerator: (req) => `${req.ip}:${req.body.email}`,
  skipSuccessfulRequests: true,
});
authRouter.post('/login', loginLimiter, login);
```

### Token-bucket vs fixed-window
- **Fixed window** — simple, but bursts at the boundary
- **Sliding window** — smoother; default in `express-rate-limit` v7
- **Token bucket** — fair under bursty traffic; use a Lua script in Redis

### Beyond rate limiting
- **Slow-down** — `express-slow-down` adds latency before blocking
- **Account lockouts** — after N failures, lock with email-based unlock
- **CAPTCHA / Turnstile** — on signup, password reset, contact forms
- **Body-size caps** at `express.json({ limit: '128kb' })` — cheapest DoS defence
- **Concurrency caps** per user with `p-limit` on expensive operations

> If you're behind a proxy, set `app.set('trust proxy', 1)` — otherwise `req.ip` is the proxy and *everyone* shares one bucket.

---

## Slide 12 — Caching

Four layers, top to bottom: **CDN**, **conditional GETs** (ETag / Last-Modified), **Redis cache-aside**, **in-process memo**.

### Cache-Control by intent

```javascript
// public, immutable assets
app.use('/static', express.static('public', { immutable: true, maxAge: '365d' }));

// private, per-user, never cached upstream
app.use('/v1/me', (req, res, next) => {
  res.set('Cache-Control', 'private, no-store, no-cache, must-revalidate');
  next();
});

// cdn-cacheable list, short TTL
app.get('/v1/posts', cache(60), listPosts);

function cache(maxAge) {
  return (_req, res, next) => {
    res.set('Cache-Control', `public, max-age=${maxAge}, stale-while-revalidate=30`);
    next();
  };
}
```

### ETag / 304

```javascript
app.set('etag', 'strong');

app.get('/v1/posts/:id', async (req, res) => {
  const p = await posts.byId(req.params.id);
  if (!p) throw NotFound();
  const tag = `"${p.id}-${p.version}"`;
  if (req.get('if-none-match') === tag) return res.status(304).end();
  res.set('ETag', tag).json(p);
});
```

### Cache-aside (Redis)

```javascript
async function postById(id) {
  const k = `post:${id}`;
  const hit = await redis.get(k);
  if (hit) return JSON.parse(hit);

  const row = await db('posts').where({ id }).first();
  if (row) await redis.set(k, JSON.stringify(row), 'EX', 60);
  return row;
}

async function updatePost(id, patch) {
  await db('posts').where({ id }).update(patch);
  await redis.del(`post:${id}`);
}
```

### Beware the stampede
When the key expires under load, every concurrent request misses and hits the DB. Mitigate with **single-flight** (one request fills the cache, others wait), **stale-while-revalidate** at the cache layer, or **jittered TTLs** (`EX 60 + rand(0,10)`).

### What not to cache
- Anything personalised behind a public URL
- Lists with cursors — tight TTL or skip
- Any 4xx / 5xx — cache successes only

---

## Slide 13 — Database Integration Patterns

One pool per process; one transaction per request that needs one; never `require('./db')` deep in services.

### Repository + service shape

```javascript
// repositories/users.js
module.exports = (db) => ({
  byEmail: (e) => db('users').where({ email: e }).first(),
  create:  (i) => db('users').insert(i).returning('*'),
});

// services/users.js
module.exports = ({ usersRepo, queue, logger }) => ({
  async signup(input) {
    const exists = await usersRepo.byEmail(input.email);
    if (exists) throw Conflict('email taken');
    const [user] = await usersRepo.create(input);
    await queue.add('welcome-email', { id: user.id });
    return user;
  }
});
```

### Transaction per request

```javascript
function withTransaction(db) {
  return async (req, _res, next) => {
    req.tx = await db.transaction();
    res.on('finish', () => {
      if (res.statusCode < 400) req.tx.commit();
      else                       req.tx.rollback();
    });
    res.on('close', () => req.tx.rollback().catch(()=>{}));
    next();
  };
}

postsRouter.post('/transfer', withTransaction(db), transfer);
```

### What not to do
- Don't wrap every request in a transaction
- Don't share a checked-out client across awaits
- Don't open a connection per query

### Read replicas
Two pools, two clients: `writer` and `reader`. Default to `writer` for the request lifetime; opt in to `reader` for clearly idempotent GETs. Be honest about replication lag.

### Health
Expose `/readyz` that runs `SELECT 1` through the pool. Liveness (`/healthz`) only checks the process is alive — *don't* let DB outages restart the pod and lose warm caches.

---

## Slide 14 — Background Jobs & Decoupling

If a request is taking > 200ms because of a side-effect (email, webhook, image resize), **enqueue, don't await**. The classic Node stack is **BullMQ** on Redis.

### Enqueue from a handler

```javascript
const { Queue } = require('bullmq');
const emails = new Queue('emails', { connection: redis });

async function signup(input) {
  const user = await usersRepo.create(input);
  await emails.add('welcome', { userId: user.id }, {
    attempts: 5,
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: 1000,
    removeOnFail:     500,
  });
  return user;
}
```

### Worker process

```javascript
const { Worker } = require('bullmq');

new Worker('emails', async (job) => {
  const u = await users.byId(job.data.userId);
  await mailer.send({ to: u.email, template: job.name });
}, { connection: redis, concurrency: 10 });
```

### Patterns that pay off
- **Idempotent jobs** — re-running must not double-charge / double-send
- **Outbox pattern** for "DB write + queue": insert into `outbox` in the same transaction; a relay process pushes to BullMQ
- **Dead-letter queue** — jobs that fail past `attempts` land in a "failed" view; alert on growth
- **Bounded concurrency** per worker, sized to downstream's tolerance

### Operational kit
- **Bull Board** for an admin UI — behind auth, never public
- **BullMQ metrics** → Prometheus: queue length, processing time, failure rate
- Run workers on *separate* deploys from the API — scale them independently
- Cap memory; let the orchestrator restart leaky workers on RSS thresholds

> Don't reach for queues for everything. If the work is < 50ms and rarely fails, in-process or just-do-it is fine.

---

## Slide 15 — Testing

Three layers: **unit** (pure functions), **HTTP integration** (supertest against a built app), **contract** (against the real OpenAPI). Most teams over-invest in unit and under-invest in HTTP.

### HTTP integration with supertest

```javascript
const request = require('supertest');
const { createApp } = require('../src/app');

let app, db;
beforeAll(async () => {
  db  = await openTestDb();
  app = createApp({ db, queue: fakeQueue, logger });
  await db.migrate.latest();
});
afterAll(async () => { await db.destroy(); });

test('POST /v1/users returns 201', async () => {
  const res = await request(app)
    .post('/v1/users')
    .send({ email: 'a@x.io', name: 'Alice' })
    .expect(201);
  expect(res.body).toMatchObject({ email: 'a@x.io' });
});

test('rejects invalid body', async () => {
  await request(app).post('/v1/users').send({}).expect(400);
});
```

### Auth in tests

```javascript
async function asUser(app, claims = {}) {
  const token = await signTestToken({ sub: 'u_1', ...claims });
  const a = request.agent(app);
  a.set('authorization', `Bearer ${token}`);
  return a;
}

const me = await asUser(app, { scope: 'users:read' });
await me.get('/v1/users/me').expect(200);
```

### Contract tests

```javascript
const OpenAPIResponseValidator = require('openapi-response-validator').default;
const spec = require('./openapi.json');

const validator = new OpenAPIResponseValidator({
  responses: spec.paths['/v1/users/{id}'].get.responses,
  components: spec.components,
});

test('GET /v1/users/{id} matches the contract', async () => {
  const res = await request(app).get('/v1/users/u_1').expect(200);
  const errs = validator.validateResponse(200, res.body);
  expect(errs).toBeUndefined();
});
```

### What to fake, what to keep real

| Dependency | In tests |
|------------|----------|
| Database | Real (SQLite or Testcontainers PG) |
| Queue | In-memory fake; assert `.add(...)` calls |
| HTTP downstream | nock / msw |
| Time | `jest.useFakeTimers()` |
| Crypto / IDs | Inject a generator; freeze in tests |

> Don't test the framework. Test *your* contract: status, body shape, side-effects.

---

## Slide 16 — Observability

Three pillars: **structured logs**, **RED metrics**, **distributed traces**. Minimum competent setup is **pino + OpenTelemetry + a request-ID middleware** — ship that, then iterate.

### Structured logs (pino)

```javascript
const pino = require('pino');
const pinoHttp = require('pino-http');

const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: ['req.headers.authorization',
           'req.headers.cookie',
           'res.headers["set-cookie"]'],
  formatters: { level: (l) => ({ level: l }) },
});

app.use(pinoHttp({
  logger,
  genReqId: (req) => req.headers['x-request-id'] ?? randomUUID(),
  customLogLevel: (_, res, err) =>
    err || res.statusCode >= 500 ? 'error'
    : res.statusCode >= 400 ? 'warn' : 'info',
}));
```

### Tracing — OpenTelemetry

```javascript
// otel.js — required *before* express
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

new NodeSDK({
  serviceName: 'api',
  instrumentations: [getNodeAutoInstrumentations()],
}).start();
```

### Request-ID propagation
- Read `x-request-id` if present; otherwise generate (ULID / UUID v7)
- Echo back in the response header
- Log it with every line; pass it to downstream calls
- Bind it to the OTel trace via `span.setAttribute('http.request_id', id)`

### Don't observe like a junior
- Don't `console.log(req.body)` — PII / tokens / passwords
- Don't unbounded-cardinality — never tag metrics by user ID
- Don't sample 100% of traces in prod; 1–5% with tail sampling on errors
- Don't emit timing logs that are also metrics — pick one

---

## Slide 17 — Security Hardening

Six controls cover most of the OWASP Top 10 for a Node API. None are optional in production.

### Helmet & CSP

```javascript
const helmet = require('helmet');

app.use(helmet({
  contentSecurityPolicy: {
    useDefaults: true,
    directives: {
      'default-src':  ["'self'"],
      'script-src':   ["'self'", "'strict-dynamic'", "'nonce-PLACEHOLDER'"],
      'style-src':    ["'self'", "'unsafe-inline'"],
      'img-src':      ["'self'", 'data:', 'https://cdn.example'],
      'connect-src':  ["'self'", 'https://api.example'],
      'object-src':   ["'none'"],
      'frame-ancestors': ["'none'"],
      'base-uri':     ["'self'"],
      'form-action':  ["'self'"],
      'upgrade-insecure-requests': [],
    },
  },
  crossOriginOpenerPolicy:   { policy: 'same-origin' },
  crossOriginResourcePolicy: { policy: 'same-site' },
  referrerPolicy:            { policy: 'no-referrer' },
}));
```

### CORS — deliberately

```javascript
const cors = require('cors');
const allow = new Set(['https://app.example.com', 'https://admin.example.com']);

app.use(cors({
  origin: (origin, cb) => cb(null, !origin || allow.has(origin)),
  credentials: true,
  methods: ['GET','POST','PUT','PATCH','DELETE'],
  allowedHeaders: ['authorization','content-type','idempotency-key'],
  maxAge: 600,
}));
```

### CSRF for cookie-auth

```javascript
const csurf = require('csurf');

app.use(csurf({
  cookie: { httpOnly: true, secure: true, sameSite: 'lax' }
}));

app.use((req, res, next) => {
  res.locals.csrfToken = req.csrfToken();
  next();
});
```

For pure JWT-Bearer APIs (no cookies), CSRF doesn't apply — but check every endpoint to be sure.

### Other essentials
- **Body size cap** — `express.json({ limit: '128kb' })` + proxy cap
- **HSTS** via helmet (1 year, includeSubDomains, preload)
- **Cookies**: `HttpOnly · Secure · SameSite=Lax`
- **Dependency hygiene**: `npm audit`, Renovate, lockfile
- **Secrets** via env / KMS — never commit, never log, never echo in errors

> The biggest mistake: trusting input. Validate it, escape it on output, parameterise SQL, never `eval` a JSON-y string from the wire.

---

## Slide 18 — Performance

Express is fast enough that the framework is rarely the bottleneck. The wins live in **CPU cores used**, **downstream parallelism**, **payload size**, and **serialisation**.

### Use all the cores

```javascript
const cluster = require('node:cluster');
const cpus = require('node:os').availableParallelism();
if (cluster.isPrimary) {
  for (let i = 0; i < cpus; i++) cluster.fork();
} else {
  require('./server');
}

// or PM2: pm2 start server.js -i max --name api
```

In Kubernetes, prefer **one process per pod** — the orchestrator handles scaling. Cluster mode is for bare-metal / single-VM deploys.

### Compression at the edge, not in Node
Default: no compression in Express. Reverse proxy (nginx / cloudfront) does gzip/brotli — faster, async. Only enable `compression()` if you have no proxy.

### Serialisation matters
- `JSON.stringify` a 1 MB object ≈ 8–15ms blocking. Stream large lists as NDJSON
- Don't return what the client doesn't ask for — field selection on list endpoints
- Use `fast-json-stringify` if you have a fixed schema and serialisation shows in profiles

### Quick wins
- **Keep-alive** — `server.keepAliveTimeout = 65_000` behind ALB / nginx
- **HTTP agent pooling** for downstream calls — `undici` / shared `http.Agent`
- **Parallelise** independent awaits — `Promise.all` or `Promise.allSettled`
- **Profile** with `--inspect` + Chrome DevTools or 0x flame graphs
- **Cap concurrency** on event-loop-blocking work — `p-limit`

> If you're event-loop blocked, move the work: `worker_threads` for CPU, queues for I/O. Don't hand-tune V8.

---

## Slide 19 — Graceful Shutdown

The orchestrator sends **SIGTERM** and waits up to `terminationGracePeriodSeconds` (default 30s in K8s). Without graceful shutdown, in-flight requests get `ECONNRESET` and clients see 5xx on every deploy.

### The shutdown sequence

```javascript
const server = app.listen(PORT, onListen);
let shuttingDown = false;

async function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;
  log.info({ signal }, 'shutting down');

  server.close((err) => { if (err) log.error({err}); });
  app.locals.ready = false;

  await worker.close();
  await db.destroy();
  await redis.quit();
  process.exit(0);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

### Liveness vs readiness

```javascript
app.get('/healthz', (_q, r) => r.json({ ok: true }));

app.get('/readyz', async (_q, r) => {
  if (shuttingDown || !app.locals.ready) return r.sendStatus(503);
  try { await db.raw('select 1'); r.json({ ok: true }); }
  catch { r.sendStatus(503); }
});
```

### Why this order matters
1. SIGTERM → flip `readyz` to 503 first — LB stops sending new traffic within ~5s
2. `server.close()` — existing connections finish on their own deadline
3. Workers close after the API does — jobs already in flight finish; new pulls stop
4. DB / cache close last — only after handlers can no longer use them

### Tunables
- K8s `terminationGracePeriodSeconds` > your slowest endpoint
- K8s `preStop` hook of `sleep 5` avoids the LB / readiness race
- `server.headersTimeout / requestTimeout` — cap how long an in-flight request can stall shutdown
- Set `keepAliveTimeout` > load balancer's idle timeout

### Don't
- Don't `process.exit()` on SIGTERM — you'll cut active sockets
- Don't combine `/healthz` and `/readyz` — they answer different questions
- Don't catch SIGTERM and *not* exit — the orchestrator escalates to SIGKILL

---

## Slide 20 — Reverse Proxy & TLS

In production, Express is almost always behind a reverse proxy — nginx, Caddy, an ALB, Cloudflare. The proxy terminates TLS, handles compression, and rewrites headers. Express needs to **trust the right ones**.

### Trust proxy

```javascript
app.set('trust proxy', 1);   // 1 proxy in front (LB)
// or specific subnets
app.set('trust proxy', 'loopback, 10.0.0.0/8');

// req.ip       → client IP (X-Forwarded-For tail)
// req.protocol → "http" or "https" from X-Forwarded-Proto
// req.secure   → true behind HTTPS-terminating proxy
```

### nginx in front of Express

```nginx
server {
  listen 443 ssl http2;
  server_name api.example.com;

  ssl_certificate     /etc/ssl/cert.pem;
  ssl_certificate_key /etc/ssl/key.pem;

  client_max_body_size 10m;

  location / {
    proxy_pass         http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   Connection        '';
    proxy_buffering    off;
    proxy_read_timeout 120s;
  }
}
```

### What the proxy should do
- **TLS termination** — certificate, OCSP stapling, modern ciphers
- **HTTP/2** — multiplexing on the wire; HTTP/1.1 to Node is fine
- **gzip / brotli** — compression with shared dictionaries; Express skips it
- **Static** — serve assets without ever hitting Node
- **WAF / rate limit** at the perimeter — cheaper than in-app

### Common bites
- Missing `trust proxy` → rate limits all share one IP, HTTPS detection wrong
- Mismatched timeouts — Node's `keepAliveTimeout` < proxy idle → 502 on every connection reuse
- Buffering on for SSE → clients see nothing until the response ends

---

## Slide 21 — OpenAPI & Contract-First

Two viable paths: **code-first** (write Express, generate spec) or **spec-first** (write OpenAPI, validate against it). Pick one per repo.

### Code-first — from Zod / TypeBox

```typescript
import { OpenAPIRegistry, OpenApiGeneratorV3 }
  from '@asteasolutions/zod-to-openapi';

const registry = new OpenAPIRegistry();
registry.register('User', UserSchema);
registry.registerPath({
  method: 'get', path: '/users/{id}',
  summary: 'Get a user by ID',
  request: { params: z.object({ id: z.string().uuid() }) },
  responses: {
    200: { description: 'OK',
           content: { 'application/json': { schema: UserSchema } } },
    404: { description: 'Not found' },
  }
});

const spec = new OpenApiGeneratorV3(registry.definitions)
  .generateDocument({ openapi: '3.0.3', info, servers });

app.get('/openapi.json', (_q, r) => r.json(spec));
```

### Spec-first — validate against it

```javascript
const OpenApiValidator = require('express-openapi-validator');

app.use(OpenApiValidator.middleware({
  apiSpec: './openapi.yaml',
  validateRequests:  true,
  validateResponses: { coerceTypes: false },
}));
```

### Either way, ship docs
- `/docs` → Swagger UI or Redoc, behind admin auth
- `/openapi.json` → the canonical document
- Pin the spec version in `info.version`; bump on contract change
- SDK generation (`openapi-typescript`) — clients get types for free

### Why this pays off
- Frontend gets types and a mock server before the API ships
- Contract tests catch breakage before clients do
- External integrators have one document to rely on
- Versioning becomes a deliberate act

---

## Slide 22 — Express vs Fastify, Koa, Hono, NestJS

| Criterion | Express | Fastify | Koa | Hono | NestJS |
|-----------|---------|---------|-----|------|--------|
| Style | Middleware chain | Plugin / hook | Async middleware | Edge-first, web-std | Decorator, IoC |
| Speed | Solid | Fastest mainstream | ~Express | Excellent on edge | Slower (DI overhead) |
| Type safety | Manual | Built-in (JSON schema) | Manual | Excellent | Decorator-based |
| Schema | BYO (Zod / Joi) | Native JSON Schema | BYO | BYO (Zod nice) | BYO |
| Edge / Workers | No | Limited | No | First-class | No |
| Ecosystem | Largest | Growing | Smaller | Modern, smaller | Strong opinions |
| Learning curve | Lowest | Medium | Low | Low (web-std) | Steep |
| Best for | Most apps | Throughput-critical APIs | Tiny middleware DAGs | Cloudflare / Bun / Deno | Large enterprise apps |

### Stay on Express when
- Big team, big ecosystem dependency
- Throughput is fine and CPU isn't the bottleneck
- Boring is a feature

### Move to Fastify when
- You need 2–3× throughput on the same box
- You want JSON-Schema-driven validation + serialisation
- You like the plugin / hook architecture

### Move to Hono / NestJS when
- **Hono**: deploying to Cloudflare Workers / Bun / Deno
- **NestJS**: large team that benefits from DI & module boundaries

---

## Slide 23 — Express 4 → 5 Migration Playbook

### Step 1 — environment
- Bump Node to 18 LTS or 20 LTS
- Bump `express` to `^5.0.0`
- Run `npm dedupe`, audit transitive Express 4 deps

### Step 2 — mechanical replacements

```bash
- app.del(...)                       app.delete(...)
- req.param('x')                     req.params.x ?? req.query.x ?? req.body.x
- res.send(404, body)                res.status(404).send(body)
- res.redirect('/foo', 301)          res.redirect(301, '/foo')

# remove these dependencies entirely
- body-parser
- express-async-errors
```

### Step 3 — routing
- Audit every `app.use('/x*', ...)` → `'/x{*splat}'`
- Audit every regex-in-string route; switch to a `RegExp` literal or named param
- Old loose patterns sometimes match more than intended

### Step 4 — async error pipeline
- Remove `require('express-async-errors')`
- Run the test suite — most pass; tests that relied on Express 4 swallowing rejections need rewriting
- Confirm your error middleware is *last*, signature `(err, req, res, next)`

### Step 5 — production switches
- **Trust proxy** — revisit your value behind LB / nginx
- **Query parser** — default is now `'simple'`; if you used nested objects, set `app.set('query parser', 'extended')`
- **Body parser limits** — reapply `{ limit: '128kb' }` when switching from `body-parser`

### Step 6 — ship behind a flag
Deploy to a single-instance canary; watch error rate & p99; promote after a clean 24 hours. Most regressions are routing-pattern bugs that surface only on real traffic.

---

## Slide 24 — Summary & Next Steps

### What "in practice" really means
- One async error pipeline — throw, don't return error envelopes
- One validation tool at the boundary; raw `req.body` never reaches services
- One composition root — no global singletons; tests build their own app
- AuthZ before the handler, never inside it; deny by default
- Stream when payloads are big; offload when work is slow; queue when retries matter

### Production checklist
- helmet + CSP + CORS allow-list + rate limits
- pino structured logs + OpenTelemetry traces + RED metrics
- Graceful shutdown sequenced: readyz → server → workers → deps
- OpenAPI shipped at `/openapi.json`, validated in CI
- Tests at the HTTP layer with supertest; contract tests in CI

### Further reading
- [Express 5 migration guide](https://expressjs.com/en/guide/migrating-5.html)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices) (Goldberg)
- [Express security best practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [OpenAPI Specification](https://www.openapis.org/) & [JSON Schema](https://json-schema.org/)
- [OpenTelemetry for Node.js](https://opentelemetry.io/docs/instrumentation/js/)

### Companion decks
- [Introduction to Express.js](https://brendanjameslynskey.github.io/Introduction_to_Express_js/) — the foundation deck
- [Introduction to Express Middleware](https://brendanjameslynskey.github.io/Introduction_to_Express_Middleware/)
- [Introduction to REST API Design](https://brendanjameslynskey.github.io/Introduction_to_REST_API_Design/)
- [Introduction to Node.js & Databases](https://brendanjameslynskey.github.io/Introduction_to_Node_Databases/)
- [Introduction to Knex](https://brendanjameslynskey.github.io/Introduction_to_Knex/)

### The single best habit
Read your own access log every morning for a week. The slowest endpoints, the loudest 5xx, the strangest paths — you'll find more wins there than in any chapter of any book.
