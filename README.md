<div align="center">

# 🤖 Android Lead Agent Skill

**Encode your team's Android engineering standards into Claude Code and other AI assistants — once, consistently, forever.**

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Compatible-blueviolet)](https://claude.ai/code)
[![Compose BOM](https://img.shields.io/badge/Compose%20BOM-2024.09.00-4DD0E1)](https://developer.android.com/jetpack/compose/bom)
[![Material 3](https://img.shields.io/badge/Material%203-Full%20Support-6750A4)](https://m3.material.io/)
[![minSdk](https://img.shields.io/badge/minSdk-26-orange)](https://apilevels.com/)
[![Kotlin](https://img.shields.io/badge/Kotlin-2.0.21-7F52FF)](https://kotlinlang.org/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

<br/>

**[📱 Sample App](https://github.com/ayush016/luminary-android)** · **[📖 Quick Start](#installation)** · **[🗺️ What's Covered](#coverage)**

</div>

---

## What This Actually Does

Claude already knows what Hilt, Room, and Compose are. You don't need to teach it Android.

What you **do** need is for every AI-generated screen in your project to follow **your** architectural decisions — your spacing grid, your state modelling pattern, your animation standards, your colour token hierarchy. Without a skill, Claude will make different choices on different days, and your codebase slowly diverges.

This skill is a **team standards document for AI**. Install it once. Every Claude Code session — yours, your teammates', your CI's — follows the same patterns.

```
Without skill → Claude produces correct-but-generic Android code
With skill    → Claude produces your project's Android code
```

---

## What You Get

| | Without Skill | With Skill |
|---|---|---|
| **Color system** | 6–8 Material roles, hardcoded hex | All 29 roles defined, 3-layer token architecture |
| **Shared element transitions** | Instant navigation or basic slide | `sharedBounds()` / `sharedElement()` with correct scope threading |
| **Loading states** | `CircularProgressIndicator` centred on grey | Skeleton shimmer mirroring real content geometry |
| **State modelling** | `var isLoading`, `var error`, `var data` | `sealed interface` with `Loading`, `Ready(data)`, `Error(message)` |
| **ViewModel events** | `StateFlow` for everything (toast replays on rotation) | `SharedFlow(replay=0)` for events, `StateFlow` for state |
| **UI quality** | Ships when it compiles | Blocked by 29-item binary checklist |
| **Spacing** | Random dp values | 4dp grid enforced via `AppSpacing.*` tokens |
| **Dark mode** | Inverted colours or forgotten | Correct tonal mapping, all 29 roles defined in both schemes |

---

## Coverage

<details>
<summary><strong>📐 Architecture & State</strong></summary>

- MVVM with unidirectional data flow
- Sealed interface UI state: `Loading / Ready(data) / Error(message, canRetry)`
- `StateFlow` for persistent state, `SharedFlow(replay=0)` for one-time events
- Hilt DI graph layering: `SingletonComponent → ViewModelComponent`
- `SavedStateHandle.toRoute<T>()` for type-safe argument extraction
- Sealed actions interface — clean screen API surface
- Clean Architecture dependency rules (domain: zero Android imports)

→ [`references/architecture.md`](references/architecture.md)
</details>

<details>
<summary><strong>🎨 Theming & Color System</strong></summary>

- 3-layer token architecture: **Primitive → Semantic → Component**
- All 29 Material 3 colour roles defined with semantic when-to-use guidance
- Material Theme Builder workflow (generate WCAG-compliant palette from one hex)
- Custom `CompositionLocal` extensions: gradient colors, warning/success roles, dimensions
- Tonal elevation system — depth without shadows
- Runtime theme switching pattern
- 6 documented common mistakes with before/after fixes
- 13-item theming quality checklist

→ [`references/theming-and-color.md`](references/theming-and-color.md)
</details>

<details>
<summary><strong>✨ Shared Element Transitions</strong></summary>

- Three-layer mental model: `SharedTransitionLayout / SharedTransitionScope / AnimatedVisibilityScope`
- `sharedElement()` vs `sharedBounds()` decision guide with comparison table
- Scope threading: explicit parameters vs `CompositionLocal` — when to use each
- 4 complete, compilable Kotlin patterns:
  - Grid thumbnail → full-screen hero image
  - Card expansion (container transform)
  - FAB → full-screen surface morph
  - Text title continuation from list to detail heading
- Navigation integration (NavHost + `SharedTransitionLayout`)
- Predictive back gesture support
- 6 pitfall → root cause → fix entries

→ [`references/shared-element-transitions.md`](references/shared-element-transitions.md)
</details>

<details>
<summary><strong>🎬 Motion & Animation</strong></summary>

- Spring physics tuning guide (`stiffness` × `dampingRatio` decision table)
- `AnimatedVisibility` stagger pattern — `delay = min(index × 40ms, 300ms)`
- `Animatable` for gesture-coupled drag-to-dismiss and swipe-to-reveal
- `updateTransition` for multi-property state machines
- `AnimatedContent` for tab switching with directional slide
- Canvas animation: arc progress rings, path morphing, shape transitions
- Motion choreography rules: ease-out enter, ease-in exit, enter-before-exit

→ [`references/motion-and-animation.md`](references/motion-and-animation.md)
</details>

<details>
<summary><strong>🧭 Navigation</strong></summary>

- Type-safe routes with `@Serializable` data classes
- Multi-module navigation — feature modules never import each other
- Deep link wiring (manifest → NavHost → `TaskStackBuilder` for notifications)
- Back stack management: `popUpTo`, `saveState`, `restoreState` for multi-backstacks
- `SavedStateHandle.toRoute<T>()` in ViewModel
- Back-stack result passing between destinations
- Predictive back gesture — manifest flag + `BackHandler`

→ [`references/navigation.md`](references/navigation.md)
</details>

<details>
<summary><strong>📱 Adaptive Layouts & Large Screens</strong></summary>

- `WindowSizeClass` — Compact / Medium / Expanded breakpoints
- `NavigationSuiteScaffold` — automatic bottom bar / rail / drawer selection
- `ListDetailPaneScaffold` — list+detail on tablet, single-pane on phone
- Foldable support — `FoldingFeature` detection, table-top mode layout
- Window insets comprehensive guide: `safeDrawingPadding`, `imePadding`, `systemGesturesPadding`
- Google Play large-screen quality requirements checklist

→ [`references/adaptive-layouts.md`](references/adaptive-layouts.md)
</details>

<details>
<summary><strong>⚡ Coroutines & Flow</strong></summary>

- Injectable `AppDispatchers` — testable dispatcher selection
- `collectAsStateWithLifecycle()` vs `collectAsState()` — why it matters
- `flatMapLatest` vs `flatMapMerge` vs `flatMapConcat` — decision table
- `stateIn(SharingStarted.WhileSubscribed(5000))` — the correct pattern
- Flow testing with Turbine: `awaitItem()`, `advanceTimeBy()`, `cancelAndIgnoreRemainingEvents()`
- Cancellation rules — `CancellationException` must always be rethrown
- `GlobalScope` prohibition with explanation

→ [`references/coroutines-and-flow.md`](references/coroutines-and-flow.md)
</details>

<details>
<summary><strong>🗄️ Data Layer</strong></summary>

- Repository as single source of truth — `Flow<DomainResult<T>>` + `suspend DomainResult<T>`
- Offline-first: Stale-While-Revalidate reads, Outbox Pattern writes
- `DomainError` sealed interface — `NetworkUnavailable`, `Unauthorized`, `NotFound`, `ServerError`
- Room: `@Upsert`, `Flow<T>` DAOs, in-memory testing
- DataStore replacing SharedPreferences
- WorkManager for background sync

→ [`references/data-layer.md`](references/data-layer.md)
</details>

<details>
<summary><strong>🔒 Security</strong></summary>

- `EncryptedSharedPreferences` + `EncryptedFile` for sensitive storage
- Biometric authentication — full `BiometricPrompt` implementation
- Certificate pinning — OkHttp `CertificatePinner` + Network Security Config XML
- Secret management — what never to commit, `BuildConfig` from env vars
- Input validation at system boundaries — Room parameterised queries, WebView config
- Pre-release security checklist — 11 binary items

→ [`references/security.md`](references/security.md)
</details>

<details>
<summary><strong>⚙️ Performance</strong></summary>

- Compose stability: `@Stable`, `@Immutable`, `ImmutableList`, lambda stability
- `derivedStateOf` — prevent threshold-driven over-recomposition
- Baseline Profiles: full setup, `BaselineProfileRule`, critical user journeys
- Macrobenchmark: `StartupTimingMetric`, `FrameTimingMetric`
- R8 full mode — ProGuard rules for Kotlin + Retrofit + Room + Hilt
- Memory management — LeakCanary, Compose leak patterns, bitmap downsampling
- Build performance — configuration cache, parallel builds, build scans

→ [`references/performance.md`](references/performance.md)
</details>

<details>
<summary><strong>🔔 Notifications, WorkManager, Image Loading, Accessibility, Testing</strong></summary>

**Notifications:** FCM setup, channels grouped by user intent, POST_NOTIFICATIONS permission, action receivers with `goAsync()`, notification groups

**Background Work:** `@HiltWorker` with retry logic, work chaining, expedited workers, Foreground Services (Android 14 service types), `AlarmManager` for exact alarms

**Image Loading:** Coil3 `ImageLoader` configuration (OkHttp, memory/disk cache), `SubcomposeAsyncImage` for custom loading states, lazy list performance traps, video thumbnails, SVG, animated GIF

**Accessibility:** Semantic roles, `contentDescription` rules, custom accessibility actions, focus management, live regions, WCAG AA contrast guide, TalkBack testing checklist

**Testing:** Testing pyramid (unit/integration/screenshot/E2E), `FakeRepository` over mocks, Roborazzi JVM screenshot tests, Turbine for Flow testing, `MainDispatcherRule`

→ [`references/notifications.md`](references/notifications.md) · [`references/background-work.md`](references/background-work.md) · [`references/image-loading.md`](references/image-loading.md) · [`references/accessibility.md`](references/accessibility.md) · [`references/testing.md`](references/testing.md)
</details>

<details>
<summary><strong>🏗️ Build, Modules & ADB</strong></summary>

**Gradle:** Convention plugins (`build-logic/` composite build), version catalog (`libs.versions.toml`), AGP↔Gradle compatibility matrix, `gradle-wrapper.properties` — the file everyone forgets

**Bootstrap:** Minimum res/ files for any app module (themes.xml, adaptive icons, strings), VectorDrawable element restrictions

**ADB / MCP:** Screenshot-driven UI review loop, logcat filtering, UI hierarchy dump, animation frame-interval debugging, responsive testing (font scale, RTL, density, foldable states)

→ [`references/build-and-modules.md`](references/build-and-modules.md) · [`references/android-mcp.md`](references/android-mcp.md)
</details>

---

## Sample App

**[Luminary](https://github.com/ayush016/luminary-android)** — a news reader built entirely with this skill. Demonstrates shared element transitions, full 29-role colour system, skeleton loading, staggered animations, and Clean Architecture end-to-end.

> Every pattern in that codebase traces to a specific section of this skill.

---

## Installation

### For Claude Code (project-local — recommended for teams)

```bash
# From your project root
git clone https://github.com/ayush016/android-lead-agent-skills.git \
  .claude/skills/android
```

Add to your project's `CLAUDE.md`:

```markdown
## Active Skills

- **android**: `.claude/skills/android/SKILL.md`
  Covers architecture, Compose, animations, theming, navigation, security, performance, testing, and more.
```

### For Claude Code (user-global — personal use)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/ayush016/android-lead-agent-skills.git \
  ~/.claude/skills/android
```

Add to `~/.claude/CLAUDE.md`:
```markdown
## Skills
- android: ~/.claude/skills/android/SKILL.md
```

### For other AI assistants (Copilot, Cursor, Gemini)

Copy `SKILL.md` into the relevant rules/instructions location for your tool:

| Tool | Location |
|------|----------|
| GitHub Copilot | `.github/copilot-instructions.md` |
| Cursor | `.cursor/rules/android.mdc` |
| Gemini Code Assist | `.gemini/context.md` |

---

## How to Invoke

### Automatic (Claude Code)

The skill auto-loads when it detects Android-related work — any `.kt` file with Compose imports, mentions of ViewModel / Hilt / Room / Coroutines, Gradle config, or ADB / MCP usage.

### Explicit reference

```
Using the android skill, build a product detail screen with a shared element 
transition from the list thumbnail. Use the offline-first data layer pattern.
```

```
Review this composable against the android skill's UI excellence checklist.
```

```
Using the android skill, set up Baseline Profiles and Macrobenchmark 
for the critical cold-start and scroll paths.
```

---

## What This Skill Does NOT Do

Be clear-eyed about limitations:

- **It does not make Claude write bug-free code.** Code generation still requires review and testing. The skill shapes patterns and architecture decisions, not syntax correctness.
- **It does not replace documentation.** The official Android Developer docs, Material 3 guidelines, and Kotlin references are always the authoritative source. This skill synthesises patterns from them.
- **It will get stale.** Android APIs evolve fast. Compose BOM, Navigation, AGP versions listed here will not be the latest in 12 months. Check the dates and update your pinned versions accordingly.
- **It won't fit entirely in one context window.** 8,500+ lines of content means Claude works from the most relevant sections. For very complex tasks, reference the specific file explicitly.

---

## File Structure

```
android-lead-agent-skills/
│
├── SKILL.md                              ← Load this into your AI tool
│
├── references/                           ← Deep-dive guides (load on demand)
│   ├── theming-and-color.md             ★ All 29 M3 roles, 3-layer tokens
│   ├── shared-element-transitions.md    ★ 4 full Kotlin patterns + 6 pitfalls
│   ├── compose-ui-system.md             ★ Design system, shimmer, gradients
│   ├── motion-and-animation.md          ★ Spring physics, Canvas, choreography
│   ├── navigation.md                      Type-safe nav, deep links, multi-module
│   ├── architecture.md                    ViewModel, state, DI, Clean Architecture
│   ├── adaptive-layouts.md               WindowSizeClass, foldables, insets
│   ├── coroutines-and-flow.md            Dispatchers, operators, cancellation
│   ├── data-layer.md                      Offline-first, Room, Retrofit
│   ├── performance.md                     Baseline Profiles, R8, recomposition
│   ├── security.md                        Encryption, biometrics, cert pinning
│   ├── background-work.md               WorkManager, Foreground Services, Alarms
│   ├── image-loading.md                  Coil3, SubcomposeAsyncImage, caching
│   ├── notifications.md                   FCM, channels, action receivers
│   ├── accessibility.md                   Semantics, TalkBack, WCAG AA
│   ├── testing.md                         Roborazzi, Turbine, fakes
│   ├── build-and-modules.md             Gradle, convention plugins, bootstrap
│   └── android-mcp.md                   ADB, screenshot review, debugging
│
└── assets/
    └── ui-excellence-checklist.md       ← 29-item binary quality gate
```

Files marked ★ are the most impactful — start here if you're evaluating the skill.

---

## Compatibility

| Component | Version |
|-----------|---------|
| AGP | 8.5.2 |
| Gradle | 8.7 (pinned via wrapper) |
| Kotlin | 2.0.21 |
| Compose BOM | 2024.09.00 |
| Navigation Compose | 2.8.4 |
| Hilt | 2.52 |
| min SDK | 26 (API 26 = ~99% device coverage) |
| target SDK | 35 |

> Versions are pinned at time of last update. Always verify against the official [AGP compatibility table](https://developer.android.com/build/releases/gradle-plugin#updating-gradle) before upgrading.

---

## Contributing

The skill improves when patterns fail in the real world. Open a PR if:

- An API listed here is deprecated or has a better replacement
- A pattern caused a compilation error or runtime bug
- A domain is missing (e.g., CameraX, Media3, Wear OS)
- You have a better code example for an existing pattern

**PR format:** Include the specific file and section, the problem with the current content, the replacement, and ideally a link to the relevant Android documentation.

---

## License

MIT © 2025 [Ayush Shrivastava](https://github.com/ayush016)

See [LICENSE](LICENSE) for full terms. TL;DR: use it, fork it, build on it — attribution appreciated but not required.

---

<div align="center">

If this skill improved your Android + AI workflow, a ⭐ on the repo helps others find it.

**[Sample App](https://github.com/ayush016/luminary-android)** · **[Issues](https://github.com/ayush016/android-lead-agent-skills/issues)** · **[Discussions](https://github.com/ayush016/android-lead-agent-skills/discussions)**

</div>
