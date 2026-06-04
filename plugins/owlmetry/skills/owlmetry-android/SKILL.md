---
name: owlmetry-android
description: >-
  Integrate the Owlmetry Android SDK (Kotlin / Jetpack Compose) into an
  Android app for analytics, event tracking, structured metrics, funnels,
  in-app questionnaires (surveys / NPS / 1–5 ratings) with progressive
  draft saves and resume-mid-flow, a drop-in user feedback view, file
  attachments, crash and error reporting, screen tracking, and user
  identity. Use when instrumenting an Android, Kotlin, or Jetpack Compose
  project with Owlmetry, building in-app surveys, resuming half-finished
  questionnaire drafts, capturing feedback, attaching files to error
  events, or wiring login/logout identity.
allowed-tools: Read, Bash, Grep, Glob
---

## What is Owlmetry?

Owlmetry is a self-hosted analytics platform. The Android SDK captures events from Android apps (Kotlin / Jetpack Compose) and delivers them to the Owlmetry server. It handles buffering, gzip compression, offline queuing, session management, and network monitoring automatically — you just call logging methods and the SDK takes care of delivery.

The SDK is a static `Owl` object (`com.owlmetry.android.Owl`) with a single runtime dependency (`kotlinx-coroutines`, the Kotlin analog of Swift Concurrency). All calls are non-blocking (events are buffered and flushed in batches). A single `Owl.configure(...)` call initialises everything. It ships as two artifacts:

- **`com.owlmetry:owlmetry-android`** — the core SDK: analytics, metrics, funnels, identity, plus the programmatic feedback/questionnaire APIs. Framework-only + coroutines, no Compose.
- **`com.owlmetry:owlmetry-android-compose`** — optional Jetpack Compose UI: `OwlFeedbackView`, `OwlQuestionnaireView` / `OwlQuestionnaireGate`, and the `Modifier.owlScreen(...)` screen-tracking modifier. Apps that don't render the drop-in UI depend on the core alone and stay free of Compose.

It mirrors the [Owlmetry Swift SDK](https://github.com/owlmetry/owlmetry-swift) feature-for-feature, with Android-native idioms (an `Application` class instead of an iOS `@main App`, FCM / Play instead of Apple Search Ads, Compose modifiers instead of SwiftUI view modifiers).

## Version Check

Run silently. Only inform the user if an update is available.

- **SDK version**: read the pinned version from the app's Gradle config (the `com.owlmetry:owlmetry-android:X.Y.Z` coordinate in `build.gradle.kts` / `build.gradle`, or the `libs.versions.toml` version catalog), then compare against `curl -sf https://api.github.com/repos/owlmetry/owlmetry-android/releases/latest | jq -r .tag_name`. If newer, inform the user.

Skill updates arrive through Claude Code's plugin marketplace (`/plugin marketplace update owlmetry-skills`).

## Prerequisite

You need an **ingest endpoint** and a **client key** (`owl_client_...`) for an Android app. Both come from the CLI setup flow.

If the user doesn't have these yet, follow the `/owlmetry-cli` skill first — it handles sign-up, project creation, and app creation. When creating the app, the platform should be **android** so events route correctly. The ingest endpoint is saved to `~/.owlmetry/config.json` (`ingest_endpoint` field) and the client key is returned when creating an app.

> **Any time you need to run an `owlmetry` CLI command** (querying events, creating metrics/funnels, listing apps, etc.), **load the `/owlmetry-cli` skill first**. Do not guess CLI syntax — it has non-obvious subcommand patterns and flags.

## Add the Gradle Dependency

**Minimum SDK:** `minSdk 24` (Android 7.0). **Build tooling:** Kotlin 2.0+, AGP 8.7+, JDK 17+. The core module has a single runtime dependency (`kotlinx-coroutines`).

**First, fetch the latest SDK release tag** — install is pinned to a published Maven release, so pin a version for reproducible builds:

```bash
curl -sf https://api.github.com/repos/owlmetry/owlmetry-android/releases/latest | jq -r .tag_name
# e.g. "v0.1.0" → strip the leading "v" → use "0.1.0" in the snippets below.
```

If the GitHub API call fails or returns no tags (early alpha may have none published yet), use the latest version listed in the repo's `OwlmetryVersion.CURRENT` / README, or ask the user which version to pin.

### Option A — version catalog (`gradle/libs.versions.toml`, preferred)

If the project uses a version catalog (most modern Android projects do), add the version + library entries there (replace `0.1.0` with the tag you fetched above):

```toml
[versions]
owlmetry = "0.1.0"

[libraries]
owlmetry-android = { module = "com.owlmetry:owlmetry-android", version.ref = "owlmetry" }
owlmetry-android-compose = { module = "com.owlmetry:owlmetry-android-compose", version.ref = "owlmetry" }
```

Then in the app module's `build.gradle.kts`:

```kotlin
dependencies {
    implementation(libs.owlmetry.android)
    // Optional — only if you want the drop-in Compose feedback/questionnaire UI
    // and the owlScreen() modifier:
    implementation(libs.owlmetry.android.compose)
}
```

### Option B — direct coordinates

If the project declares dependencies directly (no catalog), add to the app module's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.owlmetry:owlmetry-android:0.1.0")
    // Optional Compose UI:
    implementation("com.owlmetry:owlmetry-android-compose:0.1.0")
}
```

Groovy `build.gradle` equivalent:

```groovy
dependencies {
    implementation 'com.owlmetry:owlmetry-android:0.1.0'
    implementation 'com.owlmetry:owlmetry-android-compose:0.1.0'
}
```

**Only add `owlmetry-android-compose` if the app uses Jetpack Compose** and you intend to use the `Modifier.owlScreen(...)` modifier, `OwlFeedbackView`, or `OwlQuestionnaireGate` / `OwlQuestionnaireView`. The programmatic feedback and questionnaire APIs (`Owl.sendFeedback`, `Owl.saveQuestionnaireResponse`, `Owl.fetchQuestionnaire`) live in the core module and work without Compose.

Make sure Maven Central is a configured repository (it almost always already is, in `settings.gradle.kts` under `dependencyResolutionManagement { repositories { mavenCentral() } }`).

## Verify Dependency Integration

After editing the Gradle files, sync + build:

```bash
./gradlew :app:assembleDebug --quiet
# or, to just resolve dependencies without a full build:
./gradlew :app:dependencies --configuration debugRuntimeClasspath | grep owlmetry
```

Adjust `:app` to the actual application module name if it differs. If the build resolves `com.owlmetry:owlmetry-android`, proceed with configuration. Unresolved-import warnings in the editor before the first Gradle sync are expected.

## Configure

Configuration must happen once, as early as possible — in your `Application` subclass's `onCreate()`. **Do not defer it** to a later point (e.g., the first `Activity`, after async setup, or after user consent). The SDK measures app launch time (`_launch_ms`) from process start to the `configure()` call, so placing it early gives an accurate cold-start metric. It also ensures no events are dropped before configuration. Each `configure()` call generates a fresh `session_id` (UUID) that groups all subsequent events together.

If the project doesn't already have an `Application` subclass, create one and register it in the manifest (`<application android:name=".MyApp" ...>`):

```kotlin
import android.app.Application
import com.owlmetry.android.Owl
import com.owlmetry.android.OwlConfigurationError

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        try {
            Owl.configure(
                context = this,
                endpoint = "https://ingest.owlmetry.com",
                apiKey = "owl_client_...",
            )
        } catch (e: OwlConfigurationError) {
            android.util.Log.e("Owlmetry", "configuration failed", e)
        }
    }
}
```

And in `AndroidManifest.xml`:

```xml
<application
    android:name=".MyApp"
    ... >
```

**Parameters:**
- `context: Context` — an Android `Context` (required; `configure` uses `applicationContext` internally, so passing `this` from `Application` is correct)
- `endpoint: String` — server URL (required)
- `apiKey: String` — client key, must start with `owl_client_` (required)
- `flushOnBackground: Boolean` — auto-flush when the app backgrounds (default: `true`)
- `compressionEnabled: Boolean` — gzip request bodies (default: `true`)
- `networkTrackingEnabled: Boolean` — reserved for auto-tracking HTTP requests (default: `true`)
- `consoleLogging: Boolean` — echo events to Logcat (default: `true`)
- `attributionEnabled: Boolean` — install-attribution capture (default: `true`)

`configure` throws `OwlConfigurationError` on an invalid endpoint, an API key that doesn't start with `owl_client_`, or a missing bundle/package id — wrap it in `try`/`catch` as shown.

Auto-detects: package id (the app's `applicationId`), `is_dev` (true when the host app is debuggable — i.e. debug builds — false for release). Auto-generates: session ID (fresh each launch).

> **No Apple Search Ads on Android.** The Swift SDK auto-captures Apple Search Ads attribution; the Android SDK has no ASA equivalent. Install attribution on Android comes from Google Play / Play Install Referrer and is out of scope for v1 — the `attributionEnabled` flag is reserved. Likewise, **push registration on Android is Firebase Cloud Messaging (FCM)**, not APNs; the SDK does not register FCM tokens for you.

## User Identity (set up during initial configuration)

After adding `Owl.configure()`, find where the app handles authentication and add `Owl.setUser()` / `Owl.clearUser()`. This is part of the basic setup — do it now, before moving on to instrumentation.

Look for the auth state change handler (e.g., a Firebase Auth listener, a login/logout method, a sign-in `ViewModel`) and add:

```kotlin
// After successful login — claims all previous anonymous events for this user
Owl.setUser(userId)

// On logout — reverts to anonymous tracking
Owl.clearUser()

// On logout from a shared device — also mint a fresh anonymous id
Owl.clearUser(newAnonymousId = true)
```

**Where to find it:** search for login/logout methods, auth-state listeners, or session management code. Look for patterns like setting a user ID on other services (crash reporting, analytics), storing auth tokens, or clearing user state. Place `Owl.setUser()` right after the user ID becomes available; place `Owl.clearUser()` in the sign-out/logout handler.

The SDK automatically flushes buffered events before claiming identity, so anonymous events from before login are retroactively linked to the user. It also handles the "claim made while offline" case: if the claim never reached the server, the SDK re-issues it on the next launch once a saved user id is detected, and the server re-attributes any late-flushing anonymous events to the real user automatically — no manual retry needed. `setUser` / `clearUser` are safe to call before `configure()` too: the op is stashed and persisted at the next `configure()`.

Read the current user id (the real id if `setUser` was called, otherwise the persistent anonymous device id; `null` before `configure()`) via `Owl.currentUserId: String?`.

## Next Steps — Codebase Instrumentation

Once `Owl.configure()` is in place and the project builds successfully, **you MUST stop here and ask the user** which area they'd like to instrument first — even if the user's original prompt asked you to "instrument the app." Do not proceed with any code changes until the user chooses.

Use the **AskUserQuestion** tool to present the choice. Offer these labeled options (multi-select where the user may want several):

- **Screen tracking** — add `Modifier.owlScreen("ScreenName")` to every distinct Compose screen. The quickest win: automatic screen-view and time-on-screen tracking with one modifier per screen. No CLI setup needed. *(Requires the `owlmetry-android-compose` artifact.)*
- **Event & error logging** — audit the codebase for user actions, error handling, and key flows; add `Owl.info()` / `Owl.warn()` / `Owl.error()` calls at meaningful points. SDK-only, no CLI setup beyond what's already done.
- **Structured metrics** — identify operations worth measuring (data loading, image processing, API round-trips); add `Owl.startOperation()` / `Owl.recordMetric()` to track durations and success rates. **Requires CLI first:** each metric slug must be defined on the server via `owlmetry metrics create` (use the `/owlmetry-cli` skill) before the SDK emits events for it.
- **Funnel tracking** — identify user journeys (onboarding, checkout, key conversions); add `Owl.step()` calls at each step to measure drop-off. **Requires CLI first:** the funnel definition (steps + event filters) must be created via `owlmetry funnels create` (use the `/owlmetry-cli` skill) before tracking makes sense.
- **In-app feedback** — drop in `OwlFeedbackView` (or call `Owl.sendFeedback` programmatically) to collect free-text feedback. *(Compose view requires the `owlmetry-android-compose` artifact.)*
- **In-app questionnaires / NPS** — wire `OwlQuestionnaireGate` to auto-present a survey when a trigger fires. **Requires the questionnaire to exist server-side first** (dashboard / CLI / MCP). *(Requires the `owlmetry-android-compose` artifact.)*

After the user chooses, do a thorough audit of the entire codebase to find all relevant locations, then present a summary of proposed changes before making any edits.

## Screen Tracking (`Modifier.owlScreen()`)

The Compose UI module provides a `Modifier` extension that automatically tracks screen appearances and time-on-screen with zero manual event calls — the Android analog of SwiftUI's `.owlScreen(_:)`.

```kotlin
import com.owlmetry.android.compose.owlScreen

@Composable
fun HomeScreen() {
    Column(modifier = Modifier.owlScreen("Home")) { /* ... */ }
}

@Composable
fun SettingsScreen() {
    LazyColumn(modifier = Modifier.owlScreen("Settings")) { /* ... */ }
}
```

**What it does automatically** (keyed on composition lifetime via `DisposableEffect`):
- On enter: emits `sdk:screen_appeared` (debug level) with `screenName` set.
- On exit: emits `sdk:screen_disappeared` (debug level) with `screenName` set and a `_duration_ms` attribute (uptime-monotonic clock, immune to wall-clock changes).

Both events are debug-level and filtered out of the default production view — switch to dev data mode (or filter by level) to see screen flow. The disappear event with `_duration_ms` is the more useful signal; appear is retained so you can detect screens opened but never closed (e.g. a crash mid-screen).

**Where to place it:** attach `Modifier.owlScreen("ScreenName")` to the outermost composable of each screen — typically the root `Column`, `Scaffold` content, `LazyColumn`, or `Box`. Use it on every distinct screen. Choose names that are short, readable, and consistent (`"Home"`, `"Settings"`, `"Profile"`, `"Checkout"`).

**Prefer `Modifier.owlScreen()` over a manual `Owl.info()` for screen views** — it handles both appear and disappear with duration tracking. Use manual `Owl.info()` with `screenName` only for events *within* a screen (button taps, state changes), not for the screen appearance itself.

## Log Events

Events are the core unit of data in Owlmetry. Use the four log levels to capture different kinds of information:

- **`info`** — normal operations worth recording: screen views, user actions, feature usage, successful completions. Your default level.
- **`debug`** — verbose detail useful only during development: cache hits, state transitions, intermediate values. Filtered out in production data mode.
- **`warn`** — something didn't go as expected but the app can continue: failed validation, precondition checks that fail, slow responses, fallback paths, missing optional data.
- **`error`** — a caught exception or hard failure inside a `try`/`catch`: network errors, JSON decode failures, file I/O errors. Reserve for actual thrown errors, not anticipated validation outcomes.

Choose **message strings** that are specific and searchable (`"Failed to load profile image"` over `"error"`). Use `screenName` to tie events to where they happened in the UI.

`message` is silently truncated; attribute values are silently truncated (~200 chars). Put long content in `attributes`, not in `message`.

```kotlin
import com.owlmetry.android.Owl

// In a screen context — pass screenName to tie the event to the screen
Owl.info("User opened settings", screenName = "SettingsScreen")
Owl.debug("Cache hit", screenName = "HomeScreen", attributes = mapOf("key" to "user_prefs"))
Owl.warn("Invalid email format", screenName = "SignUpScreen", attributes = mapOf("input" to email))

try {
    val profile = api.loadProfile(userId)
} catch (e: Throwable) {
    // Pass the Throwable directly — the SDK extracts the runtime type, the JVM
    // stack trace, and the Throwable.cause chain (up to 5 deep) into reserved
    // `_error_*` attributes. The server's issue tracker uses `_error_type` as a
    // fingerprint discriminator, so an IOException and a JSONException with the
    // same wording stay on separate issues.
    Owl.error(e, "while loading profile", screenName = "ProfileScreen")
}

// Outside a screen context — omit screenName entirely
Owl.info("Background sync completed", attributes = mapOf("items" to "$count"))

// String-only Owl.error still works when you don't have a Throwable
// (precondition failures, manual checks, etc.)
Owl.error("Keychain returned no payload for current session")
```

All logging methods share the same shape:

```kotlin
Owl.info(message: String, screenName: String? = null, attributes: Map<String, String?> = emptyMap(), attachments: List<OwlAttachment>? = null)
```

`Owl.error` is overloaded — the first argument may be a `String` (logger-style) or a `Throwable` (exception-style; the SDK extracts type/stack/cause-chain into reserved `_error_*` attributes). When you have a `Throwable` from a `catch`, prefer the `Throwable` form — you get a richer, queryable issue and per-type fingerprinting. SDK-owned `_error_*` keys take precedence over caller-supplied same-named values.

**`screenName` is optional.** Only pass it when the event originates from a specific screen in the UI (e.g. a button-tap handler inside a composable). **Do NOT pass `screenName`** from utility functions, repositories, services, `ViewModel`s decoupled from a screen, network layers, or background work. A fabricated screen name is worse than none — it pollutes screen-level analytics.

**`attributes` accepts nullable values.** A `String?` from your domain code can flow straight into the map — `null`-valued keys are dropped before the event ships, so you don't need to unwrap or build the map conditionally:

```kotlin
val contractId: String? = session.draftId  // may be null
Owl.info("Draft created", attributes = mapOf("context" to "createDraft", "contractId" to contractId))
```

The same applies to every `Owl` / `OwlOperation` method that takes `attributes` (`info`/`debug`/`warn`/`error`, `step`, `startOperation`, `recordMetric`, `complete`/`fail`/`cancel`).

Source file, function, and line are best-effort auto-captured from the call stack (the first frame outside the SDK package).

**Avoid logging PII** (emails, phone numbers, passwords, tokens) or high-frequency events (every frame, every scroll position). Focus on actions and outcomes.

## Instrumentation Principles

Before adding `info` / `warn` / `error` calls throughout the app, internalise these four rules. They turn the SDK from a logger into a queryable analytics surface.

### 1. Log outcomes, not steps

Emit **one rich event per user-meaningful outcome**, not one event per line of code. The unit is the *thing that happened* (purchase completed, photo uploaded, document opened, sign-in succeeded), not the work your code did to make it happen.

```kotlin
// Don't narrate the action with five events
Owl.info("Buy tapped"); Owl.info("Validating receipt"); Owl.info("Calling billing") // ❌

// One event with the full context
val startedAt = SystemClock.uptimeMillis()
try {
    billing.purchase(product)
    Owl.info("Subscription purchased", screenName = "Paywall", attributes = mapOf(
        "product_id" to product.id,
        "price" to product.formattedPrice,
        "intro_offer" to if (product.hasIntroOffer) "yes" else "no",
        "duration_ms" to "${SystemClock.uptimeMillis() - startedAt}",
    ))
} catch (e: Throwable) {
    Owl.error(e, "purchase failed", screenName = "Paywall", attributes = mapOf("product_id" to product.id))
}
```

`Modifier.owlScreen()` already gives you one event per *screen visit* with `_duration_ms` — that **is** the canonical "log the outcome, not the steps" pattern for navigation. Don't supplement it with extra `Owl.info("Screen appeared")` calls. For intermediate diagnostic signals (cache hits, state transitions, fallback decisions), use `debug` level — filtered out of production automatically.

### 2. Pack attributes wide, not events deep

One event with 12 attributes beats 12 events with one attribute each. For a client-side event, think through who / what / where / how / how-much and attach whatever's relevant: `user_id` (auto from `setUser`), `subscription_status`, `product_id`, `from_screen`, `entry_point` (push / deeplink / cold_start), `source_of_truth` (cache / network), `retry_count`, `duration_ms`, `item_count`, `size_bytes`.

The SDK auto-attaches a lot for free — don't re-emit any of these manually: device model, OS version, locale (`Locale.getDefault()`), `preferred_language`, `supported_languages`, `_connection` (wifi / cellular / offline), `app_version`, `is_dev`, `environment` (`android`) — on every event. High-cardinality *attribute values* (document IDs, user IDs) are a **feature**, not a smell — they let you triage one user's broken session from a chart. Control *event frequency* (rule 3), not value uniqueness.

### 3. Aggregate hot paths; don't log per iteration

Anywhere a callback fires repeatedly — scroll listeners, `Flow` collectors, `LaunchedEffect` polls, `RecyclerView` bind, batch processors — log the **outcome**, not each invocation:

```kotlin
// Per-item — one event per photo ❌
pendingPhotos.forEach { Owl.info("Photo uploaded", attributes = mapOf("id" to it.id)) }

// Per-session — one event for the whole upload ✅
val startedAt = SystemClock.uptimeMillis()
var failed = 0
pendingPhotos.forEach { runCatching { upload(it) }.onFailure { failed++ } }
Owl.info("Photo backup completed", attributes = mapOf(
    "uploaded" to "${pendingPhotos.size - failed}",
    "failed" to "$failed",
    "duration_ms" to "${SystemClock.uptimeMillis() - startedAt}",
))
```

Same rule for retry chains: log the final outcome with `retry_count`, not one event per attempt.

### 4. Log, metric, or funnel — pick by the question you want answered

- **Log event** (`Owl.info` / `warn` / `error`) — *"show me individual records of a specific thing that happened."* User actions, error context, edge cases. Read on Dashboard → Events.
- **Lifecycle metric** (`Owl.startOperation` → `.complete` / `.fail` / `.cancel`) — *"show me p50/p95/p99 duration and success rate of this operation over time."* Image uploads, API round-trips, data syncs. Requires a server-side metric definition. Read on Dashboard → Metrics.
- **Single-shot metric** (`Owl.recordMetric`) — *"show me this point-in-time value trended."* Cold-start time, items in cart, memory usage at a checkpoint.
- **Funnel** (`Owl.step`) — *"show me where users drop off across this multi-step flow."* Onboarding, checkout. Requires a server-side funnel definition.
- **Screen view** (`Modifier.owlScreen("Name")`) — *"show me which screens are most/least visited and time-on-screen distributions."* One modifier per screen; covers appear + disappear + duration with zero manual calls. **Always prefer this over a manual `Owl.info("Screen viewed")`.**

> When in doubt, write one event with more attributes rather than several events with fewer.

## File Attachments (use sparingly)

When an error cannot be reproduced without the original input bytes — a media conversion that failed on a specific image, a file that failed to parse, a document that failed to decode — attach the file to the error event. The attachment appears on the resulting issue in the dashboard, CLI, and MCP so an engineer can download and reproduce.

```kotlin
import com.owlmetry.android.OwlAttachment
import java.io.File

try {
    PhotoConverter.convert(inputFile)
} catch (e: Throwable) {
    Owl.error(
        e,
        "image conversion failed",
        screenName = "PhotoConverterScreen",
        attributes = mapOf("stage" to "decode"),
        attachments = listOf(
            OwlAttachment.file(inputFile),                                        // from disk
            OwlAttachment.bytes(debugJson, name = "debug.json", contentType = "application/json"), // in memory
        ),
    )
}
```

**Attachments are a limited resource.** Each project has a storage quota (default **5 GB**) and each end-user has their own bucket within that project (default **250 MB per user** — uploads are auto-tagged with the currently identified `Owl.currentUserId`). Before adding `attachments` anywhere, make sure the file's bytes are *essential* to reproduce the bug.

- ✅ A failed media conversion where only the input bytes reproduce the decoder bug.
- ✅ A document / model parse failure where the file format itself is the suspect.
- ❌ Every error — routine failures (network timeouts, validation) already carry enough in `attributes`.
- ❌ Files reconstructable from event attributes alone (URLs, IDs, small config).
- ❌ Large assets that are *downloaded* rather than user-supplied — include the source URL instead.

Upload behaviour is strictly non-fatal: if the device is offline, the per-user bucket or project quota is exhausted, or the server rejects the file, the event itself still posts — the attachment is dropped silently and a warning is logged to Logcat. Uploads run on a separate coroutine so a large file never blocks event batching.

## Structured Metrics

Use structured metrics instead of plain log events when you want aggregated statistics (averages, percentiles, error rates) rather than a list of individual events. Metrics give you `p50` / `p95` / `p99` latencies, success/failure rates, and trend data over time.

**Decision: lifecycle vs single-shot:**
- **Lifecycle** — measuring something with a duration (start → end): image upload, API call, video encoding, onboarding flow. The SDK auto-tracks `duration_ms`.
- **Single-shot** — recording a point-in-time value: app cold-start time, memory usage, items in cart at checkout.

The metric definition must exist on the server **before** the SDK emits events for that slug. Create it via CLI first.

### Lifecycle operations (start → complete/fail/cancel)

```kotlin
val op = Owl.startOperation("photo-upload", attributes = mapOf("format" to "heic"))

// On success:
op.complete(attributes = mapOf("size_bytes" to "524288"))

// On failure:
op.fail(error = "timeout", attributes = mapOf("retry_count" to "3"))

// On cancellation:
op.cancel(attributes = mapOf("reason" to "user_cancelled"))
```

`duration_ms` and `tracking_id` (UUID) are auto-added.

**Rules for lifecycle operations:**
- **Every `startOperation()` must end** with exactly one `.complete()`, `.fail()`, or `.cancel()`. A start that never ends creates orphaned metric data with no duration.
- **`.complete()`** — succeeded and produced its intended result. **`.fail(error:)`** — attempted work but hit an error. **`.cancel()`** — intentionally stopped (user cancelled, screen left, became irrelevant).
- **Don't start for no-ops** — if the operation is skipped entirely (cache hit, dedup, precondition not met), don't call `startOperation()`. Only start when actual work begins.
- **Don't track duration manually** — `duration_ms` is auto-calculated. Never pass a manual duration attribute.
- **Long-lived operations** — if the operation outlives the scope where it started (recording across a screen lifecycle), hold the `OwlOperation` handle as state and end it on cleanup (`DisposableEffect` `onDispose`, `ViewModel.onCleared`) if it hasn't ended yet.

Create the metric definition first:
```bash
owlmetry metrics create --project-id <id> --name "Photo Upload" --slug photo-upload --lifecycle --format json
```

### Single-shot measurements

```kotlin
Owl.recordMetric("app-cold-start", attributes = mapOf("screen" to "home"))
```

**Slug rules:** lowercase letters, numbers, and hyphens only. Invalid slugs are auto-corrected with a Logcat warning.

## Funnel Tracking

Funnels measure how users progress through a multi-step flow (onboarding, checkout, activation) and where they drop off. Three parts:

1. **Define** the funnel server-side (via CLI or API) with ordered steps and event filters.
2. **Record** steps client-side with `Owl.step("step-name")`.
3. **Query** analytics to see conversion rates and drop-off between steps.

The step name you pass to `Owl.step()` must match the `step_name` in the funnel definition's `event_filter`. If the step filter is `{"step_name": "welcome-screen"}`, call `Owl.step("welcome-screen")`.

**Funnel design rules:**
- Each step must be a point **every user in the funnel passes through** on the way to the goal. A conditional step (a paywall shown only to free users) breaks the chain — users who skip it show as 0% conversion from there.
- Keep funnels focused on **one flow**. Don't combine separate journeys.
- **Optional interactions are not steps.** Toggling a setting or viewing info is an engagement event (`Owl.info()`), not funnel progression.
- Split alternative paths into **separate funnels**.
- Aim for **3–6 steps** per funnel.

```kotlin
Owl.step("welcome-screen")
Owl.step("create-account", attributes = mapOf("method" to "email"))
Owl.step("complete-profile")
Owl.step("first-post")
```

Define matching funnel definitions via `/owlmetry-cli`:
```bash
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

## User Properties

Attach custom key-value metadata to the current user. Properties are merged server-side — existing keys not in your call are preserved.

```kotlin
Owl.setUserProperties(mapOf(
    "plan" to "premium",
    "org" to "acme",
))
```

Set a value to `""` to delete a key. All values are strings. Max 50 properties per user, 50-char keys, 200-char values. Properties follow the current user identity — if the user is anonymous, they're set on the anonymous user and merged into the real user on `Owl.setUser()`. Use for user-level data that changes infrequently (subscription status, plan tier, company); for event-specific data, use `attributes` on events instead.

## Collect User Feedback

The `owlmetry-android-compose` artifact ships a reusable Compose view (`OwlFeedbackView`) plus a programmatic API (`Owl.sendFeedback`) for gathering free-text feedback inside your app. Submissions are linked automatically to the current session, user id, app version, device, and environment — nothing extra to pass in.

### Programmatic submit

```kotlin
import com.owlmetry.android.Owl
import com.owlmetry.android.OwlFeedbackError

try {
    val receipt = Owl.sendFeedback(
        message = "Love the new import flow!",
        name = currentUser?.displayName,   // optional
        email = currentUser?.email,        // optional
    )
    Log.d("Owlmetry", "Feedback stored: ${receipt.id}")
} catch (e: OwlFeedbackError) {
    // EmptyMessage, NotConfigured, ServerError, TransportFailure
    showSnackbar(e.message)
}
```

`sendFeedback` is a **suspend** function and is **not** offline-queued — it returns a receipt on success and throws on failure, so you can surface a retry. Call it from a coroutine.

### Drop-in Compose view

`OwlFeedbackView` is a plain `@Composable` — host it however you want: in a `ModalBottomSheet`, as a navigation destination, or inline. The host owns presentation and dismissal (Compose has no `@Environment(\.dismiss)` equivalent) — wire dismissal from `onSubmitted` / `onCancel`. Every user-facing string is overridable via `OwlFeedbackStrings`.

```kotlin
import com.owlmetry.android.compose.OwlFeedbackView
import com.owlmetry.android.compose.OwlFeedbackActionsPlacement

// 1. In a modal bottom sheet
if (showFeedback) {
    ModalBottomSheet(onDismissRequest = { showFeedback = false }) {
        OwlFeedbackView(
            name = user?.displayName,
            email = user?.email,
            onSubmitted = { showFeedback = false },
            onCancel = { showFeedback = false },
        )
    }
}

// 2. Embedded inline (no top action row, no contact fields)
Column {
    Text("Tell us what you think")
    OwlFeedbackView(
        showsContactFields = false,
        actionsPlacement = OwlFeedbackActionsPlacement.INLINE,
        onSubmitted = { /* ... */ },
    )
}
```

`actionsPlacement` controls where Submit / Cancel render: `TOOLBAR` (a top action row — use in a sheet/dialog with no app bar) or `INLINE` (buttons at the bottom — use when embedding in a parent screen). Default is `TOOLBAR`.

### Customizing strings / theming

```kotlin
// Partial override
OwlFeedbackView(strings = OwlFeedbackStrings.DEFAULT.with(header = "How are we doing?"))
```

The view reads your `MaterialTheme` — the Submit button and accents inherit your app's `colorScheme` automatically, so it matches your brand without extra code.

### Where feedback lands

Submissions show up on **Dashboard → Feedback** as a kanban with four statuses (`new → in_review → addressed → dismissed`). Humans, the CLI (`owlmetry feedback`), and MCP agents can all read and triage it.

## Collect Structured Surveys (Questionnaires)

Questionnaires are short multi-question surveys (text / single-choice / multi-choice / 1–5 rating / 0–10 NPS) shown in-app via a Compose wrapper. Where `OwlFeedbackView` collects one free-text message, an `OwlQuestionnaireView` walks the user through a typed schema you author once on the server.

### Prerequisite — create the questionnaire first

The Android SDK only reads and submits — it does **not** define questionnaires. Create one before instrumenting the app:

- Dashboard → Questionnaires → **New questionnaire** (slug + name + schema JSON)
- CLI: `owlmetry questionnaires create --project-id <id> --slug post-onboarding --name "Onboarding survey" --schema-file ./schema.json`
- MCP: the `create-questionnaire` tool

Slug is immutable after creation — pick the SDK call-site name carefully (`post-onboarding`, `weekly-checkin`, etc.).

#### What end-users see vs what's internal

**Critical for agents authoring questionnaires:** most of the spec is shown to end-users in the in-app sheet. The questionnaire's **`description`** renders verbatim in the consent prompt body (under "Quick favor?") when non-empty — it is *not* a private note. `name` is internal (dashboard tables + notifications); `slug`, `id`, `is_active`, `app_id` are internal. Question `title` / `subtitle` / `placeholder` / option `label` are all user-facing copy. If you fill `description` with `"draft — confirm wording w/ marketing"`, that exact string ships to production users.

### Auto-trigger gate (primary path)

Compose has no modifier-driven presentation, so the gate is a **wrapper composable** (`OwlQuestionnaireGate`) — the analog of Swift's `.owlQuestionnaire(...)` modifier. Render your screen as its `content` and it overlays a `ModalBottomSheet` hosting `OwlQuestionnaireView` once the trigger fires and the server returns an eligible spec.

```kotlin
import com.owlmetry.android.compose.OwlQuestionnaireGate
import com.owlmetry.android.OwlQuestionnaireTrigger

@Composable
fun RootScreen() {
    OwlQuestionnaireGate(
        slug = "post-onboarding",
        trigger = OwlQuestionnaireTrigger.afterLaunches(3),
    ) {
        // Your normal screen content goes here
        AppNavHost()
    }
}
```

**Parameters** on `OwlQuestionnaireGate(...)`:

| Parameter | Default | What it does |
|---|---|---|
| `slug` | required | Looks up the server-side spec. Must match the `slug` you created. |
| `trigger` | `.afterLaunch` | When to evaluate. ANDed conditions — see below. |
| `showsConsent` | `true` | When `true`, opens with the "Quick favor?" consent prompt (Sure / Maybe later / Don't ask again). When `false`, jumps straight to question 1. Auto-skipped when resuming a draft. |
| `isEligible` | `null` | Sync closure; return `false` to skip. App-side gating (paid status, feature flags). Re-evaluates on every foreground (`ON_RESUME`). |
| `forceShow` | `false` | Debug-only override that bypasses every local + most server gate (still respects `inactive`). Wire to a debug-menu toggle or `BuildConfig.DEBUG`. |
| `strings` | `.DEFAULT` | Override consent + flow copy via `OwlQuestionnaireStrings.DEFAULT.with(...)`. The spec's `description` wins over `strings.consentBody` when present. |
| `onSubmitted` | `null` | Fires once on the call that flips the response to submitted. Receives `OwlQuestionnaireReceipt`. |
| `onCancel` | `null` | Fires on Maybe later / Cancel / swipe-dismiss without submitting. |
| `onDismissed` | `null` | Fires on Don't ask again (global opt-out, confirmed). |

The gate evaluates once per composition entry and again on each foreground (`ON_RESUME`). Per-process dedup prevents re-presenting the same slug within a launch; cross-launch dedup is the server's job. Once accepted, questions render one-per-page with a progress bar and Back / Next / Submit; on submit, an in-sheet success page replaces the questions.

**Progressive saves + resume** — answers persist to the server on every Next tap, not just on Submit. If the user quits mid-flow, the next eligible launch's gate picks the draft up automatically: it fetches the spec, sees the saved draft, skips consent, and lands the user at the first unanswered question with prior answers pre-filled. No extra code on your side. The team `questionnaire.response_new` notification only fires on the final Submit.

### Composable triggers (ANDed conditions)

```kotlin
import com.owlmetry.android.OwlQuestionnaireTrigger
import com.owlmetry.android.OwlQuestionnaireCondition

OwlQuestionnaireGate(
    slug = "weekly-checkin",
    trigger = OwlQuestionnaireTrigger.whenAll(
        OwlQuestionnaireCondition.Launches(atLeast = 3),
        OwlQuestionnaireCondition.DaysSinceFirstLaunch(atLeast = 7),
    ),
) { content() }
```

Conditions: `Launches(atLeast)`, `Foregrounds(atLeast)`, `DaysSinceFirstLaunch(atLeast)`, `HoursSinceFirstLaunch(atLeast)`. Shortcuts: `OwlQuestionnaireTrigger.afterLaunch`, `.afterLaunches(n)`, `.whenAll(vararg conditions)`, `.manual` (never auto-trigger). All conditions in `whenAll` are ANDed — for OR logic, use `isEligible` or attach two gates with different slugs.

### Gating (free vs paid, feature flags)

```kotlin
OwlQuestionnaireGate(
    slug = "free-user-survey",
    trigger = OwlQuestionnaireTrigger.afterLaunch,
    isEligible = { !user.isPaid },
) { content() }
```

`isEligible` runs synchronously before the SDK fetches the spec; return `false` to skip this evaluation. It re-runs on the next foreground.

### Manual presentation

For ad-hoc triggers ("show after the user finishes the import wizard"), fetch + present yourself. `Owl.fetchQuestionnaire` returns a result wrapping the spec + any in-progress draft for the current user:

```kotlin
import com.owlmetry.android.Owl
import com.owlmetry.android.OwlQuestionnaire
import com.owlmetry.android.OwlQuestionnaireDraft
import com.owlmetry.android.compose.OwlQuestionnaireView

var spec by remember { mutableStateOf<OwlQuestionnaire?>(null) }
var inProgress by remember { mutableStateOf<OwlQuestionnaireDraft?>(null) }
var show by remember { mutableStateOf(false) }
val scope = rememberCoroutineScope()

Button(onClick = {
    scope.launch {
        val result = runCatching { Owl.fetchQuestionnaire("post-import") }.getOrNull()
        val q = result?.questionnaire   // null ⇒ ineligible; result.ineligibleReason carries why
        if (q != null) { spec = q; inProgress = result.inProgress; show = true }
    }
}) { Text("Take a quick survey") }

if (show) {
    val q = spec
    if (q != null) {
        ModalBottomSheet(onDismissRequest = { show = false }) {
            OwlQuestionnaireView(
                questionnaire = q,
                inProgress = inProgress,                 // resumes mid-flow
                showsConsent = inProgress == null,       // skip consent on resume
                onSubmitted = { show = false },
                onCancel = { show = false },
            )
        }
    }
}
```

### One-shot save (advanced)

For a fully custom UI on top of the SDK, `Owl.saveQuestionnaireResponse` is the underlying API (the gate and `OwlQuestionnaireView` call it for you):

```kotlin
import com.owlmetry.android.OwlQuestionnaireAnswerValue

val receipt = Owl.saveQuestionnaireResponse(
    slug = "post-import",
    answers = mapOf(
        "q_text" to OwlQuestionnaireAnswerValue.TextValue("Loved it"),
        "q_rating" to OwlQuestionnaireAnswerValue.RatingValue(5),
        "q_nps" to OwlQuestionnaireAnswerValue.NpsValue(9),
    ),
    isComplete = true,   // false to save a draft, true to finalize
)
// receipt.wasSubmitted == true exactly on the call that flipped the response to
// submitted. The server merges incoming keys onto the existing row keyed by
// (project, slug, user_id) — no client-side response_id tracking needed.
```

Answer value types: `TextValue(String)`, `ChoiceValue(String)`, `ChoicesValue(List<String>)`, `RatingValue(Int)` (1–5), `NpsValue(Int)` (0–10).

### Programmatic dismissal

```kotlin
Owl.dismissQuestionnaires()
```

Globally opts the current user out of every questionnaire. Idempotent. Survives reinstall (stored server-side on `app_users.properties`).

### Where responses land

Responses show up on **Dashboard → Questionnaires → &lt;questionnaire&gt;** with pre-aggregated per-question analytics (bar charts for choices, average for ratings, NPS score for NPS). Each response stores the schema snapshot it was submitted against, so editing the questionnaire later never retroactively breaks historical rendering. CLI: `owlmetry questionnaires …`. MCP: `list-questionnaires` / `list-questionnaire-responses` / `get-questionnaire-analytics`.

## What the SDK Tracks Automatically

Do not re-implement any of these — they are built into the SDK and emitted without code:

- **`sdk:session_started`** — emitted on `Owl.configure()`; includes `_launch_ms` (process start → configure, via `Process.getStartUptimeMillis()`).
- **`sdk:app_foregrounded`** / **`sdk:app_backgrounded`** — process lifecycle transitions (via the process `LifecycleObserver`).
- **`session_id`** — fresh UUID per `configure()` call, on every event. Readable at runtime via `Owl.sessionId: String?`. Forward it to a Node.js backend in an `X-Owl-Session-Id` request header to link client and backend events under the same session.
- **`sdk:screen_appeared`** (debug) / **`sdk:screen_disappeared`** (debug, with `_duration_ms`) — when using `Modifier.owlScreen()`.
- **Device model, OS version, locale, `preferred_language`, `supported_languages`** — on every event.
- **`_connection`** — network type (wifi, cellular, offline) via the SDK's network monitor.
- **`environment`** — `android`.
- **`app_version`** — from the host app's package info.
- **`is_dev`** — `true` when the host app is debuggable (debug builds), `false` for release.
- **`country_code`** — stamped server-side from the ingest request (the SDK does not send it).
- **`sdk_name`** (`"owlmetry-android"`) and **`sdk_version`** (the published release) — auto-stamped on every event and feedback submission. **Do not set these manually.**

You do NOT need to manually track app launch, foreground/background, session start, network type, or device info. These are already covered.

## Instrumentation Strategy

When instrumenting a new app, follow this priority:

**Always instrument (events — no CLI setup needed):**
- Screen views (`Modifier.owlScreen("ScreenName")` on every distinct Compose screen)
- Authentication events (login, logout, signup)
- Caught exceptions (`Owl.error(throwable, ...)` in `catch` blocks and error handlers)
- Validation failures and pre-checks (`warn` for bad input, missing optional data, fallback paths)
- Core business actions (purchase, share, create, delete)

**Instrument when relevant (metrics — requires CLI `owlmetry metrics create` first):**
- Lifecycle metrics where duration matters: image uploads, API calls, data syncs
- Single-shot metrics for point-in-time values: cold-start time, memory usage, items in cart

**Instrument when relevant (funnels — requires CLI `owlmetry funnels create` first):**
- Multi-step flows you want conversion on: onboarding, checkout, activation

**Where to place calls:**
- Screen views: `Modifier.owlScreen("Name")` on the outermost composable of each screen.
- User actions: button `onClick` / gesture handlers — pass `screenName` since you know the screen.
- Errors: `catch` blocks, `Result.onFailure` — pass `screenName` only if caught inside a composable; omit it in a repository, service, or `ViewModel` decoupled from a screen.
- Repositories, use-cases, background work, `WorkManager` jobs: log freely but **never pass `screenName`**.
- Metrics: wrap the async operation between `startOperation()` and `complete()` / `fail()`.

**What NOT to instrument:**

Never put these in event messages or attribute values (the caps truncate, they don't redact): auth tokens / OAuth / JWT contents, passwords / PINs / recovery phrases, raw card numbers / CVVs / SSNs, full HTTP request/response bodies, personal content the user typed (notes, messages — `Owl.sendFeedback` / `OwlFeedbackView` is the consented path for that), or a raw exception message that embeds user input (pass the `Throwable` to `Owl.error()` instead — the SDK extracts type, stack, and cause chain with proper fingerprinting).

Skip — usually noise: every tap / scroll / drag, per-frame events from `Flow` / animation callbacks, per-row events inside batch loops (log the batch outcome with counts), per-attempt retry logs (log the final outcome with `retry_count`).

## Lifecycle

```kotlin
// On a deliberate teardown (e.g. a test harness, or before process exit)
Owl.shutdown()   // suspend — flushes buffered events and stops the background-flush observer
```

`flushOnBackground = true` (the default) handles most cases automatically — the SDK flushes when the process backgrounds. Call `shutdown()` explicitly only if you need to guarantee delivery at a specific point. The SDK can be `configure()`d again afterward.
