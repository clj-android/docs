# Why Activities Use Java Shims Instead of `defactivity`

## What changed

**Before:** `defactivity` (in `neko/src/clojure/neko/activity.clj`) used Clojure's `gen-class` to emit an entire Activity subclass as AOT-compiled bytecode — no Java needed.

**After:** Activities are thin Java classes that extend `ClojureActivity` from `runtime-core`. `ClojureActivity` automatically requires a Clojure namespace derived from the class name (e.g. `MainActivity` → `main-activity`) and delegates lifecycle methods to functions in that namespace. The Java shim can be an empty class — all logic lives in Clojure.

## Why the change was made

**1. AOT/Manifest coupling.** `defactivity` required AOT compilation to produce the class that `AndroidManifest.xml` references. This created a fragile dependency: the manifest must know the exact class name at build time, tightly coupling it to Clojure's compiler output.

**2. REPL hot-reload.** The shim architecture lets the Activity stay fixed while Clojure UI code is reloaded dynamically. With `defactivity`, the Activity *was* the Clojure code — you couldn't hot-swap it without restarting the app.

**3. Dynamic classloading on Android.** When REPL-evaluated Clojure code emits bytecode, it goes through `AndroidDynamicClassLoader` → DEX conversion → runtime loading. Activities are expected to exist at APK load time, not be dynamically generated. Java shims eliminate this conflict.

**4. ART runtime strictness.** ART (replacing Dalvik since Android 5.0) has stricter class verification. Statically-compiled Java shims are safer entry points than dynamically-generated Clojure classes.

**5. Gradle/AGP compatibility.** Modern Android Gradle Plugin expects standard Java/Kotlin entry points. Generated Clojure classes caused classloader confusion.

**6. Developer ergonomics.** Java shims are standard Android — easier to understand, debug (better stack traces), and maintain. The `defactivity` macro's `gen-class` internals were opaque to most Android developers.

In short: Java handles the stable entry points that Android requires, while Clojure handles the dynamic logic and UI — playing to each language's strengths.
