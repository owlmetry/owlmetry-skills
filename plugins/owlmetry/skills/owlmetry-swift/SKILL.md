---
name: owlmetry-swift
description: >-
  Integrate the Owlmetry Swift SDK into an iOS, iPadOS, macOS, or watchOS
  app for analytics, event tracking, metrics, funnels, in-app
  questionnaires (surveys / NPS / ratings) with progressive draft saves
  and resume-mid-flow, user feedback views, file attachments, crash and
  error reporting, Apple Search Ads attribution, and push notification
  registration. Use when instrumenting a Swift or SwiftUI project with
  Owlmetry, building surveys, resuming half-finished questionnaire
  drafts, capturing feedback, attaching files to events, or wiring Apple
  Search Ads attribution.
allowed-tools: Read, Bash, Grep, Glob
---

## What is Owlmetry?

Owlmetry is a self-hosted analytics platform. The Swift SDK captures events from iOS, iPadOS, macOS, and watchOS apps and delivers them to the Owlmetry server. It handles buffering, gzip compression, offline queuing, session management, and network monitoring automatically — you just call logging methods and the SDK takes care of delivery. On watchOS the SDK also auto-relays through the paired iPhone when direct HTTP isn't available — see the "watchOS Companion Apps" section below.

The SDK is a static `Owl` enum with no external dependencies. All calls are non-blocking (events are buffered and flushed in batches). A single `configure()` call initialises everything.

## Version Check

Run silently. Only inform the user if an update is available.

- **SDK version**: Read `Package.resolved` for the current resolved revision, then compare against `curl -sf https://api.github.com/repos/owlmetry/owlmetry-swift/releases/latest | jq -r .tag_name`. If newer, inform the user.

Skill updates arrive through Claude Code's plugin marketplace (`/plugin marketplace update owlmetry-skills`).

## Prerequisite

You need an **ingest endpoint** and a **client key** (`owl_client_...`) for an Apple-platform app. Both come from the CLI setup flow.

If the user doesn't have these yet, follow the `/owlmetry-cli` skill first — it handles sign-up, project creation, and app creation. The ingest endpoint is saved to `~/.owlmetry/config.json` (`ingest_endpoint` field) and the client key is returned when creating an app.

> **Any time you need to run an `owlmetry` CLI command** (querying events, creating metrics/funnels, listing apps, etc.), **load the `/owlmetry-cli` skill first**. Do not guess CLI syntax — it has non-obvious subcommand patterns and flags.

## Add Swift Package

**Minimum platforms:** iOS 16.0, macOS 13.0, watchOS 10.0. Zero external dependencies.

**First, fetch the latest SDK release tag** — pin to a version so builds are reproducible:

```bash
curl -sf https://api.github.com/repos/owlmetry/owlmetry-swift/releases/latest | jq -r .tag_name
# e.g. "v0.1.0" → strip the leading "v" → use "0.1.0" in the snippets below.
```

If the GitHub API call fails or returns no tags (early alpha may have none), fall back to `branch: "main"` / `Branch > main` in the snippets below.

### Option A — Package.swift projects

If the project has a `Package.swift`, add the dependency there (replace `0.1.0` with the tag you fetched above):

```swift
dependencies: [
    .package(url: "https://github.com/owlmetry/owlmetry-swift.git", from: "0.1.0")
]
```
Add to your target:
```swift
.target(name: "YourApp", dependencies: [
    .product(name: "Owlmetry", package: "owlmetry-swift")
])
```

Then run `swift package resolve` to fetch the dependency.

### Option B — Xcode projects (.xcodeproj)

For `.xcodeproj`-based projects with no `Package.swift`, add the Owlmetry Swift package by editing `<Project>.xcodeproj/project.pbxproj` directly to add a remote Swift package reference for `https://github.com/owlmetry/owlmetry-swift.git` pinning `kind = upToNextMajorVersion` from the tag you fetched (e.g. `minimumVersion = 0.1.0`), product: `Owlmetry`. Do not ask the user to add it manually in Xcode.

### Option C — Ask the user (last resort)

If pbxproj editing fails or the project structure is too complex, ask the user to add the package in Xcode:

1. File > Add Package Dependencies
2. Enter URL: `https://github.com/owlmetry/owlmetry-swift.git`
3. Set rule to **Up to Next Major Version** starting at the tag from the fetch above (e.g. `0.1.0`)
4. Add **Owlmetry** to the app target

## Verify Package Integration

After adding the package, resolve dependencies and build:

```bash
xcodebuild -resolvePackageDependencies -project <path>.xcodeproj -quiet
xcodebuild -project <path>.xcodeproj -scheme <SchemeName> -destination 'platform=iOS Simulator,name=iPhone 16' build -quiet
```

If the build succeeds, proceed with configuration. The "No such module 'Owlmetry'" warning in editors (SourceKit) is expected and resolves during a real `xcodebuild`.

## Configure

Configuration must happen once, as early as possible — in the `@main` App `init()` or AppDelegate `didFinishLaunching`. **Do not defer it** to a later point (e.g., after async setup or user consent). The SDK measures app launch time (`_launch_ms`) from process start to the `configure()` call, so placing it early gives an accurate cold-start metric. It also ensures no events are dropped before configuration. Each `configure()` call generates a fresh `session_id` (UUID) that groups all subsequent events together.

```swift
import Owlmetry

@main
struct MyApp: App {
    init() {
        do {
            try Owl.configure(
                endpoint: "https://ingest.owlmetry.com",
                apiKey: "owl_client_..."
            )
        } catch {
            print("Owlmetry configuration failed: \(error)")
        }
    }
    // ...
}
```

**Parameters:**
- `endpoint: String` — server URL (required)
- `apiKey: String` — client key, must start with `owl_client_` (required)
- `flushOnBackground: Bool` — auto-flush when app backgrounds (default: `true`)
- `compressionEnabled: Bool` — gzip request bodies (default: `true`)
- `networkTrackingEnabled: Bool` — auto-track URLSession HTTP requests (default: `true`)
- `consoleLogging: Bool` — print events to console/Xcode output (default: `true`)

Auto-detects: bundle ID, debug mode (`#if DEBUG`). Auto-generates: session ID (fresh each launch).

## watchOS Companion Apps

Only relevant if the project ships a watchOS app target (typically alongside an iOS counterpart). Skip this section entirely for iOS-only / macOS-only projects.

The SDK ships native watchOS 10+ support: `Owl.configure(...)`, `Owl.track(...)`, `Owl.error(...)`, attachments, attribution, everything works identically on the watch. Delivery uses a tiered pipeline (direct HTTP → WatchConnectivity → on-disk queue) so events survive the watch being suspended, out of cellular range, or far from its iPhone. Full background: https://owlmetry.com/docs/sdks/swift/watchos.

**Watch-side**: nothing special — call `Owl.configure(...)` in `@main App.init()`. The SDK auto-activates `WCSession.default` on the watch.

**iPhone-side counterpart (REQUIRED)**: the iPhone app must forward inbound WatchConnectivity payloads into the Owlmetry pipeline. The SDK never claims `WCSession.default.delegate` on iPhone — the host app owns it. Add **one line** to the existing `WCSessionDelegate.session(_:didReceiveUserInfo:)`:

```swift
import WatchConnectivity
import Owlmetry

final class PhoneSessionDelegate: NSObject, WCSessionDelegate {
    func session(
        _ session: WCSession,
        didReceiveUserInfo userInfo: [String: Any]
    ) {
        if Owl.handleWatchUserInfo(userInfo) { return }
        // ... existing handling for non-Owlmetry payloads
    }

    func session(_ session: WCSession, activationDidCompleteWith state: WCSessionActivationState, error: Error?) {}
    func sessionDidBecomeInactive(_ session: WCSession) {}
    func sessionDidDeactivate(_ session: WCSession) { WCSession.default.activate() }
}
```

If the iPhone app doesn't use WatchConnectivity for anything else, wire the minimal delegate above in `application(_:didFinishLaunchingWithOptions:)`:

```swift
private let sessionDelegate = PhoneSessionDelegate()
// in didFinishLaunching:
if WCSession.isSupported() {
    WCSession.default.delegate = sessionDelegate
    WCSession.default.activate()
}
```

**Without the one-line forward, watch events that arrive via the WC fallback never reach the server.** Direct HTTP from the watch still works (when cellular is available), but events emitted while the watch is offline and waiting to relay through the phone are silently dropped on receive.

`Owl.handleWatchUserInfo(_:)` is safe to call before `Owl.configure(...)` — pre-configure events are buffered internally and drained once configure completes. iOS may cold-launch the host app to deliver a watch payload, racing your app's configure call.

Watch events arrive on the server with `environment: "watchos"` (distinct from `ios`/`ipados`/`macos`). The app's broad `platform` stays `apple` — no separate app needed.

## User Identity (set up during initial configuration)

After adding `Owl.configure()`, find where the app handles authentication and add `Owl.setUser()` / `Owl.clearUser()`. This is part of the basic setup — do it now, before moving on to instrumentation.

Look for the auth state change handler (e.g., Firebase Auth listener, login/logout methods) and add:

```swift
// After successful login — claims all previous anonymous events for this user
Owl.setUser(userId)

// On logout — reverts to anonymous tracking
Owl.clearUser()
```

**Where to find it:** Search for login/logout methods, auth state listeners, or session management code. Look for patterns like setting a user ID on other services (crash reporting, analytics), storing auth tokens, or clearing user state. Place `Owl.setUser()` right after the user ID becomes available. Place `Owl.clearUser()` in the sign-out/logout handler.

The SDK automatically flushes buffered events before claiming identity, so anonymous events from before login are retroactively linked to the user. It also handles the "claim made while offline" case: if the claim never reached the server, the SDK retries it on the next launch once a saved user id is detected, and the server re-attributes any late-flushing anonymous events to the real user automatically — no manual retry needed.

## Next Steps — Codebase Instrumentation

Once `Owl.configure()` is in place and the project builds successfully, **you MUST stop here and ask the user** which area they'd like to instrument first — even if the user's original prompt asked you to "instrument the app." Do not proceed with any code changes until the user chooses. Present these options:

1. **Screen tracking** — Add `.owlScreen("ScreenName")` to every distinct screen in the app. This is the quickest win — automatic screen view and time-on-screen tracking with a single modifier per screen. No CLI setup needed.
2. **Event & error logging** — Audit the codebase for user actions, error handling, and key flows. Add `Owl.info()`, `Owl.warn()`, `Owl.error()` calls at meaningful points. This is SDK-only — no CLI setup required beyond what's already done.
3. **Structured metrics** — Identify operations worth measuring (data loading, image processing, etc.). Add `Owl.startOperation()` / `Owl.recordMetric()` to track durations and success rates. **Requires CLI first:** each metric slug must be defined on the server via `owlmetry metrics create` (use the `/owlmetry-cli` skill) before the SDK can emit events for it.
4. **Funnel tracking** — Identify user journeys (onboarding, checkout, key conversions). Add `Owl.step()` calls at each step to measure drop-off. **Requires CLI first:** the funnel definition (with steps and event filters) must be created via `owlmetry funnels create` (use the `/owlmetry-cli` skill) before tracking makes sense.

After the user chooses, do a thorough audit of the entire codebase to find all relevant locations, then present a summary of proposed changes before making any edits.

## Screen Tracking (`.owlScreen()`)

The SDK provides a SwiftUI view modifier that automatically tracks screen appearances and time-on-screen with zero manual event calls.

```swift
struct HomeView: View {
    var body: some View {
        VStack { ... }
            .owlScreen("Home")
    }
}

struct SettingsView: View {
    var body: some View {
        Form { ... }
            .owlScreen("Settings")
    }
}
```

**What it does automatically:**
- On appear: emits `sdk:screen_appeared` (debug level) with `screenName` set
- On disappear: emits `sdk:screen_disappeared` (debug level) with `screenName` set and `_duration_ms` attribute

Both events are debug-level and filtered out of the default production view — switch to dev data mode (or filter by level) to see screen flow. The disappear event with `_duration_ms` is the more useful signal; appear is retained so you can detect screens opened but never closed (e.g. a crash mid-screen).

**Where to place it:** Attach `.owlScreen("ScreenName")` to the outermost view of each screen — typically on the `NavigationStack`, `Form`, `ScrollView`, or root `VStack`. Use it on every distinct screen in the app. Choose names that are short, readable, and consistent (e.g., `"Home"`, `"Settings"`, `"Profile"`, `"Checkout"`).

**Prefer `.owlScreen()` over manual `Owl.info()` for screen views** — it handles both appear and disappear with duration tracking. Use manual `Owl.info()` with `screenName:` only for events within a screen (button taps, state changes), not for screen appearances themselves.

## Network Request Tracking

The SDK automatically tracks all URLSession HTTP requests made via completion handler APIs. This is **enabled by default** — no code needed beyond `Owl.configure()`. To disable:

```swift
try Owl.configure(
    endpoint: "https://ingest.owlmetry.com",
    apiKey: "owl_client_...",
    networkTrackingEnabled: false
)
```

**What it captures automatically:**
- `_http_method` — GET, POST, etc.
- `_http_url` — sanitized URL (scheme + host + path only, query params stripped for privacy)
- `_http_status` — response status code
- `_http_duration_ms` — request duration in milliseconds
- `_http_response_size` — response body size in bytes
- `_http_error` — error description (failures only)

**Log levels:** `.info` for 2xx/3xx responses, `.warn` for 4xx/5xx, `.error` for network failures (no response).

**Safety:** The SDK's own requests to the Owlmetry ingest endpoint are automatically filtered out. Query parameters are stripped from URLs to prevent accidental logging of tokens or user IDs.

**Coverage:** Tracks requests made with `URLSession.dataTask(with:completionHandler:)` (both URL and URLRequest overloads). Delegate-based and async/await requests are not tracked in this version.

## Log Events

Events are the core unit of data in Owlmetry. Use the four log levels to capture different kinds of information:

- **`info`** — normal operations worth recording: screen views, user actions, feature usage, successful completions. This is your default level.
- **`debug`** — verbose detail useful only during development: cache hits, state transitions, intermediate values. These are filtered out in production data mode.
- **`warn`** — something didn't go as expected but the app can continue: failed validation, precondition checks that fail, slow responses, fallback paths taken, deprecated API usage, missing optional data.
- **`error`** — a caught exception or hard failure inside a `do`/`catch` block: network errors, JSON decode failures, file I/O errors, keychain access failures. Reserve for actual thrown errors, not for anticipated validation outcomes.

Choose **message strings** that are specific and searchable (`"Failed to load profile image"` over `"error"`). Use `screenName` to tie events to where they happened in the UI. For the full rubric on what to log, what to attach, and what to skip, see *Instrumentation Principles* below.

`message` is silently truncated to 2000 characters; attribute values are silently truncated to 200 characters. Put long content in `attributes`, not in `message`.

```swift
// In a screen context — pass screenName to tie the event to the screen
Owl.info("User opened settings", screenName: "SettingsView")
Owl.debug("Cache hit", screenName: "HomeView", attributes: ["key": "user_prefs"])
Owl.warn("Invalid email format", screenName: "SignUpView", attributes: ["input": email])

do {
    let profile = try await api.loadProfile(id: userId)
} catch {
    // Pass the Error directly — the SDK extracts the runtime type, NSError
    // domain/code, the underlying-error chain, and the call stack. The
    // server's issue tracker uses the type as a fingerprint discriminator,
    // so a URLError and a DecodingError with the same wording stay on
    // separate issues.
    Owl.error(error, "while loading profile", screenName: "ProfileView")
}

// Outside a screen context — omit screenName entirely
Owl.info("Background sync completed", attributes: ["items": "\(count)"])

// String-only Owl.error still works for cases where you don't have an
// Error value (precondition failures, manual checks, etc).
Owl.error("Keychain returned no payload for current session")
```

All logging methods share the same signature:
```swift
Owl.info(_ message: String, screenName: String? = nil, attributes: [String: String?] = [:], attachments: [OwlAttachment]? = nil)
```

`Owl.error` is overloaded — the first argument may be a `String` (logger-style) or an `Error` (exception-style; the SDK extracts type/stack/domain/code/cause-chain into reserved `_error_*` attributes). When you have an Error value from `do/catch`, prefer the Error form — you get a richer, queryable issue and per-type fingerprinting.

**`screenName` is optional.** Only pass it when the event originates from a specific screen in the UI (e.g., a button tap handler inside a view). **Do NOT pass `screenName`** when logging from utility functions, services, managers, network layers, background tasks, or anywhere that isn't directly tied to a visible screen. Passing a fabricated or guessed screen name is worse than omitting it — it pollutes screen-level analytics.

**`attributes` accepts optional values.** A `String?` from your domain code can flow into the literal directly — `nil`-valued keys are silently dropped before the event ships, so you don't need to unwrap or build the dict conditionally:

```swift
let contractId: String? = session.draftId  // may be nil
Owl.info("Draft created", attributes: ["context": "createDraft", "contractId": contractId])
```

The same applies to every method on `Owl` and `OwlOperation` that takes an `attributes:` parameter (`info`/`debug`/`warn`/`error`, `step`, `startOperation`, `recordMetric`, `complete`/`fail`/`cancel`).

Source file, function, and line are auto-captured.

**Avoid logging PII** (emails, phone numbers, passwords) or high-frequency events (every frame, every scroll position). Focus on actions and outcomes.

## Instrumentation Principles

Before adding `info` / `warn` / `error` calls throughout the app, internalise these four rules. They turn the SDK from a logger into a queryable analytics surface.

### 1. Log outcomes, not steps

Emit **one rich event per user-meaningful outcome**, not one event per line of code. The unit is the *thing that happened* (purchase completed, photo uploaded, document opened, sign-in succeeded), not the work your code did to make it happen.

Don't narrate the action:
```swift
Button("Buy") {
    Owl.info("Buy tapped", screenName: "Paywall")
    Owl.info("Validating receipt", screenName: "Paywall")
    Owl.info("Calling StoreKit", screenName: "Paywall")
    Owl.info("Receipt validated", screenName: "Paywall")
    Owl.info("Unlocking feature", screenName: "Paywall")
}
```

One event with the full context:
```swift
Button("Buy") {
    let startedAt = Date()
    Task {
        do {
            try await store.purchase(product)
            Owl.info("Subscription purchased", screenName: "Paywall", attributes: [
                "product_id": product.id,
                "price_locale": product.priceLocale.identifier,
                "introductory_offer": product.hasIntroOffer ? "yes" : "no",
                "duration_ms": "\(Int(Date().timeIntervalSince(startedAt) * 1000))",
            ])
        } catch {
            Owl.error(error, "purchase failed", screenName: "Paywall", attributes: [
                "product_id": product.id,
            ])
        }
    }
}
```

`.owlScreen()` already gives you one event per *screen visit* with `_duration_ms` — that **is** the canonical "log the outcome, not the steps" pattern for navigation. Don't supplement it with extra `Owl.info("View appeared")` calls. Apply the same instinct to button taps, gestures, and lifecycle handlers: one event per *user decision* with all the relevant context attached.

For intermediate diagnostic signals (cache hits, state transitions, fallback decisions), use `debug` level — filtered out of production data mode automatically.

### 2. Pack attributes wide, not events deep

One event with 12 attributes beats 12 events with one attribute each. For a client-side event, think through these axes and attach whatever's relevant:

| Axis | iOS examples |
|---|---|
| **Who** | `user_id` (auto from `setUser`), `subscription_status`, `plan_tier`, `account_age_days` |
| **What** | `product_id`, `document_id`, `playlist_id`, `feature_flag`, `notification_id` |
| **Where** | `screenName` (only when the event is tied to a visible screen), navigation source (`from_screen`) |
| **How** | `entry_point` (push / deeplink / cold_start), `gesture`, `format`, `source_of_truth` (cache / network), `retry_count` |
| **How much** | `duration_ms`, `item_count`, `size_bytes`, `network_kb`, `position_seconds` |

The SDK auto-attaches a lot for free — don't re-emit any of these manually:

- Device model, OS version, locale, `_connection` (wifi / cellular / offline), `app_version`, `build_number`, `is_dev`, `environment` (ios / ipados / macos / watchos) — on every event.
- `_http_method` / `_http_url` (query-stripped) / `_http_status` / `_http_duration_ms` / `_http_response_size` / `_http_error` — auto-captured for every completion-handler `URLSession` request via `networkTrackingEnabled` (default on).
- `screenName` + `_duration_ms` — auto-captured per screen via `.owlScreen()`.

High-cardinality *attribute values* (document IDs, playlist IDs, user IDs) are a **feature**, not a smell — they let you triage a specific user's broken session from a dashboard chart. The thing to control is *event frequency* (rule 3), not value uniqueness.

### 3. Aggregate hot paths; don't log per iteration

Anywhere a callback fires repeatedly — `CADisplayLink`, `Timer.publish`, scroll observers, animation `.onChange`, `NotificationCenter` chains, batch processors — log the **outcome**, not each invocation:

```swift
// Per-frame log — thousands of events for a single scroll
.onChange(of: scrollOffset) { newOffset in
    Owl.debug("Scroll moved", attributes: ["offset": "\(newOffset)"])
}

// Meaningful endpoint — one event when the user reaches a state worth knowing
.onChange(of: scrollOffset) { newOffset in
    if newOffset > thresholdToLoadMore {
        Owl.info("Reached bottom of feed", screenName: "Feed", attributes: [
            "items_visible": "\(visibleItems.count)",
            "loaded_pages": "\(loadedPages)",
        ])
    }
}
```

Same for batch work:

```swift
// Per-item — one event per photo
for photo in pendingPhotos {
    Owl.info("Photo uploaded", attributes: ["id": photo.id])
}

// Per-session — one event for the whole upload
let startedAt = Date()
var failed = 0
for photo in pendingPhotos {
    do { try await upload(photo) }
    catch { failed += 1 }
}
Owl.info("Photo backup completed", attributes: [
    "uploaded": "\(pendingPhotos.count - failed)",
    "failed": "\(failed)",
    "duration_ms": "\(Int(Date().timeIntervalSince(startedAt) * 1000))",
])
```

Same rule for `URLSession` retry chains, Combine `.retry(n)` operators, and SwiftUI `task` blocks that poll: log the final outcome with `retry_count`, not one event per attempt.

### 4. Log, metric, or funnel — pick by the question you want answered

- **Log event** (`Owl.info` / `warn` / `error`) — *"show me individual records of a specific thing that happened."* User actions, error context, edge cases. Read on Dashboard → Events.
- **Lifecycle metric** (`Owl.startOperation` → `.complete` / `.fail` / `.cancel`) — *"show me p50/p95/p99 duration and success rate of this operation over time."* Photo uploads, image processing, model loads, API round-trips. Requires server-side metric definition. Read on Dashboard → Metrics.
- **Single-shot metric** (`Owl.recordMetric`) — *"show me this point-in-time value trended."* Cold-start time, items in cart, memory usage at a checkpoint.
- **Funnel** (`Owl.step`) — *"show me where users drop off across this multi-step flow."* Onboarding, checkout, key conversions. Requires server-side funnel definition.
- **Screen view** (`.owlScreen("Name")`) — *"show me which screens are most/least visited and time-on-screen distributions."* One modifier per screen; covers appear + disappear + duration with zero manual calls. **Always prefer this over a manual `Owl.info("Screen viewed")`.**

The same flow can warrant several: an onboarding sequence gets `.owlScreen()` on each step, a funnel for conversion across the sequence, a lifecycle metric on the longest single step (e.g. "create-account"), and `Owl.error` on any catch path for triage.

> When in doubt, write one event with more attributes rather than several events with fewer.

## File Attachments (use sparingly)

When an error cannot be reproduced without the original input bytes — a media conversion that failed on a specific image, a 3D model that failed to parse, a document that failed to decode — you can attach the file to the error event. The attachment appears on the resulting issue in the dashboard, CLI, and MCP so an engineer can download and reproduce.

```swift
do {
    try await PhotoConverter.convert(inputURL: url)
} catch {
    Owl.error(
        "image conversion failed",
        screenName: "PhotoConverterView",
        attributes: ["stage": "decode", "error": "\(error)"],
        attachments: [
            OwlAttachment(fileURL: url),                                      // from disk
            OwlAttachment(data: debugJSON, name: "debug.json",
                          contentType: "application/json"),                  // in memory
        ]
    )
}
```

**Attachments are a limited resource.** Each project has a storage quota (default **5 GB**) and each end-user has their own bucket within that project (default **250 MB per user** — the SDK automatically tags uploads with the currently identified `Owl.userId`). Before adding `attachments:` anywhere, make sure the file's bytes are *essential* to reproduce the bug. Good candidates:

- ✅ A failed media conversion where only the input bytes can reproduce the decoder bug.
- ✅ A 3D model / document parse failure where the file format itself is the suspect.
- ✅ A CoreML or similar blob that fails to load at runtime.

Bad candidates — do not attach:

- ❌ Every error. Routine failures (network timeouts, validation) already have enough detail in `attributes`.
- ❌ Files you can reconstruct from event attributes alone (URLs, IDs, small config).
- ❌ Large asset files that are *downloaded* rather than user-supplied — include the source URL instead.
- ❌ Screens or UI state. Use `screenName` and `attributes` for that.

Upload behaviour is strictly non-fatal: if the device is offline, the user's per-user bucket or the project quota is exhausted, or the server otherwise rejects the file, the event itself still posts — the attachment is dropped silently and a warning is logged via `OSLog`. Uploads run on a separate serial queue so a 200 MB file never blocks event batching. There is no offline queue for attachments in v1: if the device is offline when the error fires, the attachment is discarded but the event queues normally.

## User Identity

Identity connects events to real users. Before `setUser()` is called, all events are tagged with an anonymous ID (`owl_anon_...`). After login, calling `setUser()` does two things:

1. Tags all future events with the real user ID.
2. Retroactively claims all previous anonymous events for that user (server-side), so you get a complete history.

Call `setUser()` right after successful authentication. Call `clearUser()` on logout to revert to anonymous tracking.

```swift
// After login — claims all previous anonymous events
Owl.setUser("user_123")

// On logout — reverts to anonymous tracking
Owl.clearUser()

// On logout with fresh anonymous ID
Owl.clearUser(newAnonymousId: true)
```

Read the current user id (real if `setUser` was called, otherwise the anonymous device id; `nil` before `configure()`) via `Owl.currentUserId: String?` — useful for wiring other SDKs or surfacing the id in debug UI.

**Important:** The SDK automatically flushes buffered events before claiming identity. If a previous `setUser()` call failed to reach the server (e.g. the device was offline), the SDK re-issues the claim on the next launch, and any anonymous events that only flush after the claim are still re-attributed to the real user automatically. `Owl.setUser(id)` just works once the device gets network.

## Funnel Tracking

Funnels measure how users progress through a multi-step flow (onboarding, checkout, activation) and where they drop off. The system has three parts:

1. **Define** the funnel server-side (via CLI or API) with ordered steps and event filters.
2. **Record** steps client-side with `Owl.step("step-name")`.
3. **Query** analytics to see conversion rates and drop-off between steps.

The step name you pass to `Owl.step()` must match the `step_name` in the funnel definition's `event_filter`. For example, if the step filter is `{"step_name": "welcome-screen"}`, then call `Owl.step("welcome-screen")`.

**Funnel design rules:**
- Each step must be a point that **every user in the funnel passes through** on the way to the goal. If a step is conditional (e.g., paywall only shown to free users), it breaks the chain — users who skip it show as 0% conversion from that point.
- Keep funnels focused on **one flow**. Don't combine "import a model" + "explore features" into one funnel — those are separate journeys with separate goals.
- **Optional interactions are not steps.** Toggling a setting, viewing info, or using a tool are engagement events (log with `Owl.info()`), not funnel progression. A funnel step should represent the user moving closer to the goal.
- Split alternative paths into **separate funnels**. If users can take a screenshot OR record a video, create two funnels — don't put both paths in one.
- Aim for **3-6 steps** per funnel. Too few = no drop-off insight. Too many = noise.

Use `attributes` when you need to segment funnel analytics later (e.g., by signup method or referral source).

```swift
Owl.step("welcome-screen")
Owl.step("create-account", attributes: ["method": "email"])
Owl.step("complete-profile")
Owl.step("first-post")
```

Define matching funnel definitions via `/owlmetry-cli`:
```bash
# Write steps to a JSON file (avoids shell quoting issues)
cat > /tmp/funnel-steps.json << 'EOF'
[
  {"name": "Welcome", "event_filter": {"step_name": "welcome-screen"}},
  {"name": "Account", "event_filter": {"step_name": "create-account"}},
  {"name": "Profile", "event_filter": {"step_name": "complete-profile"}},
  {"name": "First Post", "event_filter": {"step_name": "first-post"}}
]
EOF

owlmetry funnels create --project-id <id> --name "Onboarding" --slug onboarding \
  --steps-file /tmp/funnel-steps.json --format json
```

## Structured Metrics

Use structured metrics instead of plain log events when you want aggregated statistics (averages, percentiles, error rates) rather than just a list of individual events. Metrics give you `p50`, `p95`, `p99` latencies, success/failure rates, and trend data over time.

**Decision: lifecycle vs single-shot:**
- **Lifecycle** — when you're measuring something with a duration (start → end). Examples: image upload, API call, video encoding, onboarding flow. The SDK auto-tracks `duration_ms`.
- **Single-shot** — when you're recording a point-in-time value. Examples: app cold-start time, memory usage, items in cart at checkout.

The metric definition must exist on the server **before** the SDK emits events for that slug. Create it via CLI first.

### Lifecycle operations (start → complete/fail/cancel)

```swift
let op = Owl.startOperation("photo-upload", attributes: ["format": "heic"])

// On success:
op.complete(attributes: ["size_bytes": "524288"])

// On failure:
op.fail(error: "timeout", attributes: ["retry_count": "3"])

// On cancellation:
op.cancel(attributes: ["reason": "user_cancelled"])
```

`duration_ms` and `tracking_id` (UUID) are auto-added.

**Rules for lifecycle operations:**

- **Every `startOperation()` must end** with exactly one `.complete()`, `.fail()`, or `.cancel()`. An operation that starts but never ends creates orphaned metric data with no duration.
- **`.complete()`** — the operation succeeded and produced its intended result.
- **`.fail(error:)`** — the operation attempted work but encountered an error.
- **`.cancel()`** — the operation was intentionally stopped before completion (user cancelled, view disappeared, became irrelevant).
- **Don't start for no-ops** — if the operation is skipped entirely (cache hit, dedup, precondition not met), don't call `startOperation()` at all. Only start when actual work begins.
- **Don't track duration manually** — `duration_ms` is auto-calculated from start to complete/fail/cancel. Never pass a manual duration attribute.
- **Long-lived operations** — if the operation outlives the scope where it was started (e.g., recording that spans a view lifecycle), store the `OwlOperation` handle as a property. Cancel it on cleanup (`.onDisappear`, `deinit`) if it hasn't ended yet:

```swift
// Store handle for operations that span a lifecycle
@State private var recordingOp: OwlOperation?

func startRecording() {
    recordingOp = Owl.startOperation("video-recording")
    // ... begin recording
}

func stopRecording(url: URL) {
    recordingOp?.complete(attributes: ["format": "mp4"])
    recordingOp = nil
}

func onError(_ error: Error) {
    recordingOp?.fail(error: error.localizedDescription)
    recordingOp = nil
}

// Safety net: cancel if view disappears mid-operation
.onDisappear {
    recordingOp?.cancel()
    recordingOp = nil
}
```

Create the metric definition first:
```bash
owlmetry metrics create --project-id <id> --name "Photo Upload" --slug photo-upload --lifecycle --format json
```

### Single-shot measurements

```swift
Owl.recordMetric("app-cold-start", attributes: ["screen": "home"])
```

**Slug rules:** lowercase letters, numbers, hyphens only. Invalid slugs are auto-corrected with a console warning.

## User Properties

Attach custom key-value metadata to the current user. Properties are merged server-side — existing keys not in your call are preserved.

```swift
Owl.setUserProperties([
    "plan": "premium",
    "org": "acme",
])
```

Set a value to `""` to delete a key. All values must be strings. Max 50 properties per user, 50-char keys, 200-char values.

Properties follow the current user identity. If the user is anonymous, properties are set on the anonymous user and merged into the real user on `Owl.setUser()`.

Use for user-level data that changes infrequently (subscription status, plan tier, company). For event-specific data, use `attributes` on events instead.

**RevenueCat integration prompt** — copy-paste to set up subscription tracking:

```
Connect RevenueCat to my Owlmetry project so I can see paid vs free users:

1. Use `/owlmetry-cli` to add the RevenueCat integration with my RC V2 secret API key
   (needs Customer information → Read only AND Project configuration → Read only at the section level, everything else No access).
2. Show me the webhook setup values from the output so I can paste them into RevenueCat.
3. After I confirm the webhook is live, run a bulk sync to backfill existing subscribers.
4. Add Owl.setUserProperties() calls in my RevenueCat Purchases delegate or
   StoreKit transaction handler so the dashboard updates immediately when a user
   subscribes, without waiting for RevenueCat's webhook.
```

## Apple Search Ads Attribution

Owlmetry auto-captures Apple Search Ads attribution on `Owl.configure()` — no code required. On the first launch after install, the SDK calls `AAAttribution.attributionToken()` (iOS 14.3+) in a background task, submits the token to Owlmetry, and the server resolves it with Apple's public Attribution API. Once captured (attributed or not), the result is cached per-install so it runs exactly once.

On successful attribution the user picks up:
- `attribution_source = "apple_search_ads"` (cross-network — future Meta/Google support writes `meta`/`google_ads` into the same key)
- `asa_campaign_id`, `asa_ad_group_id`, `asa_keyword_id`, `asa_claim_type`, `asa_ad_id`, `asa_creative_set_id`

When Apple's AdServices API returns its deliberate non-production fixture (same numeric ID across campaign, ad group, and ad — structurally impossible from real Apple data), the user gets `attribution_source = "apple_test_install"` with **no** `asa_*` fields. This fires for **TestFlight builds, Xcode-deployed dev builds on real devices, and the iOS simulator** (Apple Forum #66161). Filter these out of acquisition dashboards alongside organic installs (`attribution_source IN ('none', 'apple_test_install')`), or treat them as a separate badge to spot a developer's own install showing up.

An install Apple did not attribute gets `attribution_source = "none"` and nothing else.

Human-readable names (`asa_campaign_name`, `asa_ad_group_name`, `asa_keyword`, `asa_ad_name`) come from either the **Apple Search Ads integration** (first-party; Owlmetry resolves every attributed user via Apple's Campaign Management API — configured per-project in the dashboard with OAuth credentials) or the **RevenueCat integration** (RC resolves the same names server-side and exposes them as subscriber attributes; bulk sync enriches every attributed user in RC, while RC's webhook delivery only fires on subscription events so free users only get enrichment through sync). Both sources are per-field merged — they never overwrite each other or the numeric IDs the SDK writes.

**Opt-out:** set `attributionEnabled: false` when configuring:

```swift
try Owl.configure(
    endpoint: "https://api.owlmetry.com",
    apiKey: "owl_client_…",
    attributionEnabled: false
)
```

**Manual submission:** apps that run their own token fetch can hand the token off to Owlmetry:

```swift
await Owl.sendAppleSearchAdsAttributionToken(myCapturedToken)
```

Normal apps should not need to call this — the auto-capture on `configure()` covers it.

**Dev-only reset:** call `Owl.resetAppleSearchAdsAttributionCapture()` to clear the per-install captured flag so the next `Owl.configure()` re-attempts capture — intended for development builds and UI tests, not production.

**Privacy notes:**
- AAAttribution is first-party and does **not** require App Tracking Transparency (no ATT prompt).
- No privacy manifest entries are required for token retrieval.
- Apple's attribution record may take up to ~24h to populate after install. The SDK retries across launches and gives up after 5 pending responses (writes `attribution_source = "none"`).
- In the iOS simulator `AAAttribution.attributionToken()` throws `platformNotSupported` — set `OWLMETRY_MOCK_ADSERVICES_TOKEN` in the scheme's environment (DEBUG only) to mock a token.

**Debug via Owlmetry itself:** every capture attempt emits an `sdk:attribution_capture` event so you can see success/fail from the dashboard without attaching a debugger. Attributes:

| `_outcome` | Level | Extra attributes | When |
|---|---|---|---|
| `success` | info | `_attribution_source` (`apple_search_ads` \| `none` \| `apple_test_install`) | Apple returned a decisive answer |
| `pending` | info | `_attempt`, `_max_attempts` | Apple 404 — record not ready, will retry next launch |
| `gave_up` | warn | `_attempts` | Hit the 5-pending cap; wrote `attribution_source = "none"` |
| `token_fetch_failed` | warn | `_error` | `AAAttribution.attributionToken()` threw on-device |
| `invalid_token` | warn | — | Owlmetry rejected the token as empty/malformed; never retried. Common steady state in regions where AdServices is unavailable. |
| `transport_failure` | error | — | Owlmetry POST failed after all transport retries |

Filter the dashboard Events list by `sdk:attribution_capture` (and optionally level) to spot install cohorts where capture never succeeds.

## Collect User Feedback

Owlmetry ships a reusable SwiftUI view (`OwlFeedbackView`) plus a programmatic API (`Owl.sendFeedback`) for gathering free-text feedback inside your app. Submissions are linked automatically to the current session, user id, app version, device, and environment — nothing extra to pass in.

### Programmatic submit

```swift
do {
    let receipt = try await Owl.sendFeedback(
        message: "Love the new import flow!",
        name: currentUser?.displayName,   // optional
        email: currentUser?.email         // optional
    )
    print("Feedback stored: \(receipt.id)")
} catch let error as OwlFeedbackError {
    // .emptyMessage, .notConfigured, .serverError, .transportFailure
    showAlert(error.localizedDescription)
}
```

Unlike events, `sendFeedback` is **synchronous** and **not offline-queued** — it returns a receipt on success and throws on failure so you can surface a retry.

### Drop-in SwiftUI view

`OwlFeedbackView` is a plain `View` — host it however you want: as a sheet, a navigation destination, or inline. Every user-facing string is localizable and overridable via `OwlFeedbackStrings`.

```swift
// 1. As a sheet
.sheet(isPresented: $showFeedback) {
    NavigationStack {
        OwlFeedbackView(
            name: user?.displayName,
            email: user?.email,
            onSubmitted: { _ in showFeedback = false },
            onCancel: { showFeedback = false }
        )
        .navigationTitle("Feedback")
    }
}

// 2. As a navigation destination
NavigationLink("Send feedback") {
    OwlFeedbackView()
}

// 3. Embedded inline
VStack {
    Text("Tell us what you think")
    OwlFeedbackView(showsContactFields: false)
}
```

### Customizing strings / localization

`OwlFeedbackView` ships with English defaults bundled in the SDK. To override a single string or swap in your own localized catalog:

```swift
// Partial override
OwlFeedbackView(strings: .default.with(header: "How are we doing?"))

// Point at your app's own string catalog
OwlFeedbackView(strings: OwlFeedbackStrings(
    header: LocalizedStringResource("feedback.header", table: "MyApp"),
    // …other fields keep defaults
))
```

### Theming the Submit button

Both the toolbar confirm action and the inline `.borderedProminent` Send button read the SwiftUI environment tint. Override it with `.tint()`:

```swift
OwlFeedbackView(onSubmitted: { _ in }, onCancel: {})
    .tint(.orange)
```

If you don't apply `.tint()` explicitly, the view inherits the enclosing `NavigationStack`'s tint (or your app's accent color), so in most apps it already matches your brand without any extra code. The `.tint()` modifier propagates through the environment, so you can apply it on a parent view (e.g. at the top of your `WindowGroup`) and every `OwlFeedbackView` downstream will pick it up.

### Where feedback lands

Submissions show up on **Dashboard → Feedback** as a kanban with four statuses (`new → in_review → addressed → dismissed`). Humans, the CLI (`owlmetry feedback`), and MCP agents can all read and triage it.

## Collect Structured Surveys (Questionnaires)

Questionnaires are short multi-question surveys (text / single-choice / multi-choice / 1–5 rating / 0–10 NPS) shown in-app via a SwiftUI view modifier. Where `OwlFeedbackView` collects one free-text message, an `OwlQuestionnaireView` walks the user through a typed schema you author once on the server.

### Prerequisite — create the questionnaire first

The Swift SDK only reads and submits — it does **not** define questionnaires. Create one before instrumenting the app:

- Dashboard → Questionnaires → **New questionnaire** (fill slug + name + schema JSON)
- CLI: `owlmetry questionnaires create --project-id <id> --slug post-onboarding --name "Onboarding survey" --schema-file ./schema.json`
- MCP: the `create-questionnaire` tool (agents can author short market-research surveys autonomously)

Slug is immutable after creation — pick the SDK call-site name carefully (`post-onboarding`, `weekly-checkin`, etc.).

#### What end-users see vs what's internal

**Critical for agents authoring questionnaires:** most of the spec is shown to end-users in the in-app sheet. Treat every text field below as user-facing copy unless explicitly marked internal.

| Field | Where it shows | Visibility |
|---|---|---|
| `slug` | SDK call-site only (`.owlQuestionnaire(slug: "…")`) | 🔒 internal |
| `name` | Dashboard tables + team email/in-app notifications (`questionnaire.response_new`) | 🔒 internal |
| `description` | **In-app consent sheet body** under "Quick favor?" — replaces the default `consentBody` string when non-empty | 👁️ user-facing |
| `schema.questions[].title` | Large header on the question page | 👁️ user-facing |
| `schema.questions[].subtitle` | Secondary text under the question title | 👁️ user-facing |
| `schema.questions[].placeholder` | Text-field placeholder | 👁️ user-facing |
| `schema.questions[].options[].label` | Single/multi-choice row label | 👁️ user-facing |
| `schema.questions[].id` | Wire format only (becomes the analytics column) | 🔒 internal |
| `is_active` | Gates whether the SDK can fetch it | 🔒 internal |
| `app_id` | Pins the questionnaire to a single app | 🔒 internal |

⚠️ **Common pitfall**: `description` is *not* a private note for the team — it renders verbatim in the consent prompt. If you want a hidden author note, put it in your own ticket/doc system; the spec has no internal-only text field. If you fill `description` with `"draft — confirm wording w/ marketing"`, that exact string ends up under "Quick favor?" in production.

### Question types — short vs long text

`text` questions default to a single-line `TextField`. Set `"multiline": true` on the question to render a tall, rounded `TextEditor` (~5 lines, grows with input) — reach for it whenever the expected answer is a sentence or more, not just a phrase:

```jsonc
{ "id": "q_takeaway", "type": "text", "title": "One-line takeaway?", "placeholder": "Optional" }
{ "id": "q_details",  "type": "text", "title": "Anything else?",      "multiline": true,
  "placeholder": "Optional — tell us as much as you'd like" }
```

Both share the same 4000-character server cap. Choose `multiline` based on the answer shape you want, not the field length.

### Auto-trigger view modifier (primary path)

```swift
import SwiftUI
import Owlmetry

struct RootView: View {
    var body: some View {
        TabView { ... }
            .owlQuestionnaire(
                slug: "post-onboarding",
                trigger: .afterLaunches(3)
            )
    }
}
```

#### Parameter reference

Every parameter on `.owlQuestionnaire(...)` and `OwlQuestionnaireView`. 👁️ = produces user-facing UI; 🔒 = behaviour/gating only.

| Parameter | Default | What it does |
|---|---|---|
| `slug` | required | 🔒 Looks up the server-side spec. Must match the `slug` you used in `create-questionnaire`. |
| `trigger` | `.afterLaunch` | 🔒 When to evaluate. See "Composable triggers" — `.afterLaunch`, `.afterLaunches(n)`, `.when(...)`, `.manual`. ANDed conditions. |
| `showsConsent` | `true` | 👁️ When `true`, opens with the "Quick favor?" consent detent (Sure / Maybe later / Don't ask again). When `false`, jumps straight to question 1. |
| `consentIcon` | `Image(systemName: "quote.bubble.fill")` | 👁️ Small icon at the top-left of the consent sheet. Pass `nil` to hide it. Pass any `Image` — typically a custom SF Symbol that matches the survey's tone. Ignored when `showsConsent: false`. |
| `isEligible` | `nil` | 🔒 Sync closure on main thread; return `false` to skip. Use for app-side gating (paid status, feature flags). Re-evaluates on every foreground. |
| `forceShow` | `false` | 🔒 Debug-only override that bypasses every local + most server gates (still respects `inactive`). Wire to a debug-menu toggle or `#if DEBUG`. Re-presents within the same session. |
| `tint` | `nil` | 👁️ Accent colour for the consent accept button, progress bar, rating stars, NPS chips, choice selection, Submit/Next/Done. `nil` inherits the parent's accent. |
| `strings` | `.default` | 👁️ Override every consent + flow string via `OwlQuestionnaireStrings.default.with(...)`. See "Custom consent copy" — but note `description` on the spec wins over `consentBody` when present. |
| `onSubmitted` | `nil` | 🔒 Fires once on the call that flips the response to submitted. Receives `OwlQuestionnaireReceipt` with `id`, `createdAt`, `wasSubmitted`. |
| `onCancel` | `nil` | 🔒 Fires when the user taps Maybe later, Cancel, or otherwise dismisses without submitting. |
| `onDismissed` | `nil` | 🔒 Fires when the user taps Don't ask again and confirms the global opt-out. |

When the trigger fires AND the user is server-side eligible (not already responded, not globally dismissed), the SDK opens a **non-swipe-dismissible** sheet at a small "Quick favor?" consent detent with three buttons: **Sure, happy to help** (expands to the full step flow), **Maybe later** (closes; re-evaluates next foreground), and **Don't ask again** (writes the global dismiss flag after a confirmation). Once accepted, questions render one-per-page with a top progress bar and Back / Next / Submit; on submit, an in-sheet success page (✓ + Thanks + Done) replaces the questions.

**Progressive saves + resume** — answers persist to the server on every Next tap, not just on Submit. If the user quits mid-flow, the next eligible launch's auto-trigger picks the draft up automatically: the SDK fetches the spec, sees the saved draft, skips the consent detent, and lands the user at the first unanswered question with prior answers pre-filled. No extra code on your side. Drafts survive process restarts and reinstalls (keyed server-side on user_id), and the team `questionnaire.response_new` notification only fires on the final Submit — partial saves are silent.

Pass `showsConsent: false` to skip the consent prompt and go straight to the step flow:

```swift
.owlQuestionnaire(slug: "...", trigger: .afterLaunch, showsConsent: false)
```

### Composable triggers (ANDed conditions)

```swift
.owlQuestionnaire(
    slug: "weekly-checkin",
    trigger: .when(
        .launches(atLeast: 3),
        .daysSinceFirstLaunch(atLeast: 7)
    )
)
```

Conditions: `.launches(atLeast:)`, `.foregrounds(atLeast:)`, `.daysSinceFirstLaunch(atLeast:)`, `.hoursSinceFirstLaunch(atLeast:)`. Shortcuts: `.afterLaunch`, `.afterLaunches(n)`, `.manual` (never auto-trigger).

OR-logic isn't built in. Use `isEligible: { ... }` for custom gating, or attach two modifiers with different slugs.

### Gating (free vs paid, feature flags, etc.)

```swift
.owlQuestionnaire(
    slug: "free-user-survey",
    trigger: .afterLaunch,
    isEligible: { !user.isPaid }
)
```

`isEligible` runs synchronously on the main thread before the SDK fetches the spec. Return `false` to skip this evaluation; it re-runs on the next foreground.

### Previewing the questionnaire UI in debug

Pass `forceShow: true` to bypass every local gate (trigger conditions, `isEligible`, per-process dedup) and ask the server to also ignore `alreadyResponded` and `globallyDismissed`. `inactive` is still respected. Lets a developer see the sheet without rigging up launch counts or resetting state. Wire it to a debug-menu toggle, env var, or `#if DEBUG` flag — defaults to `false`, so production users never trip it.

```swift
.owlQuestionnaire(
    slug: "post-onboarding",
    trigger: .when(.daysSinceFirstLaunch(atLeast: 7)),
    forceShow: previewQuestionnaireToggle // your @State debug toggle
)
```

Under `forceShow: true` the SDK also skips the per-process "shown" mark, so toggling off-then-on after dismissing the sheet re-presents it within the same session.

### Tint

`tint: Color?` is propagated through the sheet's environment to the consent accept button, progress bar, rating stars, NPS chips, single/multi choice selected state, and the bottom Submit / Next / Done buttons. Omit it to inherit your app's accent color.

```swift
.owlQuestionnaire(slug: "...", trigger: .afterLaunch, tint: .orange)
```

### Custom consent copy

Two places set the text the user sees on the consent sheet — and **both render to end users**, neither is internal-only:

1. **Per-questionnaire** — the spec's `description` field (set via dashboard / CLI / MCP at create or update time). When non-empty, it replaces the default consent body for that specific survey. This is the right place to write the actual pitch ("Short survey for paid-tier users — 30 seconds, helps us prioritise next month's features").
2. **App-wide** — `OwlQuestionnaireStrings` overrides every consent string at the SDK call-site. Use this to translate or rebrand the surrounding chrome (title, button labels) across all surveys.

The questionnaire's `description` always wins over `strings.consentBody` when both are set, so per-survey copy beats global defaults.

```swift
.owlQuestionnaire(
    slug: "...",
    trigger: .afterLaunch,
    strings: .default.with(
        consentTitle: "Got a minute?",
        consentAccept: "I'm in",
        consentLater: "Not right now",
        consentNever: "Stop asking me"
    )
)
```

### Manual presentation

For ad-hoc triggers ("show after the user finishes the import wizard"), fetch + present yourself. `fetchQuestionnaire` returns a result wrapping the spec + any in-progress draft for the current user:

```swift
@State private var spec: OwlQuestionnaire?
@State private var inProgress: OwlQuestionnaireDraft?
@State private var show = false

Button("Take a quick survey") {
    Task {
        let result = try? await Owl.fetchQuestionnaire(slug: "post-import")
        if let q = result?.questionnaire {
            spec = q
            inProgress = result?.inProgress  // optional draft to resume from
            show = true
        }
        // questionnaire == nil → user is ineligible (already responded /
        // globally dismissed / inactive). result.ineligibleReason carries
        // the specific reason for diagnostics.
    }
}
.sheet(isPresented: $show) {
    if let spec {
        NavigationStack {
            OwlQuestionnaireView(
                questionnaire: spec,
                inProgress: inProgress,                  // resumes mid-flow
                showsConsent: inProgress == nil,        // skip consent on resume
                onSubmitted: { _ in show = false },
                onCancel: { show = false }
            )
        }
    }
}
```

Answers persist per Next tap. If the user quits halfway, the next eligible launch's `fetchQuestionnaire` returns `inProgress` carrying their saved answers; pass it to `OwlQuestionnaireView` to resume at the first unanswered question with prior answers pre-filled.

### One-shot save (advanced)

For consumers building a fully custom UI on top of the SDK, `Owl.saveQuestionnaireResponse` is the underlying API. The auto modifier and `OwlQuestionnaireView` call this for you; only reach for it directly if you're rolling your own flow.

```swift
let receipt = try await Owl.saveQuestionnaireResponse(
    slug: "post-import",
    answers: ["q_text": .text("Loved it"), "q_rating": .rating(5)],
    isComplete: true       // false to save a draft, true to finalize
)
// receipt.wasSubmitted == true on the call that flipped the response to
// submitted (drives transition to your success view). Draft saves return
// wasSubmitted == false; the server merges incoming keys onto the existing
// row keyed by (project, slug, user_id) — no client-side response_id
// tracking needed.
```

### Programmatic dismissal

```swift
try await Owl.dismissQuestionnaires()
```

Globally opts the current user out of every questionnaire. Idempotent. Survives reinstall (stored server-side on `app_users.properties`).

### Where responses land

Responses show up on **Dashboard → Questionnaires → &lt;questionnaire&gt;** with pre-aggregated per-question analytics (bar charts for choices, average for ratings, NPS score for NPS). Each response stores the schema_snapshot it was submitted against, so editing the questionnaire later never retroactively breaks historical rendering. CLI: `owlmetry questionnaires …`. MCP: `list-questionnaires` / `list-questionnaire-responses` / `get-questionnaire-analytics`.

### Privacy

Same model as feedback — answers are user-typed into a dev-rendered form. No new automatic data categories beyond what the feedback flow already declares.

## What the SDK Tracks Automatically

Do not re-implement any of these — they are built into the SDK and emitted without any code:

- **`sdk:session_started`** — emitted on `Owl.configure()`; includes `_launch_ms`
- **`sdk:app_backgrounded`** / **`sdk:app_foregrounded`** — app state transitions
- **`session_id`** — fresh UUID per `configure()` call, included on every event. Readable at runtime via `Owl.sessionId` (String?). Forward it to a Node.js backend in an `X-Owl-Session-Id` request header and scope the backend handler with `Owl.withSession(...)` (Node SDK) to link client and backend events under the same session
- **`_launch_time_ms`** — app launch time, included in the `session_started` event
- **`_connection`** — network type (wifi, cellular, etc.), included on every event
- **Device model, OS version, locale** — included on every event
- **`is_dev`** — automatically `true` in DEBUG builds

You do NOT need to manually track app launch, app foreground/background, session start, network type, or device info. These are already covered.

## Instrumentation Strategy

When instrumenting a new app, follow this priority:

**Always instrument (events — no CLI setup needed):**
- Screen views (`.owlScreen("ScreenName")` on every distinct screen)
- Authentication events (login, logout, signup)
- Caught exceptions (`error` in `catch` blocks, error handlers)
- Validation failures and pre-checks (`warn` for bad input, missing optional data, fallback paths)
- Core business actions (purchase, share, create, delete)

**Instrument when relevant (metrics — requires CLI `owlmetry metrics create` first):**
- Lifecycle metrics for operations where duration matters: image uploads, API calls, data syncs, video encoding
- Single-shot metrics for point-in-time values: app cold-start time, memory usage, items in cart

**Instrument when relevant (funnels — requires CLI `owlmetry funnels create` first):**
- Multi-step flows you want to measure conversion on: onboarding, checkout, activation

**Where to place calls:**
- Screen views: `.owlScreen("Name")` on the outermost view of each screen (SwiftUI), `viewDidAppear` in UIKit
- User actions: button action handlers, gesture callbacks — pass `screenName` since you know which screen the user is on
- Errors: `catch` blocks, `Result.failure` handlers — pass `screenName` only if the error is caught inside a view; omit it if caught in a service, manager, or utility
- Services, utilities, background tasks: log freely but **never pass `screenName`** — these are not screen-bound
- Metrics: wrap the async operation between `startOperation()` and `complete()`/`fail()`

**What NOT to instrument — concrete list:**

Never put these in event messages or attribute values (the message and attribute caps don't redact, they just truncate):
- Auth tokens (Firebase ID tokens, OAuth tokens, JWT contents), Keychain payloads, biometric data
- Passwords, PIN codes, recovery phrases
- Raw card numbers / CVVs / bank account numbers / SSNs
- Full HTTP request/response bodies — the auto-network-tracking already emits `_http_url` (query-stripped), `_http_method`, `_http_status`, `_http_duration_ms`, `_http_response_size`, `_http_error`; don't supplement with payload bytes
- Personal-content text the user typed into your app (notes, messages, journal entries) — `Owl.sendFeedback` / `OwlFeedbackView` is the user-consented path for that
- Raw `error.localizedDescription` if it embeds user input — pass the `Error` to `Owl.error()` instead; the SDK extracts type, NSError domain/code, cause chain, and call stack into reserved `_error_*` attributes with proper fingerprinting

Skip — these are usually noise, not value:
- Every tap / scroll / drag / pinch — `.owlScreen()` already gives you one event per screen-visit; only log taps that represent a *user decision* (purchase, share, delete, submit)
- Frame-rate / per-tick events from CADisplayLink, Combine `.publisher(every:)`, or animation callbacks
- Per-row events inside batch import / sync loops (log the batch outcome with counts — see *Instrumentation Principles* rule 3)
- Per-attempt logs on URLSession retry chains (log the final outcome with `retry_count`)
- Sensitive business amounts on a per-event basis if user-aggregated revenue is what you want — use `setUserProperties` or the RevenueCat integration

## Lifecycle

```swift
// In your app's termination handler or ScenePhase .background
await Owl.shutdown()
```

`flushOnBackground: true` (default) handles most cases automatically. Call `shutdown()` explicitly only if you need to guarantee delivery at a specific point.

## Auto-Captured Data

Every event automatically includes:
- `session_id` — fresh UUID per `configure()` call
- Device model, OS version, locale
- `app_version`, `build_number` (from bundle)
- `is_dev` — `true` in DEBUG builds
- `_connection` — network type (wifi, cellular, ethernet, offline) via `NWPathMonitor`
- `environment` — specific runtime (ios, ipados, macos, watchos)
- `country_code` — ISO-3166 alpha-2 country, stamped server-side from the ingest request (SDK does not send this)
- `sdk_name` (`"owlmetry-swift"`) and `sdk_version` (the resolved SPM tag) — auto-stamped on every event and feedback submission. **Do not set these manually** — they're managed by the SDK so the server can tell which SDK and version produced each event.

**Auto-emitted lifecycle events** (no manual calls needed):
- `sdk:session_started` — on `configure()`, includes `_launch_ms` (time from process start to configure)
- `sdk:app_foregrounded` — when app enters foreground
- `sdk:app_backgrounded` — when app enters background
- `sdk:screen_appeared` (debug) / `sdk:screen_disappeared` (debug) — when using `.owlScreen()` modifier (disappear includes `_duration_ms`)
- `sdk:network_request` (info/warn/error) — URLSession HTTP requests with method, URL, status, duration (enabled by default, disable with `networkTrackingEnabled: false`)

## Before Shipping — Privacy & App Store Submission

Mention this to the developer once instrumentation is in place — it's a one-time submission task, not a code change.

The SDK bundles `PrivacyInfo.xcprivacy` automatically — SPM merges it into the app at build time. No code, no configuration, no ATT prompt required (Owlmetry doesn't use IDFA or link `AdSupport`).

On the **next** App Store submission, the developer must update **App Store Connect → App Privacy → Nutrition Label** to declare these collected data types: Crash Data, Other Diagnostic Data, Product Interaction, Performance Data, Other User Content, and — if the app calls `Owl.setUser` — User ID. Subsequent submissions are unchanged.

Full guide: https://owlmetry.com/docs/sdks/swift/privacy-compliance
