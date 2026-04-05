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
