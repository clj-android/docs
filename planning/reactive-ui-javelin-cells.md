# Done - lightweight Neko reactive library was the winner here

# Reactive UI: Javelin Cells + Neko

## Goal

UI elements that reference reactive values auto-update when those values change — the spreadsheet/dataflow model. Inspired by [Javelin cells](https://github.com/hoplon/javelin) from the Hoplon project.

## Javelin: Won't Work Directly

**Javelin is ClojureScript-only.** The source is `.cljs` files with JavaScript interop (`js/setInterval`, `.push`, `.shift`). There is no JVM Clojure implementation. A JVM port was attempted in 2015 (GitHub issue #25) but never completed. This is a dead end as a direct dependency.

## What Neko Already Has

Neko already uses the `atom + add-watch` pattern in two places, establishing a precedent:

1. **`neko.ui.adapters/ref-adapter`** (`neko/src/clojure/neko/ui/adapters.clj:11-45`) — Watches an atom and calls `on-ui (.setData adapter ...)` when data changes. This is exactly the reactive pattern we want, but limited to ListView adapters.

2. **`neko.data.shared-prefs/bind-atom-to-prefs`** (`neko/src/clojure/neko/data/shared_prefs.clj:37-53`) — Syncs an atom to SharedPreferences via `add-watch`.

Neko also has the building blocks needed:
- **`neko.ui/config`** — Updates any widget's attributes after creation: `(config widget :text "hello" :enabled false)`
- **`neko.threading/on-ui`** — Runs code on the Android UI thread
- **`neko.find-view/find-view`** — Retrieves widgets by ID

## Approaches

### Approach 1: Plain Clojure Atoms (Simplest, No New Dependencies)

No library needed. Clojure atoms already implement `IDeref`, `swap!`, `reset!`, and `add-watch` — the exact same primitives Javelin cells are built on. Write a thin binding layer:

```clojure
(ns neko.ui.reactive
  (:require [neko.ui :refer [config]]
            [neko.threading :refer [on-ui]]))

(defn bind!
  "Binds a widget attribute to an atom. When the atom changes,
   the widget is updated on the UI thread."
  [widget attr atom]
  (on-ui (config widget attr @atom))  ; set initial value
  (add-watch atom [::bind widget attr]
    (fn [_ _ old new]
      (when (not= old new)
        (on-ui (config widget attr new))))))

;; Usage:
(def counter (atom 0))
(def label (make-ui activity [:text-view {:id ::label}]))
(bind! label :text counter)  ; label auto-updates when counter changes

(swap! counter inc)  ; UI updates automatically
```

For computed/derived values (Javelin's formula cells), you chain watches:

```clojure
(def first-name (atom "John"))
(def last-name (atom "Doe"))
(def full-name (atom ""))

;; Manual formula: keep full-name in sync
(let [update! #(reset! full-name (str @first-name " " @last-name))]
  (add-watch first-name ::full update!)
  (add-watch last-name ::full update!)
  (update!))

(bind! label :text full-name)
```

**Pros:** Zero dependencies, idiomatic Clojure, works today.
**Cons:** Formula cells are manual/boilerplate-heavy. No automatic dependency tracking. No transactional propagation.

### Approach 2: Minimal Cell Implementation (Medium Effort)

Implement a ~100-line cell library directly in neko that gives Javelin-like semantics on JVM Clojure. The core idea: a `cell` is an atom, and a `formula` is an atom that auto-recomputes by tracking which cells were derefed during its body evaluation.

```clojure
(ns neko.reactive
  (:require [neko.ui :refer [config]]
            [neko.threading :refer [on-ui]]))

(def ^:dynamic *tracking* nil)  ; Set of atoms derefed during formula eval

(deftype Cell [state watches]
  clojure.lang.IDeref
  (deref [this]
    (when *tracking* (swap! *tracking* conj this))
    @state)
  clojure.lang.IRef
  (addWatch [this key f] (add-watch state key f) this)
  (removeWatch [this key] (remove-watch state key) this))

(defn cell [initial-value]
  (Cell. (atom initial-value) (atom {})))

(defn cell= [f]
  "Create a formula cell. f is a zero-arg fn whose body derefs other cells."
  (let [deps (atom #{})
        result (cell nil)
        recompute (fn []
                    (let [tracking (atom #{})
                          new-val (binding [*tracking* tracking] (f))
                          new-deps @tracking
                          old-deps @deps]
                      ;; Update dependency watches
                      (doseq [d (remove new-deps old-deps)]
                        (remove-watch d [::formula result]))
                      (doseq [d (remove old-deps new-deps)]
                        (add-watch d [::formula result]
                          (fn [_ _ _ _] (recompute))))
                      (reset! deps new-deps)
                      (reset! (.state result) new-val)))]
    (recompute)
    result))

;; Then the UI binding:
(defn bind! [widget attr cell]
  (on-ui (config widget attr @cell))
  (add-watch cell [::bind widget attr]
    (fn [_ _ _ new] (on-ui (config widget attr new)))))

;; Usage - now with automatic dependency tracking:
(def first-name (cell "John"))
(def last-name (cell "Doe"))
(def full-name (cell= #(str @first-name " " @last-name)))

(bind! label :text full-name)
(reset! (.state first-name) "Jane")  ;; label updates to "Jane Doe"
```

**Pros:** Javelin-like semantics, automatic dependency tracking, no external deps, tailored to Android/neko.
**Cons:** Need to implement and test carefully (glitch-free propagation, cycle detection, memory management/unbinding).

### Approach 3: Kenny Tilton's Matrix Library (Full-Featured, Proven)

[Matrix](https://github.com/kennytilton/matrix) is the JVM Clojure equivalent of Javelin — same spreadsheet-cell paradigm, actively maintained, explicitly supports JVM Clojure. It's the Clojure port of the Common Lisp "Cells" library.

```clojure
;; project.clj
:dependencies [[com.tiltontec/matrix "..."]
               [neko/neko "4.0.0-alpha5"]]
```

**Pros:** Mature, battle-tested, handles glitch-free propagation, cycle detection, lazy evaluation, change batching.
**Cons:** Another dependency to carry on Android (DEX size matters). May need adaptation for Android UI thread constraints. The API may feel unfamiliar compared to Javelin.

## Key Technical Challenges (All Approaches)

### 1. UI Thread Safety

All Android view mutations must happen on the main thread. Every watch callback that touches a widget must be wrapped in `on-ui`. This is the #1 gotcha.

### 2. Lifecycle Management

Cells/watches must be cleaned up when activities are destroyed, or you get memory leaks and crashes (updating views that no longer exist). Need an `unbind!` or `dispose!` that removes watches, ideally tied to `onDestroy`.

### 3. View Tree Binding

The cleanest API would let you reference cells directly in the `make-ui` tree:

```clojure
(make-ui activity
  [:linear-layout {}
    [:text-view {:text (cell= #(str "Count: " @counter))}]])
```

This requires modifying `apply-trait` / `apply-attributes` to detect cell values and auto-bind them. Each trait would need to check if its value is a cell, and if so, set up a watch instead of just applying the value once.

### 4. Batching

Multiple cell updates should ideally batch into a single UI update pass, not trigger N redraws. Javelin's `dosync` handles this; a plain atom approach doesn't.

## Recommendation

**Start with Approach 1** (plain atoms + `bind!`) — it requires zero new dependencies, fits naturally with what neko already does in `ref-adapter`, and covers 80% of use cases. If formula cells prove essential, **graduate to Approach 2** (a minimal cell implementation inside neko itself), which gives Javelin semantics without external dependencies — important on Android where DEX size and startup time matter.

The key integration point in neko would be modifying the trait system to recognize reactive references in attribute values and automatically set up watch-based bindings, rather than requiring manual `bind!` calls after `make-ui`.

## Javelin API Reference (for Design Parity)

For reference, the Javelin API that a JVM implementation would want to match:

| Javelin | Purpose |
|---------|---------|
| `(cell x)` | Input cell with initial value |
| `(cell= expr)` | Formula cell, auto-recomputes |
| `(defc name val)` | `def` an input cell |
| `(defc= name expr)` | `def` a formula cell |
| `@cell` | Read value |
| `(reset! cell v)` | Set input cell |
| `(swap! cell f)` | Update input cell |
| `(add-watch cell k f)` | Watch for changes |
| `(cell? x)` | Type check |
| `(formula? x)` | Type check |
| `(dosync & body)` | Batch updates, defer propagation |
| `(lens cell f)` | Bidirectional formula cell |
| `(cell-map f cell)` | Map over reactive collection |

**Namespace:** `javelin.core`
**Latest version:** `[hoplon/javelin "3.9.3"]` (ClojureScript only, on Clojars)
