# Module Structure

> **Owner:** Android Tech Lead  
> **Audience:** Android team  
> **Related:** `ARCHITECTURE.md` · `DATA_LAYER.md` · `CONTRIBUTING.md`  
> **Last reviewed against:** AGP 8.3, Gradle 8.6, Kotlin 2.0

---

## Table of Contents

- [Why Multi-Module](#why-multi-module)
- [Module Map](#module-map)
- [Module Descriptions](#module-descriptions)
  - [:app](#app)
  - [:build-logic](#build-logic)
  - [:feature-search](#feature-search)
  - [:feature-ride](#feature-ride)
  - [:feature-history](#feature-history)
  - [:feature-settings](#feature-settings)
  - [:core-ui](#core-ui)
  - [:core-domain](#core-domain)
  - [:core-data](#core-data)
  - [:core-location](#core-location)
- [Dependency Rules](#dependency-rules)
  - [Allowed dependencies](#allowed-dependencies)
  - [Forbidden dependencies](#forbidden-dependencies)
  - [Enforcement](#enforcement)
- [Build Logic — Convention Plugins](#build-logic--convention-plugins)
  - [Available plugins](#available-plugins)
  - [Plugin implementation example](#plugin-implementation-example)
  - [Version catalog](#version-catalog)
- [Adding a New Feature Module](#adding-a-new-feature-module)
- [Adding a New Core Module](#adding-a-new-core-module)
- [Build Performance](#build-performance)
  - [Parallelism](#parallelism)
  - [Configuration cache](#configuration-cache)
  - [Build baseline profile](#build-baseline-profile)
  - [Measuring build time](#measuring-build-time)
- [Testing Per Module](#testing-per-module)
- [Common Mistakes](#common-mistakes)
- [Decision Log](#decision-log)

---

## Why Multi-Module

This project uses multi-module structure from the start — not as premature optimisation,  
but because the domain makes it the right default.

**Reasons specific to this project:**

| Problem | How modules solve it |
|---|---|
| Location service, FGS, and GPS logic bleed into feature code | `:core-location` isolates all hardware/service concerns behind a clean interface |
| Data layer types (Room entities, DTOs) leak into ViewModels | `:core-data` cannot be imported by `:feature-*` — enforced at build time |
| Domain logic ends up in Android-dependent code | `:core-domain` has zero Android SDK imports — testable with plain JUnit |
| Build times grow as the project grows | Gradle only rebuilds changed modules and their dependents |
| New developers modify things they should not | Dependency rules are checked in CI — wrong imports fail the build |
| Feature flags need to isolate unreleased features | Feature modules are the isolation boundary — a feature can be removed by removing its module |

**When to stay single-module:**  
If the team is 1-2 people and the project is under 20 screens, the overhead of multi-module  
may not be worth it in the early weeks. The module structure described here is designed to  
be introduced gradually — start with `:app` + `:core-domain`, add modules as the codebase grows.

---

## Module Map

```
root/
├── app/                          # :app — application entry point
├── build-logic/                  # :build-logic — convention plugins
│   └── convention/
├── core/
│   ├── core-domain/              # :core-domain — pure Kotlin, no Android
│   ├── core-data/                # :core-data — Room, Retrofit, repositories
│   ├── core-ui/                  # :core-ui — design system, shared composables
│   └── core-location/            # :core-location — FGS, GPS, location bridge
└── feature/
    ├── feature-search/           # :feature-search — Idle + Searching screens
    ├── feature-ride/             # :feature-ride — Waiting + InProgress + Completed
    ├── feature-history/          # :feature-history — ride history
    └── feature-settings/         # :feature-settings — preferences, permissions

Dependency direction (arrows = "depends on"):

┌──────────────────────────────────────────────────────────────┐
│                           :app                               │
│           (depends on all modules to wire them)              │
└──────┬───────────────────────────────────────────────────────┘
       │
       ├──► :feature-search ─────┐
       ├──► :feature-ride ───────┤
       ├──► :feature-history ────┼──► :core-ui ──► :core-domain
       ├──► :feature-settings ───┘         │
       │                                   │ (also)
       ├──► :core-location ──► :core-data ──► :core-domain
       │
       └──► :core-data ──────────────────► :core-domain

Forbidden (checked by Dependency Guard):
  :feature-* ──✗──► :core-data
  :feature-* ──✗──► :core-location
  :feature-* ──✗──► :feature-* (any cross-feature dependency)
  :core-ui   ──✗──► :core-data
  :core-domain ──✗──► anything (must stay pure)
```

---

## Module Descriptions

### :app

**Purpose:** Application entry point. Wires everything together.

**Owns:**
- `Application` class (Hilt setup, WorkManager init, app-level configuration)
- `MainActivity` — single activity, hosts `NavHost`
- `AppNavHost` — top-level navigation graph linking all feature screens
- `AppScaffold` — global UI shell (offline banner, notification permission prompts)
- `AppViewModel` — process death recovery, session-level state

**Depends on:** everything (all `:feature-*`, all `:core-*`)

**Nothing depends on `:app`.**

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.micromobility.android.application)
    alias(libs.plugins.micromobility.hilt)
}

dependencies {
    implementation(projects.feature.featureSearch)
    implementation(projects.feature.featureRide)
    implementation(projects.feature.featureHistory)
    implementation(projects.feature.featureSettings)
    implementation(projects.core.coreUi)
    implementation(projects.core.coreDomain)
    implementation(projects.core.coreData)
    implementation(projects.core.coreLocation)
}
```

---

### :build-logic

**Purpose:** Gradle convention plugins. Shared build configuration.

**Owns:**
- `MicromobilityAndroidApplicationPlugin` — base app config
- `MicromobilityAndroidLibraryPlugin` — base library config  
- `MicromobilityAndroidComposePlugin` — Compose compiler + dependencies
- `MicromobilityHiltPlugin` — Hilt KSP setup
- `MicromobilityAndroidTestPlugin` — test dependencies + config

**Does NOT contribute runtime code.** Applied via `plugins {}` block.

**Everything depends on `:build-logic`** (via the plugin system).

```
build-logic/
└── convention/
    ├── src/main/kotlin/
    │   ├── MicromobilityAndroidApplicationPlugin.kt
    │   ├── MicromobilityAndroidLibraryPlugin.kt
    │   ├── MicromobilityAndroidComposePlugin.kt
    │   ├── MicromobilityHiltPlugin.kt
    │   └── MicromobilityAndroidTestPlugin.kt
    └── build.gradle.kts
```

---

### :feature-search

**Purpose:** Search flow — `Idle` and `Searching` states.

**Owns:**
- `SearchScreen` composable
- `SearchViewModel` (Orbit MVI)
- `LocationPermissionScreen` — foreground permission request entry point
- `SearchUiState`, `SearchIntent`, `SearchSideEffect`
- `StartRideUseCase` call (injected from `:core-domain`)

**Depends on:** `:core-domain`, `:core-ui`

**Used by:** `:app` (registered in `AppNavHost`)

```
feature-search/
└── src/
    ├── main/kotlin/
    │   ├── SearchScreen.kt
    │   ├── SearchViewModel.kt
    │   ├── SearchUiState.kt
    │   └── permission/
    │       └── LocationPermissionScreen.kt
    └── test/kotlin/
        └── SearchViewModelTest.kt
```

---

### :feature-ride

**Purpose:** Active ride flow — `MatchFound`, `Waiting`, `InProgress`, `Completed`, `Cancelled`.

**Owns:**
- `RideScreen` composable (adapts based on `RideState` sealed subtype)
- `RideViewModel` (the most complex ViewModel in the project)
- `WaitingContent`, `InProgressContent`, `CompletedContent`, `CancelledContent`
- Background permission rationale screen (shown at `InProgress` start)
- FGS start/stop via `SideEffect` collection

**Depends on:** `:core-domain`, `:core-ui`

**Used by:** `:app`

```
feature-ride/
└── src/
    ├── main/kotlin/
    │   ├── RideScreen.kt
    │   ├── RideViewModel.kt
    │   ├── RideUiState.kt
    │   ├── RideIntent.kt
    │   ├── RideSideEffect.kt
    │   └── content/
    │       ├── WaitingContent.kt
    │       ├── InProgressContent.kt
    │       ├── CompletedContent.kt
    │       └── CancelledContent.kt
    └── test/kotlin/
        ├── RideViewModelTest.kt
        └── RideScreenAccessibilityTest.kt
```

---

### :feature-history

**Purpose:** Ride history — list and detail screens.

**Owns:**
- `HistoryListScreen`, `HistoryDetailScreen`
- `HistoryViewModel` — cache-first, stale-while-revalidate pattern

**Depends on:** `:core-domain`, `:core-ui`

**Used by:** `:app`

---

### :feature-settings

**Purpose:** User preferences, permission management, notification settings.

**Owns:**
- `SettingsScreen`, `PermissionStatusScreen`
- `SettingsViewModel` — fully local, no network calls

**Depends on:** `:core-domain`, `:core-ui`

**Used by:** `:app`

---

### :core-ui

**Purpose:** Design system and shared Compose components.

**Owns:**
- `MicromobilityTheme` — Material 3 theming, typography, color scheme
- Shared components: `PrimaryButton`, `DriverCard`, `OfflineBanner`,  
  `LoadingOverlay`, `ErrorContent`, `RationalePoint`
- `UiText` — sealed class for string/resource text in ViewModels
- Preview utilities and `@ThemePreview` annotation

**Depends on:** `:core-domain` (for domain model display — e.g. `Driver`, `RideSummary`)

**Must NOT depend on:** `:core-data`, `:core-location`

**Used by:** `:app`, all `:feature-*`

```kotlin
// core/core-ui/build.gradle.kts
plugins {
    alias(libs.plugins.micromobility.android.library)
    alias(libs.plugins.micromobility.android.compose)
}

dependencies {
    api(projects.core.coreDomain)
    // No core-data, no core-location
}
```

---

### :core-domain

**Purpose:** Pure Kotlin business logic. No Android SDK. No Room. No Retrofit.

**Owns:**
- `RideStateMachine`
- Repository interfaces: `RideRepository`, `LocationRepository`
- UseCases: `StartRideUseCase`, `CancelRideUseCase`, `ObserveRideStateUseCase`, etc.
- Domain models: `Ride`, `Driver`, `LatLng`, `RideState`, `RideEvent`, `DomainError`
- `NetworkMonitor` interface
- `RideErrorType`, `CancellationReason`, `RideStatus` enums

**Depends on:** nothing

**Used by:** everything

```kotlin
// core/core-domain/build.gradle.kts
plugins {
    alias(libs.plugins.micromobility.kotlin.library)   // pure Kotlin — NOT android.library
}

dependencies {
    implementation(libs.kotlin.coroutines.core)
    // Zero Android dependencies. Verified by Dependency Guard.
}
```

**Verification:** The `micromobility.kotlin.library` plugin applies the Kotlin JVM plugin,  
not the Android Library plugin. This guarantees that `android.*` imports are impossible  
at compile time — not just by convention.

---

### :core-data

**Purpose:** Data layer implementation. Room, Retrofit, DataStore, repositories.

**Owns:**
- `RideRepositoryImpl`, `LocationRepositoryImpl`
- `AppDatabase`, all DAOs and entities
- `RideApiService` (Retrofit interface)
- `RideWebSocketClient`
- `PermissionStateDataStore`, `UserPreferencesDataStore`
- `ConnectivityNetworkMonitor`
- `SyncManager`, `DeferredWriteQueue`
- All mappers: DTO ↔ Entity ↔ Domain

**Depends on:** `:core-domain`

**Must NOT be imported by:** `:feature-*`, `:core-ui`

**Used by:** `:app` (provides Hilt bindings), `:core-location` (writes location to Room)

```kotlin
// core/core-data/build.gradle.kts
plugins {
    alias(libs.plugins.micromobility.android.library)
    alias(libs.plugins.micromobility.hilt)
}

dependencies {
    api(projects.core.coreDomain)
    implementation(libs.room.runtime)
    implementation(libs.room.ktx)
    ksp(libs.room.compiler)
    implementation(libs.retrofit)
    implementation(libs.okhttp)
    implementation(libs.datastore.preferences)
    implementation(libs.kotlinx.serialization.json)
}
```

---

### :core-location

**Purpose:** GPS hardware bridge. Foreground Service, FusedLocationProviderClient, location buffer.

**Owns:**
- `LocationTrackingService` (FGS with `foregroundServiceType="location"`)
- `LocationBuffer` (in-memory + Room flush)
- `RideNotificationBuilder` (persistent notification)
- `LocationPermissionManager`
- `FusedLocationProviderClientWrapper`

**Depends on:** `:core-domain`, `:core-data`

Must depend on `:core-data` to write `LocationHistoryEntity` to Room  
and read active ride state for FGS recovery.

**Must NOT be imported by:** `:feature-*`

**Used by:** `:app` (starts/stops FGS via `Intent`)

```kotlin
// core/core-location/build.gradle.kts
plugins {
    alias(libs.plugins.micromobility.android.library)
    alias(libs.plugins.micromobility.hilt)
}

dependencies {
    implementation(projects.core.coreDomain)
    implementation(projects.core.coreData)
    implementation(libs.play.services.location)
}
```

---

## Dependency Rules

### Allowed dependencies

| Module | May depend on |
|---|---|
| `:app` | All modules |
| `:build-logic` | Gradle plugins only (not runtime modules) |
| `:feature-search` | `:core-domain`, `:core-ui` |
| `:feature-ride` | `:core-domain`, `:core-ui` |
| `:feature-history` | `:core-domain`, `:core-ui` |
| `:feature-settings` | `:core-domain`, `:core-ui` |
| `:core-ui` | `:core-domain` |
| `:core-domain` | Nothing |
| `:core-data` | `:core-domain` |
| `:core-location` | `:core-domain`, `:core-data` |

### Forbidden dependencies

These are illegal. CI will fail if they are added.

| From | To | Why |
|---|---|---|
| `:feature-*` | `:core-data` | Features must not know about Room/Retrofit internals |
| `:feature-*` | `:core-location` | Features must not start/stop FGS directly |
| `:feature-*` | `:feature-*` | Cross-feature coupling — share via `:core-domain` models instead |
| `:core-ui` | `:core-data` | Design system must not know about the database |
| `:core-ui` | `:core-location` | Design system must not know about GPS hardware |
| `:core-domain` | Anything | Domain must stay pure — no leaking upward |
| `:core-data` | `:feature-*` | Inverted dependency — data layer knows nothing of features |
| `:core-data` | `:core-ui` | Data layer knows nothing of the design system |
| `:core-location` | `:feature-*` | Same as above |
| `:core-location` | `:core-ui` | Location service knows nothing of Compose |

### Enforcement

We use **[Dependency Guard](https://github.com/dropbox/dependency-guard)**  
to baseline and enforce all dependency rules.

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.dependency.guard)
}

dependencyGuard {
    configuration("releaseRuntimeClasspath")
}
```

```bash
# Generate baseline (run once after intentional change)
./gradlew dependencyGuard

# Verify in CI (fails if dependencies changed without baseline update)
./gradlew dependencyGuardBaseline
```

The baseline file `app/dependencies/releaseRuntimeClasspath.txt` is committed to version control.  
Any PR that adds a new transitive dependency causes this check to fail — the developer must  
explicitly update the baseline and justify the addition in the PR description.

**Additional enforcement via module-level `build.gradle.kts` plugin type:**

`:core-domain` uses `micromobility.kotlin.library` (pure JVM Kotlin plugin).  
This means `import android.*` literally does not compile — the Android SDK is not on the classpath.  
No convention or code review needed — the compiler enforces it.

---

## Build Logic — Convention Plugins

Instead of duplicating Gradle configuration across modules, we centralise it in  
`:build-logic` convention plugins. Each plugin applies a specific combination of  
plugins, compiler options, and dependencies.

### Available plugins

| Plugin | Applied to | What it does |
|---|---|---|
| `micromobility.android.application` | `:app` | AGP application plugin, compileSdk, minSdk, ProGuard |
| `micromobility.android.library` | All `:core-*` and `:feature-*` | AGP library plugin, compileSdk, minSdk, shared lint rules |
| `micromobility.kotlin.library` | `:core-domain` | Pure Kotlin JVM plugin — no Android SDK |
| `micromobility.android.compose` | `:core-ui`, feature modules with Compose | Compose compiler plugin, `composeOptions`, Compose BOM |
| `micromobility.hilt` | All modules using Hilt | Hilt KSP plugin, `hilt-android`, `hilt-compiler` |
| `micromobility.android.test` | All modules | JUnit 5, MockK, Turbine, ComposeTestRule setup |
| `micromobility.kotlin.test` | `:core-domain` | JUnit 5, MockK, coroutines-test (no Android dependencies) |

### Plugin implementation example

```kotlin
// build-logic/convention/src/main/kotlin/MicromobilityAndroidLibraryPlugin.kt

class MicromobilityAndroidLibraryPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<LibraryExtension> {
                compileSdk = 34

                defaultConfig {
                    minSdk = 28
                    testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                    consumerProguardFiles("consumer-rules.pro")
                }

                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                    isCoreLibraryDesugaringEnabled = true
                }

                // Disable unused build types — reduces configuration time
                buildTypes {
                    release { isMinifyEnabled = false }
                }

                // Disable unused features in library modules
                buildFeatures {
                    buildConfig = false
                    resValues = false
                }

                lint {
                    abortOnError = true
                    warningsAsErrors = true
                }
            }

            dependencies {
                add("coreLibraryDesugaring", libs.android.desugar.jdk.libs)
            }
        }
    }
}
```

```kotlin
// build-logic/convention/src/main/kotlin/MicromobilityKotlinLibraryPlugin.kt
// Used by :core-domain — pure JVM, no Android

class MicromobilityKotlinLibraryPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("org.jetbrains.kotlin.jvm")       // NOT com.android.library
            }

            extensions.configure<JavaPluginExtension> {
                sourceCompatibility = JavaVersion.VERSION_17
                targetCompatibility = JavaVersion.VERSION_17
            }

            tasks.withType<KotlinCompile>().configureEach {
                kotlinOptions {
                    jvmTarget = "17"
                    freeCompilerArgs = freeCompilerArgs + listOf(
                        "-opt-in=kotlinx.coroutines.ExperimentalCoroutinesApi",
                    )
                }
            }

            // Zero Android dependencies — intentional
        }
    }
}
```

### Version catalog

All dependency versions are centralised in `gradle/libs.versions.toml`:

```toml
[versions]
kotlin = "2.0.0"
agp = "8.3.2"
hilt = "2.51"
room = "2.6.1"
compose-bom = "2024.05.00"
orbit-mvi = "8.0.0"
retrofit = "2.11.0"
okhttp = "4.12.0"
datastore = "1.1.1"
coroutines = "1.8.1"
dependency-guard = "0.5.0"

[libraries]
kotlin-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
hilt-android = { module = "com.google.dagger:hilt-android", version.ref = "hilt" }
hilt-compiler = { module = "com.google.dagger:hilt-compiler", version.ref = "hilt" }
room-runtime = { module = "androidx.room:room-runtime", version.ref = "room" }
room-ktx = { module = "androidx.room:room-ktx", version.ref = "room" }
room-compiler = { module = "androidx.room:room-compiler", version.ref = "room" }
orbit-core = { module = "org.orbit-mvi:orbit-core", version.ref = "orbit-mvi" }
orbit-viewmodel = { module = "org.orbit-mvi:orbit-viewmodel", version.ref = "orbit-mvi" }
orbit-compose = { module = "org.orbit-mvi:orbit-compose", version.ref = "orbit-mvi" }
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
datastore-preferences = { module = "androidx.datastore:datastore-preferences", version.ref = "datastore" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
dependency-guard = { id = "com.dropbox.dependency-guard", version.ref = "dependency-guard" }
micromobility-android-application = { id = "micromobility.android.application" }
micromobility-android-library = { id = "micromobility.android.library" }
micromobility-kotlin-library = { id = "micromobility.kotlin.library" }
micromobility-android-compose = { id = "micromobility.android.compose" }
micromobility-hilt = { id = "micromobility.hilt" }
micromobility-android-test = { id = "micromobility.android.test" }
```

---

## Adding a New Feature Module

Use this checklist every time a new `:feature-*` module is created.

**1. Create the module directory:**

```bash
mkdir -p feature/feature-myfeature/src/main/kotlin
mkdir -p feature/feature-myfeature/src/test/kotlin
mkdir -p feature/feature-myfeature/src/androidTest/kotlin
```

**2. Create `build.gradle.kts`:**

```kotlin
// feature/feature-myfeature/build.gradle.kts
plugins {
    alias(libs.plugins.micromobility.android.library)
    alias(libs.plugins.micromobility.android.compose)
    alias(libs.plugins.micromobility.hilt)
    alias(libs.plugins.micromobility.android.test)
}

android {
    namespace = "com.yourorg.micromobility.feature.myfeature"
}

dependencies {
    implementation(projects.core.coreDomain)
    implementation(projects.core.coreUi)
    implementation(libs.orbit.viewmodel)
    implementation(libs.orbit.compose)
}
```

**3. Register in `settings.gradle.kts`:**

```kotlin
include(":feature:feature-myfeature")
```

**4. Add to `:app` dependencies:**

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(projects.feature.featureMyfeature)
}
```

**5. Register the screen in `AppNavHost`:**

```kotlin
composable(Screen.MyFeature.route) {
    MyFeatureScreen(
        onNavigateBack = { navController.popBackStack() }
    )
}
```

**6. Update Dependency Guard baseline:**

```bash
./gradlew dependencyGuard
```

**7. Checklist before first PR:**

- [ ] `build.gradle.kts` uses only permitted plugins and dependencies
- [ ] No imports of `:core-data`, `:core-location`, or other `:feature-*`
- [ ] ViewModel follows MVI pattern (State / Intent / SideEffect)
- [ ] Screen composable is stateless — all state via `collectAsState()`
- [ ] Accessibility checklist in `ACCESSIBILITY.md` completed
- [ ] At least one ViewModel unit test

---

## Adding a New Core Module

Use when: a concern is shared across multiple features but is neither domain logic  
nor UI components. Examples: analytics abstraction, crash reporting, feature flags.

**Steps are similar to feature module, with differences:**

```kotlin
// core/core-analytics/build.gradle.kts
plugins {
    alias(libs.plugins.micromobility.android.library)
    alias(libs.plugins.micromobility.hilt)
    // No compose plugin — core modules generally don't have UI
}

android {
    namespace = "com.yourorg.micromobility.core.analytics"
}
```

**Dependency rule for new core modules:**

Before creating the module, answer:
- Can it depend on `:core-domain`? (usually yes)
- Can `:feature-*` depend on it? (only if it contains no Android hardware or data layer code)
- Can it depend on `:core-data` or `:core-location`? (only if it bridges those layers)

Document the rule in this file before merging.

---

## Build Performance

### Parallelism

Gradle builds modules in parallel by default when they have no dependency on each other.  
The current module graph enables parallel compilation of:

- All `:feature-*` modules (after `:core-domain` and `:core-ui` are built)
- `:core-data` and `:core-ui` in parallel (both only depend on `:core-domain`)

**Critical path:**  
`:core-domain` → `:core-data` + `:core-ui` (parallel) → `:feature-*` (parallel) → `:app`

Avoid adding dependencies that extend the critical path without a strong reason.

### Configuration cache

Enable in `gradle.properties`:

```properties
org.gradle.configuration-cache=true
org.gradle.configuration-cache.problems=warn   # warn in early adoption, error later
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.daemon=true
org.gradle.jvmargs=-Xmx4g -Dfile.encoding=UTF-8
```

### Build baseline profile

For the `:app` module, maintain a Baseline Profile to improve ART compilation  
and reduce app startup time:

```bash
./gradlew :app:generateBaselineProfile
```

Baseline profiles are committed to `app/src/main/baseline-prof.txt`.

### Measuring build time

```bash
# Full build with timing
./gradlew assembleDebug --profile

# Task-level timing (Gradle 7.6+)
./gradlew assembleDebug --scan

# Which tasks are slowest
./gradlew assembleDebug --profile 2>&1 | grep "BUILD"
```

Target build times (from clean build on CI):

| Task | Target |
|---|---|
| `:core-domain:compileKotlin` | < 15s |
| `:core-data:compileKotlin` | < 30s |
| `:feature-ride:compileKotlin` | < 20s |
| Full `assembleDebug` | < 3 min |
| Incremental change in `:feature-ride` | < 45s |

---

## Testing Per Module

Each module owns its tests. Tests never cross module boundaries.

| Module | Test type | Location | Runner |
|---|---|---|---|
| `:core-domain` | Unit | `src/test/` | JUnit 5, plain JVM |
| `:core-data` | Unit + Room (in-memory) | `src/test/` + `src/androidTest/` | JUnit 5 + AndroidJUnit4 |
| `:core-location` | Unit | `src/test/` | JUnit 5 |
| `:core-ui` | Compose UI | `src/androidTest/` | ComposeTestRule |
| `:feature-*` | Unit (ViewModel) + Compose UI | `src/test/` + `src/androidTest/` | JUnit 5 + ComposeTestRule |
| `:app` | Integration / E2E | `src/androidTest/` | Espresso + AndroidJUnit4 |

```bash
# Run all unit tests
./gradlew test

# Run tests for a specific module
./gradlew :feature-ride:test

# Run instrumented tests (requires device)
./gradlew :core-data:connectedAndroidTest

# Run all checks (lint + test + dependency guard)
./gradlew check
```

---

## Common Mistakes

| Mistake | Detection | Fix |
|---|---|---|
| `:feature-*` importing from `:core-data` | Dependency Guard CI check fails | Move data access to a UseCase in `:core-domain` |
| Adding Android SDK import to `:core-domain` | Compile error (plugin is `kotlin.jvm`, not `android.library`) | Use domain wrappers (`LatLng` instead of `android.location.Location`) |
| Cross-feature dependency (`:feature-ride` importing from `:feature-search`) | Dependency Guard | Extract shared model to `:core-domain` |
| Hardcoding dependency version in `build.gradle.kts` | Code review | Add to `libs.versions.toml`, use `libs.xyz` alias |
| Forgetting to update Dependency Guard baseline | CI fails | Run `./gradlew dependencyGuard` and commit the updated baseline |
| Creating a Compose screen in `:core-domain` | Compile error (no Compose on classpath) | Move to `:core-ui` or a `:feature-*` module |
| Using `implementation` instead of `api` for transitive domain types in `:core-ui` | Compile errors in feature modules that use domain models | `:core-ui` should `api(projects.core.coreDomain)` |
| Room entity in `:core-domain` | Breaks domain purity; Room annotation requires Android | Keep entities in `:core-data`, map to domain models |
| Starting `LocationTrackingService` from a `:feature-*` composable | No compile error — runtime issue only | Start via `SideEffect` in `:app` screen layer |

---

## Decision Log

| Decision | Rationale |
|---|---|
| `:core-domain` uses `kotlin.jvm` plugin, not `android.library` | Enforces domain purity at compile time — not by convention |
| `:feature-*` cannot see `:core-data` | Prevents Room entities, DAOs, and DTOs from appearing in ViewModel code |
| No `:feature-*` → `:feature-*` dependency | Prevents spaghetti coupling; shared state goes through `:core-domain` |
| Convention plugins in `:build-logic` rather than `buildSrc` | `buildSrc` is always rebuilt on any change; `:build-logic` uses Gradle caching |
| Single `settings.gradle.kts` with `include` rather than composite builds | Simpler mental model for a team under 10; composite builds add complexity without clear benefit at this scale |
| Dependency Guard over manual inspection | Automated, runs in CI, catches transitive dependency changes that humans miss |

---

*Last updated: <!-- date -->*  
*Owner: Android Tech Lead*  
*Review: Required before creating or removing any module, or changing module dependencies*
