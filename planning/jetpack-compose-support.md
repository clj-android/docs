# Jetpack Compose Support in Neko — Investigation & Plan

## Executive Summary

Jetpack Compose **cannot be used directly from Clojure**. The `@Composable` annotation is processed by a Kotlin compiler plugin that transforms function signatures at compile time (injecting `Composer` and `$changed` parameters, wrapping bodies with `startReplaceGroup`/`endGroup` calls). Clojure compiles to JVM bytecode without going through the Kotlin compiler, so it cannot produce valid Composable functions.

However, a **bridge approach** is viable: pre-compile Composable components in Kotlin, then host them in `ComposeView` (a regular Android `View`) from Clojure via Neko's existing View-based DSL.

## The Problem

The task asks specifically about `androidx.compose.material3.adaptive`, which provides adaptive layout components like `ListDetailPaneScaffold` that respond to window size classes. These are all `@Composable` functions — there is no View-based API for them.

## Why Direct Usage is Impossible

1. **Kotlin compiler plugin required**: The Compose compiler transforms `@Composable fun Screen()` into `fun Screen($composer: Composer, $changed: Int)` with additional runtime bookkeeping. Without this transformation, Compose functions cannot participate in the composition/recomposition lifecycle.

2. **@Composable calls @Composable**: Composable functions can only be called from other Composable functions. There is no entry point from plain JVM code except through `ComposeView.setContent {}`.

3. **No precedent**: No one has successfully used Jetpack Compose from Clojure, Scala, Groovy, or pure Java. Community consensus (2024-2026) is that Compose requires Kotlin.

4. **Manual Composer injection infeasible**: The `Composer` interface and slot table are complex internal structures with no public API for manual usage. They change between Compose versions.

## Option A: ComposeView Bridge (Recommended)

### Architecture

```
neko (Clojure library)
├── neko.ui.compose           — ComposeView wrapper for Neko DSL
│   registers :compose-view keyword

neko-compose (NEW Kotlin library)
├── Pre-built @Composable functions
├── Exposes components as factory functions callable from Java/Clojure
└── Depends on androidx.compose.material3.adaptive

App (Clojure + Kotlin)
├── Uses neko DSL for View-based UI
├── Embeds Compose content via :compose-view
└── Can define custom @Composable functions in src/kotlin/
```

### How It Works

**Kotlin side** (neko-compose library):
```kotlin
// Pre-built composable wrapped for Java/Clojure interop
object AdaptiveLayouts {
    @JvmStatic
    fun setAdaptiveListDetail(
        composeView: ComposeView,
        listContent: (ViewGroup) -> Unit,   // callbacks into Clojure
        detailContent: (ViewGroup) -> Unit
    ) {
        composeView.setContent {
            ListDetailPaneScaffold(
                listPane = { AndroidView(factory = { ctx -> FrameLayout(ctx).also { listContent(it) } }) },
                detailPane = { AndroidView(factory = { ctx -> FrameLayout(ctx).also { detailContent(it) } }) }
            )
        }
    }
}
```

**Clojure side** (neko DSL):
```clojure
;; Register ComposeView as a Neko widget
(kw/defelement :compose-view
  :classname androidx.compose.ui.platform.ComposeView
  :inherits :view
  :traits [:compose-content])

;; Usage in app
(make-ui [:compose-view {:compose-content
                         (fn [^ComposeView cv]
                           (AdaptiveLayouts/setAdaptiveListDetail
                             cv
                             (fn [container] (add-view container (make-ui [...])))
                             (fn [container] (add-view container (make-ui [...])))))}])
```

### What's Required

1. **New Kotlin module** (`neko-compose/`): Built with Gradle + Kotlin + Compose compiler plugin. Contains pre-built Composable wrappers that expose a Java-friendly API.

2. **Kotlin compiler in build**: The `android-clojure-plugin` or app's `build.gradle` must apply `org.jetbrains.kotlin.android` and `org.jetbrains.kotlin.plugin.compose`.

3. **Neko wrapper** (`neko.ui.compose`): A small Clojure namespace registering `:compose-view` with the mapping system and providing a `:compose-content` trait.

4. **Dependencies**: `androidx.compose.material3.adaptive:adaptive-layout`, `androidx.compose.ui:ui`, `androidx.compose.material3:material3`, plus the Kotlin stdlib.

### Tradeoffs

| Pro | Con |
|-----|-----|
| Uses official Android APIs | Requires Kotlin in the build pipeline |
| Follows Neko's existing optional-dependency pattern | Can't write Composables in Clojure |
| Pre-built components are reusable | Data passing across the Clojure/Kotlin boundary adds friction |
| `ComposeView` is a stable, supported API | Compose adds ~5-8 MB to APK size |
| Granular: apps only add Compose deps if they use it | Two UI paradigms in one app |

### Complexity Assessment

- **neko.ui.compose wrapper**: Small (~50 lines). Just registers ComposeView and a trait.
- **neko-compose Kotlin module**: Medium. Requires Gradle setup, Compose compiler, and wrapper functions for each component. The wrappers need careful API design for Clojure interop.
- **Build system changes**: The `android-clojure-plugin` already handles AARs. Apps would need to add the Kotlin and Compose plugins.
- **Sample app integration**: Needs Kotlin plugin added to `build.gradle.kts`, plus Compose dependencies.

## Option B: Kotlin Source Generation (Not Recommended)

Generate `.kt` files from Clojure macros at build time, then compile with the Kotlin compiler.

**Why not**: Adds enormous build complexity, breaks REPL-driven development, debugging is near-impossible, and the generated code would be fragile across Compose version changes. The bridge approach achieves the same result with far less complexity.

## Option C: Do Nothing (Viable)

The Material 3 adaptive components solve a narrow problem (responsive list/detail layouts). Neko's View-based DSL can achieve similar layouts using `DrawerLayout`, configuration-dependent resource files, or programmatic layout switching. Compose support could wait until the ecosystem matures or until a Kotlin-to-Clojure interop layer emerges.

**When this makes sense**: If no apps in the ecosystem actually need adaptive layouts today, the engineering cost of Option A is hard to justify.

## Recommendation

**Option A (ComposeView Bridge)** if there's demand for Material 3 adaptive components. It follows Neko's established pattern of optional dependencies and doesn't force Compose on apps that don't want it. Start with a minimal `neko-compose` module wrapping just `ListDetailPaneScaffold`, and expand based on usage.

**Option C (Do Nothing)** if adaptive layouts aren't an immediate need. The bridge approach will still be viable later — nothing about the current architecture precludes it.

## Implementation Steps (if proceeding with Option A)

1. Create `neko-compose/` Kotlin module with Gradle + Compose compiler setup
2. Write wrapper for `ListDetailPaneScaffold` with Java-friendly API
3. Add `neko.ui.compose` namespace to neko registering `:compose-view`
4. Add `compileOnly` dependency on `androidx.compose.ui:ui` to neko for `ComposeView` class
5. Update `android-clojure-plugin` if needed (likely no changes — AARs already handled)
6. Add sample demonstrating adaptive layout in the sample app
7. Document the Kotlin build requirements for apps using Compose widgets
