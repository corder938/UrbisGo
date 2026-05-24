# ADR-000 — Template

> Copy this file, rename it `ADR-NNN-short-title.md`, fill in every section.  
> Do not leave placeholder text in a committed ADR.

---

**Status:** `Draft` | `Proposed` | `Accepted` | `Deprecated` | `Superseded by ADR-NNN`  
**Date:** YYYY-MM-DD  
**Deciders:** <!-- names or roles -->  
**Consulted:** <!-- people whose input was sought but who didn't decide -->  
**Informed:** <!-- people notified after the decision -->

---

## Context

<!-- 1-3 paragraphs. What is the situation that requires a decision?
     What constraints exist? What will happen if we don't decide?
     Write in present tense. Do not describe the decision itself here. -->

## Decision

<!-- One clear sentence stating what was decided.
     Example: "We will use X for Y."
     Optionally 1-2 sentences of immediate rationale. -->

## Options Considered

<!-- Table or list. For each option: what it is, pros, cons.
     Be fair — list genuine advantages of rejected options.
     Do not bury the chosen option. -->

## Consequences

### Positive
<!-- What becomes easier, faster, or safer as a result. -->

### Negative
<!-- What becomes harder, slower, or riskier. Be honest. -->

### Risks & Mitigations
<!-- What could go wrong, and what we do about it. -->

## Validation

<!-- How will we know the decision was correct?
     What metrics, signals, or review points will we check? -->

## Links

<!-- Related ADRs, docs, GitHub issues, benchmarks, library repos. -->

---
---

# ADR-001 — MVI Library Selection

**Status:** Accepted  
**Date:** 2024-06-01  
**Deciders:** Android Tech Lead, Senior Android Developer  
**Consulted:** —  
**Informed:** Full Android team

---

## Context

We are building a micromobility and logistics tracking app from scratch.  
The UI layer requires a predictable state management pattern to handle:

- A **ride lifecycle state machine** with multiple states (`Idle`, `Searching`, `Waiting`,  
  `InProgress`, `Completed`, `Cancelled`, `Error`) and strict transition rules.
- **Real-time updates** from a WebSocket — driver location, ETA changes, ride status events —  
  all arriving concurrently and needing to update a single immutable UI state.
- **One-shot side effects** that must not be replayed on recomposition — starting the  
  Foreground Service, navigating between screens, showing snackbars.
- **Background state changes** — the FGS writes location to Room, Room notifies the ViewModel  
  via a Flow, the ViewModel updates state — all while the user is not interacting.

We chose **MVI (Model-View-Intent)** as the pattern because:
- Unidirectional data flow prevents the race conditions that bidirectional patterns  
  introduce when concurrent WebSocket events and user actions arrive simultaneously.
- Immutable state makes every UI render deterministic and easy to test.
- The state machine naturally maps to MVI: each `RideEvent` is an `Intent`,  
  each `RideState` transition produces a new `State`.

The remaining question is **which library — if any — to use to implement MVI**.

The three realistic options are:

1. **Orbit MVI** — a library purpose-built for MVI on Android with Compose support
2. **MVI-Kotlin** — a more explicit, opinionated MVI framework by Arkadii Ivanov
3. **Manual implementation** — plain `ViewModel` + `StateFlow` + `Channel`

This ADR documents why we chose Orbit MVI.

---

## Decision

**We will use Orbit MVI (`orbit-mvi`) as the MVI framework for all ViewModels.**

Orbit was chosen because it provides the correct primitives (`intent {}`, `reduce {}`,  
`postSideEffect {}`) with minimal boilerplate, first-class Compose support, and a  
testing API (`test {}`) that makes ViewModel tests readable without mocking Orbit itself.

---

## Options Considered

### Option A — Orbit MVI ✅ Chosen

**What it is:**  
A lightweight MVI library built around `ContainerHost` and a `container` delegate.  
State, side effects, and intents are handled via a structured DSL.  
GitHub: https://github.com/orbit-mvi/orbit-mvi

**Core API:**

```kotlin
@HiltViewModel
class RideViewModel @Inject constructor(
    private val rideStateMachine: RideStateMachine,
    private val startRideUseCase: StartRideUseCase,
) : ViewModel(), ContainerHost<RideUiState, RideSideEffect> {

    override val container = container<RideUiState, RideSideEffect>(
        initialState = RideUiState(),
    )

    fun onIntent(intent: RideIntent) = when (intent) {
        RideIntent.StartSearch -> handleStartSearch()
        RideIntent.CancelRide  -> handleCancelRide()
        // ...
    }

    private fun handleStartSearch() = intent {       // orbit `intent` block
        reduce { state.copy(isLoading = true) }      // atomic state update
        startRideUseCase()
            .onSuccess { ride ->
                val next = rideStateMachine.transition(state.rideState, RideEvent.SearchStarted)
                reduce { state.copy(rideState = next, isLoading = false) }
                postSideEffect(RideSideEffect.StartLocationTrackingService)
            }
            .onFailure { error ->
                reduce { state.copy(isLoading = false, error = error.toUiError()) }
            }
    }
}
```

**Composable side:**

```kotlin
@Composable
fun RideScreen(viewModel: RideViewModel = hiltViewModel()) {
    val state by viewModel.collectAsState()          // orbit-compose extension
    viewModel.collectSideEffect { effect ->          // delivered exactly once
        when (effect) {
            is RideSideEffect.StartLocationTrackingService -> context.startFgs()
            is RideSideEffect.NavigateToComplete -> onNavigateComplete()
        }
    }
    RideScreenContent(state = state, onIntent = viewModel::onIntent)
}
```

**Test API:**

```kotlin
@Test
fun `StartSearch transitions to Searching and posts FGS side effect`() = runTest {
    val viewModel = RideViewModel(
        rideStateMachine = RideStateMachine(),
        startRideUseCase = FakeStartRideUseCase(Result.success(testRide())),
    )

    viewModel.test(this) {
        expectInitialState()
        viewModel.onIntent(RideIntent.StartSearch)
        expectState { copy(isLoading = true) }
        expectState { copy(rideState = RideState.Searching, isLoading = false) }
        expectSideEffect(RideSideEffect.StartLocationTrackingService)
    }
}
```

**Pros:**
- `intent {}` block handles coroutine lifecycle automatically — no `viewModelScope.launch` noise
- `reduce {}` is thread-safe and atomic — no race conditions between concurrent intents
- `postSideEffect {}` uses a `Channel` internally — guarantees exactly-once delivery, not replayed
- `collectSideEffect {}` in Compose respects lifecycle — effects are not delivered when the composable is stopped
- `test {}` DSL is readable and does not require mocking Orbit internals
- Actively maintained, Kotlin 2.0 compatible, Compose multiplatform support in roadmap
- Small binary size: ~80KB AAR

**Cons:**
- One more library dependency (though small)
- `ContainerHost` interface is required on ViewModel — minor coupling
- `orbit-compose` artifact needed separately (two artifacts, not one)
- Less explicit than manual: the `intent {}` block's coroutine scheduling is opaque  
  (runs on a single-threaded sequential `CoroutineDispatcher` by default)

---

### Option B — MVI-Kotlin

**What it is:**  
A more explicit MVI framework by Arkadii Ivanov (JetBrains). Defines `Store`, `Intent`,  
`State`, `Label` (equivalent to side effects) as first-class types with interfaces.  
GitHub: https://github.com/arkivanov/MVIKotlin

**Core API sketch:**

```kotlin
// Requires separate Store, Executor, Reducer, Bootstrapper components
// Each is a class implementing a specific interface

class RideStoreFactory @Inject constructor() : StoreFactory {
    fun create(): RideStore =
        object : RideStore, Store<RideIntent, RideState, RideLabel>
        by storeFactory.create(
            name = "RideStore",
            initialState = RideState.Idle,
            bootstrapper = SimpleBootstrapper(Action.ObserveRideEvents),
            executorFactory = ::RideExecutor,
            reducer = RideReducer,
        ) {}
}

// Executor handles Intents and Actions, dispatches Msgs to Reducer
// Reducer applies Msgs to State immutably
// Labels are one-shot outputs (equivalent to SideEffects)
```

**Pros:**
- Highly explicit — every component has a single responsibility
- Strong separation: Executor never mutates state directly
- Good for very large teams where enforcement of single-responsibility matters
- Kotlin Multiplatform support (KMP) — runs on iOS, Desktop, Web
- Excellent for codebases that will be shared across platforms

**Cons:**
- Significantly more boilerplate: Store, Executor, Reducer, Bootstrapper, Msg sealed class —  
  a minimum of 5 separate constructs per screen vs Orbit's 1 ViewModel
- Steeper learning curve — new developers spend days understanding the architecture  
  before writing productive code
- `Store` is not a `ViewModel` — requires a separate binding layer to Android ViewModel lifecycle
- Android + Compose integration requires additional wiring not provided by the library
- Overkill for a single-platform Android app with no KMP plans

**Verdict:** MVI-Kotlin's structure is valuable at scale or in KMP projects.  
For a single-platform Android app with a team of 3-5, the boilerplate cost is not justified.

---

### Option C — Manual implementation (ViewModel + StateFlow + Channel)

**What it is:**  
No MVI library. A `ViewModel` exposes a `StateFlow<UiState>` and a `Channel<SideEffect>`.  
Intents are regular functions. State updates use `update {}` on `MutableStateFlow`.

```kotlin
@HiltViewModel
class RideViewModel @Inject constructor(
    private val rideStateMachine: RideStateMachine,
    private val startRideUseCase: StartRideUseCase,
) : ViewModel() {

    private val _state = MutableStateFlow(RideUiState())
    val state: StateFlow<RideUiState> = _state.asStateFlow()

    private val _sideEffects = Channel<RideSideEffect>(Channel.BUFFERED)
    val sideEffects: Flow<RideSideEffect> = _sideEffects.receiveAsFlow()

    fun onStartSearch() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            startRideUseCase()
                .onSuccess { ride ->
                    val next = rideStateMachine.transition(_state.value.rideState, RideEvent.SearchStarted)
                    _state.update { it.copy(rideState = next, isLoading = false) }
                    _sideEffects.send(RideSideEffect.StartLocationTrackingService)
                }
                .onFailure { error ->
                    _state.update { it.copy(isLoading = false, error = error.toUiError()) }
                }
        }
    }
}
```

**Pros:**
- Zero library dependencies — only Kotlin and AndroidX
- Every team member already knows `StateFlow` and `Channel` — no learning curve
- Full control over coroutine scheduling, dispatcher, and cancellation
- No library abstractions to debug when something goes wrong
- No risk of library abandonment

**Cons:**
- State updates are NOT atomic by default.  
  `_state.update { }` is atomic, but if two concurrent coroutines call `_state.update`  
  simultaneously (e.g. WebSocket event + user tap), they can interleave incorrectly  
  because `update {}` is a CAS loop, not a sequential queue.  
  Orbit's `intent {}` block runs on a single-threaded sequential dispatcher — this is safe by design.
- Side effect delivery has subtle bugs.  
  `Channel.BUFFERED` can drop events if the buffer is full. `Channel.UNLIMITED` leaks.  
  `Channel.RENDEZVOUS` blocks the sender. Orbit handles this correctly internally.  
  Getting it right manually requires understanding these tradeoffs and writing tests for them.
- Testing requires mocking or turbine directly — more verbose than Orbit's `test {}` DSL
- Every new ViewModel author recreates the same pattern — inconsistency grows over time
- The "manual MVI" pattern in teams of 3+ people tends to drift: one developer uses  
  `LaunchedEffect` for navigation, another uses `Channel`, a third uses `SharedFlow` —  
  all doing slightly different things that interact badly

**Verdict:** The manual approach is correct in principle but fragile in practice.  
The concurrency correctness problems are subtle, found late, and fixed inconsistently.  
The value of a library here is not abstraction — it is **consistency and correctness guarantees**.

---

## Options Comparison

| Criterion | Orbit MVI | MVI-Kotlin | Manual |
|---|---|---|---|
| Boilerplate per screen | Low | High | Medium |
| Concurrency correctness | ✅ Sequential dispatcher | ✅ Explicit Executor | ⚠️ Manual CAS, easy to get wrong |
| Side effect delivery | ✅ Channel, exactly-once | ✅ Label channel | ⚠️ Manual, subtle bugs |
| Testing API | ✅ `test {}` DSL | ✅ Good | ⚠️ Manual turbine setup |
| Compose integration | ✅ `orbit-compose` | ⚠️ Manual binding | ✅ Native |
| Learning curve | Low | High | Zero (for Kotlin devs) |
| Kotlin Multiplatform | Roadmap | ✅ Full KMP | ✅ Full KMP |
| Library risk | Low (active) | Low (active) | None |
| Team consistency | ✅ Enforced by API | ✅ Enforced by API | ⚠️ Convention only |
| Binary size impact | ~80KB | ~120KB | 0KB |

---

## Consequences

### Positive

- Every ViewModel follows the same structure — new developers read one example and understand all others.
- `reduce {}` atomicity and `intent {}` sequential execution mean we cannot accidentally  
  introduce state corruption from concurrent events — a real risk given WebSocket + user + FGS  
  all updating the same ViewModel simultaneously.
- `test {}` DSL makes ViewModel tests read as specifications:  
  "given intent X, expect state Y, then side effect Z" — useful as documentation.
- Orbit is a thin layer: if we ever need to move to manual implementation,  
  every `intent {}` block becomes a `viewModelScope.launch {}` and every `reduce {}` becomes  
  `_state.update {}`. The migration cost is low.

### Negative

- New developers must learn Orbit's API before they can write their first ViewModel.  
  Estimated ramp-up: 1-2 hours reading Orbit docs + reviewing one existing ViewModel.
- `orbit-compose` is a second artifact — two Gradle entries per module that uses it.
- Orbit's internal `CoroutineDispatcher` is sequential, which means that if a long-running  
  operation inside an `intent {}` block blocks the dispatcher, other intents queue up.  
  **Rule:** never do blocking IO inside `intent {}` directly — always call a `suspend fun`  
  that runs on the IO dispatcher.

### Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Orbit becomes unmaintained | Low | The library is thin; migration to manual MVI is straightforward (see above). We do not use deep Orbit internals. |
| Developer misuses `intent {}` for blocking IO | Medium | Enforced via code review checklist + `MVI.md` documents the anti-pattern explicitly. |
| Orbit's sequential dispatcher causes subtle ordering issues | Low | Documented in `MVI.md`. All intents are unit-tested. Sequential dispatch is a feature, not a bug, for this use case. |
| New developer writes MVI without using Orbit | Low | All feature module templates include Orbit dependency. `CONTRIBUTING.md` mandates Orbit for ViewModels. |

---

## Validation

We will know this decision was correct at the 3-month mark if:

- [ ] ViewModel unit tests are written without mocking Orbit internals
- [ ] Zero reported race conditions in ride state during testing
- [ ] New developers can write a ViewModel independently after 1 day of onboarding
- [ ] No PR comments requesting manual state management workarounds

If we find that Orbit's sequential dispatcher causes performance issues  
(e.g. intents queuing for >100ms), we will revisit with a targeted benchmark.

---

## Links

- [Orbit MVI GitHub](https://github.com/orbit-mvi/orbit-mvi)
- [Orbit MVI documentation](https://orbit-mvi.org)
- [MVI-Kotlin GitHub](https://github.com/arkivanov/MVIKotlin)
- [`docs/arch/MVI.md`](../arch/MVI.md) — how MVI is implemented in this project
- [`docs/arch/STATE_MACHINE.md`](../arch/STATE_MACHINE.md) — ride lifecycle state machine
- [ADR-002](ADR-002-di-framework.md) — Dependency injection framework selection
- Related issue: #12 — "Choose state management approach for ride screen"

---
---

# ADR-000 Usage Notes

## Status lifecycle

```
Draft → Proposed → Accepted
                 → Rejected (document why — rejected decisions are valuable history)

Accepted → Deprecated (no longer applies — e.g. library removed)
         → Superseded by ADR-NNN (replaced by a new decision)
```

Never delete an ADR. A superseded decision is still useful context.  
Add "Superseded by ADR-NNN" to the status line and leave the file in place.

## When to write an ADR

Write an ADR when the decision:

- Affects more than one module or more than one developer
- Would be surprising to a new team member without context
- Has been discussed and has rejected alternatives worth documenting
- Could reasonably be revisited in 6-12 months

Do NOT write an ADR for:
- Naming conventions (go in `CONTRIBUTING.md`)
- Implementation details that don't affect module boundaries or patterns
- Decisions with only one realistic option

## Suggested first ADRs for this project

| ADR | Decision to document |
|---|---|
| ADR-001 | MVI library (this document) |
| ADR-002 | Dependency injection: Hilt vs Koin |
| ADR-003 | Local database: Room vs SQLDelight |
| ADR-004 | Serialization: kotlinx.serialization vs Gson vs Moshi |
| ADR-005 | Navigation: Compose Navigation vs Voyager vs Decompose |
| ADR-006 | Image loading: Coil vs Glide vs Picasso |
| ADR-007 | Map SDK: Google Maps Compose vs Mapbox vs OSM |
| ADR-008 | Background location sync: WorkManager vs manual retry |
| ADR-009 | WebSocket library: OkHttp vs Ktor vs Scarlet |
| ADR-010 | Testing: JUnit 5 vs JUnit 4 |

---

*ADR template and ADR-001 — Micromobility & Logistics Tracker*  
*Owner: Android Tech Lead*
