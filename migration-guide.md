# Migrating an Old Clojure-Android App to the Current System

This guide walks through updating a lein-droid-based Clojure-Android project
to the current Gradle-based build system. It covers every change: build
tooling, project layout, dependencies, activity patterns, REPL setup, and
release configuration.

## Overview of What Changed

| Aspect | Old | New |
|--------|-----|-----|
| Build tool | Leiningen + lein-droid 0.4.6 | Gradle 8.7+ / AGP 8.9+ |
| Clojure | `com.goodanser.clj-android/clojure` 1.7.0 fork | Stock `org.clojure:clojure` 1.12.0 |
| neko | `neko/neko` 4.0.0-alpha5 | `com.goodanser.clj-android:neko` 5.0.0-SNAPSHOT |
| nREPL | `org.clojure/tools.nrepl` 0.2.10 | `nrepl:nrepl` 1.0.0 (auto-included) |
| Min Android | ~API 14 (Ice Cream Sandwich) | API 26 (Oreo 8.0) |
| Target Android | API 18 (Jelly Bean) | API 35 |
| Java level | 1.6 | 11 |
| Activity pattern | `defactivity` macro (gen-class) | Thin Java shims + Clojure functions |
| Application class | `neko.App` | `com.goodanser.clj_android.runtime.ClojureApp` |
| Dexing | SDK `dx` binary (manual) | D8 (handled by AGP) |
| Shrinking | ProGuard (manual) | R8 (handled by AGP) |
| MultiDex | Explicit `com.android.support/multidex` | Native (API 21+) |
| Dynamic classloader | `DalvikDynamicClassLoader` (in Clojure fork) | `AndroidDynamicClassLoader` (in runtime-repl) |

---

## Step 1: Set Up the New Toolchain

Before converting your app, install the Clojure-Android toolchain to your
local Maven repository. The sample project includes a task that builds and
publishes all dependencies in the correct order:

```bash
cd sample
./gradlew publishDepsToMavenLocal
```

This clones all dependencies into `build/deps/` (if not already present),
then runs `publishToMavenLocal` in each one. Alternatively, build each
component individually from its own repo:

```bash
cd clojure-patched        && ./gradlew publishToMavenLocal
cd ../android-clojure-plugin && ./gradlew publishToMavenLocal
cd ../runtime-core        && ./gradlew publishToMavenLocal
cd ../runtime-repl        && ./gradlew publishToMavenLocal
cd ../neko                && ./gradlew publishToMavenLocal
```

Requirements:
- **JDK 17+** (JDK 22+ also works with the `--enable-native-access` flag)
- **Android SDK** with Build Tools and API 26+
- **Gradle 8.7+** (the Gradle wrapper is recommended)

---

## Step 2: Replace project.clj with Gradle Build Files

An old project has a single `project.clj`. The new system uses three Gradle
files at the project root plus one per module.

### Old: project.clj

```clojure
(defproject com.example/myapp "1.0.0"
  :source-paths ["src/clojure" "src"]
  :java-source-paths ["src/java"]
  :javac-options ["-target" "1.6" "-source" "1.6"]
  :plugins [[lein-droid "0.4.6"]]
  :dependencies [[com.goodanser.clj-android/clojure "1.7.0-RC1" :use-resources true]
                 [neko/neko "4.0.0-alpha5"]]
  :profiles {:dev
             {:dependencies [[org.clojure/tools.nrepl "0.2.10"]]
              :android {:aot :all-with-unused
                        :rename-manifest-package "com.example.myapp.debug"}}
             :release
             {:android {:aot :all
                        :build-type :release}}}
  :android {:target-version 18
            :aot-exclude-ns ["clojure.parallel" "clojure.core.reducers"]
            :dex-opts ["-JXmx4096M" "--incremental"]
            :manifest-options {:app-name "@string/app_name"}})
```

### New: Three files replace this

**settings.gradle.kts** (project root):

```kotlin
pluginManagement {
    repositories {
        mavenLocal()
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositories {
        mavenLocal()
        google()
        mavenCentral()
        maven { url = uri("https://clojars.org/repo") }
    }
}

rootProject.name = "my-app"
include(":app")
```

**build.gradle.kts** (project root):

```kotlin
plugins {
    id("com.android.application") version "8.9.0" apply false
    id("com.goodanser.clj-android.android-clojure") version "0.5.0-SNAPSHOT" apply false
}
```

**app/build.gradle.kts** (app module):

```kotlin
plugins {
    id("com.android.application")
    id("com.goodanser.clj-android.android-clojure")
}

android {
    namespace = "com.example.myapp"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }

    packaging {
        resources {
            pickFirsts += listOf("nrepl/socket.clj", "nrepl/socket/dynamic.clj")
        }
    }
}

clojureOptions {
    warnOnReflection.set(true)
}

dependencies {
    implementation("org.clojure:clojure:1.12.0")
    implementation("com.goodanser.clj-android:neko:5.0.0-SNAPSHOT")
}
```

**gradle.properties** (project root, needed for JDK 22+):

```properties
org.gradle.jvmargs=--enable-native-access=ALL-UNNAMED
```

### What you no longer configure manually

The following `project.clj` / `:android` map settings have no equivalent
because the Gradle plugin and AGP handle them automatically:

| Old setting | Disposition |
|-------------|-------------|
| `:sdk-path` | Set via `ANDROID_HOME` env var or `local.properties` |
| `:build-tools-version` | Managed by AGP |
| `:dex-opts` | D8 handles dexing; no manual flags needed |
| `:aot-exclude-ns` | Use `clojureOptions { aotExcludeNamespaces.set(listOf(...)) }` if needed |
| `:aot :all` / `:all-with-unused` | All detected `.clj` files are AOT-compiled by default |
| `:multi-dex true` | Native since minSdk 21; no configuration needed |
| `:rename-manifest-package` | Use `applicationIdSuffix` in build types instead |
| `:manifest-options` | Use `resValue` or standard Android resource merging |
| Profiles (`:dev`, `:release`, `:lean`) | Use Gradle `buildTypes` and `productFlavors` |
| `:use-resources true` | Not needed; resource handling is automatic |

### What the plugin handles automatically

When you apply `com.goodanser.clj-android.android-clojure`, the plugin:

1. Registers `src/{sourceSet}/clojure/` as source directories
2. Creates `compile{Variant}Clojure` tasks that AOT-compile `.clj` to `.class`
3. Wires compiled classes into AGP's dex pipeline
4. Adds `runtime-core` to all variants
5. Adds `runtime-repl` (nREPL + dynamic classloader + dx) to debug variants only
6. Substitutes stock Clojure with the patched version in debug builds

---

## Step 3: Restructure the Source Layout

### Old layout

```
myapp/
├── project.clj
├── AndroidManifest-template.xml
├── res/
│   └── values/strings.xml
├── src/
│   └── clojure/
│       └── com/example/myapp/
│           └── main.clj              # defactivity here
└── src/
    └── java/
        └── com/example/myapp/
            └── (nothing, or R.java)
```

### New layout

```
myapp/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── app/
│   ├── build.gradle.kts
│   └── src/main/
│       ├── AndroidManifest.xml       # (not a template)
│       ├── clojure/
│       │   └── com/example/myapp/
│       │       └── main_activity.clj # activity logic (matches MainActivity)
│       ├── java/
│       │   └── com/example/myapp/
│       │       └── MainActivity.java # thin ClojureActivity shim
│       └── res/
│           └── values/
│               └── strings.xml
└── gradle/
    └── wrapper/                      # Gradle wrapper (check in to VCS)
```

Key differences:
- Sources move under `app/src/main/`
- `AndroidManifest.xml` is a real file, not a Clostache template
- Java shim files are added alongside Clojure sources
- Gradle wrapper files should be checked in

### Moving files

```bash
mkdir -p app/src/main/clojure app/src/main/java app/src/main/res

# Move Clojure sources
cp -r src/clojure/* app/src/main/clojure/

# Move Java sources (if any)
cp -r src/java/* app/src/main/java/

# Move resources
cp -r res/* app/src/main/res/
```

---

## Step 4: Replace defactivity with Java Shims

This is the most significant code change. The old system used neko's
`defactivity` macro to generate Activity classes via `gen-class`. The new
system uses thin Java classes that call into Clojure.

### Why the change

1. **AOT/Manifest coupling eliminated** -- the manifest no longer depends on
   Clojure's compiler output
2. **REPL hot-reload works** -- the Activity stays fixed while Clojure UI
   code reloads dynamically
3. **ART compatibility** -- statically compiled Java shims are safer entry
   points than dynamically generated Clojure classes
4. **Better developer ergonomics** -- standard Android patterns, clearer
   stack traces

### Old: defactivity

```clojure
(ns com.example.myapp.main
  (:require [neko.activity :refer [defactivity set-content-view!]]
            [neko.debug :refer [*a]]
            [neko.threading :refer [on-ui]]
            [neko.find-view :refer [find-view]]
            [neko.notify :refer [toast]]))

(defactivity com.example.myapp.MainActivity
  :key :main

  (onCreate [this bundle]
    (.superOnCreate this bundle)
    (neko.debug/keep-screen-on this)
    (on-ui
      (set-content-view! (*a)
        [:linear-layout {:orientation :vertical
                         :layout-width :fill
                         :layout-height :wrap}
         [:edit-text {:id ::user-input
                      :hint "Type text here"
                      :layout-width :fill}]
         [:button {:text "Press me"
                   :on-click (fn [_] (toast "Hello!" :short))}]]))))
```

### New: ClojureActivity shim + Clojure namespace

**Java shim** (`app/src/main/java/com/example/myapp/MainActivity.java`):

```java
package com.example.myapp;

import com.goodanser.clj_android.runtime.ClojureActivity;

/**
 * ClojureActivity maps this class to the Clojure namespace
 * com.example.myapp.main-activity (CamelCase → kebab-case by convention).
 */
public class MainActivity extends ClojureActivity {
    // All behavior is defined in the Clojure namespace
    // com.example.myapp.main-activity
}
```

`ClojureActivity` (from `runtime-core`) automatically:
- Derives a Clojure namespace from the class name (`MainActivity` →
  `main-activity`)
- Requires the namespace on `onCreate`
- Delegates lifecycle methods (`on-create`, `on-resume`, etc.) to functions
  in that namespace
- Provides `reloadUi()` which calls `make-ui` on the UI thread
- Tracks instances for REPL access via `ClojureActivity.getInstance(ns)`

Override `getClojureNamespace()` in the Java class to use a custom namespace
instead of the convention-based one.

**Clojure namespace** (`app/src/main/clojure/com/example/myapp/main_activity.clj`):

```clojure
(ns com.example.myapp.main-activity
  (:require [neko.ui :as ui]
            [neko.find-view :refer [find-view]]
            [neko.log :as log])
  (:import android.app.Activity
           com.goodanser.clj_android.runtime.ClojureActivity))

(defn make-ui
  "Builds the UI tree. Called by on-create and ClojureActivity.reloadUi()."
  [^Activity activity]
  (ui/make-ui activity
    [:linear-layout {:id-holder true
                     :orientation :vertical
                     :padding [32 32 32 32]}
     [:text-view {:text "Hello from Clojure!"
                  :text-size [24 :sp]}]
     [:button {:text "Press me"
               :on-click (fn [_] (log/i "Button clicked"))}]]))

(defn on-create
  "Called automatically by ClojureActivity when the activity is created."
  [^Activity activity saved-instance-state]
  (.setContentView activity (make-ui activity)))

(defn reload-ui!
  "Hot-reload the UI from the REPL."
  []
  (when-let [activity (ClojureActivity/getInstance
                        "com.example.myapp.main-activity")]
    (.reloadUi ^ClojureActivity activity)))
```

### Conversion pattern

For each `defactivity` in your old code:

1. Create a Java file extending `ClojureActivity` with the same class name
   referenced in your manifest — the Java class body can be empty
2. Create a Clojure namespace matching the convention
   (`MyActivity` → `my-activity` in the same package)
3. Define `(on-create [activity bundle])` for initialization and
   `(make-ui [activity])` for REPL-driven hot-reload
4. Other lifecycle functions (`on-resume`, `on-pause`, etc.) are optional

### Multiple activities

Repeat the pattern for each activity. Each gets its own Java shim and
Clojure namespace, matched by naming convention:

```
java/.../MainActivity.java     → clojure/.../main_activity.clj
java/.../SettingsActivity.java → clojure/.../settings_activity.clj
java/.../DetailActivity.java   → clojure/.../detail_activity.clj
```

---

## Step 5: Update AndroidManifest.xml

### Old: template with Clostache placeholders

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="{{package}}"
    android:versionCode="{{version-code}}"
    android:versionName="{{version-name}}">

    <uses-sdk android:minSdkVersion="{{min-sdk-version}}"
              android:targetSdkVersion="{{target-sdk-version}}" />

    <application android:label="{{app-name}}"
                 android:name="neko.App">
        <activity android:name="{{activity}}"
                  android:label="{{app-name}}">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

### New: standard Android manifest

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
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

Key changes:
- **No Clostache templates** -- `package`, `versionCode`, `versionName`,
  `minSdkVersion`, `targetSdkVersion` are all set in `build.gradle.kts`
- **Application class**: `neko.App` → `com.goodanser.clj_android.runtime.ClojureApp`
- **Activity names**: Reference your Java shim classes, not gen-class output
- **`android:exported="true"`**: Required since API 31 for activities with
  intent filters
- **`INTERNET` permission**: Needed for nREPL over ADB; can be moved to
  `src/debug/AndroidManifest.xml` if you don't need it in release

---

## Step 6: Update Clojure Namespace References

### nREPL namespace change

If your code references nREPL namespaces directly (e.g., for middleware
configuration), update them:

```clojure
;; Old
(:require [clojure.tools.nrepl.server :as nrepl])

;; New
(:require [nrepl.server :as nrepl])
```

In practice, you probably don't need to reference nREPL directly. The
`ClojureApp` application class starts it automatically in debug builds.

### neko internal namespace rename

The `neko.-utils` namespace was renamed to `neko.internal` because AGP's
resource merger excludes files with `_` prefixes:

```clojure
;; Old (if you referenced it directly)
(:require [neko.-utils :as utils])

;; New
(:require [neko.internal :as utils])
```

Most apps don't import this namespace directly.

### neko.notify API changes

The notification API was rewritten for modern Android (API 26+ requires
notification channels):

```clojure
;; Old (deprecated APIs)
(neko.notify/notification {:icon R$drawable/ic_launcher
                           :ticker-text "Hello"
                           :content-title "Title"
                           :content-text "Body"})

;; New (notification channels, PendingIntent.FLAG_IMMUTABLE)
;; The neko.notify namespace handles channel creation automatically.
;; See neko source for the updated API.
```

### Removed: neko.debug/*a and defactivity :key

The `*a` dynamic var and activity `:key` registry from `neko.debug` are tied
to the `defactivity` pattern. With Java shims, you manage the activity
reference yourself (see the `currentInstance` pattern in the Java shim).

### Removed: res/import-all

The old `(res/import-all)` macro imported Android R class fields. In the new
system, access resources through standard Android APIs in the Java shim, or
use `(.getResources activity)` from Clojure.

---

## Step 7: Update the REPL Workflow

### Old workflow

```bash
lein droid doall          # Build, install, run
lein droid forward-port   # Set up ADB port forwarding
lein droid repl           # Connect via REPLy
```

### New workflow

```bash
./gradlew :app:installDebug                    # Build and install
adb shell am start -n com.example.myapp/.MainActivity  # Launch app
adb forward tcp:7888 tcp:7888                  # Forward nREPL port
```

Then connect with your editor:

```
# Emacs + CIDER
M-x cider-connect → localhost:7888

# VS Code + Calva
Calva: Connect to a Running REPL Server → localhost:7888

# Command line
lein repl :connect 7888
```

Key differences:
- The nREPL server starts **automatically** when `ClojureApp` is the
  application class (no need for neko.init flags or dev profile setup)
- Default port is **7888** (configurable via `clojureOptions { nreplPort.set(N) }`)
- nREPL is automatically excluded from release builds
- Dynamic classloading uses `InMemoryDexClassLoader` (API 26+) instead of
  `DexFile.loadDex()` with temp files

### Hot-reload from the REPL

```clojure
;; Verify connection
(+ 1 2) ;;=> 3
(System/getProperty "java.vm.name") ;;=> "Dalvik"

;; Reload UI after making changes
(require '[com.example.myapp.main-activity :as ma])
(ma/reload-ui!)
```

---

## Step 8: Update Release Build Configuration

### Old: lein-droid release profile

```clojure
:profiles {:release
           {:android {:keystore-path "/path/to/keystore"
                      :key-alias "mykey"
                      :aot :all
                      :build-type :release}}}
```

```bash
lein with-profile release droid doall
```

### New: Gradle release build type

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

```bash
./gradlew :app:assembleRelease
```

Add ProGuard/R8 rules for Clojure in `app/proguard-rules.pro`:

```proguard
-keep class clojure.** { *; }
-keep class com.example.myapp.** { *; }
-keep class neko.** { *; }
-dontwarn clojure.**
-dontwarn neko.**
```

The Gradle plugin automatically excludes nREPL, the dynamic classloader,
and the dx library from release builds. No manual configuration is needed.

### Old: Skummet/lean profile

The old `:lean` profile with Skummet minification is no longer needed. R8
(handled by AGP) provides equivalent or better tree-shaking. Remove any
Skummet-related configuration.

---

## Step 9: Remove Old Build Artifacts

Once migration is complete, remove lein-droid artifacts:

```bash
rm project.clj
rm -rf target/
rm -rf .lein-*
rm AndroidManifest-template.xml  # if using template
```

And remove from `.gitignore` any lein-specific entries:
```
# Remove these:
/target
/classes
/.lein-*
/.nrepl-port
```

Add Gradle entries:
```
# Add these:
/.gradle/
/build/
/app/build/
local.properties
```

---

## Complete Migration Checklist

```
[ ] Install toolchain (clojure-patched, runtime-core, runtime-repl, neko, plugin)
[ ] Create settings.gradle.kts
[ ] Create root build.gradle.kts
[ ] Create app/build.gradle.kts
[ ] Create gradle.properties (if JDK 22+)
[ ] Move Clojure sources to app/src/main/clojure/
[ ] Move Java sources to app/src/main/java/
[ ] Move resources to app/src/main/res/
[ ] Replace each defactivity with a Java shim + Clojure namespace
[ ] Update AndroidManifest.xml (remove template syntax, update app class)
[ ] Update nREPL namespace references (clojure.tools.nrepl → nrepl)
[ ] Update neko.-utils → neko.internal (if referenced)
[ ] Update notification code for API 26+ channels (if using neko.notify)
[ ] Add Gradle wrapper (gradle wrapper --gradle-version 8.7)
[ ] Test debug build: ./gradlew :app:installDebug
[ ] Test REPL connection: adb forward tcp:7888 tcp:7888 → connect
[ ] Test release build: ./gradlew :app:assembleRelease
[ ] Verify release APK has no nREPL: zipinfo ... | grep nrepl
[ ] Remove project.clj and lein-droid artifacts
[ ] Update .gitignore
```

---

## Side-by-Side: Old vs. New Build Commands

| Task | Old (lein-droid) | New (Gradle) |
|------|-----------------|--------------|
| Full build + install + run | `lein droid doall` | `./gradlew :app:installDebug` |
| Compile only | `lein droid compile` | `./gradlew :app:compileDebugClojure` |
| Build APK | `lein droid apk` | `./gradlew :app:assembleDebug` |
| Install on device | `lein droid install` | `./gradlew :app:installDebug` |
| Release build | `lein with-profile release droid doall` | `./gradlew :app:assembleRelease` |
| Connect REPL | `lein droid repl` | `adb forward tcp:7888 tcp:7888` + editor connect |
| Run tests | `lein droid local-test` | (Robolectric via standard Gradle test tasks) |
| Verbose output | `DEBUG=1 lein droid ...` | `./gradlew ... --info` |

---

## Dependency Mapping

| Old (project.clj) | New (build.gradle.kts) |
|-------------------|----------------------|
| `[com.goodanser.clj-android/clojure "1.7.0-RC1"]` | `implementation("org.clojure:clojure:1.12.0")` |
| `[neko/neko "4.0.0-alpha5"]` | `implementation("com.goodanser.clj-android:neko:5.0.0-SNAPSHOT")` |
| `[org.clojure/tools.nrepl "0.2.10"]` (dev) | Automatic (runtime-repl includes nrepl 1.0.0) |
| `[com.android.support/multidex "1.0.0"]` | Not needed (native on minSdk 26) |

Note: You declare `org.clojure:clojure:1.12.0` (stock Clojure) in your
dependencies. The Gradle plugin automatically substitutes the patched
version (`com.goodanser.clj-android:clojure:1.12.0-1`) in debug builds for REPL
support. Release builds use stock Clojure with no modifications.

---

## Troubleshooting

**App crashes on startup with `Reflector` error**
The patched Clojure fixes a `canAccess` crash on ART. Make sure you're
using the toolchain's patched Clojure (installed via
`clojure-patched/gradlew publishToMavenLocal`). The Gradle plugin handles
substitution automatically in debug builds; for release, stock Clojure
1.12.0 works because `Reflector` only crashes when `canAccess` is actually
invoked.

**nREPL not starting**
Check that `android:name="com.goodanser.clj_android.runtime.ClojureApp"` is set
in your manifest's `<application>` tag. Check logcat:
`adb logcat -s ClojureApp`.

**`_`-prefixed `.clj` files missing from APK**
AGP's resource merger excludes files starting with `_`. Rename them (e.g.,
`_utils.clj` → `internal.clj`) and update the namespace.

**AAR dependency classes not found during AOT compilation**
The plugin extracts `classes.jar` from AARs automatically. If you have
non-standard AAR dependencies that fail, check the plugin's compile task
logs with `--info`.

**`UnixDomainSocketAddress` crash**
The `runtime-repl` module includes TCP-only stubs that replace nREPL's Unix
domain socket code. Ensure the `pickFirsts` packaging option is set in your
`build.gradle.kts` (see Step 2).
