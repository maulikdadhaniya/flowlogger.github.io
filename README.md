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
## Automatic ViewModel event logging (`AutoLogTracker`)

These rules apply only if you rely on **reflection-based** tracking of flows on a **ViewModel** (same mechanism as in the sample app).

| Requirement | Detail |
|-------------|--------|
| **Property name** | The **`Flow` / `SharedFlow` / `StateFlow` / `LiveData`** backing field (or property) must be named **`event`** or **`events`** (case-insensitive). Other names are **not** auto-discovered as event streams. |
| **Emission type** | **Any** type is allowed (`String`, `Map`, sealed classes, data classes, etc.). There is **no** requirement for a sealed class. |
| **Lifecycle** | Tracking is tied to a **`LifecycleOwner`** (e.g. screen) when you register the tracker — the flow is collected in **`STARTED`** or equivalent per implementation. |

**Not required for auto-tracking:** naming something `state` — **state** streams are **not** selected by `AutoLogTracker` (only **`event`** / **`events`**).

**Bypassing the name rule:** use **`FlowLogger.log`**, **`Flow.collectWithLog`**, **`ComposeTracker`**, or **`ViewModelTracker`** on any flow you choose; those APIs do **not** require the property to be named `event` or `events`.

---
