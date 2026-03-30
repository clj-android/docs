# F-Droid Reproducible Builds

Apps built with clj-android can be published on F-Droid with reproducible
builds. This document covers the required configuration.

## Reproducible Build Support

The clj-android build system produces deterministic output:

- Clojure AOT compilation generates identical bytecode across builds (via a
  patched `Compiler.java` with deterministic `LocalBinding.hashCode()`)
- Namespace compilation order is sorted, eliminating filesystem ordering
  dependencies

## F-Droid Metadata

The F-Droid source scanner flags certain patterns in the dependency source
trees that are cloned into `build/deps/`. Add these fields to your app's
F-Droid metadata file to handle them:

```yaml
Builds:
  - versionName: ...
    versionCode: ...
    gradle:
      - yes
    scandelete:
      # Gradle compiles the plugin at configuration time, creating binary
      # artifacts before the scanner runs. They are recompiled during the
      # actual build.
      - build/deps/android-clojure-plugin/build/
    scanignore:
      # InMemoryDexClassLoader triggers a substring match on "DexClassLoader".
      # This class is only used in debug builds for REPL support.
      - build/deps/runtime-repl/src/main/java/clojure/lang/AndroidDynamicClassLoader.java
```

### Why these entries are needed

**`scandelete` for android-clojure-plugin/build/**: The android-clojure-plugin
is an included Gradle build. Gradle must compile it to apply the plugin during
project configuration, which happens before the scanner runs. The `scandelete`
removes the compiled artifacts; they are regenerated when the actual build
starts.

**`scanignore` for AndroidDynamicClassLoader.java**: This file imports
`dalvik.system.InMemoryDexClassLoader` (an Android SDK class for loading DEX
from memory). F-Droid's scanner flags the substring "DexClassLoader" within
that class name. The class is only included in debug builds via the
`runtime-repl` dependency and is not present in release APKs.

## Signing

Use `apksigner` from build-tools **34.x**, not 35.0.0+. The newer
`apksigner` produces signature padding that `apksigcopier` (used by
F-Droid to verify reproducible builds) cannot transplant correctly,
causing verification failure even when the unsigned APKs are
byte-identical.

```bash
~/Android/Sdk/build-tools/34.0.0/apksigner sign \
  -ks your-keystore.jks --ks-key-alias your-alias \
  app/build/outputs/apk/release/your-app.apk
```

## Baseline Profile

Disable ART profile generation — it is non-deterministic across build
environments. In `app/build.gradle.kts`:

```kotlin
tasks.whenTaskAdded {
    if (name.contains("ArtProfile")) {
        enabled = false
    }
}
```

## Gradle Wrapper JARs

F-Droid automatically removes `gradle-wrapper.jar` files from cloned
dependencies. No metadata configuration is needed for these.
