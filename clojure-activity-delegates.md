# ClojureActivity Delegate Functions

`ClojureActivity` automatically bridges Android `Activity` callbacks into a
Clojure namespace. Every function below is optional -- define only what you need.

## Namespace Convention

The Clojure namespace is derived from the Java subclass name:

```
com.example.foo.MainActivity  ->  com.example.foo.main-activity
com.example.foo.NekoActivity  ->  com.example.foo.neko-activity
```

Override `getClojureNamespace()` in Java to use a custom namespace.

## Return Values

Functions marked **returns boolean** use Clojure truthiness: `nil` and `false`
are falsy, everything else is truthy. When such a function is not defined, the
default `super` behavior is used.

---

## Lifecycle

| Clojure fn | Java method | Description |
|---|---|---|
| `(on-create [activity ^Bundle bundle])` | [`onCreate`][onCreate] | Activity created. If absent, falls back to `make-ui`. |
| `(on-restart [activity])` | [`onRestart`][onRestart] | Activity restarting after stop (not destroyed). |
| `(on-start [activity])` | [`onStart`][onStart] | Activity becoming visible. |
| `(on-resume [activity])` | [`onResume`][onResume] | Activity in foreground, interactive. |
| `(on-pause [activity])` | [`onPause`][onPause] | Activity losing focus. |
| `(on-stop [activity])` | [`onStop`][onStop] | Activity no longer visible. |
| `(on-destroy [activity])` | [`onDestroy`][onDestroy] | Activity being destroyed. |
| `(on-save-instance-state [activity ^Bundle out-state])` | [`onSaveInstanceState`][onSaveInstanceState] | Save transient state before destruction. |
| `(on-restore-instance-state [activity ^Bundle saved-state])` | [`onRestoreInstanceState`][onRestoreInstanceState] | Restore state from bundle. |
| `(make-ui [activity])` | -- | Return a `View`; used by `reloadUi()` and as `on-create` fallback. |

[onCreate]: https://developer.android.com/reference/android/app/Activity#onCreate(android.os.Bundle)
[onRestart]: https://developer.android.com/reference/android/app/Activity#onRestart()
[onStart]: https://developer.android.com/reference/android/app/Activity#onStart()
[onResume]: https://developer.android.com/reference/android/app/Activity#onResume()
[onPause]: https://developer.android.com/reference/android/app/Activity#onPause()
[onStop]: https://developer.android.com/reference/android/app/Activity#onStop()
[onDestroy]: https://developer.android.com/reference/android/app/Activity#onDestroy()
[onSaveInstanceState]: https://developer.android.com/reference/android/app/Activity#onSaveInstanceState(android.os.Bundle)
[onRestoreInstanceState]: https://developer.android.com/reference/android/app/Activity#onRestoreInstanceState(android.os.Bundle)

## Intent & Navigation

| Clojure fn | Java method | Description |
|---|---|---|
| `(on-activity-result [activity request-code result-code ^Intent data])` | [`onActivityResult`][onActivityResult] | Result from `startActivityForResult`. |
| `(on-new-intent [activity ^Intent intent])` | [`onNewIntent`][onNewIntent] | New intent delivered (singleTop/singleTask, deep links). |
| `(on-back-pressed [activity])` | [`onBackPressed`][onBackPressed] | Back button pressed. If defined, replaces default back behavior entirely; your fn must handle navigation. |
| `(on-request-permissions-result [activity request-code ^String[] permissions ^int[] grant-results])` | [`onRequestPermissionsResult`][onRequestPermissionsResult] | Runtime permission request result (API 23+). |

[onActivityResult]: https://developer.android.com/reference/android/app/Activity#onActivityResult(int,%20int,%20android.content.Intent)
[onNewIntent]: https://developer.android.com/reference/android/app/Activity#onNewIntent(android.content.Intent)
[onBackPressed]: https://developer.android.com/reference/android/app/Activity#onBackPressed()
[onRequestPermissionsResult]: https://developer.android.com/reference/android/app/Activity#onRequestPermissionsResult(int,%20java.lang.String[],%20int[])

## Menus

| Clojure fn | Java method | Description |
|---|---|---|
| `(on-create-options-menu [activity ^Menu menu])` | [`onCreateOptionsMenu`][onCreateOptionsMenu] | Populate the options menu. **Returns boolean** -- truthy to display the menu. |
| `(on-prepare-options-menu [activity ^Menu menu])` | [`onPrepareOptionsMenu`][onPrepareOptionsMenu] | Update menu before display. **Returns boolean** -- truthy to show. |
| `(on-options-item-selected [activity ^MenuItem item])` | [`onOptionsItemSelected`][onOptionsItemSelected] | Options menu item tapped. **Returns boolean** -- truthy if consumed. |
| `(on-create-context-menu [activity ^ContextMenu menu ^View view menu-info])` | [`onCreateContextMenu`][onCreateContextMenu] | Populate a context (long-press) menu. |
| `(on-context-item-selected [activity ^MenuItem item])` | [`onContextItemSelected`][onContextItemSelected] | Context menu item tapped. **Returns boolean** -- truthy if consumed. |

[onCreateOptionsMenu]: https://developer.android.com/reference/android/app/Activity#onCreateOptionsMenu(android.view.Menu)
[onPrepareOptionsMenu]: https://developer.android.com/reference/android/app/Activity#onPrepareOptionsMenu(android.view.Menu)
[onOptionsItemSelected]: https://developer.android.com/reference/android/app/Activity#onOptionsItemSelected(android.view.MenuItem)
[onCreateContextMenu]: https://developer.android.com/reference/android/app/Activity#onCreateContextMenu(android.view.ContextMenu,%20android.view.View,%20android.view.ContextMenu.ContextMenuInfo)
[onContextItemSelected]: https://developer.android.com/reference/android/app/Activity#onContextItemSelected(android.view.MenuItem)

## Configuration & Memory

| Clojure fn | Java method | Description |
|---|---|---|
| `(on-configuration-changed [activity ^Configuration config])` | [`onConfigurationChanged`][onConfigurationChanged] | Device configuration changed (rotation, locale, etc.). |
| `(on-low-memory [activity])` | [`onLowMemory`][onLowMemory] | System is running low on memory. |
| `(on-trim-memory [activity level])` | [`onTrimMemory`][onTrimMemory] | System requests memory trimming at the given level. |

[onConfigurationChanged]: https://developer.android.com/reference/android/app/Activity#onConfigurationChanged(android.content.res.Configuration)
[onLowMemory]: https://developer.android.com/reference/android/app/Activity#onLowMemory()
[onTrimMemory]: https://developer.android.com/reference/android/app/Activity#onTrimMemory(int)

## Window

| Clojure fn | Java method | Description |
|---|---|---|
| `(on-window-focus-changed [activity has-focus])` | [`onWindowFocusChanged`][onWindowFocusChanged] | Window gained or lost focus. `has-focus` is boolean. |
| `(on-attached-to-window [activity])` | [`onAttachedToWindow`][onAttachedToWindow] | Activity's window attached to the window manager. |
| `(on-detached-from-window [activity])` | [`onDetachedFromWindow`][onDetachedFromWindow] | Activity's window detached from the window manager. |

[onWindowFocusChanged]: https://developer.android.com/reference/android/app/Activity#onWindowFocusChanged(boolean)
[onAttachedToWindow]: https://developer.android.com/reference/android/app/Activity#onAttachedToWindow()
[onDetachedFromWindow]: https://developer.android.com/reference/android/app/Activity#onDetachedFromWindow()

## Multi-window

| Clojure fn | Java method | Description |
|---|---|---|
| `(on-multi-window-mode-changed [activity in-multi-window ^Configuration config])` | [`onMultiWindowModeChanged`][onMultiWindowModeChanged] | Entered or exited multi-window mode. `in-multi-window` is boolean. |
| `(on-picture-in-picture-mode-changed [activity in-pip ^Configuration config])` | [`onPictureInPictureModeChanged`][onPictureInPictureModeChanged] | Entered or exited picture-in-picture. `in-pip` is boolean. |
| `(on-user-leave-hint [activity])` | [`onUserLeaveHint`][onUserLeaveHint] | User pressed Home or otherwise hinted at leaving. Useful for auto-entering PiP. |

[onMultiWindowModeChanged]: https://developer.android.com/reference/android/app/Activity#onMultiWindowModeChanged(boolean,%20android.content.res.Configuration)
[onPictureInPictureModeChanged]: https://developer.android.com/reference/android/app/Activity#onPictureInPictureModeChanged(boolean,%20android.content.res.Configuration)
[onUserLeaveHint]: https://developer.android.com/reference/android/app/Activity#onUserLeaveHint()

## REPL Helpers

These are Java methods on `ClojureActivity`, not Clojure delegate functions:

| Method | Description |
|---|---|
| `ClojureActivity/getInstance(namespace)` | Get the live activity instance for a namespace (for REPL use). |
| `.reloadUi` | Re-invoke `make-ui` and set the result as content view. Thread-safe. |
| `ClojureActivity/reloadAll` | Reload UI for all tracked activity instances. |
