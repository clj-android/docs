# Adding Clojure Dependencies

A quick-start reference for Clojure developers used to `project.clj` or
`deps.edn` who want to add libraries to a Clojure-Android project.

## The short version

Add a line to the `dependencies` block in `app/build.gradle.kts`:

```kotlin
dependencies {
    implementation("org.clojure:clojure:1.12.0")
    implementation("com.goodanser.clj-android:neko:5.0.0-SNAPSHOT")

    // Add libraries exactly like you would in deps.edn — group:artifact:version
    implementation("hiccup:hiccup:2.0.0-RC3")
    implementation("metosin:malli:0.16.4")
    implementation("clj-http:clj-http:3.13.0")
}
```

That's it. Run `./gradlew :app:assembleDebug` and Gradle fetches, caches,
and bundles everything.

## Mapping from project.clj / deps.edn

The coordinates are the same — just different syntax:

| project.clj | deps.edn | build.gradle.kts |
|---|---|---|
| `[hiccup "2.0.0-RC3"]` | `hiccup/hiccup {:mvn/version "2.0.0-RC3"}` | `implementation("hiccup:hiccup:2.0.0-RC3")` |
| `[metosin/malli "0.16.4"]` | `metosin/malli {:mvn/version "0.16.4"}` | `implementation("metosin:malli:0.16.4")` |
| `[org.clojure/data.json "2.5.1"]` | `org.clojure/data.json {:mvn/version "2.5.1"}` | `implementation("org.clojure:data.json:2.5.1")` |

The rule: Maven coordinates are `group:artifact:version` separated by
colons.  When the Clojure library has no explicit group (like `hiccup`), the
group equals the artifact name — same convention as Clojars.

## Where artifacts come from

Repositories are declared once in `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        mavenLocal()                                    // ~/.m2/repository
        google()                                        // Android libraries
        mavenCentral()                                  // Most Java/Clojure libs
        maven { url = uri("https://clojars.org/repo") } // Clojure-specific libs
    }
}
```

With these four repositories, virtually any Clojure library published to
Clojars or Maven Central resolves automatically.  There is no separate
`lein deps` or `clj -P` step — Gradle fetches and caches dependencies on
the first build.

## What the plugin adds for you

You do **not** declare these — the `android-clojure` Gradle plugin injects
them automatically based on the build variant:

| Dependency | When added | Why |
|---|---|---|
| `runtime-core` | All builds | `ClojureApp` base class, runtime init |
| `runtime-repl` | Debug only | nREPL server, `AndroidDynamicClassLoader`, dx |
| `clojure-patched` | Debug only | Substitutes stock Clojure with Android-aware classloader |

You declare stock `org.clojure:clojure:1.12.0` in your dependencies.  In
debug builds the plugin silently swaps it for the patched version so the
REPL works; release builds use stock Clojure unmodified.

## How Clojure sources in dependencies work

Clojure libraries typically ship `.clj` files as resources inside their
JAR.  In a Clojure-Android project:

1. **Your app's** `.clj` files (under `app/src/main/clojure/`) are
   **AOT-compiled** to `.class` files by the plugin, then dexed into the
   APK.
2. **Library** `.clj` files are **not** AOT-compiled.  They are packaged
   into the APK as resources and loaded at runtime by Clojure's `require`.

This means `(:require [some.library :as lib])` in your namespace works the
same as it does on the JVM — Clojure finds the `.clj` file on the
classpath and compiles it on first load.  In debug builds this happens via
`AndroidDynamicClassLoader`; in release builds, transitive requires
triggered during your app's AOT compilation are compiled ahead of time too.

## Scope keywords (implementation vs compileOnly)

| Gradle scope | project.clj equivalent | Use when |
|---|---|---|
| `implementation` | `:dependencies` | Normal dependency — compiled and bundled in APK |
| `compileOnly` | `:provided` | Needed at compile time only (e.g. annotation processors) |
| `debugImplementation` | `:dev` profile `:dependencies` | Debug builds only |
| `releaseImplementation` | `:release` profile `:dependencies` | Release builds only |

Most Clojure libraries use `implementation`.

## Excluding transitive dependencies

```kotlin
// Exclude a transitive dependency
implementation("some-group:some-lib:1.0.0") {
    exclude(group = "org.clojure", module = "clojurescript")
}
```

Equivalent to `:exclusions [org.clojure/clojurescript]` in project.clj.

## Using a SNAPSHOT or local library

**SNAPSHOT from a remote repo** — just use the SNAPSHOT version string.
Gradle checks for updates according to its caching policy (default: 24h,
override with `--refresh-dependencies`):

```kotlin
implementation("com.goodanser.clj-android:neko:5.0.0-SNAPSHOT")
```

**Library built locally** — publish it to your local Maven repository, then
reference it normally:

```bash
# In the library's project directory:
./gradlew publishToMavenLocal   # Gradle project
# or: lein install              # Leiningen project
# or: clj -T:build install      # tools.build project
```

Make sure `mavenLocal()` is listed in your repositories (it is by default
in the sample app).

**Library as a sibling checkout (composite build)** — for active
development on a dependency, add it to `settings.gradle.kts`:

```kotlin
// At the bottom of settings.gradle.kts
includeBuild("../my-library")
```

Gradle builds it from source and substitutes it for the published artifact.
This is how the sample app consumes neko, runtime-core, and the other
clojure-android components during development.

## Auto-fetch of clojure-android components

The sample app's `settings.gradle.kts` automatically clones the
clojure-android dependencies (neko, runtime-core, runtime-repl,
clojure-patched, android-clojure-plugin) from GitHub if they are not
already checked out as sibling directories.  On the first
`./gradlew assembleDebug`, you will see:

```
Cloning neko from https://github.com/clj-android/neko.git ...
Cloning runtime-core from https://github.com/clj-android/runtime-core.git ...
...
```

These are shallow clones (`--depth 1`).  If cloning fails (e.g. no
network), Gradle falls back to resolving artifacts from `mavenLocal()` and
the configured remote repositories.

For your own projects, you can either:
- Copy the auto-clone pattern from the sample's `settings.gradle.kts`
- Use `includeBuild()` pointing at local checkouts
- Publish the toolchain to mavenLocal with `./gradlew publishToMavenLocal`
  in each component

## Android-specific gotchas

**Not every Clojure library works on Android.** Libraries that depend on
these will fail at runtime:

- `javax.swing`, `java.awt` — no GUI toolkit on Android
- `java.lang.management` — not available on ART
- Unix domain sockets (`java.net.UnixDomainSocketAddress`) — not available
  before API 33, and nREPL's Unix socket support is stubbed out by
  `runtime-repl`
- Desktop filesystem assumptions (`/tmp`, `user.home` semantics)

Pure-Clojure data processing libraries (spec, malli, data.json,
core.async, etc.) generally work without issues.

**APK size** — every dependency adds to the APK.  Clojure itself is ~4 MB
of classes.  Use `isMinifyEnabled = true` with R8 in release builds to
tree-shake unused code.

**Startup time** — loading many Clojure namespaces at startup affects app
launch time.  AOT compilation (which the plugin does for your code)
mitigates this, but transitive runtime requires from libraries still incur
load time.  Profile with `adb shell am start -W` if startup feels slow.
