# App Widget Guidelines — Production Reference

> Official API reference: https://developer.android.com/develop/ui/compose/glance

Jetpack Glance is the right tool when you need a home screen widget that is small, useful, and cheap to update. The rule is simple: make the widget answer one question fast, then hand the user back to the app for anything deeper.

---

## Mental Model — Three Layers

Understanding the widget as a layered system makes every decision easier.

### Layer 1: `GlanceAppWidget` — the renderer

`GlanceAppWidget` is the entry point that turns your widget state into `RemoteViews`. It should stay passive and stateless. Treat it as a renderer, not a place for business logic.

### Layer 2: app state — the source of truth

The widget should read from app-owned data such as cached repository state, database state, or a small persisted widget preference. The widget should not invent its own important state in memory.

### Layer 3: the launcher — the host

The launcher decides how much space the widget gets, when it is refreshed, and how it appears in the picker. That means the widget must be resilient to size changes, partial data, and occasional missed updates.

### The Key Contract

If a behavior feels too interactive, too dynamic, or too detailed for a home screen surface, it belongs in the app instead of the widget.

---

## Widget-First Use Cases

Use a widget when the user benefits from seeing a compact summary without opening the app.

### Good fits

- Active item, order, or task status
- Upcoming appointment, booking, or reminder summary
- Quick progress, timeline, or ETA display
- One-tap deep link into the app
- Optional configuration when personalization matters

### Poor fits

- Full maps or rich multi-step flows
- Permission-heavy actions
- Long-running work in the receiver
- Live polling inside the UI layer
- Dense lists that do not fit small surfaces

---

## Setup Rules

Glance widgets should be wired like normal Android widgets, with Compose enabled and Glance handling the widget lifecycle.

### Required pieces

- Add the Glance app widget dependency in the app module.
- Enable Compose in the module build config.
- Register a `GlanceAppWidgetReceiver` in `AndroidManifest.xml`.
- Add widget provider metadata XML in `res/xml`.
- Use a loading layout for `initialLayout`.

### Manifest contract

The receiver must listen for `android.appwidget.action.APPWIDGET_UPDATE` and point to the provider XML through `android:resource`.

```xml
<receiver
    android:name=".widget.ExampleWidgetReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/example_widget_info" />
</receiver>
```

### Widget provider contract

Define the metadata with the default size, resize behavior, description, preview, and optional configuration flags that the widget actually needs.

```xml
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="40dp"
    android:minHeight="40dp"
    android:initialLayout="@layout/glance_default_loading_layout"
    android:widgetCategory="home_screen" />
```

---

## State and Data Rules

Widgets are not app screens. They must be stateless, passive, and backed by application data.

### Source of truth

- Read from application state, cache, database, or repository data.
- Do not rely on in-memory state for important widget content.
- Use app-side refresh logic when the underlying data changes.
- Keep widget state separate from full-screen UI state.

### Widget state model

Prefer a single widget state model that can represent:

- empty
- loading
- error
- summary
- active/live

### Data shaping rule

The widget should receive already-compacted data. It should not need to know how to query large graphs, assemble complex screens, or derive business logic that belongs in the app layer.

---

## Layout Rules

Glance layouts must stay compact, responsive, and readable.

### Core layouts

- Use `Box`, `Column`, and `Row` as the primary building blocks.
- Use `LazyColumn` only when the content truly needs scrolling.
- Keep spacing and hierarchy simple enough to read at small sizes.

```kotlin
@Composable
fun WidgetContent(title: String, subtitle: String) {
    Column {
        Text(text = title)
        Text(text = subtitle)
    }
}
```

### Size mode choice

- Use `SizeMode.Single` for fixed content.
- Use `SizeMode.Responsive` when you need a small set of size-aware layouts.
- Use `SizeMode.Exact` only when per-size rendering is necessary.

```kotlin
class ExampleWidget : GlanceAppWidget() {
    override val sizeMode = SizeMode.Responsive(
        setOf(
            DpSize(100.dp, 100.dp),
            DpSize(250.dp, 120.dp),
            DpSize(250.dp, 250.dp)
        )
    )
}
```

### Practical rule

Design for the smallest supported widget size first, then let larger sizes reveal extra context instead of changing the widget into a different product.

---

## Update Rules

Widget updates should be conservative and data-driven.

### Recommended update paths

- Use `GlanceAppWidget.update()`, `updateAll()`, or `updateIf()` from app code or background workers.
- Use `WorkManager` for long-running fetches or periodic refreshes.
- Update immediately when the app is already awake and the relevant data changes.

```kotlin
class WidgetSyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        ExampleWidget().updateAll(applicationContext)
        return Result.success()
    }
}
```

### Battery rule

- Avoid frequent updates that drain battery.
- Keep `updatePeriodMillis` conservative and use it only for coarse refreshes.
- Never use widget updates as a substitute for real-time foreground UI behavior.

### Receiver rule

Do not run heavy work directly in receiver callbacks. If the update needs network or database work, offload it to a worker and update the widget after the work completes.

---

## Interaction Rules

Interactions in widgets must stay simple and predictable.

### Preferred interactions

- Use widget taps mainly to open an activity.
- Pass identifiers through action parameters so the app can open the correct screen.
- Use `ActionCallback` for small callback-driven actions.

```kotlin
@Composable
fun WidgetCard(itemId: Long) {
    Button(
        text = "Open",
        onClick = actionStartActivity<MainActivity>(
            actionParametersOf(
                ActionParameters.Key<Long>("item_id") to itemId
            )
        )
    )
}
```

### Restricted interactions

- Use service or broadcast actions only for small, safe operations.
- Do not start long work directly from receiver callbacks.
- Avoid multi-step flows that require full screen navigation logic inside the widget.

### Tap-through principle

If the widget shows a summary, the tap should open the exact place in the app where the user can continue from that summary.

---

## Configuration Rules

Add widget configuration only when it meaningfully improves the experience.

### When to configure

- Use `android:configure` when the widget has setup options.
- Mark configuration as optional when a sensible default exists.
- Mark it reconfigurable only when users should be able to change it later.

### When to skip configuration

- If the widget has no user-specific setup, skip configuration entirely.
- If the widget can infer a good default from app state, prefer that over setup friction.

---

## Preview Rules

The widget picker should show a useful, believable preview.

### Picker requirements

- Add a widget name and description for the picker.
- Provide `previewLayout` or `previewImage` for launcher compatibility.
- Keep previews representative of the real widget content.

### Generated previews

- Provide generated previews with `providePreview` on supported Android versions.
- Use preview sizes that match the smallest important widget states.
- Refresh previews when the widget experience changes materially.

---

## Theme and Styling Rules

Widgets should look polished, but styling should never overpower clarity.

### Styling guidance

- Wrap content in `GlanceTheme` when theme colors are needed.
- Prefer resource-backed colors and drawables.
- Keep typography simple and readable.
- Use shapes and backgrounds that work on multiple launchers and sizes.

```kotlin
override suspend fun provideGlance(context: Context, id: GlanceId) {
    provideContent {
        GlanceTheme {
            WidgetContent(title = "Today", subtitle = "3 items pending")
        }
    }
}
```

### Design restraint

- Avoid over-designing the widget surface.
- Favor contrast, hierarchy, and clarity over visual density.

---

## Interoperability Rules

Use interoperability only when Glance cannot reasonably express the required UI.

### Fallback rule

- Use `AndroidRemoteViews` only when a required UI piece is not practical in Glance.
- Keep interoperability as a fallback, not the primary design.
- Do not mix normal Jetpack Compose UI into Glance content directly.

```kotlin
Column {
    Text("Summary")
    AndroidRemoteViews(RemoteViews(context.packageName, R.layout.example_view))
}
```

---

## Error Handling Rules

Widgets must fail gracefully.

### Error behavior

- Show a graceful error or fallback state if data is unavailable.
- Keep the widget usable even when refresh fails.
- Prefer clear copy over empty surfaces.
- Handle missing or partial data explicitly.

### Recovery rule

When possible, keep the widget actionable even in an error state, usually by opening the app or letting the user retry from the app side.

---

## Testing Rules

Treat widget behavior as testable product logic.

### What to test

- Empty, loading, summary, error, and active/live states
- Tap actions and intent routing
- Compact and expanded size behavior

### How to test

- Add unit tests for the widget composables.
- Verify click actions with Glance testing matchers.
- Keep composables small enough to test in isolation.

---

## Metrics Rules

Track widget usage only when it gives you a real product signal.

### What to measure

- Widget impressions
- Tap and interaction events
- Usage patterns for updates, if needed

### Metrics discipline

- Keep metrics collection lightweight and periodic.
- Measure only the signals that help improve the widget.

---

## Reference Snippets

Use these as starting points when implementing a new widget.

### Minimal widget

```kotlin
class ExampleWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent {
            WidgetContent(title = "Example", subtitle = "Ready")
        }
    }
}

class ExampleWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = ExampleWidget()
}
```

### Open an activity

```kotlin
Button(
    text = "Open app",
    onClick = actionStartActivity<MainActivity>()
)
```

### Responsive sizing

```kotlin
override val sizeMode = SizeMode.Responsive(
    setOf(
        DpSize(100.dp, 100.dp),
        DpSize(250.dp, 120.dp)
    )
)
```

### Update from a worker

```kotlin
ExampleWidget().updateAll(context)
```

---

## Implementation Checklist

- Define the widget use case and the one-sentence user value.
- Map app data into a compact widget state model.
- Build the widget UI with Glance layouts.
- Register the receiver and provider metadata.
- Add previews, configuration, and deep links if needed.
- Add refresh/update wiring through app code or WorkManager.
- Add tests for the main widget states.

