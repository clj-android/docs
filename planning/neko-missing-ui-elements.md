# Neko UI Elements Audit & Implementation Plan

Audit of Android UI elements not yet implemented in Neko's UI DSL (`mapping.clj`, `traits.clj`), with a prioritized implementation plan.

**Target API level:** 18 (Android 4.3 Jelly Bean)

## Currently Supported (16 elements)

| Keyword | Android Class | Traits |
|---------|--------------|--------|
| `:view` | View | click, drag, touch, key, focus, padding, layout-params |
| `:view-group` | (inherits :view) | container, id-holder |
| `:button` | Button | (inherits :text-view) |
| `:check-box` | CheckBox | (inherits :text-view) |
| `:text-view` | TextView | text, text-size, on-editor-action |
| `:edit-text` | EditText | input type values |
| `:image-view` | ImageView | image, scale-type |
| `:linear-layout` | LinearLayout | (inherits :view-group) |
| `:relative-layout` | RelativeLayout | (inherits :view-group) |
| `:frame-layout` | FrameLayout | (inherits :view-group) |
| `:list-view` | ListView | on-item-click |
| `:grid-view` | GridView | on-item-click |
| `:scroll-view` | ScrollView | (inherits :view) |
| `:search-view` | SearchView | on-query-text |
| `:progress-bar` | ProgressBar | (inherits :view) |
| `:web-view` | WebView | (inherits :view) |
| `:action-bar` | ActionBar | display-options (separate module) |

## Already-implemented but unwired listener wrappers

These exist in `neko.listeners.adapter-view` but have **no corresponding traits**:
- `on-item-long-click-call` -- not wired to any element
- `on-item-selected-call` -- not wired to any element (critical for Spinner)

---

## Tier 1: High-priority missing widgets (API <= 18, no new deps)

Standard `android.widget.*` classes available at the target API level. Sorted by usage frequency:

| # | Keyword | Android Class | Why essential | New traits/listeners needed |
|---|---------|--------------|---------------|----------------------------|
| 1 | `:spinner` | Spinner | Dropdown selector -- one of the most common form elements | `:on-item-selected` trait (listener already exists) |
| 2 | `:switch` | Switch (API 14+) | Toggle control, ubiquitous in settings | `:on-checked-change` listener + trait |
| 3 | `:toggle-button` | ToggleButton | On/off toggle | `:on-checked-change` (shared) |
| 4 | `:radio-button` | RadioButton | Radio selection | `:on-checked-change` (shared) |
| 5 | `:radio-group` | RadioGroup | Groups radio buttons | `:on-checked-change` (group-level) |
| 6 | `:seek-bar` | SeekBar | Slider/range input | `:on-seek-bar-change` listener + trait |
| 7 | `:rating-bar` | RatingBar | Star ratings | `:on-rating-bar-change` listener + trait |
| 8 | `:auto-complete-text-view` | AutoCompleteTextView | Text input with suggestions | Inherits :edit-text |
| 9 | `:horizontal-scroll-view` | HorizontalScrollView (API 3+) | Horizontal scrolling | Inherits :frame-layout |
| 10 | `:table-layout` | TableLayout | Table-based layouts | Inherits :linear-layout |
| 11 | `:table-row` | TableRow | Row within TableLayout | Inherits :linear-layout |
| 12 | `:date-picker` | DatePicker | Date selection | `:on-date-changed` listener + trait |
| 13 | `:time-picker` | TimePicker | Time selection | `:on-time-changed` listener + trait |
| 14 | `:number-picker` | NumberPicker (API 11+) | Numeric selection | `:on-value-change` listener + trait |
| 15 | `:grid-layout` | GridLayout (API 14+) | Flexible grid | Grid layout-params trait |
| 16 | `:space` | Space (API 14+) | Invisible spacer | None (inherits :view) |
| 17 | `:view-flipper` | ViewFlipper | Animated view switching | Inherits :view-group |
| 18 | `:checked-text-view` | CheckedTextView | Checkable text (used in list items) | Inherits :text-view |
| 19 | `:video-view` | VideoView | Video playback | `:on-prepared`, `:on-completion` |
| 20 | `:chronometer` | Chronometer | Timer display | Inherits :text-view |
| 21 | `:expandable-list-view` | ExpandableListView | Grouped list items | Several group/child click listeners |

## Tier 2: Missing traits for EXISTING elements

These are arguably **more impactful** than new widgets -- they make the current elements actually usable:

| # | Trait | Applies to | What it does |
|---|-------|-----------|--------------|
| 1 | `:background` / `:background-color` | :view | Set background color or drawable -- currently impossible via DSL |
| 2 | `:visibility` | :view | Show/hide elements (value namespace exists, no trait) |
| 3 | `:enabled` | :view | Enable/disable interaction |
| 4 | `:hint` | :edit-text | Placeholder text -- essential for any form |
| 5 | `:input-type` | :edit-text | Keyboard type (number, email, password) |
| 6 | `:text-color` | :text-view | Text color -- surprisingly missing |
| 7 | `:gravity` | :text-view | Text alignment within widget |
| 8 | `:max-lines` / `:single-line` | :text-view | Line count control | **DONE** |
| 9 | `:checked` | :check-box (and CompoundButton subclasses) | Set initial checked state |
| 10 | `:on-checked-change` | :check-box | React to checkbox toggle -- currently missing |
| 11 | `:on-item-long-click` | :list-view, :grid-view | Long-press on list items (listener exists, unwired) |
| 12 | `:on-item-selected` | :spinner (new), :list-view | Selection change (listener exists, unwired) |
| 13 | `:progress` / `:max` | :progress-bar | Set progress value -- currently ProgressBar is a no-op |
| 14 | `:content-description` | :view | Accessibility -- important for production apps |
| 15 | `:alpha` | :view (API 11+) | Transparency | **DONE** |
| 16 | `:min-height` / `:min-width` | :view | Minimum dimensions | **DONE** |
| 17 | `:on-scroll` | :list-view | Scroll event listener | **DONE** |

## Tier 3: Support library widgets (require new dependencies)

These require adding `com.android.support` dependencies to the project. High value but higher effort:

| # | Keyword | Library artifact | Why valuable |
|---|---------|-----------------|--------------|
| 1 | `:recycler-view` | recyclerview-v7 | Modern list/grid -- replaces ListView |
| 2 | `:card-view` | cardview-v7 | Material design cards |
| 3 | `:toolbar` | appcompat-v7 | Modern ActionBar replacement |
| 4 | `:drawer-layout` | support-v4 | Navigation drawer |
| 5 | `:view-pager` | support-v4 | Swipeable pages |
| 6 | `:swipe-refresh-layout` | support-v4 | Pull-to-refresh |
| 7 | `:floating-action-button` | design | FAB button |
| 8 | `:tab-layout` | design | Tab navigation |
| 9 | `:coordinator-layout` | design | Coordinated scrolling |
| 10 | `:app-bar-layout` | design | App bar with scrolling behavior |
| 11 | `:nested-scroll-view` | support-v4 | Nested scrolling |

---

## Implementation Plan

### Phase 1: Wire up existing unwired code + essential missing traits (highest ROI)

1. **New listener file: `listeners/compound_button.clj`**
   - `on-checked-change-call` / `on-checked-change` -- CompoundButton.OnCheckedChangeListener

2. **New listener file: `listeners/seek_bar.clj`**
   - `on-seek-bar-change-call` -- SeekBar.OnSeekBarChangeListener (onProgressChanged, onStartTrackingTouch, onStopTrackingTouch)

3. **New listener file: `listeners/rating_bar.clj`**
   - `on-rating-bar-change-call` -- RatingBar.OnRatingBarChangeListener

4. **New traits in `traits.clj`**:
   - `:background` / `:background-color` -- call `.setBackgroundColor` or `.setBackground`/`.setBackgroundDrawable`
   - `:visibility` -- call `.setVisibility` with View.VISIBLE/INVISIBLE/GONE
   - `:enabled` -- call `.setEnabled`
   - `:hint` -- call `.setHint` on EditText
   - `:input-type` -- call `.setInputType` on EditText
   - `:text-color` -- call `.setTextColor` on TextView
   - `:gravity` -- call `.setGravity` on TextView
   - `:checked` -- call `.setChecked` on CompoundButton
   - `:on-checked-change` -- wire to new listener
   - `:on-item-long-click` -- wire to existing `adapter-view/on-item-long-click-call`
   - `:on-item-selected` -- wire to existing `adapter-view/on-item-selected-call`
   - `:progress` -- call `.setProgress` / `.setMax` on ProgressBar
   - `:content-description` -- call `.setContentDescription`

5. **Update existing mappings in `mapping.clj`**:
   - Add new traits to `:view` -- `:background`, `:visibility`, `:enabled`, `:content-description`
   - Add new traits to `:text-view` -- `:text-color`, `:gravity`
   - Add new traits to `:edit-text` -- `:hint`, `:input-type`
   - Add new traits to `:check-box` -- `:checked`, `:on-checked-change`
   - Add new traits to `:progress-bar` -- `:progress`
   - Add `:on-item-long-click` to `:list-view` and `:grid-view`

### Phase 2: Add Tier 1 widget mappings (no new dependencies)

6. **Add to `mapping.clj`** (imports + `keyword-mapping` + `reverse-mapping`):
   - `:spinner` inherits `:view-group`, traits: `[:on-item-selected]`
   - `:switch` inherits `:check-box` (-> CompoundButton -> TextView)
   - `:toggle-button` inherits `:check-box`
   - `:radio-button` inherits `:check-box`
   - `:radio-group` inherits `:linear-layout`
   - `:seek-bar` inherits `:progress-bar`, traits: `[:on-seek-bar-change]`
   - `:rating-bar` inherits `:view`, traits: `[:on-rating-bar-change]`
   - `:auto-complete-text-view` inherits `:edit-text`
   - `:horizontal-scroll-view` inherits `:frame-layout`
   - `:table-layout` inherits `:linear-layout`
   - `:table-row` inherits `:linear-layout`
   - `:date-picker` inherits `:view-group`
   - `:time-picker` inherits `:view-group`
   - `:number-picker` inherits `:view-group`
   - `:grid-layout` inherits `:view-group`
   - `:space` inherits `:view`
   - `:view-flipper` inherits `:view-group`
   - `:checked-text-view` inherits `:text-view`

### Phase 3: Support library widgets (separate PR, new deps)

7. Create `neko/src/clojure/neko/ui/support.clj` -- optional namespace users require explicitly
8. Add support library deps as `:provided` scope to `project.clj`
9. Define mappings for RecyclerView, CardView, Toolbar, DrawerLayout, ViewPager, SwipeRefreshLayout
10. Create `neko/src/clojure/neko/listeners/recycler_view.clj` for scroll listeners
11. Create RecyclerView adapter helpers in `neko/ui/adapters.clj` (extend existing)

---

**Recommended implementation order**: Phase 1 -> Phase 2 -> Phase 3, with Phase 1 delivering the most immediate value since it makes already-supported elements actually useful.
