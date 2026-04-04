# Clojure-Android Developer Guide

This guide covers how to build Android apps with Clojure — from project
structure through live REPL development to store-ready release builds.

## Prerequisites

- **JDK 17+** (for Gradle and Clojure compilation)
- **Android SDK** with Build Tools and platform API 26+ installed
- **Gradle 8.7+** (the wrapper is included in the sample project)
- A physical Android device or emulator running Android 8.0+ (API 26+)
- A Clojure-aware editor (Emacs+CIDER, VS Code+Calva, IntelliJ+Cursive, or
  any nREPL client)

Install the Clojure-Android toolchain to your local Maven repository before
creating an app. The sample project includes a task that builds and publishes
all dependencies in the correct order:

```bash
cd sample
./gradlew publishDepsToMavenLocal
```

This clones all dependencies into `build/deps/` (if not already present),
then runs `clean publishToMavenLocal` in each one. The sample app's
`settings.gradle.kts` also auto-clones these dependencies on any build, so
you can skip this step and build the sample directly — dependencies are
fetched and built from source automatically.

---

## 1. App Structure

A Clojure-Android app is a standard Gradle Android project with Clojure
source directories added alongside (or instead of) Java/Kotlin sources.

### Directory layout

```
my-app/
├── settings.gradle.kts
├── build.gradle.kts            # Root build file (plugin declarations)
├── gradle.properties
├── app/
│   ├── build.gradle.kts        # App module (Android + Clojure plugin)
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── clojure/             # Clojure sources (AOT-compiled)
│       │   └── com/myapp/
│       │       └── main_activity.clj  # Activity logic (matches MainActivity)
│       ├── java/                # Java sources (optional)
│       │   └── com/myapp/
│       │       └── MainActivity.java
│       └── res/
│           ├── values/
│           │   └── strings.xml
│           └── drawable/        # Icons, images
└── gradle/
    └── wrapper/                 # Gradle wrapper (checked in)
```

### settings.gradle.kts

```kotlin
pluginManagement {
    repositories {
        mavenLocal()          // Clojure-Android plugin lives here
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositories {
        mavenLocal()          // Runtime libraries live here
        google()
        mavenCentral()
        maven { url = uri("https://clojars.org/repo") }  // Clojure libraries
    }
}

rootProject.name = "my-app"
include(":app")
```

### Root build.gradle.kts

```kotlin
plugins {
    id("com.android.application") version "8.9.0" apply false
    id("com.goodanser.clj-android.android-clojure") version "0.5.0-SNAPSHOT" apply false
}
```

### app/build.gradle.kts

```kotlin
plugins {
    id("com.android.application")
    id("com.goodanser.clj-android.android-clojure")
}

android {
    namespace = "com.myapp"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.myapp"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true    // R8 tree-shaking for smaller APK
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }

    packaging {
        resources {
            // Our Android-compatible stubs shadow nREPL's originals
            pickFirsts += listOf("nrepl/socket.clj", "nrepl/socket/dynamic.clj")
        }
    }
}

clojureOptions {
    warnOnReflection.set(true)

    // replEnabled: include nREPL server. Default: true for debug, false for release.
    // Set to true in release builds if you want REPL access in production.
    // replEnabled.set(true)

    // dynamicCompilationEnabled: include AndroidDynamicClassLoader and dx.
    // Automatically enabled when replEnabled is true. Set independently if
    // your app needs to eval Clojure at runtime without nREPL.
    // dynamicCompilationEnabled.set(true)

    // nreplPort: device-side nREPL port. Default: 7888.
    // nreplPort.set(9999)

    // sourceResourceExcludes: strip specific .clj files from the APK/AAR.
    // Use this for proprietary code that should be AOT-compiled but not
    // distributed as source. Patterns match resource paths in the archive.
    // sourceResourceExcludes.set(listOf("com/myapp/internal/**"))
}

dependencies {
    implementation("org.clojure:clojure:1.12.0")
    implementation("com.goodanser.clj-android:neko:5.0.0-SNAPSHOT")
}
```

### AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Required for nREPL connections over ADB -->
    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name="com.goodanser.clj_android.runtime.ClojureApp"
        android:label="@string/app_name"
        android:supportsRtl="true">

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

Key points:
- **`android:name="com.goodanser.clj_android.runtime.ClojureApp"`** — This
  Application subclass initializes the Clojure runtime before any Activity
  starts. You can subclass it if you need custom Application logic.
- **`INTERNET` permission** — Required for nREPL to accept connections via
  ADB port forwarding. You can remove this in release builds if your app
  doesn't otherwise need it.

### Writing activities

Activities extend `ClojureActivity` from `runtime-core`. This base class
automatically requires a Clojure namespace derived from the class name and
delegates lifecycle methods to functions in that namespace. The Java class
is just a thin shim — all logic lives in Clojure:

```java
// app/src/main/java/com/myapp/MainActivity.java
package com.myapp;

import com.goodanser.clj_android.runtime.ClojureActivity;

/**
 * ClojureActivity maps this class to the Clojure namespace
 * com.myapp.main-activity (CamelCase → kebab-case by convention).
 */
public class MainActivity extends ClojureActivity {
    // All behavior is defined in the Clojure namespace
    // com.myapp.main-activity
}
```

`ClojureActivity` converts the class name to a namespace using this
convention: `com.myapp.MainActivity` becomes `com.myapp.main-activity`.
Override `getClojureNamespace()` in the Java class to use a custom namespace
instead.

### Writing Clojure UI code

The Clojure namespace corresponding to an activity defines lifecycle
functions that `ClojureActivity` calls automatically. The neko library
provides a declarative DSL where UI layouts are Clojure data structures:

```clojure
;; app/src/main/clojure/com/myapp/main_activity.clj
(ns com.myapp.main-activity
  (:require [neko.ui :as ui]
            [neko.find-view :refer [find-view]]
            [neko.log :as log])
  (:import android.app.Activity
           com.goodanser.clj_android.runtime.ClojureActivity))

(def ^:private counter (atom 0))

(defn make-ui
  "Builds the UI tree using neko's declarative DSL.
  Called by on-create and by ClojureActivity.reloadUi()."
  [^Activity activity]
  (ui/make-ui activity
    [:linear-layout {:id-holder true
                     :orientation :vertical
                     :padding [32 32 32 32]}
     [:text-view {:text "Hello from Clojure!"
                  :text-size [24 :sp]}]
     [:button {:text "Click me"
               :on-click (fn [_]
                           (let [n (swap! counter inc)]
                             (log/i "Button clicked:" n)))}]]))

(defn on-create
  "Called automatically by ClojureActivity when the activity is created."
  [^Activity activity saved-instance-state]
  (.setContentView activity (make-ui activity)))

(defn reload-ui!
  "Hot-reload the UI from the REPL."
  []
  (when-let [activity (ClojureActivity/getInstance
                        "com.myapp.main-activity")]
    (.reloadUi ^ClojureActivity activity)))
```

The Clojure namespace may define any of these functions (all optional):

- `(on-create [activity bundle])` — called from `onCreate`
- `(on-start [activity])`, `(on-resume [activity])`, `(on-pause [activity])`,
  `(on-stop [activity])`, `(on-destroy [activity])`
- `(on-save-instance-state [activity bundle])`,
  `(on-restore-instance-state [activity bundle])`
- `(make-ui [activity])` — returns a `View`; used by `reloadUi()` and as a
  fallback if `on-create` is absent

### What the Gradle plugin does automatically

When you apply `com.goodanser.clj-android.android-clojure`, the plugin:

1. Registers `src/{sourceSet}/clojure/` as source directories
2. Creates a `compile{Variant}Clojure` task that AOT-compiles all `.clj`
   files to JVM `.class` files
3. Wires the compiled classes into Android's dex pipeline
4. Adds `runtime-core` (all builds) and `runtime-repl` (debug builds by
   default — see `clojureOptions` below)
5. In debug builds (by default), substitutes stock Clojure with a patched
   version that delegates to `AndroidDynamicClassLoader` for REPL support
6. Bundles `.clj` source files as resources only in builds with dynamic
   compilation enabled (debug by default). Release builds contain only
   AOT-compiled classes — no source files in the APK

---

## 2. Development with nREPL over ADB

The REPL workflow lets you modify a running app without rebuilding. You
edit Clojure code, send it to the device, and see changes immediately.

### How it works

```
 Your Editor                     ADB                      Android Device
┌──────────┐              ┌─────────────┐              ┌────────────────┐
│  nREPL   │──tcp:9999──▶│ adb forward │──tcp:9999──▶│  nREPL server  │
│  client  │              │             │              │                │
│ (CIDER/  │◀─results────│  tunnels    │◀─results────│  Evaluates     │
│  Calva)  │              │  over USB   │              │  Clojure code  │
└──────────┘              └─────────────┘              │  on ART via    │
                                                       │  AndroidDyn.CL │
                                                       └────────────────┘
```

In debug builds (by default), the app includes:
- An **nREPL server** listening on port 7888 (configurable) on the device
- **AndroidDynamicClassLoader** — translates JVM bytecode emitted by the
  Clojure compiler into DEX format on-the-fly using the dx library, then
  loads it via `InMemoryDexClassLoader`
- The **dx library** (from AOSP) for JVM-to-DEX bytecode translation

By default, none of these components are included in release builds.
However, you can opt in to dynamic compilation and/or nREPL in release
builds via `clojureOptions`:

```kotlin
clojureOptions {
    // Include nREPL + dynamic compilation in release builds
    replEnabled.set(true)

    // Or include only dynamic compilation (no nREPL)
    dynamicCompilationEnabled.set(true)
}
```

This is useful for apps that evaluate user-provided Clojure expressions
at runtime, or for field-testing with REPL access.

### Step-by-step setup

#### 1. Build and install the debug APK

```bash
cd my-app
./gradlew :app:installDebug
```

This builds the APK with REPL support enabled and installs it on the
connected device.

#### 2. Launch the app

Start the app from the device's launcher, or via ADB:

```bash
adb shell am start -n com.myapp/.MainActivity
```

The nREPL server starts automatically on the device when `ClojureApp` is
the application class (set via `android:name` in your manifest — see the
AndroidManifest.xml section above). You can verify it in logcat:

```bash
adb logcat -s ClojureApp
# Should show: "nREPL server started on port 7888"
```

The default nREPL port is **7888**. You can change it via
`clojureOptions { nreplPort.set(9999) }` in your `build.gradle.kts`.

#### 3. Forward the port over ADB

ADB port forwarding creates a TCP tunnel from your development machine
to the device over USB:

```bash
adb forward tcp:7888 tcp:7888
```

This maps `localhost:7888` on your machine to port 7888 on the device.
Now any nREPL client connecting to `localhost:7888` is talking to the
device.

#### 4. Connect your editor

**Emacs + CIDER:**
```
M-x cider-connect
Host: localhost
Port: 7888
```

**VS Code + Calva:**
- Command Palette → "Calva: Connect to a Running REPL Server"
- Select "Generic" project type
- Enter: `localhost:7888`

**Command line (lein/reply):**
```bash
lein repl :connect 7888
```

**Command line (raw nREPL):**
```bash
clj -Sdeps '{:deps {nrepl/nrepl {:mvn/version "1.0.0"}}}' \
    -M -m nrepl.cmdline --connect --host localhost --port 7888
```

#### 5. Develop interactively

Once connected, you can evaluate Clojure code that runs on the device:

```clojure
;; Verify the connection
(+ 1 2)
;;=> 3

;; Check you're on Android
(System/getProperty "java.vm.name")
;;=> "Dalvik"

;; Require your activity's namespace
(require '[com.myapp.main-activity :as ma])

;; Modify the UI function, then hot-reload
(ma/reload-ui!)
```

The `reload-ui!` pattern calls `ClojureActivity.reloadUi()`, which invokes
`make-ui` on the UI thread and sets the result as the content view. Because
the Clojure code is evaluated dynamically (compiled to DEX on-device),
changes take effect immediately.

### Tips for REPL development

- **UI thread**: Android requires UI mutations on the main thread. Use
  `Handler`/`Looper` or neko's `neko.threading/on-ui` macro to post work
  to the UI thread from the REPL.
- **State**: Use atoms for app state. They survive code reloads. The
  `reload-ui!` function rebuilds the view tree from current state.
- **Logcat**: Use `neko.log/i`, `neko.log/d`, etc. Output appears in
  `adb logcat`.
- **Multiple devices**: If you have multiple devices/emulators, specify
  which one: `adb -s <serial> forward tcp:7888 tcp:7888`. Get serials
  with `adb devices`.
- **Wi-Fi debugging**: ADB port forwarding also works over Wi-Fi ADB
  (`adb connect <device-ip>:5555`), so you can develop wirelessly.

---

## 3. Gradle Build for F-Droid

[F-Droid](https://f-droid.org) is a free software app repository that
builds apps from source. F-Droid requires reproducible Gradle builds with
no proprietary dependencies.

### F-Droid compatibility checklist

The Clojure-Android Gradle plugin is designed to be F-Droid compatible:

- **Standard Gradle build** — F-Droid's build system (`fdroidserver`)
  invokes `gradle assembleRelease`, which works out of the box
- **No proprietary dependencies** — Clojure, neko, and the build plugin
  are all open source (EPL). The Android SDK build tools are included in
  F-Droid's build environment
- **Deterministic output** — AOT compilation is deterministic; the same
  sources produce the same `.class` files
- **No REPL in release (by default)** — The plugin excludes nREPL, dx,
  and the dynamic classloader from release builds unless overridden via
  `clojureOptions`

### F-Droid metadata

Create an F-Droid metadata file at `metadata/com.myapp.yml` in your
F-Droid data repo (or include it in your source repo for
[`fdroidserver`](https://f-droid.org/docs/Build_Metadata_Reference/)):

```yaml
Categories:
  - System
License: EPL-1.0
SourceCode: https://github.com/yourname/my-app
IssueTracker: https://github.com/yourname/my-app/issues

AutoName: My App

Builds:
  - versionName: '1.0'
    versionCode: 1
    commit: v1.0
    subdir: app
    gradle:
      - yes
    prebuild:
      # Install Clojure-Android toolchain to local Maven
      - cd .. && ./install-deps.sh

AllowedAPKSigningKeys: <your-key-fingerprint>
```

### Handling the local Maven dependencies

F-Droid builds happen in a clean environment without access to your local
`~/.m2`. You need to either:

**Option A: Publish to a public Maven repository**

Publish `android-clojure-plugin`, `runtime-core`, `runtime-repl`, and
`neko` to Maven Central or a public repository. Then F-Droid's build can
fetch them like any other dependency.

**Option B: Include as composite builds (recommended)**

The sample project's `settings.gradle.kts` already does this: it auto-clones
all dependencies into `build/deps/` and includes them as Gradle composite
builds. Everything builds from source with no external binary dependencies.

For F-Droid, add a prebuild step that publishes to Maven local:

```bash
./gradlew publishDepsToMavenLocal
```

This is the most F-Droid-friendly approach because all source is fetched
and built deterministically.

**Option C: Vendor JARs**

Include pre-built JARs in a `libs/` directory and reference them as
`files(...)` dependencies. This is the simplest approach but less clean.

### Release build configuration

> Targeting `minSdk` 24 is supported on an experimental basis — see
> [API 24 Release Builds](api-24-release.md) for the required
> desugaring setup and a full config sample.

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}

// REPL is excluded from release by default — no action needed.
// To opt in: clojureOptions { replEnabled.set(true) }
```

### ProGuard/R8 rules for Clojure

Clojure uses reflection and dynamic class loading. Add these R8 rules:

```proguard
# proguard-rules.pro

# Keep Clojure runtime core
-keep class clojure.** { *; }

# Keep AOT-compiled namespaces (adjust package to match yours)
-keep class com.myapp.** { *; }

# Keep neko UI mapping classes
-keep class neko.** { *; }

# Clojure uses reflection for protocol dispatch
-dontwarn clojure.**
-dontwarn neko.**
```

### Building the release APK

```bash
./gradlew :app:assembleRelease
```

The unsigned APK is at `app/build/outputs/apk/release/app-release-unsigned.apk`.
For F-Droid, the build server handles signing. For manual distribution, see
the next section.

---

## 4. Building for Google Play Store

Google Play requires APKs (or App Bundles) to be signed with your private
key and to meet current target SDK requirements.

### Signing configuration

#### Generate a keystore (one-time)

```bash
keytool -genkey -v \
    -keystore my-release-key.jks \
    -keyalg RSA -keysize 2048 \
    -validity 10000 \
    -alias my-app-key
```

Store this keystore securely. If you lose it, you cannot update your app
on Play Store.

#### Configure signing in Gradle

```kotlin
// app/build.gradle.kts
android {
    signingConfigs {
        create("release") {
            storeFile = file("../my-release-key.jks")
            storePassword = System.getenv("STORE_PASSWORD")
            keyAlias = "my-app-key"
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

Set passwords via environment variables to avoid checking secrets into
version control:

```bash
export STORE_PASSWORD="your-store-password"
export KEY_PASSWORD="your-key-password"
./gradlew :app:assembleRelease
```

#### Using an App Bundle (recommended for Play Store)

Google Play prefers Android App Bundles (`.aab`) over APKs. Bundles let
Play generate optimized APKs per device:

```bash
./gradlew :app:bundleRelease
```

Output: `app/build/outputs/bundle/release/app-release.aab`

### Play Store requirements checklist

1. **Target SDK**: `targetSdk = 35` (or current Play Store minimum). Set
   in `defaultConfig`.

2. **64-bit support**: Clojure runs on the JVM/ART so there are no
   native library concerns unless you add NDK dependencies.

3. **No dynamic code loading in release (by default)**: The Clojure-Android
   plugin excludes `runtime-repl` (which contains dx and nREPL) from release
   builds by default. Release APKs contain only AOT-compiled code, satisfying
   Play Store policies against downloading executable code. If you opt in
   to `replEnabled` or `dynamicCompilationEnabled` for release, ensure your
   use case complies with Play Store policies.

4. **Permissions**: Remove `INTERNET` permission if your app doesn't need
   network access in production. The REPL needs it, but it's not included
   in release builds. You can use a manifest placeholder or build-type
   specific manifest to add it only in debug:

   ```xml
   <!-- app/src/debug/AndroidManifest.xml -->
   <manifest xmlns:android="http://schemas.android.com/apk/res/android">
       <uses-permission android:name="android.permission.INTERNET" />
   </manifest>
   ```

5. **App signing**: Use [Play App Signing](https://support.google.com/googleplay/android-developer/answer/9842756)
   (Google manages the signing key, you upload with an upload key).

6. **ProGuard/R8**: Enable minification to reduce APK size. Clojure
   includes many namespaces; R8 tree-shaking removes unused code.

### Build and verify

```bash
# Build release bundle
./gradlew :app:bundleRelease

# Or build release APK
./gradlew :app:assembleRelease

# Verify the release APK has no REPL components or .clj source files
# (all should return empty — no matches)
zipinfo app/build/outputs/apk/release/app-release.apk | grep -i nrepl
zipinfo app/build/outputs/apk/release/app-release.apk | grep -i "AndroidDynamicClassLoader"
zipinfo app/build/outputs/apk/release/app-release.apk | grep '\.clj$'
```

### CI pipeline example

```bash
#!/bin/bash
set -euo pipefail

# Build debug (with REPL) and release (without)
./gradlew :app:assembleDebug :app:assembleRelease

# Verify release APK excludes REPL infrastructure
if zipinfo app/build/outputs/apk/release/app-release.apk | grep -q nrepl; then
    echo "ERROR: nREPL found in release APK"
    exit 1
fi

if zipinfo app/build/outputs/apk/release/app-release.apk | grep -q '\.clj$'; then
    echo "ERROR: .clj source files found in release APK"
    exit 1
fi

echo "Build OK: debug has REPL, release is clean"
```

---

## 5. Troubleshooting

### `NoClassDefFoundError` during AOT compilation

If `./gradlew assembleDebug` fails with an error like:

```
java.lang.NoClassDefFoundError: com/example/SomeClass
	at java.base/java.lang.Class.getDeclaredConstructors0(Native Method)
	...
> Task :app:compileDebugClojure FAILED
```

This typically means a **transitive dependency** is missing from the compile
classpath. The Clojure AOT compiler resolves classes eagerly — if your code
(or a library you depend on) references a class whose transitive dependency
is declared as runtime-only in its POM, the compiler will fail even though
the class would be available at runtime on the device.

**Example:** `androidplot-core` depends on `com.halfhp.fig:figlib`, but
declares it as a runtime-only dependency. During AOT compilation, the
Clojure compiler encounters androidplot classes that reference
`FigException` and fails with `NoClassDefFoundError:
com/halfhp/fig/FigException`.

**Fix:** Add the missing transitive dependency as an explicit
`implementation` dependency in your `app/build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.androidplot:androidplot-core:1.5.11")
    implementation("com.halfhp.fig:figlib:1.0.11")  // required at compile time by androidplot
}
```

**Diagnosing:** To find which dependency pulls in the missing class, run:

```bash
./gradlew app:dependencies --configuration debugRuntimeClasspath | grep <library-name>
```

Compare with the compile classpath to confirm it's missing there:

```bash
./gradlew app:dependencies --configuration debugCompileClasspath | grep <library-name>
```

If the library appears in the runtime classpath but not the compile
classpath, add it as an explicit `implementation` dependency.

---

## Quick Reference

| Task | Command |
|------|---------|
| Build debug APK | `./gradlew :app:assembleDebug` |
| Install debug on device | `./gradlew :app:installDebug` |
| Launch app via ADB | `adb shell am start -n com.myapp/.MainActivity` |
| Forward nREPL port | `adb forward tcp:7888 tcp:7888` |
| Connect REPL | `lein repl :connect 7888` (or editor connect) |
| Build release APK | `./gradlew :app:assembleRelease` |
| Build release bundle | `./gradlew :app:bundleRelease` |
| View device logs | `adb logcat -s ClojureREPL` |
| List connected devices | `adb devices` |
