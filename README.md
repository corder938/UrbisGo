# UrbisGo — Android

> Real-time ride and delivery tracking for the European micromobility market.  
> Built with Jetpack Compose · MVI · Foreground Location Service · Android 14+

---

## Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Module Structure](#module-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Setup](#setup)
  - [Running the App](#running-the-app)
- [Key Features & Domain](#key-features--domain)
- [Permissions](#permissions)
- [Configuration](#configuration)
- [Testing](#testing)
- [CI/CD](#cicd)
- [Documentation Index](#documentation-index)
- [Contributing](#contributing)
- [Team Contacts](#team-contacts)

---

## Project Overview

**Micromobility & Logistics Tracker** is an Android application for end users and couriers/drivers  
operating in the European micromobility and on-demand delivery market (think: Bolt, Tier, Wolt).

Core product flows:

| Flow | States |
|------|--------|
| Ride/Delivery | `Idle → Searching → MatchFound → Waiting → InProgress → Completed` |
| Cancellation | `Searching / Waiting / InProgress → Cancelled` |
| Error recovery | `Any → Error → Retry → previous state` |

The app is designed for **hands-free use** — couriers and drivers operate it while moving,  
so accessibility (TalkBack, large touch targets, semantic grouping) is a first-class concern,  
not an afterthought.

**Assumptions:**
- Backend communicates via REST + WebSocket (real-time state updates)
- European market: GDPR-compliant permission flows, multilingual (en, de, fr, pl baseline)
- Android 9+ minimum SDK, Android 14 target SDK

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| UI | Jetpack Compose |
| Architecture | MVI (Orbit MVI) |
| DI | Hilt |
| Navigation | Compose Navigation |
| Networking | Retrofit + OkHttp + kotlinx.serialization |
| Real-time | OkHttp WebSocket |
| Local storage | Room + DataStore Preferences |
| Location | Google Play Services Location (FusedLocationProviderClient) |
| Background tracking | Foreground Service (`foregroundServiceType="location"`) |
| Maps | Google Maps Compose |
| Images | Coil |
| Testing | JUnit 5 · MockK · Turbine · ComposeTestRule |
| CI | GitHub Actions |
| Crash reporting | Firebase Crashlytics |
| Analytics | Firebase Analytics (event names defined in `docs/ops/ANALYTICS.md`) |
| Lint / static analysis | Detekt + Android Lint |

> **ADR log:** All major technology choices are documented in [`docs/adr/`](docs/adr/).  
> Start with [ADR-001 (MVI library)](docs/adr/ADR-001-mvi-library.md) and [ADR-002 (DI)](docs/adr/ADR-002-di-framework.md).

---

## Architecture

The project follows **Clean Architecture** with an **MVI pattern** on the UI layer.

```
┌─────────────────────────────────────────────────────┐
│                     UI Layer                        │
│  Composable Screens  ←→  ViewModel (Orbit MVI)      │
│  State · Intent · SideEffect                        │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────┐
│                   Domain Layer                      │
│  UseCases · Repository interfaces · Models          │
│  RideStateMachine · Business rules                  │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────┐
│                    Data Layer                       │
│  RemoteDataSource (Retrofit / WebSocket)            │
│  LocalDataSource (Room / DataStore)                 │
│  Repository implementations                         │
└─────────────────────────────────────────────────────┘
```

**Key architectural decisions:**

- **MVI**: every screen has an immutable `State`, user actions are `Intent`s,  
  one-shot effects (navigation, FGS start) are `SideEffect`s via `Channel`.
- **State machine**: `RideState` is a sealed class hierarchy. Illegal state transitions  
  are caught at compile time or throw `IllegalStateTransitionException` at runtime.
- **Foreground Service**: `LocationTrackingService` runs with `foregroundServiceType="location"`.  
  It is started/stopped as a `SideEffect` — never directly from a composable.
- **Offline-first**: active ride state is persisted in Room. If the app is killed mid-ride,  
  it restores state from the database on next launch.

→ Full architecture doc: [`docs/arch/ARCHITECTURE.md`](docs/arch/ARCHITECTURE.md)  
→ MVI pattern: [`docs/arch/MVI.md`](docs/arch/MVI.md)  
→ State machine: [`docs/arch/STATE_MACHINE.md`](docs/arch/STATE_MACHINE.md)

---

## Module Structure

```
app/                        # Application module, Hilt setup, MainActivity
├── core/
│   ├── core-ui/            # Design system, theme, shared Compose components
│   ├── core-domain/        # Base UseCases, Result wrapper, shared domain models
│   ├── core-data/          # Network client, DB setup, base Repository
│   └── core-location/      # FusedLocationProviderClient wrapper, LocationTrackingService
├── feature/
│   ├── feature-search/     # Search screen: Idle + Searching states
│   ├── feature-ride/       # Active ride screen: Waiting + InProgress + Completed
│   ├── feature-history/    # Past rides list and detail
│   └── feature-settings/   # Profile, permissions, preferences
└── build-logic/            # Convention plugins, shared Gradle config
```

> **Note:** Multi-module setup is intentional for build-time parallelism and  
> feature isolation. New features always go into a dedicated `feature-*` module.  
> See [`docs/arch/MODULE_STRUCTURE.md`](docs/arch/MODULE_STRUCTURE.md) for the full dependency graph.

---

## Getting Started

### Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Android Studio | Hedgehog 2023.1.1+ | or latest stable |
| JDK | 17 | bundled with AS, or via SDKMAN |
| Android SDK | API 34 (target), API 28 (min) | install via SDK Manager |
| Google Maps API key | — | see Setup below |
| `google-services.json` | — | Firebase config, see Setup below |

### Setup

**1. Clone the repository**

```bash
git clone git@github.com:your-org/micromobility-tracker-android.git
cd micromobility-tracker-android
```

**2. Set up local secrets**

Copy the template and fill in your values:

```bash
cp local.properties.template local.properties
```

Edit `local.properties`:

```properties
# Google Maps
MAPS_API_KEY=your_key_here

# Backend
BASE_URL=https://api.your-backend.dev/v1/
WS_URL=wss://ws.your-backend.dev/v1/

# Feature flags (optional, overrides remote config)
FEATURE_FLAG_BACKGROUND_TRACKING=true
```

> ⚠️ `local.properties` is `.gitignore`d. Never commit secrets.

**3. Add Firebase config**

Place `google-services.json` from [Firebase Console](https://console.firebase.google.com)  
into the `app/` directory:

```bash
cp ~/Downloads/google-services.json app/google-services.json
```

**4. Sync and build**

```bash
./gradlew assembleDebug
```

First build downloads ~300MB of dependencies. Subsequent builds are cached.

### Running the App

**From Android Studio:** Press ▶ with a connected device or emulator.

**From terminal:**

```bash
# Install debug build
./gradlew installDebug

# Launch activity
adb shell am start -n com.yourorg.tracker.debug/com.yourorg.tracker.MainActivity
```

**Testing location on emulator:**

Use the Extended Controls → Location panel in Android Studio, or push a GPX file:

```bash
adb emu geo nmea < assets/test-route-berlin.gpx
```

---

## Key Features & Domain

### Ride Lifecycle State Machine

The core of the app is a state machine that mirrors the backend ride lifecycle.  
State lives in `RideStateMachine` in the `:core-domain` module.

```
         ┌──────────────────────────────────────┐
         │                                      │
  Idle ──► Searching ──► MatchFound ──► Waiting ──► InProgress ──► Completed
         │                                      │
         └──────────── Cancelled ───────────────┘
                           │
                         Error ──► (retry) ──► Searching
```

Transitions are driven by:
- User `Intent`s (start search, cancel)
- WebSocket events from the server (`driver_matched`, `ride_started`, `ride_completed`)
- Timeouts (no driver found within N seconds → back to `Idle` with `NoDriverAvailableError`)

→ [`docs/arch/STATE_MACHINE.md`](docs/arch/STATE_MACHINE.md) — full transition table and Kotlin code

### Background Location Tracking

The app tracks location while a ride is `InProgress` using a Foreground Service  
with `foregroundServiceType="location"`. This works even when the app is backgrounded.

Permission flow:
1. Foreground location (`ACCESS_FINE_LOCATION`) — requested at ride search
2. Background location (`ACCESS_BACKGROUND_LOCATION`) — requested separately, with explicit rationale

→ [`docs/platform/LOCATION_PERMISSIONS.md`](docs/platform/LOCATION_PERMISSIONS.md) — full step-by-step flow  
→ [`docs/platform/FOREGROUND_SERVICE.md`](docs/platform/FOREGROUND_SERVICE.md) — FGS lifecycle

### Accessibility

The app is designed for **TalkBack-first operation** for couriers and drivers.

Key patterns used:
- `Modifier.semantics(mergeDescendants = true)` — groups driver/order cards into single TalkBack nodes
- `stateDescription` — describes dynamic state (ETA, ride status) in accessibility announcements
- `liveRegion = LiveRegionMode.Polite` — announces real-time updates without interrupting speech
- `traversalIndex` — controls TalkBack reading order on the active ride screen
- All interactive elements: minimum 48×48dp touch targets

→ [`docs/platform/ACCESSIBILITY.md`](docs/platform/ACCESSIBILITY.md)  
→ [`docs/platform/COMPOSE_SEMANTICS.md`](docs/platform/COMPOSE_SEMANTICS.md)

---

## Permissions

| Permission | When requested | Required |
|-----------|---------------|----------|
| `ACCESS_FINE_LOCATION` | Before first ride search | Yes |
| `ACCESS_COARSE_LOCATION` | Fallback if fine denied | Partial |
| `ACCESS_BACKGROUND_LOCATION` | Separately, after foreground granted | For background tracking |
| `FOREGROUND_SERVICE` | Declared in manifest | Yes (automatic) |
| `FOREGROUND_SERVICE_LOCATION` | Declared in manifest (Android 14+) | Yes |
| `POST_NOTIFICATIONS` | App launch (Android 13+) | For FGS notification |

> **Play Store note:** Background location requires a Privacy Policy and  
> a justification form in Play Console. See [`docs/platform/LOCATION_PERMISSIONS.md`](docs/platform/LOCATION_PERMISSIONS.md).

---

## Configuration

### Build Variants

| Variant | Backend | Logging | ProGuard |
|---------|---------|---------|---------|
| `debug` | dev / local | verbose | off |
| `staging` | staging | info | off |
| `release` | production | error only | on |

Switch variant in Android Studio via **Build → Select Build Variant**.

### Feature Flags

Feature flags are managed via Firebase Remote Config.  
Local overrides (for development) go in `local.properties`:

```properties
FEATURE_FLAG_BACKGROUND_TRACKING=true
FEATURE_FLAG_NEW_RIDE_CARD=false
```

Flag definitions: [`docs/ops/FEATURE_FLAGS.md`](docs/ops/FEATURE_FLAGS.md) *(add later)*

---

## Testing

```bash
# Unit tests
./gradlew test

# Instrumented tests (requires connected device or emulator)
./gradlew connectedAndroidTest

# Specific module
./gradlew :feature-ride:test

# Run with coverage report
./gradlew testDebugUnitTestCoverage
```

Test philosophy:

- **Unit tests**: all ViewModels, UseCases, state machine transitions, Repository (fakes, no mocks where possible)
- **Integration tests**: Room DB, DataStore
- **UI tests**: ComposeTestRule for critical screens; accessibility checks via `AccessibilityChecks.enable()`
- **State machine**: every legal transition has a test; illegal transitions are tested to throw

→ [`docs/quality/TESTING.md`](docs/quality/TESTING.md) — full testing strategy

---

## CI/CD

Every pull request runs:

```
Lint → Detekt → Unit Tests → Assemble Debug → (on main) Assemble Release
```

Pipelines: [`.github/workflows/`](.github/workflows/)

| Workflow | Trigger | What it does |
|---------|---------|-------------|
| `ci.yml` | Push to any branch, PR | Lint, tests, build |
| `release.yml` | Tag `v*` | Build release AAB, upload to Play Store (internal) |
| `nightly.yml` | Scheduled 03:00 UTC | Full E2E suite on Firebase Test Lab |

→ [`docs/ops/CICD.md`](docs/ops/CICD.md)

---

## Documentation Index

| Document | What it covers |
|---------|---------------|
| [`docs/ONBOARDING.md`](docs/ONBOARDING.md) | First day setup and orientation |
| [`docs/arch/ARCHITECTURE.md`](docs/arch/ARCHITECTURE.md) | Layer structure, DI, dependency rules |
| [`docs/arch/MVI.md`](docs/arch/MVI.md) | MVI pattern, State/Intent/SideEffect |
| [`docs/arch/STATE_MACHINE.md`](docs/arch/STATE_MACHINE.md) | Ride lifecycle state machine |
| [`docs/arch/DATA_LAYER.md`](docs/arch/DATA_LAYER.md) | Room, DataStore, offline-first |
| [`docs/arch/ERROR_HANDLING.md`](docs/arch/ERROR_HANDLING.md) | Result type, error hierarchy, recovery |
| [`docs/platform/LOCATION_PERMISSIONS.md`](docs/platform/LOCATION_PERMISSIONS.md) | Step-by-step permission flow, Android 14+ |
| [`docs/platform/FOREGROUND_SERVICE.md`](docs/platform/FOREGROUND_SERVICE.md) | FGS lifecycle, notification, kill scenarios |
| [`docs/platform/ACCESSIBILITY.md`](docs/platform/ACCESSIBILITY.md) | TalkBack, semantics, hands-free UX |
| [`docs/platform/COMPOSE_SEMANTICS.md`](docs/platform/COMPOSE_SEMANTICS.md) | Semantics code patterns for this project |
| [`docs/product/RIDE_LIFECYCLE.md`](docs/product/RIDE_LIFECYCLE.md) | Business rules, edge cases, cancellation |
| [`docs/api/API_CONTRACT.md`](docs/api/API_CONTRACT.md) | REST endpoints, WebSocket events, error codes |
| [`docs/quality/TESTING.md`](docs/quality/TESTING.md) | Test pyramid, coverage targets, tooling |
| [`docs/ops/CICD.md`](docs/ops/CICD.md) | Pipelines, checks, deployment |
| [`docs/ops/RELEASE.md`](docs/ops/RELEASE.md) | Release branches, Play Store, versioning |
| [`docs/ops/ANALYTICS.md`](docs/ops/ANALYTICS.md) | Event taxonomy, naming conventions |
| [`docs/adr/`](docs/adr/) | Architecture Decision Records |

---

## Contributing

Please read [`CONTRIBUTING.md`](CONTRIBUTING.md) before opening a PR.

Quick checklist:
- [ ] Branch from `develop`, not `main`
- [ ] Branch naming: `feature/`, `fix/`, `chore/`, `docs/`
- [ ] Commit messages: [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `refactor:`, etc.)
- [ ] New screen → update `docs/product/UI_STATES.md`
- [ ] New permission usage → update `docs/platform/LOCATION_PERMISSIONS.md`
- [ ] New architecture decision → add ADR to `docs/adr/`
- [ ] PR description: fill in the template (What / Why / How to test)

---

## Team Contacts

| Role | Name | Contact |
|------|------|---------|
| Tech Lead / Android Architect | — | @handle |
| Android Developer | — | @handle |
| Product Manager | — | @handle |
| Designer | — | @handle |
| Backend Lead | — | @handle |
| QA | — | @handle |

> **New to the project?** Start with [`docs/ONBOARDING.md`](docs/ONBOARDING.md).  
> It covers environment setup, a tour of the codebase, and your first tasks.

---

*Last updated: <!-- date -->*  
*Android team · [your-org](https://github.com/your-org)*
