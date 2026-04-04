# Release builds targeting API 24 (experimental)

> **Status: experimental.** The default supported floor is **API 26**. API 24
> works in practice but relies on core library desugaring and has not been
> exercised as broadly as the mainline configuration. Test on a real API 24
> device (or emulator) before shipping.
>
> For older devices (API 19–23), see the `legacy` branches of `clojure-patched`,
> `runtime-repl`, and `sample` — those require additional runtime workarounds
> (DEX 035 fixup, reflective `FunctionalInterface` lookup, multidex keep rules
> for the primary DEX) that are *not* needed at API 24+.

This doc is a focused companion to [GUIDE.md § Release builds](GUIDE.md#release-build-configuration).
It covers only the delta required to drop `minSdk` from 26 to 24 on a release
build that does **not** ship the REPL or the dynamic classloader.

## What the plugin already does for you

The `android-clojure` Gradle plugin distinguishes debug and release variants
automatically:

- `runtime-core` is always added.
- Stock `org.clojure:clojure` is always substituted with the patched fork.
  The fork's `RT.makeClassLoader` falls back to the stock
  `DynamicClassLoader` when `AndroidDynamicClassLoader` is absent, so it is
  safe for release.
- `runtime-repl` is added as **`compileOnly`** for non-debug variants — code
  that references `clj-android.repl.server` still compiles, but nothing
  REPL-related ships in the APK.
- When `clojureOptions.dynamicCompilationEnabled` is false (the default for
  non-debug variants), all `.clj` source files are stripped from packaging.
  Only the AOT-compiled `.class` files are dexed into the APK.

So a release APK built from `main` with the default `clojureOptions` already
contains no `AndroidDynamicClassLoader`, no nREPL server, and no `.clj`
sources. Dropping to API 24 is a matter of (a) the `minSdk`, (b) enabling
core library desugaring, and (c) standard R8 keep rules.

## Why API 24 is the experimental floor

API 24 (Nougat) introduced native support for DEX format 037, which is what
the bundled `dalvik-dx` library emits. That removes the single biggest
runtime compatibility issue we hit on API 19–23 (the `"Unrecognized version
number in .dex: 0 3 6"` error from ART rejecting the version byte).

However, Clojure 1.12's AOT output references types from `java.time`,
`java.nio.file`, and `java.util.function` that were not added to the Android
platform until API 26. At API 24 these must be supplied by
[core library desugaring](https://developer.android.com/studio/write/java8-support-table).
The `desugar_jdk_libs_nio` artifact covers the full set; the smaller
`desugar_jdk_libs` variant may work if your code path does not touch
`java.nio.file`, but has not been verified against the full Clojure core.

## `app/build.gradle.kts`

```kotlin
plugins {
    id("com.android.application")
    id("com.goodanser.clj-android.android-clojure")
}

android {
    namespace = "com.example.clojuredroid"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.clojuredroid"
        minSdk = 24                    // experimental floor; default is 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
        // Clojure's runtime plus your AOT output will typically exceed
        // 65k methods before R8 runs. Native multidex is free on API 21+.
        multiDexEnabled = true
    }

    signingConfigs {
        create("release") {
            // Read from ~/.gradle/gradle.properties or the environment —
            // never commit credentials.
            val keystore = System.getenv("CLJDROID_KEYSTORE")
                ?: (findProperty("cljdroid.keystore") as String?)
            if (keystore != null) {
                storeFile = file(keystore)
                storePassword = System.getenv("CLJDROID_KEYSTORE_PASSWORD")
                    ?: (findProperty("cljdroid.keystore.password") as String?)
                keyAlias = System.getenv("CLJDROID_KEY_ALIAS")
                    ?: (findProperty("cljdroid.key.alias") as String?)
                keyPassword = System.getenv("CLJDROID_KEY_PASSWORD")
                    ?: (findProperty("cljdroid.key.password") as String?)
            }
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro",
            )
            signingConfig = signingConfigs.getByName("release")
        }
    }

    compileOptions {
        // Required at minSdk < 26. Clojure 1.12's AOT output references
        // java.time.*, java.nio.file.*, and java.util.function types that
        // are not available on API 24-25 without desugaring. Remove this
        // block once minSdk >= 26.
        isCoreLibraryDesugaringEnabled = true
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }
}

clojureOptions {
    warnOnReflection.set(true)
    // Explicit is nice even though these match the defaults for release:
    replEnabled.set(false)
    dynamicCompilationEnabled.set(false)
}

dependencies {
    implementation("org.clojure:clojure:1.12.0")       // substituted by plugin
    implementation("com.goodanser.clj-android:neko:5.0.0-SNAPSHOT")
    implementation("com.google.android.material:material:1.12.0")

    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs_nio:2.1.4")
}
```

## `app/proguard-rules.pro`

```proguard
# ---- Clojure on Android: R8 keep rules ----------------------------------
#
# Clojure resolves classes reflectively by name everywhere (RT.classForName,
# Compiler.compile, var metadata). Without these rules R8 will strip AOT
# output that looks dead and the app will NoClassDefFoundError at startup.

# Clojure runtime internals.
-keep class clojure.lang.** { *; }
-keep class clojure.java.api.** { *; }
-keep class clojure.core.** { *; }
-dontwarn clojure.**

# Our runtime shim.
-keep class com.goodanser.clj_android.runtime.** { *; }

# neko — uses reflection for Android interop.
-keep class neko.** { *; }
-dontwarn neko.**

# Every Clojure namespace AOT-compiles to an `__init` class that is loaded
# by name via RT.classForName. These MUST be kept. Compiled fn classes are
# reachable from the __init's constants table, so standard tree-shaking
# picks them up once __init is rooted.
-keep class **__init { *; }
-keepclassmembers class ** {
    public static final clojure.lang.Var const__*;
}

# Your application namespaces — keep anything loaded by name from Java.
# Add one line per namespace ClojureApp.loadNamespaces(...) references.
# Munging rule: dots → slashes, hyphens → underscores.
# Example: com.example.my-app.core  →  com/example/my_app/core__init
-keep class com.example.clojuredroid.core__init { *; }
# ...etc

# Patched-Clojure optional holder probed reflectively on older APIs.
-keep class clojure.lang.RT$DesugarPrefixes { *; }
-dontwarn clojure.lang.RT$DesugarPrefixes

# Desugar shims.
-dontwarn j$.**

# Android framework classes that Clojure AOT may reference but aren't on
# the build classpath.
-dontwarn java.lang.management.**
-dontwarn java.beans.**
-dontwarn javax.**
-dontwarn sun.**
```

## Building and verifying

```bash
# Generate a keystore once (skip if you already have one).
keytool -genkey -v -keystore ~/.android/release.jks \
    -keyalg RSA -keysize 2048 -validity 10000 -alias cljdroid

export CLJDROID_KEYSTORE=~/.android/release.jks
export CLJDROID_KEYSTORE_PASSWORD=...
export CLJDROID_KEY_ALIAS=cljdroid
export CLJDROID_KEY_PASSWORD=...

./gradlew :app:assembleRelease
```

Sanity-check the output APK:

```bash
APK=app/build/outputs/apk/release/app-release.apk

# Should be empty — no REPL classes or stubs shipped:
unzip -l "$APK" | grep -iE 'AndroidDynamic|nrepl|clj_android/repl'

# Should be empty — no .clj sources shipped (stripped by the plugin):
unzip -l "$APK" | grep '\.clj$'

# Sanity check: AOT classes for your namespaces are present:
unzip -l "$APK" | grep 'com/example/clojuredroid.*__init'
```

Install and watch logcat on an API 24 device or emulator:

```bash
adb install -r "$APK"
adb logcat -s ClojureApp AndroidRuntime
```

## Gotchas

1. **R8 will strip your namespaces on first try.** You will see
   `ClassNotFoundException: …__init` on launch. Add one `-keep class
   …__init` rule per namespace loaded from Java, re-build, repeat. Expect
   two or three iterations the first time.

2. **No `eval`, no `load-string`, no `load-file`.** Without
   `AndroidDynamicClassLoader` on the classpath, any code path that invokes
   the Clojure compiler at runtime will fail with a
   `ClassFormatError` or similar. Audit dependencies for macros that call
   `eval` unexpectedly.

3. **Reflection warnings become load-time failures** in a minified build
   if R8 strips a reflectively accessed method. Resolve `warn-on-reflection`
   findings during development rather than chasing them in release.

4. **Desugaring adds ~1.5 MB** to the APK. If you are confident your AOT
   output does not reference `java.nio.file`, try swapping
   `desugar_jdk_libs_nio` for plain `desugar_jdk_libs` and re-testing. R8
   will fail the build with "Type is not present" if a required shim is
   missing.

5. **First-launch cost.** AOT-compiled Clojure still pays a significant
   static-initializer cost (RT, Compiler, `core.clj` load). Use
   `ClojureApp.loadNamespacesAsync` from a splash activity rather than
   blocking the main thread.

## Going further: API 19–23

The `legacy` branches of `clojure-patched`, `runtime-repl`, and `sample`
carry additional workarounds for Android 4.4 through 6.0:

- DEX version byte rewrite to `035` in `AndroidDynamicClassLoader`
- Reflective `java.lang.FunctionalInterface` lookup in `Compiler.java`
- Temp-file DEX loading on API < 26 (no `InMemoryDexClassLoader`)
- Primary-DEX multidex keep rules for Clojure runtime classes
- Android-compatible `nrepl.socket` stubs that avoid `ProtocolFamily`

Those branches are **not** merged to `main` and are maintained on a
best-effort basis. If you do not specifically need pre-Nougat support,
stay on `main` with `minSdk = 24` or higher.
