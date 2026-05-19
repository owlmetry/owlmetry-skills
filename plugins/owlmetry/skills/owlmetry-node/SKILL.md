---
name: owlmetry-node
description: >-
  Integrate the Owlmetry Node.js SDK into a backend service for server-side
  analytics, event tracking, metrics, and funnels. Use when
  instrumenting a Node.js, Express, Fastify, or serverless project with Owlmetry.
allowed-tools: Read, Bash, Grep, Glob
---

## What is Owlmetry?

Owlmetry is a self-hosted analytics platform. The Node.js SDK captures events from backend services — API servers, job workers, serverless functions — and delivers them to the Owlmetry server. It buffers events in memory and flushes them in batches to avoid blocking your application's request path.

The SDK has zero runtime dependencies. All calls are non-blocking — events are buffered internally and sent in the background. A single `Owl.configure()` call initialises everything including a background flush timer.

## Version Check

Run silently. Only inform the user if an update is available.

- **SDK version**: `npm ls @owlmetry/node --json 2>/dev/null` for current version, `npm view @owlmetry/node version 2>/dev/null` for latest. If newer, offer `npm install @owlmetry/node@latest`.

Skill updates arrive through Claude Code's plugin marketplace (`/plugin marketplace update owlmetry-skills`).

## Prerequisite

You need an **ingest endpoint** and a **client key** (`owl_client_...`) for a backend app. Both come from the CLI setup flow.

If the user doesn't have these yet, follow the `/owlmetry-cli` skill first — it handles sign-up, project creation, and app creation. The ingest endpoint is saved to `~/.owlmetry/config.json` (`ingest_endpoint` field) and the client key is returned when creating an app.

> **Any time you need to run an `owlmetry` CLI command** (querying events, creating metrics/funnels, listing apps, etc.), **load the `/owlmetry-cli` skill first**. Do not guess CLI syntax — it has non-obvious subcommand patterns and flags.

## Install

```bash
npm install @owlmetry/node
```

Zero runtime dependencies. Node.js 20+. ESM only.

## Configure

Place `Owl.configure()` in the **main entry point** of your service — the root `index.ts`, `server.ts`, or `app.ts` file where your server starts. It must run once at process startup, before any other `Owl` calls. Do **not** put it in a helper/utility file — it should be alongside your other top-level initialization (e.g., `express()`, `Fastify()`, Firebase `setGlobalOptions`).

```typescript
import { Owl } from '@owlmetry/node';

Owl.configure({
  endpoint: 'https://ingest.owlmetry.com',
  apiKey: 'owl_client_...',
  serviceName: 'my-api',          // optional, default: "unknown"
  appVersion: '1.2.0',            // optional
  isDev: false,                   // optional, default: process.env.NODE_ENV !== "production"
  flushIntervalMs: 5000,          // optional, default: 5000
  flushThreshold: 20,             // optional, default: 20
  maxBufferSize: 10000,           // optional, default: 10000
  consoleLogging: true,           // optional, default: true — print events to console
});
```

- `apiKey` must start with `owl_client_`
- `isDev` defaults to `true` when `NODE_ENV !== "production"`
- Generates a fresh `sessionId` (UUID) on each `configure()` call. This is the process-wide default; per-request overrides are available via `Owl.withSession(sessionId)` (returns a `ScopedOwl`) or the `options.sessionId` parameter on `info/debug/warn/error`. Use them to link backend events to the session of the client that triggered them
- Registers a `beforeExit` handler to auto-flush on graceful shutdown

**Serverless (Firebase Cloud Functions, AWS Lambda, Vercel):** After adding `Owl.configure()`, also wrap your exported handler functions with `Owl.wrapHandler()` to guarantee events are flushed before the runtime freezes. This is essential boilerplate for serverless — without it, buffered events are lost:

```typescript
// AWS Lambda / generic serverless:
export const handler = Owl.wrapHandler(async (event, context) => { ... });

// Firebase Cloud Functions v2:
// IMPORTANT: Explicitly type the request parameter — TypeScript cannot infer
// the CallableRequest type through wrapHandler's generics.
import { onCall, type CallableRequest } from 'firebase-functions/v2/https';

export const myFunction = onCall(
  Owl.wrapHandler(async (request: CallableRequest) => {
    const { data, auth } = request;  // works — TypeScript knows the type
    // ...
  })
);
```

**TypeScript note:** `wrapHandler()` uses generic rest parameters (`<TArgs extends unknown[]>`), which means TypeScript sometimes infers handler parameters as `unknown` when the outer function (like Firebase's `onCall`) expects a specific callback type. If you see type errors like `Property 'data' does not exist on type 'unknown'`, explicitly annotate the handler's parameters (e.g., `request: CallableRequest`, `event: APIGatewayEvent`).

## Next Steps — Codebase Instrumentation

Once `Owl.configure()` is in place and the project builds successfully, **you MUST stop here and ask the user** which area they'd like to instrument first — even if the user's original prompt asked you to "instrument the app." Do not proceed with any code changes until the user chooses. Present these three options:

1. **Event & error logging** — Audit the codebase for request handlers, error paths, background jobs, and key operations. Add `Owl.info()`, `Owl.warn()`, `Owl.error()` calls at meaningful points throughout the service. This is SDK-only — no CLI setup required beyond what's already done.
2. **Structured metrics** — Identify operations worth measuring (API response times, database queries, external service calls, etc.). Add `Owl.startOperation()` / `Owl.recordMetric()` to track durations and success rates. **Requires CLI first:** each metric slug must be defined on the server via `owlmetry metrics create` (use the `/owlmetry-cli` skill) before the SDK can emit events for it.
3. **Funnel tracking** — Identify server-side user journeys (sign-up flow, payment processing, onboarding steps). Add `Owl.step()` calls at each step to measure drop-off. **Requires CLI first:** the funnel definition (with steps and event filters) must be created via `owlmetry funnels create` (use the `/owlmetry-cli` skill) before tracking makes sense.

After the user chooses, do a thorough audit of the entire codebase to find all relevant locations, then present a summary of proposed changes before making any edits.

## Buffering and Delivery

The SDK buffers events in memory and sends them to the server in batches. Understanding how buffering works helps you avoid lost events:

- **Buffer size** (`maxBufferSize: 10000`): maximum events held in memory. If the buffer fills up, new events are dropped. Increase this if your service produces bursts of events.
- **Flush interval** (`flushIntervalMs: 5000`): a background timer fires every 5 seconds and sends all buffered events. The timer uses `.unref()` so it won't keep the process alive.
- **Flush threshold** (`flushThreshold: 20`): if the buffer reaches this many events before the timer fires, an immediate flush is triggered.
- **HTTP timeout**: each flush request has a 10-second timeout to prevent hanging on slow or unreachable servers. Failed flushes are silently dropped (events are not retried).
- **Graceful exit**: a `beforeExit` handler auto-flushes remaining events on normal process termination. For SIGTERM/SIGINT, you must call `Owl.shutdown()` explicitly (see Graceful Shutdown).

The key implication: if your process crashes or is force-killed, any events still in the buffer are lost. For critical routes, use `wrapHandler()` to flush after each request.

## Log Events

Events are the core data unit. Use the four log levels to capture different kinds of backend activity:

- **`info`** — normal operations: server started, request handled, job completed, user action processed.
- **`debug`** — verbose detail for development: cache lookups, query plans, config loading, intermediate state.
- **`warn`** — something didn't go as expected but the process can continue: failed validation, precondition checks that fail, slow queries, rate limits approaching, fallback paths, deprecated API usage, missing optional config.
- **`error`** — a caught exception or hard failure inside a `try`/`catch`: database connection errors, external API timeouts, unhandled rejections, file system errors. Reserve for actual thrown errors, not for anticipated validation outcomes.

Choose **message strings** that are specific and searchable (`"Payment processing failed"` over `"error occurred"`). For the full rubric on what to log, what to attach, and what to skip, see *Instrumentation Principles* below.

```typescript
Owl.info('Server started', { port: 4000 });
Owl.debug('Cache miss', { key: 'user:123' });
Owl.warn('Invalid request payload', { field: 'email', reason: 'missing' });

try {
  await db.connect();
} catch (err) {
  // Pass the Error directly — the SDK extracts type, full stack, Error.cause
  // chains, AggregateError children, and Node ErrnoException fields
  // (code/syscall/path) into reserved `_error_*` attributes. The server
  // uses _error_type as a fingerprint discriminator so different error
  // classes with the same message stay on separate issues.
  Owl.error(err, 'while connecting to database', { host: 'db.example.com' });
}
```

All info/debug/warn methods: `Owl.info/debug/warn(message, attrs?, options?)`. `Owl.error` is overloaded — first argument may be a `string` (logger-style) or any value (`Error`, `unknown`, etc — exception-style; the SDK extracts structured fields from Error instances). The third `options` argument supports `{ attachments }` for uploading files alongside the event — see *File Attachments* below.

**Auto-capture of unhandled errors is on by default.** The SDK installs `process.on('uncaughtException')` and `process.on('unhandledRejection')` listeners that capture and forward as `Owl.error` events tagged with `_unhandled`. Crash semantics are preserved — your app keeps crashing on uncaught errors exactly like it would without the SDK, and any pre-existing handlers you've registered still run. Opt out via `Owl.configure({ ..., captureUnhandled: false })`.

`message` is silently truncated to 2000 characters; attribute values are silently truncated to 200 characters. Put long content in attributes, not in `message`.

Source module (file:line) is auto-captured from the call stack. `country_code` is **always `null`** for events from this SDK — the app's `platform` is `backend`, and the request reaches Owlmetry from the customer's hosting datacenter rather than an end user, so capturing the Cloudflare-derived country would be misleading. If you need per-user geography on backend events, attach it as a custom attribute on the event or as a user property.

`sdk_name` (`"owlmetry-node"`) and `sdk_version` (read from the package's `package.json` at build time) are auto-stamped on every event and feedback submission. **Do not set these manually** — they're managed by the SDK so the server can tell which SDK and version produced each event.

**Backend-specific examples:**
```typescript
Owl.info('Request handled', { method: 'POST', path: '/api/orders', status: 201 });
Owl.info('Background job completed', { job: 'send-emails', processed: 150 });
Owl.warn('Rate limit approaching', { current: 95, limit: 100, client_id: 'abc' });

try {
  await stripe.charges.create(params);
} catch (err) {
  Owl.error(err, 'Stripe charge failed', { endpoint: '/charges' });
}
```

## Instrumentation Principles

Before adding `info` / `warn` / `error` calls throughout the service, internalise these four rules. They turn the SDK from a logger into a queryable analytics surface.

### 1. Log outcomes, not steps

Emit **one rich event per request or job outcome**, not one event per line of code. The unit is the *thing the service did* (request handled, job completed, webhook processed, payment captured), not the work your code did to get there.

Don't narrate the request:
```ts
app.post('/api/orders', async (req, res) => {
  Owl.info('Order request received');
  Owl.info('Validating payload');
  Owl.info('Looking up user');
  Owl.info('Charging card');
  Owl.info('Order created');
  res.json(order);
});
```

One event per outcome, scoped to the request:
```ts
app.post('/api/orders', Owl.wrapHandler(async (req, res) => {
  const startedAt = Date.now();
  const order = await createOrder(req.body);
  req.owl.info('Order placed', {
    order_id: order.id,
    item_count: order.items.length,
    total_cents: order.totalCents,
    payment_method: order.paymentMethod,
    duration_ms: Date.now() - startedAt,
  });
  res.json(order);
}));
```

`req.owl` is the per-request scoped instance set up in middleware (see *Per-Request User Scoping* and *Per-Request Session Scoping*). It carries `user_id` + client `session_id` automatically, so every event inside the request lifecycle correlates without re-passing them.

For intermediate diagnostic signals (cache lookups, query plans, fallback decisions), use `debug` level — filtered out of production view by default. Don't pad `info` with code-narration noise.

### 2. Pack attributes wide, not events deep

One event with 15 attributes beats 15 events with one attribute each. For a server-side event, think through these axes and attach whatever's relevant:

| Axis | Backend examples |
|---|---|
| **Who** | `user_id` (auto-tagged via `withUser(req.user.id)`), `org_id`, `plan_tier`, `api_key_id`, `tenant_id` |
| **What** | `order_id`, `subscription_id`, `webhook_event_id`, `queue_message_id`, `feature_flag` |
| **Where** | route `path`, `method` (auto if you log inside a request handler), `region`, `pod`, `service_name` (auto from configure) |
| **How** | `payment_provider`, `auth_method`, `retry_count`, `fallback_taken`, `idempotency_key` |
| **How much** | `duration_ms`, `status_code`, `response_size_bytes`, `rows_processed`, `amount_cents` |

High-cardinality *attribute values* (user IDs, order IDs, request IDs) are a **feature**, not a smell — they let you click from a dashboard chart down to a specific failing request when triaging. The thing to control is *event frequency* (rule 3), not the uniqueness of values.

### 3. Aggregate batch work; don't log per iteration

If you're inside a `for` / `.forEach` / queue-drain / cron-fanout and reaching for `Owl.info`, stop. Either:

- Log a single summary event after the batch with counts and totals, **or**
- Use a structured lifecycle metric for the operation as a whole.

```ts
// Per-row — 10,000 events for one nightly job
for (const row of rows) {
  Owl.info('Processed row', { id: row.id });
}

// Per-batch — one event
const startedAt = Date.now();
let failed = 0;
const failureSamples: string[] = [];
for (const row of rows) {
  try { await process(row); }
  catch (err) {
    failed++;
    if (failureSamples.length < 5) failureSamples.push(String(err));
  }
}
Owl.info('Nightly sync completed', {
  job: 'user-export',
  processed: rows.length,
  failed,
  failure_samples: failureSamples.join(' | '),
  duration_ms: Date.now() - startedAt,
});
```

Same rule for retries: log the final outcome with `retry_count`, not one event per attempt. Same for SSE / WebSocket frames, polling loops, and stream chunks — log the connection outcome (status, frame count, duration), not each frame.

### 4. Log, metric, or funnel — pick by the question you want answered

- **Log event** (`Owl.info` / `warn` / `error`) — *"show me individual records when a specific request went wrong."* One-off occurrences, exception context, audit trail. Read on Dashboard → Events.
- **Lifecycle metric** (`Owl.startOperation` → `.complete` / `.fail` / `.cancel`) — *"show me p50/p95/p99 latency and success rate for this operation over time."* Requires server-side metric definition. Read on Dashboard → Metrics.
- **Single-shot metric** (`Owl.recordMetric`) — *"show me this point-in-time value trended over time."* Queue depth, cache hit rate, active connections.
- **Funnel** (`Owl.step`) — *"show me where users drop off across this multi-step backend flow."* Signup pipeline, payment processing, onboarding sequence. Requires server-side funnel definition. **Always inside a `withUser(...)` scope** — funnel analytics need a user ID to compute conversion.

The same operation can warrant several: a checkout pipeline gets a funnel for conversion, a lifecycle metric on `process-payment` for latency + failure rate, and an `Owl.error` on the catch path for triage.

> When in doubt, write one event with more attributes rather than several events with fewer.

## File Attachments (use sparingly)

When an error's root cause is bound up in a file the server received — a PDF that failed to parse, an upload that failed to decode, a model file that failed to load — you can attach the file to the error event so an engineer can download it from the dashboard, CLI, or MCP and reproduce locally.

```typescript
try {
  await PdfParser.parse(inputPath);
} catch (err) {
  Owl.error(err, 'pdf parse failed', undefined, {
    attachments: [
      { path: inputPath, name: 'input.pdf' },                               // from disk
      { buffer: diagnosticsJson, name: 'debug.json', contentType: 'application/json' }, // in memory
    ],
  });
}
```

**Attachments are a limited resource.** Each project has a storage quota (default **5 GB**) and each end-user has their own bucket within that project (default **250 MB per user**; reservations without a user_id are bounded only by the project ceiling). Before adding `attachments:` anywhere, make sure the file's bytes are *essential* to reproduce the bug. Good candidates:

- ✅ A parse failure on a user-uploaded document whose bytes you cannot reconstruct.
- ✅ A ML model file that fails to load — the bytes themselves are the suspect.
- ✅ A fixture-like artefact that reveals the bug when inspected.

Bad candidates — do not attach:

- ❌ Every error. Most backend failures are well-described by `attrs`.
- ❌ Data you can reconstruct from the request (URLs, IDs, small config).
- ❌ Large files that are downloaded rather than user-supplied — include the source URL.
- ❌ Logs or stack traces. Put them in `attrs` or stream them separately.

Uploads are strictly non-fatal: network errors and quota rejections (`user_quota_exhausted`, `quota_exhausted`) are logged (when `debug: true`) but the event still posts normally. Uploads use `fetch` and run on a serial queue so a 200 MB file never blocks event batching. `Owl.flush()` and `Owl.shutdown()` await pending uploads, so `wrapHandler()` and explicit flushes in serverless handlers deliver attachments before the runtime freezes. Attachments are not queued offline — if the process is killed without reaching `flush()`/`shutdown()`/`beforeExit`, the attachment is lost but the event is not.

## Per-Request User Scoping

In a multi-user backend, many requests are processed concurrently. Without scoping, you'd need to pass a user ID to every log call. `Owl.withUser()` creates a scoped instance that automatically tags all events with the user ID — create one per request in your auth middleware.

Use scoped instances whenever you know who the user is. Use the global `Owl` for system-level events (startup, shutdown, background jobs without a user context).

```typescript
const owl = Owl.withUser('user_42');
owl.info('User logged in');
owl.warn('Rate limit approaching', { requests: 95 });
owl.error('Payment failed', { reason: 'insufficient_funds' });
```

`ScopedOwl` has the same logging methods as `Owl` (`info`, `debug`, `warn`, `error`, `step`, `startOperation`, `recordMetric`, `setUserProperties`).

## Per-Request Session Scoping (Cross-SDK Correlation)

By default, the Node SDK stamps every event with one session ID generated at `configure()`. For a server handling many clients, that default is almost never what you want — you'd like backend events to share the client's session ID so you can see client + server activity together in one session view.

`Owl.withSession(sessionId)` returns a `ScopedOwl` that overrides the session ID on every event it emits. It's chainable with `withUser()` in either order:

```typescript
const owl = Owl.withSession(clientSessionId).withUser(userId);
owl.info('Request received', { path: req.url });
```

The typical pattern: have the client propagate its session ID via an HTTP header, then scope the request. Swift clients expose `Owl.sessionId` publicly for exactly this purpose.

```typescript
// Fastify
app.addHook('onRequest', async (request) => {
  const clientSessionId = request.headers['x-owl-session-id'] as string | undefined;
  const base = request.user?.id ? Owl.withUser(request.user.id) : Owl;
  request.owl = clientSessionId ? base.withSession(clientSessionId) : base;
});
```

```swift
// Swift client
var request = URLRequest(url: apiURL)
if let sessionId = Owl.sessionId {
    request.setValue(sessionId, forHTTPHeaderField: "X-Owl-Session-Id")
}
```

For callers that can't scope, `info/debug/warn/error` also accept a per-call `sessionId` in their options object:

```typescript
Owl.info('Webhook received', { provider: 'stripe' }, { sessionId: clientSessionId });
```

Precedence: per-call `options.sessionId` > scope `withSession(...)` > default process session ID.

Session IDs should be UUID strings (the Swift SDK's `Owl.sessionId` already is). Non-UUID values passed to `withSession()` or `options.sessionId` are silently ignored — the event falls back to the SDK's default session ID. This means a malformed or missing client header will never crash a request handler. Invalid values are logged via `console.error` only when `debug: true` is set on configure.

## Funnel Tracking

Backend funnels track server-side conversion flows — signup pipelines, payment processing, onboarding sequences, data processing stages. Unlike client-side funnels where users interact with UI, backend funnels typically track state transitions in response to API calls or background jobs.

**User ID is critical for backend funnels.** Funnel analytics group by user to calculate conversion — without a user ID, events can't be attributed. Always use a scoped instance (`Owl.withUser()`) or ensure user identity is set.

```typescript
Owl.step('signup-started');
Owl.step('email-verified', { method: 'link' });
Owl.step('profile-completed');

// With user scoping:
const owl = Owl.withUser(userId);
owl.step('checkout-completed', { item_count: '3' });
```

The step name you pass to `step()` must match the `step_name` in the funnel definition's `event_filter`. For example, if the step filter is `{"step_name": "signup-started"}`, then call `Owl.step('signup-started')`. Define matching funnels via `/owlmetry-cli`.

**Note:** `step()` attributes must be `Record<string, string>` (string values only).

## Structured Metrics

Use structured metrics when you want aggregated performance data (averages, percentiles, error rates) rather than individual event logs. Metrics give you trend data and alerting signals; events give you individual records for debugging.

**Decision: lifecycle vs single-shot:**
- **Lifecycle** — when measuring something with a duration (start → end). Backend examples: HTTP request handling, database queries, external API calls, file processing, queue job execution.
- **Single-shot** — when recording a point-in-time measurement. Backend examples: queue depth, cache hit rate, active connections, deployment markers.

The metric definition must exist on the server **before** the SDK emits events for that slug. Create it via CLI first.

### Lifecycle operations (start -> complete/fail/cancel)

```typescript
const op = Owl.startOperation('database-query', { table: 'users' });
try {
  const result = await db.query('SELECT * FROM users');
  op.complete({ rows: result.length });
} catch (err) {
  op.fail(String(err), { table: 'users' });
}

// Or cancel:
op.cancel({ reason: 'timeout' });
```

`duration_ms` and `tracking_id` (UUID) are auto-added. Create the metric definition first:
```bash
owlmetry metrics create --project-id <id> --name "Database Query" --slug database-query --lifecycle --format json
```

### Single-shot measurements

```typescript
Owl.recordMetric('cache-hit-rate', { rate: '0.95', cache: 'redis' });
```

Works with scoped instances too: `owl.startOperation(...)`, `owl.recordMetric(...)`.

**Slug rules:** lowercase letters, numbers, hyphens only. Invalid slugs are auto-corrected with a warning.

## User Properties

Attach custom key-value metadata to users. Properties are merged server-side — existing keys not in your call are preserved.

```typescript
// With explicit user ID
Owl.setUserProperties('user_123', { plan: 'premium', org: 'acme' });

// With scoped instance
const owl = Owl.withUser('user_123');
owl.setUserProperties({ plan: 'premium', org: 'acme' });
```

Set a value to `""` to delete a key. All values must be strings. Max 50 properties per user, 50-char keys, 200-char values.

Use for user-level data that changes infrequently (subscription status, plan tier, company). For event-specific data, use custom attributes on events instead.

**RevenueCat integration prompt** — copy-paste to set up subscription tracking:

```
Connect RevenueCat to my Owlmetry project so I can see paid vs free users:

1. Use `/owlmetry-cli` to add the RevenueCat integration with my RC V2 secret API key
   (needs Customer information → Read only AND Project configuration → Read only at the section level, everything else No access).
2. Show me the webhook setup values from the output so I can paste them into RevenueCat.
3. After I confirm the webhook is live, run a bulk sync to backfill existing subscribers.
4. Add Owl.setUserProperties() calls in my Node.js webhook handler or purchase
   callback so the dashboard updates immediately when a user subscribes, without
   waiting for RevenueCat's webhook.
```

## User Feedback

Forward free-text feedback collected by your own frontend (form, chat widget, support page) into the Owlmetry feedback tracker. Use this when your Node server is the pass-through between the end user and Owlmetry.

```typescript
try {
  const receipt = await Owl.sendFeedback(req.body.message, {
    name: req.body.name,
    email: req.body.email,
    userId: req.auth?.userId,
    sessionId: req.headers['x-owl-session-id'], // UUID; non-UUIDs are silently dropped
  });
  res.json({ id: receipt.id });
} catch (err) {
  res.status(400).json({ error: err.message });
}

// Scoped form — auto-attaches userId + sessionId from the scope
const owl = Owl.withUser(userId).withSession(sessionHeader);
await owl.sendFeedback(message, { email });
```

Unlike `info` / `error` / `setUserProperties`, `sendFeedback` **throws on failure** — the caller is waiting on a user action, so errors must propagate. Always wrap in `try/catch`.

`message` is required and trimmed; max 4000 chars (server-enforced). The server validates the email and environment and returns a descriptive error if either is rejected. Returns `{ id, createdAt }` — surface `id` back to the user if you want to reference the submission later (for example, in a "Thanks, we got your feedback #abcd" toast).

Feedback shows up on `/dashboard/feedback`, in `owlmetry feedback list`, and via the MCP `list-feedback` tool.

## Serverless Support

Serverless environments (AWS Lambda, Vercel Functions, Cloud Functions) freeze the runtime after each invocation. Events still in the buffer are lost when the runtime freezes — the flush timer and `beforeExit` handler may never fire.

`wrapHandler()` solves this by calling `Owl.flush()` in a `finally` block after every invocation, guaranteeing delivery before the runtime freezes.

**When to use `wrapHandler()`:**
- **Serverless functions**: always wrap — this is the only reliable way to flush.
- **Long-running servers, critical routes**: wrap routes where losing events is unacceptable (payment processing, user signup). The 5-second flush timer handles the rest.
- **Long-running servers, general routes**: not needed — the background timer and shutdown handler cover normal operation.

```typescript
// AWS Lambda
export const handler = Owl.wrapHandler(async (event, context) => {
  Owl.info('Lambda invoked', { functionName: context.functionName });
  // ... handle request ...
  return { statusCode: 200, body: JSON.stringify({ ok: true }) };
});

// Express route
app.post('/api/checkout', Owl.wrapHandler(async (req, res) => {
  const owl = Owl.withUser(req.user?.id);
  const op = owl.startOperation('checkout');
  // ... process ...
  op.complete();
  res.json({ ok: true });
}));
```

`wrapHandler` calls `Owl.flush()` in a `finally` block, ensuring events are sent even if the handler throws.

## Graceful Shutdown

Without explicit shutdown, events buffered since the last flush (up to 5 seconds worth) are lost when the process exits via SIGTERM or SIGINT. The `beforeExit` handler only covers graceful exits (no pending work, no signal termination).

Always add a shutdown handler for production services:

```typescript
process.on('SIGTERM', async () => {
  await Owl.shutdown();
  process.exit(0);
});
```

- `shutdown()` stops the flush timer, flushes remaining events, and clears state.
- The `beforeExit` handler auto-flushes on graceful process exit even without explicit shutdown.

## Integration Patterns

These patterns show how to wire Owlmetry into common Node.js frameworks. The key ideas:
- **Create a scoped logger in auth middleware** so all route handlers get user-attributed events automatically.
- **Use `wrapHandler()` on critical routes** where losing events is unacceptable.
- **Call `shutdown()` in the framework's close hook** to flush on process termination.

### Express middleware

```typescript
import express from 'express';
import { Owl } from '@owlmetry/node';

const app = express();

// Scoped logging middleware
app.use((req, res, next) => {
  req.owl = req.user?.id ? Owl.withUser(req.user.id) : Owl;
  next();
});

app.post('/api/order', Owl.wrapHandler(async (req, res) => {
  const op = req.owl.startOperation('create-order', { items: req.body.items?.length });
  try {
    const order = await createOrder(req.body);
    op.complete({ order_id: order.id });
    res.json(order);
  } catch (err) {
    op.fail(String(err));
    res.status(500).json({ error: 'Order failed' });
  }
}));

process.on('SIGTERM', async () => {
  await Owl.shutdown();
  process.exit(0);
});
```

### Fastify hooks

```typescript
import Fastify from 'fastify';
import { Owl } from '@owlmetry/node';

const fastify = Fastify();

fastify.addHook('onRequest', (request, reply, done) => {
  request.owl = request.user?.id ? Owl.withUser(request.user.id) : Owl;
  done();
});

fastify.addHook('onClose', async () => {
  await Owl.shutdown();
});

fastify.post('/api/process', async (request, reply) => {
  request.owl.info('Processing request', { path: request.url });
  // ... handle request ...
  await Owl.flush();
  return { ok: true };
});
```

## Instrumentation Strategy

When instrumenting a backend service, follow this priority:

**Always instrument (events — no CLI setup needed):**
- Server startup and shutdown (`info`)
- Request handling: key route hits, responses sent (`info` with method/path/status)
- Caught exceptions: catch blocks, unhandled rejections, external API failures (`error`)
- Validation failures and pre-checks: bad input, missing optional config, rate limits (`warn`)
- Authentication events: login, logout, token refresh (`info`)
- Core business actions: order placed, payment processed, email sent (`info`)
- Background jobs: started, completed, failed (`info`/`error`)

**Instrument when relevant (metrics — requires CLI `owlmetry metrics create` first):**
- Lifecycle metrics for operations where duration matters: API response time, database queries, external service calls, file processing
- Single-shot metrics for point-in-time values: queue depth, cache hit rate, active connections

**Instrument when relevant (funnels — requires CLI `owlmetry funnels create` first):**
- Multi-step server-side flows you want to measure conversion on: signup pipeline, payment processing, onboarding sequence
- Always use `Owl.withUser()` for funnel events — funnel analytics require user IDs to calculate conversion

**What NOT to instrument — concrete list:**

Never put these in event messages or attribute values (the message and attribute caps don't redact, they just truncate):
- Secrets, API keys, bearer tokens, auth cookies, JWT contents, refresh tokens
- Passwords (cleartext or hashed)
- Raw card numbers / CVVs / bank account numbers / SSNs / national IDs
- Full request bodies, full response bodies — log status + sizes + resource IDs instead
- `Authorization` / `Cookie` / `Set-Cookie` header values
- Raw IP addresses — the server stamps `country_code` automatically; finer geo doesn't belong in events
- Raw `error.stack` as an attribute value — pass the `Error` to `Owl.error()` instead; the SDK extracts the stack into `_error_stack` with a 16K cap and the proper fingerprinting key

Skip — these are usually noise, not value:
- High-frequency health checks, heartbeats, keep-alives
- Per-row logs inside batch jobs (log the batch outcome — see *Instrumentation Principles* rule 3)
- Per-attempt logs on retries (log the final outcome with `retry_count`)
- Every database query (instrument categories or wrap with a metric — see *Instrumentation Principles* rule 4)
- Sensitive business amounts on a per-event basis if user-aggregated revenue is what you actually want — use user properties or the RevenueCat integration
