# Ride & Delivery State Machine

> **Owner:** Android Tech Lead  
> **Audience:** Android team, backend team, QA  
> **Related:** `ARCHITECTURE.md` · `MVI.md` · `RIDE_LIFECYCLE.md` · `docs/adr/ADR-001`

---

## Table of Contents

- [Purpose](#purpose)
- [State Hierarchy](#state-hierarchy)
- [Event Hierarchy](#event-hierarchy)
- [Transition Table](#transition-table)
- [State Diagram](#state-diagram)
- [Kotlin Implementation](#kotlin-implementation)
  - [RideState sealed interface](#ridestate-sealed-interface)
  - [RideEvent sealed interface](#rideevent-sealed-interface)
  - [RideStateMachine](#ridestate-machine)
  - [IllegalStateTransitionException](#illegalstatetransitionexception)
- [Edge Cases & Guard Conditions](#edge-cases--guard-conditions)
- [State Restoration on Process Death](#state-restoration-on-process-death)
- [Integration with MVI](#integration-with-mvi)
- [Mapping: Server Events → RideEvent](#mapping-server-events--rideevent)
- [Testing](#testing)
  - [Happy path tests](#happy-path-tests)
  - [Illegal transition tests](#illegal-transition-tests)
  - [Edge case tests](#edge-case-tests)
- [Checklist for Adding a New State](#checklist-for-adding-a-new-state)

---

## Purpose

The **RideStateMachine** is the single source of truth for the lifecycle of a ride or delivery.

It answers one question for every situation: **is this transition valid, and what is the next state?**

Without a formal state machine, state logic scatters across ViewModel, Repository, and UI composables — each making local decisions. The result is subtle bugs: a `Completed` ride that can somehow be cancelled; an `InProgress` ride that resets to `Idle` on screen rotation; a retry that takes the user back to the wrong state.

**What this document defines:**

- Every valid state the app can be in
- Every event that can trigger a transition
- Every valid transition (from → event → to)
- Every illegal transition and what happens when one is attempted
- Edge cases: timeouts, driver cancellation, process death, network loss mid-ride

**Scope:**

This state machine covers the **ride/delivery lifecycle** from the user's perspective.  
It does not cover: authentication state, onboarding, payment flow, or settings.  
Those are separate, simpler flows handled by their own ViewModels.

**Assumption:** the backend is the authority on ride state. The client state machine  
mirrors server state; it does not invent state independently.  
When a WebSocket event arrives, the client transitions. If the client and server disagree,  
the server wins — the client re-syncs via REST on reconnect.

---

## State Hierarchy

```
RideState
├── Idle                          No active ride. Initial state, and state after completion/cancellation.
├── Searching                     User requested a ride. Waiting for driver match from server.
├── MatchFound(rideId, driver, etaSeconds)
│                                 Driver assigned. Brief intermediate state before Waiting UI appears.
│                                 Exists to separate "assignment event received" from "user sees driver info".
├── Waiting(rideId, driver, etaSeconds, updatedAt)
│                                 Driver is en route to pickup. ETA updates arrive via WebSocket.
├── InProgress(rideId, driver, startedAt, routePolyline?)
│                                 Ride/delivery is active. Location tracking service is running.
├── Completed(rideId, summary)    Ride finished. Terminal state — no transitions out except back to Idle
│                                 after user dismisses the summary screen.
├── Cancelled(rideId?, reason, cancellerId)
│                                 Ride cancelled — by user, driver, or system. Terminal state.
└── Error(type, previousState)    Something went wrong. Non-terminal — can retry back to previousState
                                  or reset to Idle depending on error type.
```

**Terminal states:** `Completed`, `Cancelled`  
From these, the only valid next event is `UserDismissedSummary` → `Idle`.

**Quasi-terminal:** `Error`  
Can retry (`RetryRequested` → depends on `previousState`) or reset (`ResetRequested` → `Idle`).

---

## Event Hierarchy

Events come from three sources, marked below.

```
RideEvent
│
├── [USER]    SearchStarted               User tapped "Request ride"
├── [USER]    SearchCancelled             User tapped "Cancel" during Searching
├── [USER]    RideCancelled               User cancelled during Waiting or InProgress
├── [USER]    RetryRequested              User tapped "Retry" on error screen
├── [USER]    ResetRequested              User tapped "Start over" on error screen
├── [USER]    SummaryDismissed            User dismissed Completed/Cancelled summary screen
│
├── [SERVER]  DriverMatched(rideId, driver, etaSeconds)
│                                         WebSocket: server assigned a driver
├── [SERVER]  DriverEtaUpdated(seconds)   WebSocket: ETA refreshed (Waiting state)
├── [SERVER]  DriverArrived               WebSocket: driver is at pickup point
├── [SERVER]  RideStarted(startedAt)      WebSocket: driver started the trip
├── [SERVER]  RideCompleted(summary)      WebSocket: server confirmed completion
├── [SERVER]  DriverCancelled(reason)     WebSocket: driver cancelled after match
├── [SERVER]  SystemCancelled(reason)     WebSocket: system cancelled (no driver, timeout, etc.)
├── [SERVER]  RideSynced(ride)            REST: full ride state from server on reconnect/restore
│
└── [SYSTEM]  SearchTimedOut              Internal timeout — no driver found within configured window
               NetworkLost                ConnectivityManager — connection dropped
               NetworkRestored            ConnectivityManager — connection regained
               ErrorOccurred(type)        Data layer surfaced an unrecoverable error
```

---

## Transition Table

Legend: ✅ valid · ❌ illegal (throws) · — not applicable

| Current State | Event | Next State | Notes |
|---|---|---|---|
| `Idle` | `SearchStarted` | `Searching` | FGS not yet started |
| `Idle` | any other | ❌ | Nothing to act on |
| `Searching` | `DriverMatched` | `MatchFound` | Brief intermediate |
| `Searching` | `SearchCancelled` | `Idle` | Clean cancel, no penalty |
| `Searching` | `SearchTimedOut` | `Error(NoDriverAvailable, Searching)` | Retry → `Idle` |
| `Searching` | `NetworkLost` | `Error(NetworkLost, Searching)` | Retry when restored |
| `Searching` | `SystemCancelled` | `Cancelled(reason=SYSTEM)` | |
| `Searching` | `ErrorOccurred` | `Error(type, Searching)` | |
| `MatchFound` | `RideCancelled` [user] | `Cancelled(reason=USER)` | Cancellation window |
| `MatchFound` | `DriverCancelled` | `Cancelled(reason=DRIVER)` | Re-search automatically |
| `MatchFound` | *(auto after delay)* | `Waiting` | ViewModel transitions after UI ack |
| `Waiting` | `DriverEtaUpdated` | `Waiting` (updated eta) | In-place update, same state |
| `Waiting` | `DriverArrived` | `Waiting` (eta=0) | ETA → 0, UI changes label |
| `Waiting` | `RideStarted` | `InProgress` | **FGS starts here** |
| `Waiting` | `RideCancelled` [user] | `Cancelled(reason=USER)` | May incur penalty |
| `Waiting` | `DriverCancelled` | `Cancelled(reason=DRIVER)` | Show re-search prompt |
| `Waiting` | `NetworkLost` | `Error(NetworkLost, Waiting)` | Persist Waiting in Room |
| `Waiting` | `ErrorOccurred` | `Error(type, Waiting)` | |
| `InProgress` | `RideCompleted` | `Completed` | **FGS stops here** |
| `InProgress` | `RideCancelled` [user] | `Cancelled(reason=USER)` | **FGS stops here** |
| `InProgress` | `SystemCancelled` | `Cancelled(reason=SYSTEM)` | **FGS stops here** |
| `InProgress` | `NetworkLost` | `Error(NetworkLost, InProgress)` | FGS keeps running |
| `InProgress` | `DriverEtaUpdated` | ❌ | ETA is not meaningful InProgress |
| `InProgress` | `SearchStarted` | ❌ | Cannot start new ride mid-ride |
| `Completed` | `SummaryDismissed` | `Idle` | Clean slate |
| `Completed` | `SearchStarted` | ❌ | Must dismiss summary first |
| `Cancelled` | `SummaryDismissed` | `Idle` | Clean slate |
| `Cancelled` | `SearchStarted` | ❌ | Must dismiss summary first |
| `Error` | `RetryRequested` | depends on `previousState`* | See retry logic below |
| `Error` | `ResetRequested` | `Idle` | Full reset |
| `Error` | `NetworkRestored` | retry if `type == NetworkLost` | Automatic retry |
| `Error(_, InProgress)` | `RideCompleted` | `Completed` | Server event arrives late |

*Retry targets by previous state:

| `previousState` in Error | Retry target |
|---|---|
| `Idle` / `Searching` | `Idle` (restart search) |
| `Waiting` | `Waiting` (re-sync from server via `RideSynced`) |
| `InProgress` | `InProgress` (re-sync, FGS stays running) |

---

## State Diagram

```
                              SearchStarted
         ┌─────────────────────────────────────────────┐
         │                                             │
    ┌────▼────┐  DriverMatched   ┌───────────┐         │
    │  Idle   │─────────────────►│ MatchFound│         │
    └────┬────┘                  └─────┬─────┘         │
         │                            │ (auto)         │
         │  SearchStarted             ▼                │
         │                     ┌─────────────┐         │
         └────────────────────►│  Searching  │         │
                               └──────┬──────┘         │
                                      │ DriverMatched   │
                                      ▼                 │
                               ┌─────────────┐         │
                               │  MatchFound │         │
                               └──────┬──────┘         │
                                      │ (auto → UI ack) │
                                      ▼                 │
                               ┌─────────────┐         │
                               │   Waiting   │◄────────┘
                               └──────┬──────┘  (EtaUpdated stays here)
                                      │ RideStarted
                                      ▼
                               ┌─────────────┐
                               │  InProgress │ ← FGS running
                               └──────┬──────┘
                                      │ RideCompleted
                                      ▼
                               ┌─────────────┐
                               │  Completed  │──► (SummaryDismissed) ──► Idle
                               └─────────────┘

    Cancellation (from Searching / Waiting / InProgress):
                               ┌─────────────┐
                               │  Cancelled  │──► (SummaryDismissed) ──► Idle
                               └─────────────┘

    Error (from any non-terminal state):
                               ┌─────────────┐
                               │    Error    │──► (Retry) ──► previousState or Idle
                               └─────────────┘    (Reset) ──► Idle
```

---

## Kotlin Implementation

### RideState sealed interface

```kotlin
// :core-domain/src/main/kotlin/.../statemachine/RideState.kt

import java.time.Instant

sealed interface RideState {

    /** No active ride. App is ready to start a new search. */
    data object Idle : RideState

    /** Search request sent to server. Awaiting driver assignment. */
    data object Searching : RideState

    /**
     * Driver assigned by server. Brief intermediate state.
     * ViewModel transitions to [Waiting] after the UI has acknowledged
     * the match (e.g. after showing a "Driver found!" animation).
     */
    data class MatchFound(
        val rideId: String,
        val driver: Driver,
        val etaSeconds: Int,
    ) : RideState

    /**
     * Driver is en route to the pickup point.
     * [etaSeconds] is updated in-place via [RideEvent.DriverEtaUpdated].
     * [updatedAt] tracks when the last ETA update arrived — used to detect
     * stale ETA when the WebSocket reconnects.
     */
    data class Waiting(
        val rideId: String,
        val driver: Driver,
        val etaSeconds: Int,
        val updatedAt: Instant = Instant.now(),
    ) : RideState

    /**
     * Active ride or delivery in progress.
     * LocationTrackingService (FGS) is running while in this state.
     * [routePolyline] may be null initially and populated once the server
     * sends the planned route.
     */
    data class InProgress(
        val rideId: String,
        val driver: Driver,
        val startedAt: Instant,
        val routePolyline: List<LatLng>? = null,
    ) : RideState

    /**
     * Ride finished successfully. Terminal — transitions only to [Idle]
     * after the user dismisses the summary screen.
     */
    data class Completed(
        val rideId: String,
        val summary: RideSummary,
    ) : RideState

    /**
     * Ride cancelled by user, driver, or system.
     * [rideId] may be null if cancellation happened before a driver was assigned.
     */
    data class Cancelled(
        val rideId: String?,
        val reason: CancellationReason,
        val cancellerId: CancellerType,
    ) : RideState

    /**
     * An error occurred. Non-terminal.
     * [previousState] enables context-aware retry:
     * retrying from a [Waiting] error re-syncs with the server rather than
     * resetting all the way to [Idle].
     */
    data class Error(
        val type: RideErrorType,
        val previousState: RideState,
    ) : RideState
}

// ── Supporting types ──────────────────────────────────────────────

enum class CancellationReason {
    USER_REQUESTED,
    DRIVER_CANCELLED,
    NO_DRIVER_AVAILABLE,
    SYSTEM_TIMEOUT,
    PAYMENT_FAILED,
    UNKNOWN,
}

enum class CancellerType { USER, DRIVER, SYSTEM }

sealed interface RideErrorType {
    data object NetworkLost : RideErrorType
    data object NoDriverAvailable : RideErrorType
    data object ServerError : RideErrorType
    data class Unknown(val cause: String) : RideErrorType
}

data class RideSummary(
    val durationSeconds: Int,
    val distanceMeters: Int,
    val fareAmount: Money,
    val route: List<LatLng>,
)
```

---

### RideEvent sealed interface

```kotlin
// :core-domain/src/main/kotlin/.../statemachine/RideEvent.kt

import java.time.Instant

sealed interface RideEvent {

    // ── User-initiated ────────────────────────────────────────────
    data object SearchStarted : RideEvent
    data object SearchCancelled : RideEvent
    data object RideCancelledByUser : RideEvent
    data object RetryRequested : RideEvent
    data object ResetRequested : RideEvent
    data object SummaryDismissed : RideEvent
    data object MatchAcknowledgedByUi : RideEvent   // MatchFound → Waiting trigger

    // ── Server-originated (WebSocket / REST) ──────────────────────
    data class DriverMatched(
        val rideId: String,
        val driver: Driver,
        val etaSeconds: Int,
    ) : RideEvent

    data class DriverEtaUpdated(val etaSeconds: Int) : RideEvent
    data object DriverArrived : RideEvent

    data class RideStarted(val startedAt: Instant) : RideEvent
    data class RideCompleted(val summary: RideSummary) : RideEvent

    data class DriverCancelled(val reason: CancellationReason) : RideEvent
    data class SystemCancelled(val reason: CancellationReason) : RideEvent

    /** Full ride snapshot from REST — used after reconnect or process restore. */
    data class RideSynced(val ride: Ride) : RideEvent

    // ── System / internal ─────────────────────────────────────────
    data object SearchTimedOut : RideEvent
    data object NetworkLost : RideEvent
    data object NetworkRestored : RideEvent
    data class ErrorOccurred(val type: RideErrorType) : RideEvent
}
```

---

### RideStateMachine

```kotlin
// :core-domain/src/main/kotlin/.../statemachine/RideStateMachine.kt

import java.time.Instant
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class RideStateMachine @Inject constructor() {

    /**
     * Computes the next [RideState] given the current state and an incoming event.
     *
     * @throws IllegalStateTransitionException if the transition is not defined.
     *         This is a programming error — all reachable transitions must be
     *         handled. Crash in debug; log + no-op in release (see error handling doc).
     */
    fun transition(current: RideState, event: RideEvent): RideState = when {

        // ── From Idle ─────────────────────────────────────────────────────────
        current is RideState.Idle && event is RideEvent.SearchStarted ->
            RideState.Searching

        // ── From Searching ────────────────────────────────────────────────────
        current is RideState.Searching && event is RideEvent.DriverMatched ->
            RideState.MatchFound(
                rideId = event.rideId,
                driver = event.driver,
                etaSeconds = event.etaSeconds,
            )

        current is RideState.Searching && event is RideEvent.SearchCancelled ->
            RideState.Idle

        current is RideState.Searching && event is RideEvent.SearchTimedOut ->
            RideState.Error(
                type = RideErrorType.NoDriverAvailable,
                previousState = current,
            )

        current is RideState.Searching && event is RideEvent.SystemCancelled ->
            RideState.Cancelled(
                rideId = null,
                reason = event.reason,
                cancellerId = CancellerType.SYSTEM,
            )

        // ── From MatchFound ───────────────────────────────────────────────────
        current is RideState.MatchFound && event is RideEvent.MatchAcknowledgedByUi ->
            RideState.Waiting(
                rideId = current.rideId,
                driver = current.driver,
                etaSeconds = current.etaSeconds,
            )

        current is RideState.MatchFound && event is RideEvent.RideCancelledByUser ->
            RideState.Cancelled(
                rideId = current.rideId,
                reason = CancellationReason.USER_REQUESTED,
                cancellerId = CancellerType.USER,
            )

        current is RideState.MatchFound && event is RideEvent.DriverCancelled ->
            RideState.Cancelled(
                rideId = current.rideId,
                reason = event.reason,
                cancellerId = CancellerType.DRIVER,
            )

        // ── From Waiting ──────────────────────────────────────────────────────
        current is RideState.Waiting && event is RideEvent.DriverEtaUpdated ->
            current.copy(
                etaSeconds = event.etaSeconds,
                updatedAt = Instant.now(),
            )

        current is RideState.Waiting && event is RideEvent.DriverArrived ->
            current.copy(etaSeconds = 0, updatedAt = Instant.now())

        current is RideState.Waiting && event is RideEvent.RideStarted ->
            RideState.InProgress(
                rideId = current.rideId,
                driver = current.driver,
                startedAt = event.startedAt,
            )

        current is RideState.Waiting && event is RideEvent.RideCancelledByUser ->
            RideState.Cancelled(
                rideId = current.rideId,
                reason = CancellationReason.USER_REQUESTED,
                cancellerId = CancellerType.USER,
            )

        current is RideState.Waiting && event is RideEvent.DriverCancelled ->
            RideState.Cancelled(
                rideId = current.rideId,
                reason = event.reason,
                cancellerId = CancellerType.DRIVER,
            )

        // ── From InProgress ───────────────────────────────────────────────────
        current is RideState.InProgress && event is RideEvent.RideCompleted ->
            RideState.Completed(
                rideId = current.rideId,
                summary = event.summary,
            )

        current is RideState.InProgress && event is RideEvent.RideCancelledByUser ->
            RideState.Cancelled(
                rideId = current.rideId,
                reason = CancellationReason.USER_REQUESTED,
                cancellerId = CancellerType.USER,
            )

        current is RideState.InProgress && event is RideEvent.SystemCancelled ->
            RideState.Cancelled(
                rideId = current.rideId,
                reason = event.reason,
                cancellerId = CancellerType.SYSTEM,
            )

        // ── From Completed / Cancelled ────────────────────────────────────────
        (current is RideState.Completed || current is RideState.Cancelled) &&
            event is RideEvent.SummaryDismissed ->
            RideState.Idle

        // ── From Error ────────────────────────────────────────────────────────
        current is RideState.Error && event is RideEvent.ResetRequested ->
            RideState.Idle

        current is RideState.Error && event is RideEvent.RetryRequested ->
            retryTarget(current.previousState)

        // NetworkRestored only auto-retries network errors
        current is RideState.Error &&
            current.type is RideErrorType.NetworkLost &&
            event is RideEvent.NetworkRestored ->
            retryTarget(current.previousState)

        // Late server event arriving while in Error(_, InProgress) — honour it
        current is RideState.Error &&
            current.previousState is RideState.InProgress &&
            event is RideEvent.RideCompleted ->
            RideState.Completed(
                rideId = current.previousState.rideId,
                summary = event.summary,
            )

        // ── Cross-cutting: network and generic errors ──────────────────────────
        event is RideEvent.NetworkLost && current !is RideState.Idle ->
            RideState.Error(
                type = RideErrorType.NetworkLost,
                previousState = current,
            )

        event is RideEvent.ErrorOccurred ->
            RideState.Error(
                type = event.type,
                previousState = current,
            )

        // ── RideSynced: reconcile client state with server ────────────────────
        event is RideEvent.RideSynced ->
            reconcileWithServer(current, event.ride)

        // ── Illegal transition ────────────────────────────────────────────────
        else -> throw IllegalStateTransitionException(
            currentState = current,
            event = event,
        )
    }

    /**
     * Where to go after a retry, depending on the state we were in when the error occurred.
     * Searching/Idle → restart from Idle.
     * Waiting/InProgress → re-sync from server (caller will dispatch RideSynced next).
     */
    private fun retryTarget(previousState: RideState): RideState = when (previousState) {
        is RideState.Idle,
        is RideState.Searching,
        is RideState.MatchFound -> RideState.Idle

        is RideState.Waiting -> previousState   // hold position; ViewModel will re-sync
        is RideState.InProgress -> previousState // FGS still running; ViewModel will re-sync

        // Should not happen — Error.previousState is never Completed/Cancelled/Error
        else -> RideState.Idle
    }

    /**
     * Reconciles client state with the authoritative server snapshot.
     * Called when a full [Ride] arrives via REST after reconnect or process restore.
     * Server always wins.
     */
    private fun reconcileWithServer(current: RideState, ride: Ride): RideState {
        return when (ride.status) {
            RideStatus.SEARCHING -> RideState.Searching
            RideStatus.DRIVER_ASSIGNED -> RideState.Waiting(
                rideId = ride.id,
                driver = requireNotNull(ride.driver),
                etaSeconds = ride.etaSeconds ?: 0,
            )
            RideStatus.IN_PROGRESS -> RideState.InProgress(
                rideId = ride.id,
                driver = requireNotNull(ride.driver),
                startedAt = requireNotNull(ride.startedAt),
            )
            RideStatus.COMPLETED -> RideState.Completed(
                rideId = ride.id,
                summary = requireNotNull(ride.summary),
            )
            RideStatus.CANCELLED -> RideState.Cancelled(
                rideId = ride.id,
                reason = ride.cancellationReason ?: CancellationReason.UNKNOWN,
                cancellerId = ride.cancelledBy ?: CancellerType.SYSTEM,
            )
        }
    }
}
```

---

### IllegalStateTransitionException

```kotlin
// :core-domain/src/main/kotlin/.../statemachine/IllegalStateTransitionException.kt

class IllegalStateTransitionException(
    currentState: RideState,
    event: RideEvent,
) : IllegalStateException(
    "Illegal state transition: ${currentState::class.simpleName} + ${event::class.simpleName} → no rule defined.\n" +
    "Current state: $currentState\n" +
    "Event: $event\n" +
    "Check STATE_MACHINE.md transition table before adding a new rule."
)
```

**Handling policy:**

```kotlin
// In RideViewModel — wraps every state machine call
private fun safeTransition(event: RideEvent) = intent {
    val nextState = try {
        rideStateMachine.transition(state.rideState, event)
    } catch (e: IllegalStateTransitionException) {
        if (BuildConfig.DEBUG) throw e          // crash in debug — surface the bug immediately
        logger.error("Illegal transition", e)   // log in release — never silently swallow
        return@intent                           // stay in current state
    }
    reduce { state.copy(rideState = nextState) }
    handleSideEffects(nextState)
}
```

---

## Edge Cases & Guard Conditions

### Duplicate events

WebSocket connections can reconnect and replay recent events. The state machine  
must be idempotent for certain events.

| Scenario | Handling |
|---|---|
| `DriverEtaUpdated` arrives twice with same value | `copy(etaSeconds)` — no visible change, safe |
| `RideStarted` arrives while already `InProgress` | Current `when` clause hits `else` → `IllegalStateTransition`. ViewModel logs + ignores. |
| `RideCompleted` arrives while already `Completed` | Same — log + ignore |
| `DriverMatched` arrives while in `Waiting` | Illegal transition → log + ignore. Driver was already assigned. |

**Rule:** make every handler check the full `current` type. Never handle just the event.

---

### Search timeout

The server does not always send an explicit "no driver" event — the client implements  
its own timeout as a safety net.

```kotlin
// In RideViewModel
private fun startSearchTimeout() {
    searchTimeoutJob = viewModelScope.launch {
        delay(SEARCH_TIMEOUT_MS)                // e.g. 90_000
        if (container.stateFlow.value.rideState is RideState.Searching) {
            onIntent(RideIntent.SystemEventReceived(RideEvent.SearchTimedOut))
        }
    }
}

private fun cancelSearchTimeout() {
    searchTimeoutJob?.cancel()
    searchTimeoutJob = null
}
```

Start `startSearchTimeout()` on `SearchStarted`. Cancel on `DriverMatched`, `SearchCancelled`,  
`SystemCancelled`, or any transition away from `Searching`.

---

### Driver cancels after match

`DriverCancelled` arriving in `Waiting` or `MatchFound` transitions to `Cancelled(reason=DRIVER)`.

The UI should then offer the user a "Search again" button.  
The button fires `SummaryDismissed` → `Idle`, then immediately `SearchStarted`.

Do **not** auto-restart the search silently — the user must confirm.

---

### Network loss mid-ride

When `NetworkLost` fires during `InProgress`:

1. → `Error(NetworkLost, InProgress)`
2. `LocationTrackingService` continues running — FGS is not stopped
3. Location updates are buffered in `LocationRepository` (in-memory queue, max 50 points)
4. `NetworkRestored` auto-retries → `InProgress` restored
5. ViewModel dispatches `RideSynced` to reconcile any server-side changes

Do **not** stop FGS on `NetworkLost` during an active ride. The ride may still be ongoing.

---

### App killed during InProgress (process death)

Android can kill the process at any time while in the background.

Recovery flow on next launch:

```
App starts
    └─► AppViewModel.init()
           └─► rideRepository.getActiveRide()
                  ├─► null      →  Idle (normal launch)
                  └─► Ride      →  RideEvent.RideSynced(ride)
                                      └─► reconcileWithServer()
                                             └─► InProgress / Waiting / Completed
                                                    └─► navigate to correct screen
                                                    └─► restart FGS if InProgress
```

`LocationTrackingService` also restores itself if it was killed:  
- Use `START_STICKY` so the system restarts the service
- On `onStartCommand`, check `rideRepository.getActiveRide()` — if null, stop self

---

## Integration with MVI

The ViewModel is the only caller of `RideStateMachine.transition()`.  
The state machine does not know about coroutines, Hilt, or Compose.

```
User action (tap "Cancel")
    └─► composable fires: onIntent(RideIntent.CancelRide)
           └─► RideViewModel.onIntent()
                  └─► intent {
                         val next = rideStateMachine.transition(state.rideState, RideEvent.RideCancelledByUser)
                         reduce { state.copy(rideState = next) }
                         handleSideEffects(next)            ← stops FGS, navigates
                      }

WebSocket event ("ride_started")
    └─► RideWebSocketClient emits RideServerEvent.RideStarted
           └─► ObserveRideEventsUseCase maps to RideEvent.RideStarted
                  └─► RideViewModel collects via Flow
                         └─► onIntent(RideIntent.ServerEventReceived(RideEvent.RideStarted(...)))
                                └─► transition() → InProgress
                                       └─► SideEffect: StartLocationTrackingService
```

### Side effects triggered by state transitions

```kotlin
// In RideViewModel
private fun handleSideEffects(nextState: RideState) = intent {
    when (nextState) {
        is RideState.InProgress ->
            postSideEffect(RideSideEffect.StartLocationTrackingService)

        is RideState.Completed -> {
            postSideEffect(RideSideEffect.StopLocationTrackingService)
            postSideEffect(RideSideEffect.NavigateToCompletedScreen)
        }

        is RideState.Cancelled -> {
            postSideEffect(RideSideEffect.StopLocationTrackingService)
            postSideEffect(RideSideEffect.NavigateToCancelledScreen)
        }

        is RideState.Error ->
            if (nextState.type is RideErrorType.NetworkLost) {
                // FGS stays running if previousState was InProgress
                if (nextState.previousState !is RideState.InProgress) {
                    postSideEffect(RideSideEffect.StopLocationTrackingService)
                }
            }

        else -> Unit
    }
}
```

---

## Mapping: Server Events → RideEvent

WebSocket messages arrive as JSON strings. They are parsed in `RideWebSocketClient`  
and mapped to `RideEvent` in `ObserveRideEventsUseCase` — not in the ViewModel.

```kotlin
// :core-data/src/main/.../remote/RideEventMapper.kt

fun RideServerEventDto.toRideEvent(): RideEvent = when (this.type) {
    "driver_matched"   -> RideEvent.DriverMatched(
                              rideId = this.rideId,
                              driver = this.driver!!.toDomain(),
                              etaSeconds = this.etaSeconds!!,
                          )
    "eta_updated"      -> RideEvent.DriverEtaUpdated(etaSeconds = this.etaSeconds!!)
    "driver_arrived"   -> RideEvent.DriverArrived
    "ride_started"     -> RideEvent.RideStarted(startedAt = Instant.parse(this.startedAt!!))
    "ride_completed"   -> RideEvent.RideCompleted(summary = this.summary!!.toDomain())
    "driver_cancelled" -> RideEvent.DriverCancelled(reason = this.reason!!.toCancellationReason())
    "system_cancelled" -> RideEvent.SystemCancelled(reason = this.reason!!.toCancellationReason())
    else               -> RideEvent.ErrorOccurred(type = RideErrorType.Unknown(cause = this.type))
}
```

**Rule:** the ViewModel never parses raw server strings.  
It receives strongly-typed `RideEvent`s only.

---

## Testing

Add these to `:core-domain/src/test/.../statemachine/RideStateMachineTest.kt`

### Happy path tests

```kotlin
class RideStateMachineTest {

    private val machine = RideStateMachine()

    // Helper
    private fun RideState.on(event: RideEvent) = machine.transition(this, event)

    @Test
    fun `Idle + SearchStarted → Searching`() {
        val next = RideState.Idle.on(RideEvent.SearchStarted)
        assertThat(next).isEqualTo(RideState.Searching)
    }

    @Test
    fun `Searching + DriverMatched → MatchFound with correct driver`() {
        val driver = testDriver()
        val next = RideState.Searching.on(RideEvent.DriverMatched("ride-1", driver, 180))
        assertThat(next).isInstanceOf(RideState.MatchFound::class.java)
        assertThat((next as RideState.MatchFound).driver).isEqualTo(driver)
        assertThat(next.etaSeconds).isEqualTo(180)
    }

    @Test
    fun `MatchFound + MatchAcknowledgedByUi → Waiting`() {
        val state = RideState.MatchFound("ride-1", testDriver(), 120)
        val next = state.on(RideEvent.MatchAcknowledgedByUi)
        assertThat(next).isInstanceOf(RideState.Waiting::class.java)
        assertThat((next as RideState.Waiting).rideId).isEqualTo("ride-1")
    }

    @Test
    fun `Waiting + RideStarted → InProgress`() {
        val state = RideState.Waiting("ride-1", testDriver(), 30)
        val now = Instant.now()
        val next = state.on(RideEvent.RideStarted(now))
        assertThat(next).isInstanceOf(RideState.InProgress::class.java)
        assertThat((next as RideState.InProgress).startedAt).isEqualTo(now)
    }

    @Test
    fun `InProgress + RideCompleted → Completed`() {
        val state = RideState.InProgress("ride-1", testDriver(), Instant.now())
        val summary = testRideSummary()
        val next = state.on(RideEvent.RideCompleted(summary))
        assertThat(next).isInstanceOf(RideState.Completed::class.java)
        assertThat((next as RideState.Completed).summary).isEqualTo(summary)
    }

    @Test
    fun `Completed + SummaryDismissed → Idle`() {
        val state = RideState.Completed("ride-1", testRideSummary())
        assertThat(state.on(RideEvent.SummaryDismissed)).isEqualTo(RideState.Idle)
    }

    @Test
    fun `Waiting + DriverEtaUpdated → Waiting with updated eta`() {
        val state = RideState.Waiting("ride-1", testDriver(), etaSeconds = 200)
        val next = state.on(RideEvent.DriverEtaUpdated(90)) as RideState.Waiting
        assertThat(next.etaSeconds).isEqualTo(90)
        assertThat(next.rideId).isEqualTo("ride-1")  // rideId unchanged
    }

    @Test
    fun `full happy path: Idle → Searching → MatchFound → Waiting → InProgress → Completed → Idle`() {
        val driver = testDriver()
        val summary = testRideSummary()
        val now = Instant.now()

        val states = buildList {
            add(RideState.Idle)
            add(last().on(RideEvent.SearchStarted))
            add(last().on(RideEvent.DriverMatched("r1", driver, 120)))
            add(last().on(RideEvent.MatchAcknowledgedByUi))
            add(last().on(RideEvent.RideStarted(now)))
            add(last().on(RideEvent.RideCompleted(summary)))
            add(last().on(RideEvent.SummaryDismissed))
        }

        assertThat(states.map { it::class.simpleName }).containsExactly(
            "Idle", "Searching", "MatchFound", "Waiting", "InProgress", "Completed", "Idle"
        )
    }
}
```

### Illegal transition tests

```kotlin
    @Test
    fun `Idle + any event other than SearchStarted throws`() {
        val illegalEvents = listOf(
            RideEvent.DriverMatched("r", testDriver(), 0),
            RideEvent.RideStarted(Instant.now()),
            RideEvent.RideCompleted(testRideSummary()),
            RideEvent.SummaryDismissed,
        )
        illegalEvents.forEach { event ->
            assertThrows<IllegalStateTransitionException> {
                RideState.Idle.on(event)
            }
        }
    }

    @Test
    fun `InProgress + SearchStarted throws`() {
        val state = RideState.InProgress("r", testDriver(), Instant.now())
        assertThrows<IllegalStateTransitionException> {
            state.on(RideEvent.SearchStarted)
        }
    }

    @Test
    fun `Completed + SearchStarted throws — must dismiss summary first`() {
        val state = RideState.Completed("r", testRideSummary())
        assertThrows<IllegalStateTransitionException> {
            state.on(RideEvent.SearchStarted)
        }
    }

    @Test
    fun `InProgress + DriverEtaUpdated throws — ETA not meaningful in InProgress`() {
        val state = RideState.InProgress("r", testDriver(), Instant.now())
        assertThrows<IllegalStateTransitionException> {
            state.on(RideEvent.DriverEtaUpdated(30))
        }
    }
```

### Edge case tests

```kotlin
    @Test
    fun `Error(NetworkLost, InProgress) + NetworkRestored → InProgress (preserves state)`() {
        val inProgress = RideState.InProgress("r", testDriver(), Instant.now())
        val errorState = RideState.Error(RideErrorType.NetworkLost, inProgress)
        val recovered = errorState.on(RideEvent.NetworkRestored)
        assertThat(recovered).isEqualTo(inProgress)
    }

    @Test
    fun `Error + ResetRequested always → Idle regardless of previousState`() {
        listOf(
            RideState.Searching,
            RideState.Waiting("r", testDriver(), 60),
            RideState.InProgress("r", testDriver(), Instant.now()),
        ).forEach { previous ->
            val errorState = RideState.Error(RideErrorType.ServerError, previous)
            assertThat(errorState.on(RideEvent.ResetRequested)).isEqualTo(RideState.Idle)
        }
    }

    @Test
    fun `Error(NetworkLost, InProgress) + RideCompleted → Completed (late server event)`() {
        val inProgress = RideState.InProgress("r", testDriver(), Instant.now())
        val errorState = RideState.Error(RideErrorType.NetworkLost, inProgress)
        val summary = testRideSummary()
        val next = errorState.on(RideEvent.RideCompleted(summary))
        assertThat(next).isInstanceOf(RideState.Completed::class.java)
    }

    @Test
    fun `NetworkLost from any non-Idle state → Error preserving previousState`() {
        listOf(
            RideState.Searching,
            RideState.Waiting("r", testDriver(), 60),
            RideState.InProgress("r", testDriver(), Instant.now()),
        ).forEach { state ->
            val next = state.on(RideEvent.NetworkLost) as RideState.Error
            assertThat(next.type).isEqualTo(RideErrorType.NetworkLost)
            assertThat(next.previousState).isEqualTo(state)
        }
    }

    @Test
    fun `cancellation valid from Searching, MatchFound, Waiting, InProgress`() {
        val cancellableStates = listOf(
            RideState.Searching,
            RideState.MatchFound("r", testDriver(), 60),
            RideState.Waiting("r", testDriver(), 60),
            RideState.InProgress("r", testDriver(), Instant.now()),
        )
        cancellableStates.forEach { state ->
            val next = state.on(RideEvent.RideCancelledByUser)
            assertThat(next).isInstanceOf(RideState.Cancelled::class.java)
            assertThat((next as RideState.Cancelled).cancellerId).isEqualTo(CancellerType.USER)
        }
    }

// ── Test fixtures ──────────────────────────────────────────────────────────

fun testDriver() = Driver(
    id = "driver-1",
    name = "Alex M.",
    rating = 4.9f,
    vehiclePlate = "B-MT 123",
    location = null,
)

fun testRideSummary() = RideSummary(
    durationSeconds = 720,
    distanceMeters = 3400,
    fareAmount = Money(amount = 12_50, currency = "EUR"),
    route = emptyList(),
)
```

---

## Checklist for Adding a New State

Before adding a new `RideState` subtype, answer every question:

- [ ] What is the business reason for this state? Is it distinct from existing states?
- [ ] What events can lead **into** this state? Add rows to the transition table.
- [ ] What events can lead **out of** this state? Add rows to the transition table.
- [ ] Does this state need to be persisted in Room for process death recovery?
- [ ] Does this state affect `LocationTrackingService` (start / stop)?
- [ ] Is this state terminal? If yes — what is the only exit event?
- [ ] Update `RideState` sealed interface
- [ ] Update `RideEvent` sealed interface (new events if needed)
- [ ] Update `RideStateMachine.transition()` with new `when` branches
- [ ] Update `reconcileWithServer()` if the server can report this status
- [ ] Add happy path test for every new transition
- [ ] Add illegal transition tests for events that must not reach this state
- [ ] Update the transition table in this document
- [ ] Update the state diagram in this document
- [ ] Update `UI_STATES.md` — how does the UI render this state?
- [ ] Notify backend team if this state requires a new WebSocket event type

---

*Last updated: <!-- date -->*  
*Owner: Android Tech Lead*  
*Review: Required before any change to `RideStateMachine.transition()`*
