# FlowLogger — Quick setup

**Not for production.** FlowLogger is a **development-only** tool (local debugging and QA). Do **not** ship it in store or production builds. Use **`debugImplementation`** so it is **never** included in **release** variants.

---

## 1. Dependency

```kotlin
// app/build.gradle.kts
dependencies {
    debugImplementation("io.github.maulikdadhaniya:flowlogger:<version>")
}
```

**Version catalog** — in `gradle/libs.versions.toml`:

```toml
[versions]
flowlogger = "1.0.0"  # match the version you use

[libraries]
flowlogger = { group = "io.github.maulikdadhaniya", name = "flowlogger", version.ref = "flowlogger" }
```

Then: `debugImplementation(libs.flowlogger)`.

---

## 2. Initialize (Application)

```kotlin
import android.content.pm.ApplicationInfo
import com.maulik.flowlogger.notification.FlowLoggerAutoInstall

override fun onCreate() {
    super.onCreate()
    FlowLoggerAutoInstall.install(
        application = this,
        scope = appScope,
        enabled = (applicationInfo.flags and ApplicationInfo.FLAG_DEBUGGABLE) != 0,
    )
}
```

---

## 3. Only for developement usage

A **production-level** app should **not** embed this library for end users. It exists only for **development usage** while you build and test.

**`debugImplementation`** keeps FlowLogger **out of release** APKs and App Bundles: in **release**, the dependency is **not** applied — nothing is shipped and FlowLogger is **not** on the classpath. That is **on purpose**, so your production build stays clean.

---

## 4. Optional: app-icon shortcut

**Add** — on your **launcher** activity only (often `src/debug/AndroidManifest.xml`):

```xml
<meta-data
    android:name="android.app.shortcuts"
    android:resource="@xml/flowlogger_shortcuts" />
```
---

## ‼️ Automatic ViewModel event logging (`AutoLogTracker`) ‼️

These rules apply only if you rely on **reflection-based** tracking of flows on a **ViewModel** (same mechanism as in the sample app).

| Requirement | Detail |
|-------------|--------|
| **Property name** | The **`Flow` / `SharedFlow` / `StateFlow` / `LiveData`** backing field (or property) must be named **`event`** or **`events`** (case-insensitive). Other names are **not** auto-discovered as event streams. |
| **Emission type** | **Any** type is allowed (`String`, `Map`, sealed classes, data classes, etc.). There is **no** requirement for a sealed class. |
| **Lifecycle** | Tracking is tied to a **`LifecycleOwner`** (e.g. screen) when you register the tracker — the flow is collected in **`STARTED`** or equivalent per implementation. |

**Not required for auto-tracking:** naming something `state` — **state** streams are **not** selected by `AutoLogTracker` (only **`event`** / **`events`**).

**Bypassing the name rule:** use **`FlowLogger.log`**, **`Flow.collectWithLog`**, **`ComposeTracker`**, or **`ViewModelTracker`** on any flow you choose; those APIs do **not** require the property to be named `event` or `events`.

---

## Manual logging APIs — parameters and examples

Use these when you want logging **without** renaming fields to `event` / `events`. Imports: `com.maulik.flowlogger.core.FlowLogger`, `com.maulik.flowlogger.core.collectWithLog`, `com.maulik.flowlogger.compose.ComposeTracker`, `com.maulik.flowlogger.tracker.ViewModelTracker` (as needed). **`ScreenTracker.currentScreen`** is only relevant if **`FlowLoggerAutoInstall`** / **`ComposeTracker.TrackScreen`** set the current screen; otherwise pass a fixed `screen` string or your own `screenProvider`.

### `FlowLogger.log` — one-off entry

| Parameter | Required | Meaning |
|-----------|----------|---------|
| `screen` | Yes | Label for “where” (often screen name). |
| `source` | Yes | Owner of the event (e.g. `"LoginViewModel"`). |
| `event` | Yes | Event name (e.g. `"SubmitClicked"`). |
| `value` | No | Any payload (`Map`, `String`, JSON, data class, …); formatted for the timeline. |
| `error` | No | `Throwable`; stack trace is stored. |
| `diffKey` | No | If set, duplicate **values** for the same key may be skipped (dedup). |

```kotlin
FlowLogger.log(
    screen = "Login",
    source = "LoginViewModel",
    event = "CredentialsSubmitted",
    value = mapOf("email" to "user@example.com"),
)

FlowLogger.log(
    screen = "Login",
    source = "LoginViewModel",
    event = "NetworkError",
    error = throwable,
)
```

### `Flow.collectWithLog` — log every emission of a cold `Flow` (core extension)

| Parameter | Required | Meaning |
|-----------|----------|---------|
| `screen` | Yes | Used for every log line from this chain. |
| `source` | Yes | Same as `FlowLogger.log`. |
| `name` | Yes | Prefix for this stream; also used in `diffKey` if not overridden. |
| `diffKey` | No | Defaults to `"$source.$name"`. Dedup key for repeated values. |
| `valueToLog` | No | Maps each emission to what is stored (default: identity). |

Also logs synthetic events: `name(start)`, `name(error)`, `name(complete)` around the stream.

```kotlin
import com.maulik.flowlogger.core.collectWithLog

someFlow
    .collectWithLog(
        screen = "Home",
        source = "Repo",
        name = "userStream",
        valueToLog = { it }, // optional: trim or map payload
    )
    .collect { /* ... */ } // you must still collect somewhere
```

### `ComposeTracker` — Compose screens

| API | What to pass | Purpose |
|-----|----------------|--------|
| **`TrackScreen(screenName)`** | `screenName: String` | Updates **`ScreenTracker`** so default `screenProvider` logs use this screen. |
| **`LogState(state, source, name, …)`** | `State<T>` from `remember { mutableStateOf }` / similar | Logs whenever the **state** value changes (via `snapshotFlow`). |
| **`Flow.collectWithLog` (extension)** | `source`, `name`, optional `diffKey`, `screenProvider`, `valueToLog` | Same idea as core `collectWithLog`, but **`screen`** defaults to **`ScreenTracker.currentScreen`** — call **`TrackScreen`** first or override **`screenProvider`**. |
| **`TrackObjectFlows(target)`** | `ViewModel` / any object | Runs **`AutoLogTracker`** for **`event`** / **`events`** only (same rules as automatic VM tracking). |

```kotlin
import com.maulik.flowlogger.compose.ComposeTracker
import com.maulik.flowlogger.compose.ComposeTracker.collectWithLog

@Composable
fun ProfileScreen(vm: ProfileViewModel) {
    ComposeTracker.TrackScreen("Profile")

    val uiState = vm.uiState.collectAsState()
    ComposeTracker.LogState(
        state = uiState,
        source = "ProfileViewModel",
        name = "uiState",
    )

    LaunchedEffect(Unit) {
        vm.sideEffects.collectWithLog(
            source = "ProfileViewModel",
            name = "sideEffects",
        ).collect { }
    }
}
```

Minimal pattern for a **`Flow`** (screen comes from **`ScreenTracker`** after **`TrackScreen`**):

```kotlin
LaunchedEffect(Unit) {
    vm.myActions.collectWithLog(
        source = "ProfileViewModel",
        name = "myActions",
    ).collect { }
}
```

For **`LogState`**, pass a Compose **`State<T>`** (e.g. return value of **`collectAsState()`**, or **`remember { mutableStateOf(...) }`**), not a bare value.

### `ViewModelTracker` — Fragment / `ComponentActivity` + arbitrary `Flow` / `LiveData`

| API | What to pass | Purpose |
|-----|----------------|--------|
| **`trackStateFlow`** | `LifecycleOwner`, `Flow<T>`, `source`, `name`, optional `diffKey`, `screenProvider`, `valueToLog` | Collects while lifecycle is at least **`STARTED`**; each emission → **`FlowLogger.log`**. |
| **`trackLiveData`** | `LifecycleOwner`, `LiveData<T>`, `source`, `name`, same optional params | Observes LiveData; each value → **`FlowLogger.log`**. Returns an **`Observer`** (usually you ignore return if you only need side effect). |

Default **`screenProvider`** is **`{ ScreenTracker.currentScreen.value }`**. Override if you do not use **`ScreenTracker`**.

```kotlin
import androidx.fragment.app.Fragment
import com.maulik.flowlogger.tracker.ViewModelTracker

class HomeFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val vm: HomeViewModel by viewModels()
        ViewModelTracker.trackStateFlow(
            owner = viewLifecycleOwner,
            flow = vm.userClicks, // any Flow name — NOT required to be "events"
            source = "HomeViewModel",
            name = "userClicks",
            screenProvider = { "Home" }, // or { ScreenTracker.currentScreen.value }
            valueToLog = { it },
        )
    }
}
```

```kotlin
ViewModelTracker.trackLiveData(
    owner = this, // Activity or Fragment as LifecycleOwner
    liveData = vm.status,
    source = "HomeViewModel",
    name = "status",
)
```

---

## Notifications (conditional)

| Requirement | Detail |
|-------------|--------|
| **Android 13+ (API 33+)** | **`POST_NOTIFICATIONS`** is required for the ongoing notification. The library requests it via bootstrap; the user can deny (logging elsewhere still works). |
| **Manifest** | Library **merges** `FlowLoggerNotificationReceiver` and related entries; you **must not** duplicate them unless you know you need an override. |

---

## Launcher shortcuts (optional)

| Requirement | Detail |
|-------------|--------|
| **Static shortcut** | If you want the long-press app icon shortcut, add **`android.app.shortcuts`** `meta-data` on **your** launcher activity pointing to **`@xml/flowlogger_shortcuts`**. |
| **If omitted** | Notifications, in-app open, and manual logging **still work**; only the launcher shortcut is missing. |

---
