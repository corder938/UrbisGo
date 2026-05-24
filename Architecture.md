# Architecture

> Micromobility & Logistics Tracker — Android  
> Architecture reference for the Android team and tech lead.

---

## Table of Contents

- [Guiding Principles](#guiding-principles)
- [Layer Overview](#layer-overview)
- [UI Layer — MVI](#ui-layer--mvi)
  - [State](#state)
  - [Intent](#intent)
  - [SideEffect](#sideeffect)
  - [ViewModel](#viewmodel)
  - [Composable Screen](#composable-screen)
- [Domain Layer](#domain-layer)
  - [UseCases](#usecases)
  - [Repository Interfaces](#repository-interfaces)
  - [Models](#models)
  - [RideStateMachine](#ridestate-machine)
- [Data Layer](#data-layer)
  - [Remote](#remote)
  - [Local](#local)
  - [Repository Implementations](#repository-implementations)
- [State Machine — Ride Lifecycle](#state-machine--ride-lifecycle)
- [Foreground Service Integration](#foreground-service-integration)
- [Dependency Injection](#dependency-injection)
- [Navigation](#navigation)
- [Error Handling](#error-handling)
- [Offline-First Strategy](#offline-first-strategy)
- [Module Graph](#module-graph)
- [Dependency Rules](#dependency-rules)
- [Anti-Patterns — What We Do Not Do](#anti-patterns--what-we-do-not-do)
- [Decision Log](#decision-log)

---

## Guiding Principles

These principles drive every architectural decision in this project.

**1. Unidirectional data flow — always.**  
UI never mutates state directly. Every state change passes through the ViewModel.  
`Intent → ViewModel → State / SideEffect → UI`

**2. The domain layer owns business logic — no exceptions.**  
ViewModels orchestrate; they do not contain if/else business rules.  
Business rules live in UseCases and `RideStateMachine`.

**3. Illegal states are unrepresentable.**  
`RideState` is a sealed class hierarchy. A `Completed` ride cannot transition to `InProgress`.  
Compiler catches it. If it somehow reaches runtime, it throws — never silently continues.

**4. The UI layer is dumb.**  
Composables render `State`. They fire `Intent`s. That is all.  
No network calls, no DB access, no business logic in composables.

**5. Foreground Service is a side effect — not a component.**  
`LocationTrackingService` is started and stopped via `SideEffect` from the ViewModel.  
It is never started directly from a composable or from within a UseCase.

**6. Accessibility is architecture, not styling.**  
Semantic structure is defined at the same level as visual structure.  
Every composable that carries user-facing information has explicit `Modifier.semantics`.

---

## Layer Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                          UI Layer                               │
│                                                                 │
│   RideScreen (Composable)   ◄──── RideViewModel (Orbit MVI)    │
│       renders State               processes Intent              │
│       fires Intent                emits State + SideEffect      │
│                                                                 │
│   LocationPermissionScreen        SearchViewModel               │
│   SearchScreen                    ...                           │
└────────────────────────────┬────────────────────────────────────┘
                             │  UseCases only (no direct repo access)
┌────────────────────────────▼────────────────────────────────────┐
│                        Domain Layer                             │
│                                                                 │
│   StartRideUseCase         CancelRideUseCase                    │
│   ObserveRideStateUseCase  GetDriverLocationUseCase             │
│                                                                 │
│   RideStateMachine         (sealed RideState hierarchy)         │
│   Repository interfaces    RideRepository, LocationRepository   │
│   Domain models            Ride, Driver, LatLng, RideState      │
└────────────────────────────┬────────────────────────────────────┘
                             │  Implementation only (no domain logic)
┌────────────────────────────▼────────────────────────────────────┐
│                         Data Layer                              │
│                                                                 │
│   RideRepositoryImpl       LocationRepositoryImpl               │
│                                                                 │
│   RemoteDataSource         LocalDataSource                      │
│   ├─ RideApiService        ├─ RideDao (Room)                    │
│   ├─ RideWebSocketClient   ├─ DriverLocationDao                 │
│   └─ Retrofit / OkHttp     └─ DataStore (prefs, permission state│
└─────────────────────────────────────────────────────────────────┘

   LocationTrackingService (FGS)
   ├─ runs independently of the Activity lifecycle
   ├─ reads from FusedLocationProviderClient
   └─ writes to LocationRepository (via injection)
```

---

## UI Layer — MVI

We use **[Orbit MVI](https://github.com/orbit-mvi/orbit-mvi)** as the MVI framework.  
Every screen has exactly one ViewModel. The ViewModel is the single source of truth for that screen.

### State

`State` is an **immutable data class** that fully describes everything the UI needs to render.  
No derived state in composables. No `remember { }` holding business data.

```kotlin
// feature-ride/src/main/.../RideUiState.kt

data class RideUiState(
    val rideState: RideState = RideState.Idle,
    val driverLocation: LatLng? = null,
    val etaSeconds: Int? = null,
    val isLoading: Boolean = false,
    val error: RideError? = null,
    val permissionState: LocationPermissionState = LocationPermissionState.Unknown,
)
```

Rules for `State`:
- Always has sensible defaults (no nullable fields where a sealed subtype works better)
- Never contains Android framework types (`Context`, `View`, `Drawable`)
- Never contains lambdas or callbacks
- Nested objects are themselves immutable data classes

### Intent

`Intent` represents **a user action or a system event** that the ViewModel should react to.  
Naming convention: verb in imperative form.

```kotlin
// feature-ride/src/main/.../RideIntent.kt

sealed interface RideIntent {
    // User actions
    data object StartSearch : RideIntent
    data object CancelRide : RideIntent
    data object RetryAfterError : RideIntent
    data object ConfirmCompletion : RideIntent

    // System / reactive events
    data class LocationPermissionResult(
        val granted: Boolean,
        val backgroundGranted: Boolean,
    ) : RideIntent

    data class DriverLocationUpdated(val location: LatLng) : RideIntent
    data class ServerRideEventReceived(val event: RideServerEvent) : RideIntent
    data class EtaUpdated(val seconds: Int) : RideIntent
}
```

Rules for `Intent`:
- Never carries Android framework types
- System events (WebSocket updates, location callbacks) are also modelled as `Intent`s
- One `sealed interface` per screen/ViewModel; do not share Intents across screens

### SideEffect

`SideEffect` covers **one-shot events** — things that happen once and are not part of the rendered State.  
Delivered via `Channel` (not `StateFlow`), so they are not replayed on recomposition.

```kotlin
// feature-ride/src/main/.../RideSideEffect.kt

sealed interface RideSideEffect {
    data object StartLocationTrackingService : RideSideEffect
    data object StopLocationTrackingService : RideSideEffect
    data object NavigateToRideCompleteScreen : RideSideEffect
    data object NavigateBack : RideSideEffect
    data class ShowSnackbar(val message: UiText) : RideSideEffect
    data class RequestLocationPermission(val type: LocationPermissionType) : RideSideEffect
    data class OpenAppSettings(val reason: SettingsReason) : RideSideEffect
}
```

Rules for `SideEffect`:
- Navigation is always a `SideEffect`, never embedded in `State`
- Starting/stopping `LocationTrackingService` is always a `SideEffect`
- Snackbars, dialogs, permission requests are `SideEffect`s
- Never use `SideEffect` for things that should survive recomposition — that belongs in `State`

### ViewModel

```kotlin
// feature-ride/src/main/.../RideViewModel.kt

@HiltViewModel
class RideViewModel @Inject constructor(
    private val startRideUseCase: StartRideUseCase,
    private val cancelRideUseCase: CancelRideUseCase,
    private val observeRideStateUseCase: ObserveRideStateUseCase,
    private val rideStateMachine: RideStateMachine,
) : ViewModel(), ContainerHost<RideUiState, RideSideEffect> {

    override val container = container<RideUiState, RideSideEffect>(RideUiState())

    init {
        observeRideState()
    }

    fun onIntent(intent: RideIntent) = when (intent) {
        is RideIntent.StartSearch -> handleStartSearch()
        is RideIntent.CancelRide -> handleCancelRide()
        is RideIntent.ServerRideEventReceived -> handleServerEvent(intent.event)
        is RideIntent.RetryAfterError -> handleRetry()
        // ...
    }

    private fun handleStartSearch() = intent {
        val nextState = rideStateMachine.transition(
            current = state.rideState,
            event = RideEvent.SearchStarted,
        )
        reduce { state.copy(rideState = nextState, isLoading = true) }
        postSideEffect(RideSideEffect.RequestLocationPermission(LocationPermissionType.Foreground))

        startRideUseCase()
            .onSuccess { reduce { state.copy(isLoading = false) } }
            .onFailure { error ->
                reduce { state.copy(isLoading = false, error = error.toRideError()) }
            }
    }

    private fun observeRideState() = intent {
        observeRideStateUseCase()
            .collect { serverState ->
                val nextState = rideStateMachine.transition(
                    current = state.rideState,
                    event = serverState.toRideEvent(),
                )
                reduce { state.copy(rideState = nextState) }

                if (nextState is RideState.InProgress) {
                    postSideEffect(RideSideEffect.StartLocationTrackingService)
                }
                if (nextState is RideState.Completed) {
                    postSideEffect(RideSideEffect.StopLocationTrackingService)
                    postSideEffect(RideSideEffect.NavigateToRideCompleteScreen)
                }
            }
    }
}
```

### Composable Screen

```kotlin
// feature-ride/src/main/.../RideScreen.kt

@Composable
fun RideScreen(
    viewModel: RideViewModel = hiltViewModel(),
    onNavigateToComplete: () -> Unit,
    onNavigateBack: () -> Unit,
) {
    val state by viewModel.collectAsState()
    val context = LocalContext.current

    // SideEffect handling — the ONLY place we interact with Android framework
    viewModel.collectSideEffect { effect ->
        when (effect) {
            is RideSideEffect.StartLocationTrackingService ->
                context.startLocationTrackingService()
            is RideSideEffect.StopLocationTrackingService ->
                context.stopLocationTrackingService()
            is RideSideEffect.NavigateToRideCompleteScreen ->
                onNavigateToComplete()
            is RideSideEffect.ShowSnackbar ->
                // show snackbar via SnackbarHostState
        }
    }

    RideScreenContent(
        state = state,
        onIntent = viewModel::onIntent,
    )
}

// Stateless content composable — easier to preview and test
@Composable
private fun RideScreenContent(
    state: RideUiState,
    onIntent: (RideIntent) -> Unit,
) {
    // render based on state.rideState sealed subtype
    when (val rideState = state.rideState) {
        is RideState.Idle -> IdleContent(onIntent)
        is RideState.Searching -> SearchingContent(state.isLoading, onIntent)
        is RideState.Waiting -> WaitingContent(rideState, state.etaSeconds, onIntent)
        is RideState.InProgress -> InProgressContent(rideState, state.driverLocation, onIntent)
        is RideState.Completed -> {} // handled via SideEffect navigation
        is RideState.Cancelled -> CancelledContent(rideState.reason, onIntent)
        is RideState.Error -> ErrorContent(state.error, onIntent)
    }
}
```

---

## Domain Layer

The domain layer has **no Android dependencies** — it is pure Kotlin.  
It can be tested with plain JUnit without Robolectric or any Android framework.

### UseCases

Each UseCase does one thing. It receives domain inputs, calls one or more repositories, returns a `Result`.

```kotlin
// core-domain/src/main/.../StartRideUseCase.kt

class StartRideUseCase @Inject constructor(
    private val rideRepository: RideRepository,
    private val locationRepository: LocationRepository,
) {
    suspend operator fun invoke(): Result<Ride> {
        val location = locationRepository.getLastKnownLocation()
            ?: return Result.failure(DomainError.LocationUnavailable)

        return rideRepository.startSearch(origin = location)
    }
}
```

UseCase rules:
- `operator fun invoke()` as the single entry point
- Returns `Result<T>` — never throws (except programming errors)
- Suspending or returns `Flow` — never blocking
- No UI imports, no Android imports (exception: `android.location.Location` domain type is ok)

### Repository Interfaces

```kotlin
// core-domain/src/main/.../RideRepository.kt

interface RideRepository {
    suspend fun startSearch(origin: LatLng): Result<Ride>
    suspend fun cancelRide(rideId: String): Result<Unit>
    fun observeRideEvents(): Flow<RideServerEvent>
    suspend fun getActiveRide(): Ride?         // for state restoration on launch
    suspend fun persistRideState(state: RideState)
}

// core-domain/src/main/.../LocationRepository.kt

interface LocationRepository {
    fun observeLocation(interval: LocationInterval): Flow<LatLng>
    suspend fun getLastKnownLocation(): LatLng?
}
```

### Models

```kotlin
// core-domain/src/main/.../models/

data class Ride(
    val id: String,
    val status: RideStatus,
    val origin: LatLng,
    val destination: LatLng?,
    val driver: Driver?,
    val startedAt: Instant?,
    val completedAt: Instant?,
)

data class Driver(
    val id: String,
    val name: String,
    val rating: Float,
    val vehiclePlate: String,
    val location: LatLng?,
)

// Never use android.location.Location in domain — use this wrapper
data class LatLng(val latitude: Double, val longitude: Double)
```

### RideState Machine

`RideStateMachine` lives in the domain layer — it contains pure business logic.

```kotlin
// core-domain/src/main/.../RideStateMachine.kt

class RideStateMachine @Inject constructor() {

    fun transition(current: RideState, event: RideEvent): RideState {
        return when {
            current is RideState.Idle && event is RideEvent.SearchStarted ->
                RideState.Searching

            current is RideState.Searching && event is RideEvent.DriverMatched ->
                RideState.Waiting(
                    rideId = event.rideId,
                    driver = event.driver,
                    etaSeconds = event.etaSeconds,
                )

            current is RideState.Waiting && event is RideEvent.RideStarted ->
                RideState.InProgress(
                    rideId = current.rideId,
                    driver = current.driver,
                    startedAt = event.startedAt,
                )

            current is RideState.InProgress && event is RideEvent.RideCompleted ->
                RideState.Completed(
                    rideId = current.rideId,
                    summary = event.summary,
                )

            // Cancellation is valid from Searching, Waiting, InProgress
            (current is RideState.Searching ||
             current is RideState.Waiting ||
             current is RideState.InProgress) && event is RideEvent.Cancelled ->
                RideState.Cancelled(reason = event.reason)

            // Any state can go to Error
            event is RideEvent.ErrorOccurred ->
                RideState.Error(type = event.errorType, previousState = current)

            // Recovery from Error
            current is RideState.Error && event is RideEvent.RetryRequested ->
                RideState.Idle

            else -> throw IllegalStateTransitionException(
                current = current,
                event = event,
            )
        }
    }
}

// Sealed state hierarchy
sealed interface RideState {
    data object Idle : RideState
    data object Searching : RideState

    data class Waiting(
        val rideId: String,
        val driver: Driver,
        val etaSeconds: Int,
    ) : RideState

    data class InProgress(
        val rideId: String,
        val driver: Driver,
        val startedAt: Instant,
    ) : RideState

    data class Completed(
        val rideId: String,
        val summary: RideSummary,
    ) : RideState

    data class Cancelled(val reason: CancellationReason) : RideState

    data class Error(
        val type: RideErrorType,
        val previousState: RideState,  // enables contextual recovery
    ) : RideState
}
```

→ Full transition table and edge cases: [`docs/arch/STATE_MACHINE.md`](STATE_MACHINE.md)

---

## Data Layer

The data layer implements the repository interfaces from domain.  
It knows about Retrofit, Room, DataStore, OkHttp — domain does not.

### Remote

```kotlin
// core-data/src/main/.../remote/RideApiService.kt

interface RideApiService {
    @POST("rides/search")
    suspend fun startSearch(@Body body: StartSearchRequest): Response<RideDto>

    @DELETE("rides/{rideId}")
    suspend fun cancelRide(@Path("rideId") rideId: String): Response<Unit>
}

// core-data/src/main/.../remote/RideWebSocketClient.kt

class RideWebSocketClient @Inject constructor(private val okHttpClient: OkHttpClient) {
    fun observeEvents(rideId: String): Flow<RideServerEvent> = callbackFlow {
        val ws = okHttpClient.newWebSocket(
            request = buildWebSocketRequest(rideId),
            listener = object : WebSocketListener() {
                override fun onMessage(webSocket: WebSocket, text: String) {
                    trySend(parseEvent(text))
                }
                override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
                    close(t)
                }
            }
        )
        awaitClose { ws.cancel() }
    }
}
```

### Local

```kotlin
// core-data/src/main/.../local/RideDao.kt

@Dao
interface RideDao {
    @Query("SELECT * FROM rides WHERE status = 'ACTIVE' LIMIT 1")
    suspend fun getActiveRide(): RideEntity?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsertRide(ride: RideEntity)

    @Query("UPDATE rides SET status = :status WHERE id = :rideId")
    suspend fun updateRideStatus(rideId: String, status: String)
}
```

DataStore is used for:
- Permission flow progress (which step the user reached)
- User preferences (notification sound, map type)
- Last known location (for fast startup)

### Repository Implementations

```kotlin
// core-data/src/main/.../repository/RideRepositoryImpl.kt

class RideRepositoryImpl @Inject constructor(
    private val apiService: RideApiService,
    private val webSocketClient: RideWebSocketClient,
    private val rideDao: RideDao,
    private val dispatcher: CoroutineDispatcher,  // injected, testable
) : RideRepository {

    override suspend fun startSearch(origin: LatLng): Result<Ride> =
        withContext(dispatcher) {
            runCatching {
                apiService.startSearch(origin.toRequest()).bodyOrThrow().toDomain()
            }
        }

    override fun observeRideEvents(): Flow<RideServerEvent> =
        webSocketClient.observeEvents(/* rideId from active ride */)

    override suspend fun getActiveRide(): Ride? =
        withContext(dispatcher) { rideDao.getActiveRide()?.toDomain() }
}
```

---

## Foreground Service Integration

`LocationTrackingService` is a `Service` with `foregroundServiceType="location"`.  
It is architecturally **outside** the MVI loop — it communicates via the repository, not via the ViewModel.

```
ViewModel
  └─► SideEffect: StartLocationTrackingService
         └─► Screen collects SideEffect
                └─► context.startForegroundService(Intent(context, LocationTrackingService::class.java))

LocationTrackingService
  └─► FusedLocationProviderClient (location updates)
         └─► LocationRepository.emitLocation(latLng)   ← shared StateFlow / Channel
                └─► ObserveDriverLocationUseCase
                       └─► ViewModel receives via Flow
                              └─► reduce { state.copy(driverLocation = latLng) }
```

The Service **does not hold a reference to the ViewModel**.  
The ViewModel **does not hold a reference to the Service**.  
They communicate only through the repository layer.

→ Full FGS lifecycle doc: [`docs/platform/FOREGROUND_SERVICE.md`](../platform/FOREGROUND_SERVICE.md)

---

## Dependency Injection

We use **Hilt** for DI. Every layer has its own Hilt module.

```
app/
  AppModule.kt          — OkHttpClient, Retrofit, DB instance
  CoroutineModule.kt    — dispatchers (IO, Default, Main)

feature-ride/
  RideModule.kt         — RideApiService, RideWebSocketClient

core-domain/
  DomainModule.kt       — UseCases, RideStateMachine

core-location/
  LocationModule.kt     — FusedLocationProviderClient, LocationTrackingService binding
```

Rules:
- UseCases are `@Inject constructor` — no manual `@Provides` needed
- Repository bindings: `@Binds` to bind interface to implementation
- Dispatchers are injected via `@Named("IO")` / `@Named("Default")` — never hardcoded

---

## Navigation

We use **Compose Navigation** with a single `NavHost` in `MainActivity`.

```kotlin
// app/src/main/.../navigation/AppNavHost.kt

@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(navController, startDestination = Screen.Search.route) {
        composable(Screen.Search.route) {
            SearchScreen(
                onRideStarted = { navController.navigate(Screen.ActiveRide.route) }
            )
        }
        composable(Screen.ActiveRide.route) {
            RideScreen(
                onNavigateToComplete = { navController.navigate(Screen.RideComplete.route) {
                    popUpTo(Screen.Search.route)
                }},
                onNavigateBack = { navController.popBackStack() }
            )
        }
        composable(Screen.RideComplete.route) { RideCompleteScreen() }
        composable(Screen.Settings.route) { SettingsScreen() }
    }
}
```

Navigation rules:
- Navigation is always triggered by a `SideEffect` — never by a state flag or from a composable directly
- Back stack manipulation (popUpTo, launchSingleTop) is defined at the NavHost level
- Deep links for the FGS persistent notification are registered in the NavHost

---

## Error Handling

All errors in the domain layer are modelled as typed sealed classes.  
We never catch `Exception` and swallow it silently.

```kotlin
// core-domain/src/main/.../error/

sealed interface DomainError {
    data object LocationUnavailable : DomainError
    data object NetworkUnavailable : DomainError
    data class ServerError(val code: Int, val message: String) : DomainError
    data class RideNotFound(val rideId: String) : DomainError
    data object DriverCancelledRide : DomainError
}

// Result extension — used everywhere in the data layer
inline fun <T> Result<T>.onFailure(
    action: (DomainError) -> Unit
): Result<T> { /* ... */ }
```

Error flow:
1. Data layer catches network/DB exceptions, maps to `DomainError`, returns `Result.failure()`
2. UseCase propagates `Result` to ViewModel — never throws
3. ViewModel maps `DomainError` to `RideError` (UI-friendly) via extension function
4. `reduce { state.copy(error = rideError) }` — Error is rendered by the composable
5. User retries → `RideIntent.RetryAfterError` → ViewModel clears error, retries UseCase

→ Full strategy: [`docs/arch/ERROR_HANDLING.md`](ERROR_HANDLING.md)

---

## Offline-First Strategy

| Data | Online | Offline |
|------|--------|---------|
| Active ride state | Server via WebSocket | Room — restored on launch |
| Driver location | WebSocket stream | Last known position from Room |
| Ride history | Server | Room cache (stale-while-revalidate) |
| User preferences | DataStore | DataStore (always local) |
| Permission state | DataStore | DataStore (always local) |

Network state is observed via `ConnectivityManager.registerNetworkCallback()`,  
wrapped in a `NetworkMonitor` interface (injectable, testable with a fake).

When connectivity is restored:
1. `NetworkMonitor` emits `NetworkState.Available`
2. Repository layer triggers re-sync of in-flight operations
3. ViewModel receives updated state via `observeRideStateUseCase()`

→ Full strategy: [`docs/arch/OFFLINE_FIRST.md`](OFFLINE_FIRST.md)

---

## Module Graph

```
         ┌──────────────────────────┐
         │           app            │
         └──┬────────────────────┬──┘
            │                    │
     ┌──────▼──────┐      ┌──────▼──────┐
     │ feature-ride │      │feature-search│  ...other features
     └──────┬──────┘      └──────┬──────┘
            │                    │
     ┌──────▼────────────────────▼──────┐
     │           core-domain            │
     │   (no Android, pure Kotlin)      │
     └──────────────┬───────────────────┘
                    │
     ┌──────────────▼───────────────────┐
     │            core-data             │
     │   (Retrofit, Room, DataStore)    │
     └──────────────┬───────────────────┘
                    │
     ┌──────────────▼───────────────────┐
     │          core-location           │
     │   (FGS, FusedLocation, GPS)      │
     └──────────────────────────────────┘

     ┌──────────────────────────────────┐
     │            core-ui               │
     │  (Design system, shared compos.) │
     └──────────────────────────────────┘
     (used by all feature modules, no upward deps)
```

→ Full module breakdown: [`docs/arch/MODULE_STRUCTURE.md`](MODULE_STRUCTURE.md)

---

## Dependency Rules

| Module | Can depend on | Cannot depend on |
|--------|--------------|-----------------|
| `feature-*` | `core-domain`, `core-ui` | other `feature-*`, `core-data` directly |
| `core-domain` | nothing (pure Kotlin) | `core-data`, `core-ui`, Android SDK |
| `core-data` | `core-domain` | `feature-*`, `core-ui` |
| `core-location` | `core-domain`, `core-data` | `feature-*`, `core-ui` |
| `core-ui` | `core-domain` (for models) | `core-data`, `feature-*` |
| `app` | everything | — |

These rules are enforced at build time via [Dependency Guard](https://github.com/dropbox/dependency-guard)  
and checked in CI on every PR.

---

## Anti-Patterns — What We Do Not Do

| Anti-pattern | Why | What to do instead |
|-------------|-----|--------------------|
| Business logic in `@Composable` | Breaks testability, hard to track | Move to UseCase or ViewModel |
| Mutable state in composable (`remember { mutableStateOf() }` for business data) | Creates a second source of truth | All business state lives in `UiState` |
| `LiveData` in new code | Orbit MVI uses `StateFlow` natively | Use `collectAsState()` |
| Starting FGS from a composable | UI layer must not manage service lifecycle | Use `SideEffect` → collect in Screen-level composable |
| Catching `Exception` in UseCase and returning default value silently | Hides errors, impossible to react | Return `Result.failure(DomainError)` |
| Sharing `Intent` sealed classes across multiple screens | Tight coupling, unclear ownership | Each ViewModel has its own sealed `Intent` |
| Nullable fields in `UiState` where a sealed subtype would work | Forces `if (x != null)` chains in UI | Model state with sealed subtypes |
| `ViewModel.viewModelScope.launch` without structured concurrency | Leaks if ViewModel is cleared | Always use Orbit's `intent { }` or cancel via `viewModelScope` |
| Android framework types in domain models | Breaks domain purity, forces Robolectric | Use domain wrappers (`LatLng`, `Instant`) |

---

## Decision Log

Key decisions recorded as ADRs. Read these before changing anything significant.

| ADR | Decision | Status |
|-----|---------|--------|
| [ADR-001](../adr/ADR-001-mvi-library.md) | Orbit MVI over MVI-Kotlin / manual | Accepted |
| [ADR-002](../adr/ADR-002-di-framework.md) | Hilt over Koin | Accepted |
| [ADR-003](../adr/ADR-003-database.md) | Room over SQLDelight | Accepted |
| [ADR-004](../adr/ADR-004-serialization.md) | kotlinx.serialization over Gson/Moshi | Accepted |
| [ADR-005](../adr/ADR-005-navigation.md) | Compose Navigation over Jetpack Nav + Fragments | Accepted |

> When you make a decision that affects the architecture — add an ADR.  
> Template: [`docs/adr/ADR-000-template.md`](../adr/ADR-000-template.md)

---

*Last updated: <!-- date -->*  
*Owner: Android Tech Lead*
