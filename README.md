# Clojure-Android Documentation

Documentation for the [Clojure-Android](https://github.com/clj-android) toolchain — build Android apps with Clojure.

## Contents

| Document | Description |
|---|---|
| [Developer Guide](GUIDE.md) | Complete guide: project structure, Gradle config, nREPL workflow, F-Droid and Play Store builds |
| [API 24 Release Builds](api-24-release.md) | *Experimental.* Dropping `minSdk` to 24 for release builds without the REPL |
| [Adding Dependencies](adding-dependencies.md) | Quick-start for Clojure developers: translating project.clj / deps.edn to Gradle, auto-fetch, Android gotchas |
| [Migration Guide](migration-guide.md) | Step-by-step migration from lein-droid to the modern Gradle build system |
| [Why Java Activity Shims](why-java-activity-shims.md) | Design rationale for thin Java activity classes instead of defactivity/gen-class |
| [Cache Bug](cache-bug.md) | Technical writeup of the Gradle cache invalidation issue and fix |

## Related Repositories

| Repository | Role |
|---|---|
| [sample](https://github.com/clj-android/sample) | Sample app — start here |
| [android-clojure-plugin](https://github.com/clj-android/android-clojure-plugin) | Gradle plugin for AOT compilation |
| [neko](https://github.com/clj-android/neko) | Clojure wrappers for Android APIs |
| [runtime-core](https://github.com/clj-android/runtime-core) | Core runtime (ClojureApp, ClojureActivity) |
| [runtime-repl](https://github.com/clj-android/runtime-repl) | nREPL server with on-device DEX compilation |
| [clojure-patched](https://github.com/clj-android/clojure-patched) | Clojure 1.12.0 with Android-aware classloader |

## License

Eclipse Public License 1.0
