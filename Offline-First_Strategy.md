# Offline-First Strategy

> **Owner:** Android Tech Lead  
> **Audience:** Android team, QA  
> **Related:** `DATA_LAYER.md` · `STATE_MACHINE.md` · `FOREGROUND_SERVICE.md` · `ERROR_HANDLING.md`  
> **Last reviewed against:** Android 14, Room 2.6, WorkManager 2.9, Kotlin Coroutines 1.8

---

## Table of Contents

- [What Offline-First Means Here](#what-offline-first-means-here)
- [What Is and Is Not Offline-Capable](#what-is-and-is-not-offline-capable)
- [Offline Scenarios by Ride State](#offline-scenarios-by-ride-state)
- [Network Monitor](#network-monitor)
- [Cache Architecture](#cache-architecture)
  - [Cache layers](#cache-layers)
  - [Cache policy per data type](#cache-policy-per-data-type)
  - [Staleness detection](#staleness-detection)
- [Read Strategies](#read-strategies)
  - [Cache-first (offline-safe reads)](#cache-first-offline-safe-reads)
  - [Network-first with cache fallback](#network-first-with-cache-fallback)
  - [Stale-while-revalidate](#stale-while-revalidate)
- [Write Strategies](#write-strategies)
  - [Write-through (immediate sync)](#write-through-immediate-sync)
  - [Write-back with local buffer](#write-back-with-local-buffer)
  - [Optimistic writes](#optimistic-writes)
  - [Deferred writes with WorkManager](#deferred-writes-with-workmanager)
- [Reconnect & Sync Flow](#reconnect--sync-flow)
  - [Reconnect sequence](#reconnect-sequence)
  - [Priority-ordered sync](#priority-ordered-sync)
  - [Reconcile with server](#reconcile-with-server)
- [Conflict Resolution](#conflict-resolution)
  - [Server-wins policy](#server-wins-policy)
  - [When client data is newer](#when-client-data-is-newer)
  - [Handling duplicate server events](#handling-duplicate-server-events)
- [UI Behaviour While Offline](#ui-behaviour-while-offline)
  - [Offline banner](#offline-banner)
  - [Degraded states per screen](#degraded-states-per-screen)
  - [Error vs offline distinction](#error-vs-offline-distinction)
- [Location Tracking Offline](#location-tracking-offline)
  - [FGS during network loss](#fgs-during-network-loss)
  - [Location buffer](#location-buffer)
  - [Track reconstruction on reconnect](#track-reconstruction-on-reconnect)
- [Kotlin Implementation](#kotlin-implementation)
  - [OfflineFirstRideRepository](#offlinefirstriderepository)
  - [SyncManager](#syncmanager)
  - [DeferredWriteQueue](#deferredwritequeue)
  - [OfflineAwareUseCase pattern](#offlineawareusecase-pattern)
- [WorkManager — Deferred Operations](#workmanager--deferred-operations)
- [Testing Offline Behaviour](#testing-offline-behaviour)
  - [Unit tests](#unit-tests)
  - [Integration tests](#integration-tests)
  - [QA scenarios](#qa-scenarios)
- [Decision Log](#decision-log)

---

## What Offline-First Means Here

Offline-first is a **spectrum**, not a binary property. This document defines exactly  
where this app sits on that spectrum and why.

**Our definition:**  
The app never shows a broken, empty, or crashed screen because of a network condition.  
Users always see the last known state of their ride, clearly labelled if it may be stale.  
Critical in-flight data (active ride state, location track) survives process death and reconnect.

**What this is NOT:**  
A full peer-to-peer, always-works-without-internet experience.  
Starting a new ride requires network — you cannot match with a driver without a server.

**Why this matters for couriers and drivers specifically:**  
A courier riding through a tunnel, an underground parking structure, or a rural area  
with poor signal must not lose their active ride state. The app must degrade gracefully  
and recover seamlessly — not crash, not show an empty screen, not require a restart.

---

## What Is and Is Not Offline-Capable

| Feature | Offline-capable | Degraded behaviour |
|---|---|---|
| View active ride state | ✅ | Shows last cached state with staleness banner |
| View driver location | ✅ partial | Shows last known position; pin stops updating |
| ETA display | ✅ partial | Shows last known ETA; count-up timer shows elapsed time |
| View ride history | ✅ | Shows cached list; no new entries until reconnect |
| Location tracking (FGS) | ✅ | Continues writing to buffer; uploads on reconnect |
| Cancel active ride | ⚠️ deferred | Queued locally; executed on reconnect |
| Start new ride search | ❌ | Blocked; clear "No connection" message |
| Submit ride rating | ⚠️ deferred | Queued locally; executed on reconnect |
| View profile / settings | ✅ | Fully local; no network needed |
| Change preferences | ✅ | DataStore; no network needed |

**Assumptions:**  
Authentication tokens are stored locally in `EncryptedSharedPreferences`.  
A token refresh requires network — if the token expires while offline, the user will need to reconnect  
before performing authenticated operations. Token expiry during offline is an edge case outside this scope.

---

## Offline Scenarios by Ride State

These are the canonical scenarios that must be handled correctly.  
Each is covered by QA test cases at the end of this document.

### Scenario 1 — Network lost while Idle / Searching

The least critical case. Search requires network.

```
Searching → NetworkLost event
    └─► RideStateMachine: Searching + NetworkLost → Error(NetworkLost, Searching)
    └─► UI: show "No connection" overlay, disable search button
    └─► Room: no active ride to persist (Searching has no driver yet)
    └─► NetworkMonitor: waits for restoration
    └─► On reconnect: Error + NetworkRestored → Idle (restart from beginning)
```

**Why we don't try to resume the search:** the server-side search session has a short TTL.  
By the time connectivity is restored, the session is expired. Restarting is cleaner.

---

### Scenario 2 — Network lost while Waiting (driver en route)

Critical: the driver is still coming. The user must know this.

```
Waiting(rideId, driver, eta) → NetworkLost event
    └─► RideStateMachine: Waiting + NetworkLost → Error(NetworkLost, Waiting)
    └─► Room: Waiting state persisted (rideId, driver info, last eta)
    └─► FGS: not running yet (FGS starts on InProgress)
    └─► UI: show staleness banner, ETA shows "last known: X min" + elapsed timer
    └─► WebSocket: auto-reconnect loop with backoff
    └─► On reconnect:
           ├─► GET /rides/active → reconcile
           ├─► If still Waiting: restore Waiting with fresh ETA
           ├─► If InProgress: transition to InProgress + start FGS
           └─► If Cancelled: show cancellation screen
```

**Key:** we never transition automatically to `InProgress` during offline.  
The client waits for server confirmation via WebSocket or REST reconcile.

---

### Scenario 3 — Network lost while InProgress (active ride)

Most critical. FGS is running. Track must be preserved.

```
InProgress(rideId, driver, startedAt) → NetworkLost event
    └─► RideStateMachine: InProgress + NetworkLost → Error(NetworkLost, InProgress)
    └─► FGS: CONTINUES running — never stops on network loss
    └─► Location buffer: accumulates GPS points in memory (max 500 points)
    └─► Room: writes to location_history every 30 points (periodic flush)
    └─► UI: "Connection lost — tracking continues" persistent banner
    └─► WebSocket: auto-reconnect loop
    └─► On reconnect:
           ├─► Reconcile ride state with server
           ├─► Upload buffered location points (POST /rides/{id}/track-points)
           ├─► Resume WebSocket events
           └─► Clear buffer after confirmed upload
```

**FGS must never stop due to network loss.**  
If FGS stops during an active ride, the track has permanent gaps.  
Stopping FGS requires an explicit `StopLocationTrackingService` SideEffect,  
which only comes from `Completed` or `Cancelled` state transitions.

---

### Scenario 4 — Process death during InProgress

Android kills the process (OOM, system resource pressure).

```
InProgress → process killed
    └─► FGS: system may restart it via START_STICKY
    └─► Room: has last committed ride entity (status = IN_PROGRESS)
    └─► location_history: has points up to the last periodic flush
    └─► in-memory buffer: LOST

On next app open:
    └─► AppViewModel.init()
           └─► rideRepository.getActiveRide() → returns InProgress ride from Room
                  └─► RideSynced event dispatched
                         └─► RideStateMachine reconciles to InProgress
                                └─► Navigate to active ride screen
                                └─► SideEffect: StartLocationTrackingService (if not running)
                                └─► SyncManager: upload any points in location_history
```

**Acceptable data loss:** GPS points between the last periodic flush and the process kill.  
This is bounded by the flush interval (default: every 30 points ≈ every 60-150 seconds).  
We document this tradeoff — perfect track fidelity is not guaranteed in extreme OOM scenarios.

---

### Scenario 5 — App opened with no network, no active ride

Simplest degraded case.

```
App opens → no network → no active ride in Room
    └─► UI: Idle screen with "No connection" banner
    └─► Search button: disabled with tooltip "Connect to the internet to find a ride"
    └─► Ride history: loaded from Room cache
    └─► Banner dismisses automatically when network is restored
```

---

### Scenario 6 — User force-stops app during InProgress

User goes to Settings → App Info → Force Stop.

```
Process killed by user
    └─► FGS: START_STICKY does NOT restart after force stop
    └─► Room: has last state (may be slightly stale)
    └─► In-memory buffer: LOST (bounded by last flush)

On next app open:
    └─► Same as Scenario 4, but FGS must be restarted manually
    └─► SyncManager attempts to reconcile and upload what we have
    └─► If server marked ride as completed in the interim: show completion screen
```

---

## Network Monitor

The `NetworkMonitor` is the single source of truth for connectivity state.  
It is consumed by the data layer (repositories) and domain layer (via a domain interface).

```kotlin
// :core-domain/src/main/kotlin/.../network/NetworkMonitor.kt

interface NetworkMonitor {
    /** Hot flow. Emits true when validated internet is available. */
    val isOnline: Flow<Boolean>

    /** Current state — suspends until first emission if not yet known. */
    suspend fun isCurrentlyOnline(): Boolean
}
```

```kotlin
// :core-data/src/main/kotlin/.../network/ConnectivityNetworkMonitor.kt

@Singleton
class ConnectivityNetworkMonitor @Inject constructor(
    @ApplicationContext private val context: Context,
    @AppScope private val appScope: CoroutineScope,
) : NetworkMonitor {

    override val isOnline: Flow<Boolean> = callbackFlow {
        val cm = context.getSystemService<ConnectivityManager>()
            ?: run { trySend(false); awaitClose {}; return@callbackFlow }

        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network)  { trySend(true) }
            override fun onLost(network: Network)       { trySend(false) }
            override fun onUnavailable()                { trySend(false) }
            override fun onCapabilitiesChanged(
                network: Network,
                capabilities: NetworkCapabilities,
            ) {
                val validated = capabilities.hasCapability(
                    NetworkCapabilities.NET_CAPABILITY_VALIDATED
                )
                trySend(validated)
            }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .addCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
            .build()

        cm.registerNetworkCallback(request, callback)
        trySend(cm.isCurrentlyConnected())      // emit immediately on subscribe

        awaitClose { cm.unregisterNetworkCallback(callback) }
    }
        .distinctUntilChanged()
        .shareIn(
            scope = appScope,                   // app-scoped — outlives ViewModels
            started = SharingStarted.Eagerly,   // start immediately, always active
            replay = 1,                         // new collectors get current state immediately
        )

    override suspend fun isCurrentlyOnline(): Boolean =
        isOnline.first()

    private fun ConnectivityManager.isCurrentlyConnected(): Boolean =
        activeNetwork
            ?.let { getNetworkCapabilities(it) }
            ?.run {
                hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) &&
                hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
            }
            ?: false
}
```

**`SharingStarted.Eagerly` vs `WhileSubscribed`:**  
NetworkMonitor uses `Eagerly` because connectivity state is critical infrastructure.  
It must be active even when no screen is collecting — the FGS and background sync  
need to know about connectivity changes regardless of UI lifecycle.

**`onCapabilitiesChanged` + `NET_CAPABILITY_VALIDATED`:**  
This handles the captive portal case (hotel WiFi, airport). The network is "available"  
but not yet validated (user hasn't accepted the portal). We emit `false` until validated.

---

## Cache Architecture

### Cache layers

```
┌─────────────────────────────────────────────────────────────────┐
│                    In-Memory Cache (L1)                         │
│                                                                 │
│  SharedFlow / StateFlow in Repositories                         │
│  LocationRepository._locationFlow (replay=1)                   │
│  LocationBuffer: ArrayDeque<LatLng> (max 500 points)           │
│                                                                 │
│  Lifetime: process lifetime. Lost on process death.             │
│  Access time: ~0ms                                              │
└────────────────────────────┬────────────────────────────────────┘
                             │ miss / flush
┌────────────────────────────▼────────────────────────────────────┐
│                    Room Database (L2)                           │
│                                                                 │
│  rides · driver_locations · location_history                   │
│  last_known_location                                            │
│                                                                 │
│  Lifetime: indefinite (until explicit deletion)                 │
│  Access time: ~1-5ms                                            │
│  Survives: process death, app restart                           │
└────────────────────────────┬────────────────────────────────────┘
                             │ miss / sync
┌────────────────────────────▼────────────────────────────────────┐
│                    Remote Server (L3)                           │
│                                                                 │
│  REST API (Retrofit)                                            │
│  WebSocket (real-time events)                                   │
│                                                                 │
│  Lifetime: authoritative / indefinite                           │
│  Access time: 100ms - 10s (network dependent)                  │
│  Available: only when online                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

### Cache policy per data type

| Data | L1 (memory) | L2 (Room) | L3 (server) | TTL | Eviction |
|---|---|---|---|---|---|
| Active ride state | `SharedFlow(replay=1)` | Always | On reconnect | Until ride ends | On completion/cancel |
| Driver location | `SharedFlow(replay=1)` | `last_known_location` | WebSocket stream | 30s staleness | On new fix |
| Location track | `ArrayDeque(max=500)` | `location_history` | Upload on reconnect | Until ride ends | On confirmed upload |
| Ride history | — | `rides` table | On reconnect sync | 5 min | On sync |
| ETA | In UiState | In `rides.eta_seconds` | WebSocket updates | 30s | On update |
| User preferences | DataStore in-memory | DataStore file | — | Indefinite | User changes |
| Permission state | DataStore in-memory | DataStore file | — | Indefinite | User grants/revokes |

---

### Staleness detection

```kotlin
// :core-data/src/main/kotlin/.../util/StalenessChecker.kt

object StalenessChecker {
    // How long we trust cached data before requiring a refresh
    val RIDE_HISTORY_TTL = Duration.ofMinutes(5)
    val DRIVER_LOCATION_TTL = Duration.ofSeconds(30)
    val ETA_TTL = Duration.ofSeconds(60)

    fun isStale(lastUpdated: Instant, ttl: Duration): Boolean =
        Instant.now().isAfter(lastUpdated.plus(ttl))
}

// Usage in Repository
suspend fun getDriverLocation(rideId: String): DriverLocationResult {
    val cached = driverLocationDao.getLatest(rideId)
    return when {
        cached == null ->
            DriverLocationResult.Unknown
        StalenessChecker.isStale(cached.recordedAt, DRIVER_LOCATION_TTL) ->
            DriverLocationResult.Stale(cached.toDomain())
        else ->
            DriverLocationResult.Fresh(cached.toDomain())
    }
}

sealed interface DriverLocationResult {
    data object Unknown : DriverLocationResult
    data class Stale(val location: LatLng) : DriverLocationResult
    data class Fresh(val location: LatLng) : DriverLocationResult
}
```

The `Stale` vs `Fresh` distinction propagates to the ViewModel, which shows the  
degraded-tracking banner only when the location is `Stale` — not for brief update gaps.

---

## Read Strategies

### Cache-first (offline-safe reads)

Use for: data that must work offline regardless of network state.

```kotlin
// Active ride — always read from Room first
override suspend fun getActiveRide(): Ride? =
    withContext(ioDispatcher) {
        rideDao.getActiveRide()?.toDomain()
        // No network call — Room is the source of truth for the current session.
        // Server reconciliation happens separately via reconcileWithServer().
    }

// Ride history — observe Room, refresh in background
override fun observeRideHistory(): Flow<List<Ride>> =
    rideDao.observeHistory()
        .map { it.map { entity -> entity.toDomain() } }
        .flowOn(ioDispatcher)
        // Room's Flow auto-emits when data changes.
        // The background refresh (syncRideHistory) writes to Room,
        // which Room then propagates through this Flow — no explicit re-emit needed.
```

---

### Network-first with cache fallback

Use for: data that should be fresh, but the cached version is acceptable if offline.

```kotlin
// ETA for waiting screen — prefer fresh from server, fall back to cache
override suspend fun getCurrentEta(rideId: String): Result<Int> =
    withContext(ioDispatcher) {
        if (networkMonitor.isCurrentlyOnline()) {
            runCatching {
                rideApiService.getEta(rideId).bodyOrThrow().etaSeconds
            }.recoverCatching {
                // Network call failed — fall back to cache
                rideDao.getById(rideId)?.etaSeconds
                    ?: throw DomainError.DataUnavailable
            }
        } else {
            // Offline — use cache immediately without attempting network
            runCatching {
                rideDao.getById(rideId)?.etaSeconds
                    ?: throw DomainError.DataUnavailable
            }
        }.mapFailure { it.toDomainError() }
    }
```

---

### Stale-while-revalidate

Use for: lists and history where showing stale content is better than showing a loader.

```kotlin
// Ride history — emit cache, then update in background
override fun observeRideHistoryWithRefresh(): Flow<ListResult<Ride>> = flow {
    // Step 1: emit cached data immediately
    val cached = rideDao.observeHistory().first()
    emit(ListResult.Cached(cached.map { it.toDomain() }))

    // Step 2: check if refresh is needed
    val lastSync = syncMetadataStore.getLastHistorySync()
    val needsRefresh = lastSync == null ||
        StalenessChecker.isStale(lastSync, RIDE_HISTORY_TTL)

    if (!needsRefresh || !networkMonitor.isCurrentlyOnline()) return@flow

    // Step 3: refresh from server
    runCatching {
        val fresh = rideApiService.getRideHistory().bodyOrThrow()
        rideDao.replaceHistory(fresh.map { it.toEntity() })
        syncMetadataStore.markHistorySynced()
        // Room's Flow in observeHistory() will auto-emit — no explicit emit here
    }.onFailure { /* suppress — cached data already emitted */ }
}.flatMapLatest {
    // Step 4: return the live Room Flow from this point forward
    rideDao.observeHistory()
        .map { entities -> ListResult.Fresh(entities.map { it.toDomain() }) }
}

sealed interface ListResult<out T> {
    data class Cached<T>(val data: List<T>) : ListResult<T>
    data class Fresh<T>(val data: List<T>) : ListResult<T>
}
```

---

## Write Strategies

### Write-through (immediate sync)

Use for: operations where data integrity is critical and we can wait for network.

```kotlin
// Cancel ride — user must see confirmation; do not proceed until server confirms
override suspend fun cancelRide(
    rideId: String,
    reason: CancellationReason,
): Result<Unit> = withContext(ioDispatcher) {
    runCatching {
        // 1. Call server first
        rideApiService.cancelRide(rideId, reason.toDto()).bodyOrThrow()
        // 2. Update local state only on server success
        rideDao.updateStatus(rideId, "CANCELLED")
    }.mapFailure { it.toDomainError() }
    // If offline: Result.failure(DomainError.NetworkUnavailable)
    // ViewModel shows "Cannot cancel — no connection" error
}
```

---

### Write-back with local buffer

Use for: high-frequency writes where immediate server sync is impractical.

```kotlin
// Location history — buffer in memory, flush periodically
class LocationBuffer @Inject constructor(
    private val locationHistoryDao: LocationHistoryDao,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,
) {
    private val buffer = ArrayDeque<LocationHistoryEntity>()
    private val FLUSH_THRESHOLD = 30    // flush every 30 points (~60-150 seconds at 2-5s interval)
    private val MAX_BUFFER_SIZE = 500   // hard cap — prevent OOM on long offline periods

    suspend fun add(point: LocationHistoryEntity) = withContext(ioDispatcher) {
        buffer.addLast(point)
        if (buffer.size > MAX_BUFFER_SIZE) buffer.removeFirst()  // drop oldest on overflow
        if (buffer.size >= FLUSH_THRESHOLD) flushToRoom()
    }

    suspend fun flushToRoom() = withContext(ioDispatcher) {
        if (buffer.isEmpty()) return@withContext
        val toWrite = buffer.toList()
        locationHistoryDao.insertAll(toWrite)
        buffer.clear()
    }

    fun size(): Int = buffer.size
    fun isEmpty(): Boolean = buffer.isEmpty()
}
```

---

### Optimistic writes

Use for: low-risk operations where immediate visual feedback is important.

```kotlin
// Rating submission — update UI immediately, sync in background
override suspend fun submitRating(rideId: String, rating: Int): Result<Unit> =
    withContext(ioDispatcher) {
        // 1. Update local cache immediately — user sees confirmation instantly
        rideDao.updateRating(rideId, rating)

        // 2. Attempt server sync
        val serverResult = runCatching {
            rideApiService.submitRating(rideId, rating).bodyOrThrow()
        }

        if (serverResult.isFailure) {
            // 3. Queue for retry if server sync fails
            deferredWriteQueue.enqueue(
                DeferredWrite.RatingSubmission(rideId, rating)
            )
        }

        // Return success regardless — local write succeeded
        Result.success(Unit)
    }
```

**When to use optimistic writes:**  
Only for operations where: (a) the user experience matters more than guaranteed sync,  
(b) the operation is unlikely to be rejected by the server, and  
(c) failure can be handled gracefully (retry queue, eventual consistency).  
Do NOT use optimistic writes for `cancelRide` — the user must know if the cancel succeeded.

---

### Deferred writes with WorkManager

Use for: non-critical operations that can wait for connectivity.

```kotlin
// Rating submission deferred write — executed by WorkManager when online
class SubmitRatingWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted workerParams: WorkerParameters,
    private val rideApiService: RideApiService,
) : CoroutineWorker(context, workerParams) {

    override suspend fun doWork(): Result {
        val rideId = inputData.getString(KEY_RIDE_ID) ?: return Result.failure()
        val rating = inputData.getInt(KEY_RATING, -1).takeIf { it > 0 } ?: return Result.failure()

        return runCatching {
            rideApiService.submitRating(rideId, rating).bodyOrThrow()
            Result.success()
        }.getOrElse { throwable ->
            if (runAttemptCount < MAX_RETRIES) Result.retry()
            else {
                // Log permanent failure — rating lost after max retries
                logger.e(TAG, "Rating submission permanently failed", throwable)
                Result.failure()
            }
        }
    }

    companion object {
        const val KEY_RIDE_ID = "ride_id"
        const val KEY_RATING  = "rating"
        const val MAX_RETRIES = 5
        const val TAG = "SubmitRatingWorker"

        fun buildRequest(rideId: String, rating: Int): OneTimeWorkRequest =
            OneTimeWorkRequestBuilder<SubmitRatingWorker>()
                .setInputData(workDataOf(KEY_RIDE_ID to rideId, KEY_RATING to rating))
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(NetworkType.CONNECTED)
                        .build()
                )
                .setBackoffCriteria(
                    BackoffPolicy.EXPONENTIAL,
                    WorkRequest.MIN_BACKOFF_MILLIS,
                    TimeUnit.MILLISECONDS,
                )
                .build()
    }
}
```

---

## Reconnect & Sync Flow

### Reconnect sequence

```
NetworkMonitor emits: isOnline = true
        │
        ▼
SyncManager.onNetworkRestored()
        │
        ├─► [Priority 1] reconcileActiveRide()
        │       └─► GET /rides/active
        │               ├─► 200: upsert to Room, dispatch RideSynced to ViewModel
        │               ├─► 404: no active ride — clear Room if stale
        │               └─► error: log, retry up to 3× with exponential backoff
        │
        ├─► [Priority 2] flushLocationBuffer()      (only if InProgress)
        │       └─► POST /rides/{id}/track-points
        │               ├─► 200: clear buffer and location_history for this ride
        │               └─► error: keep buffer, retry on next reconnect
        │
        ├─► [Priority 3] reconnectWebSocket()
        │       └─► RideWebSocketClient auto-reconnects via channelFlow loop
        │               └─► events resume, ViewModel state updates
        │
        └─► [Priority 4] WorkManager processes deferred writes
                └─► SubmitRatingWorker, etc. — non-critical, run when convenient
```

---

### Priority-ordered sync

```kotlin
// :core-data/src/main/kotlin/.../sync/SyncManager.kt

@Singleton
class SyncManager @Inject constructor(
    private val rideRepository: RideRepository,
    private val locationRepository: LocationRepository,
    private val networkMonitor: NetworkMonitor,
    private val workManager: WorkManager,
    @AppScope private val appScope: CoroutineScope,
    private val logger: Logger,
) {
    init {
        appScope.launch {
            networkMonitor.isOnline
                .distinctUntilChanged()
                .filter { isOnline -> isOnline }
                .collect { onNetworkRestored() }
        }
    }

    private suspend fun onNetworkRestored() {
        logger.d(TAG, "Network restored — starting sync sequence")

        // Priority 1: ride state — most critical, must be first
        withRetry(maxAttempts = 3, tag = "reconcile") {
            rideRepository.reconcileWithServer()
        }

        // Priority 2: location buffer — only if actively tracking
        val activeRide = rideRepository.getActiveRide()
        if (activeRide?.status == RideStatus.IN_PROGRESS) {
            withRetry(maxAttempts = 3, tag = "flush_locations") {
                locationRepository.uploadPendingTrack(activeRide.id)
            }
        }

        // Priority 3: WebSocket — auto-reconnects via channelFlow, no explicit call needed

        // Priority 4: ride history — non-critical, best-effort
        appScope.launch {
            rideRepository.syncRideHistory()
                .onFailure { logger.w(TAG, "History sync failed — will retry later") }
        }
    }

    private suspend fun withRetry(
        maxAttempts: Int,
        tag: String,
        backoffMs: Long = 1_000L,
        block: suspend () -> Result<*>,
    ) {
        repeat(maxAttempts) { attempt ->
            val result = block()
            if (result.isSuccess) return
            logger.w(TAG, "$tag failed (attempt ${attempt + 1}/$maxAttempts)")
            if (attempt < maxAttempts - 1) delay(backoffMs * (attempt + 1))
        }
        logger.e(TAG, "$tag failed after $maxAttempts attempts")
    }

    companion object { private const val TAG = "SyncManager" }
}
```

---

### Reconcile with server

```kotlin
// In RideRepositoryImpl

override suspend fun reconcileWithServer(): Result<Unit> =
    withContext(ioDispatcher) {
        runCatching {
            val serverRide = runCatching {
                rideApiService.getActiveRide().bodyOrThrow()
            }.getOrNull()

            val localRide = rideDao.getActiveRide()

            when {
                // Server has no active ride
                serverRide == null && localRide != null -> {
                    // Check if local ride is terminal — may have been completed offline
                    if (localRide.status !in listOf("COMPLETED", "CANCELLED")) {
                        // Server doesn't know about our ride — mark it lost
                        logger.w(TAG, "Server has no active ride; local has ${localRide.status}")
                        rideDao.updateStatus(localRide.id, "CANCELLED")
                        _reconcileEvent.emit(ReconcileEvent.RideLostOnServer)
                    }
                }

                // Server has a ride, we have none — restore from server
                serverRide != null && localRide == null -> {
                    rideDao.upsert(serverRide.toEntity())
                    _reconcileEvent.emit(ReconcileEvent.RideRestored(serverRide.toDomain()))
                }

                // Both exist — server wins on conflicts
                serverRide != null && localRide != null -> {
                    val serverUpdatedAt = Instant.parse(serverRide.updatedAt)
                    val localUpdatedAt = localRide.updatedAt

                    if (serverUpdatedAt > localUpdatedAt ||
                        serverRide.status != localRide.status
                    ) {
                        rideDao.upsert(serverRide.toEntity())
                        _reconcileEvent.emit(ReconcileEvent.RideUpdated(serverRide.toDomain()))
                    }
                    // Local is same or newer — keep local (optimistic write in flight)
                }

                // Neither has a ride — nothing to reconcile
                else -> { /* no-op */ }
            }
        }.mapFailure { it.toDomainError() }
    }

// ReconcileEvent — consumed by AppViewModel to update RideStateMachine
sealed interface ReconcileEvent {
    data object RideLostOnServer : ReconcileEvent
    data class RideRestored(val ride: Ride) : ReconcileEvent
    data class RideUpdated(val ride: Ride) : ReconcileEvent
}
```

---

## Conflict Resolution

### Server-wins policy

The server is the authoritative source for ride state. Always.

| Conflict | Resolution | Reason |
|---|---|---|
| Client says `Waiting`, server says `InProgress` | Adopt `InProgress` | Driver started the ride — server is correct |
| Client says `InProgress`, server says `Completed` | Adopt `Completed` | Ride ended — even if client didn't receive the WebSocket event |
| Client says `Waiting`, server says `Cancelled` | Adopt `Cancelled` | Driver cancelled — client missed the event |
| Client has `eta=120`, server has `eta=90` | Adopt `90` | Server has fresher data from driver's device |
| Client has local driver location, server has newer | Adopt server's | Server aggregates from driver's GPS |

The only exception: if the client has an **optimistic write in flight** (e.g. cancel request sent  
but not yet confirmed), we hold the local state for up to 5 seconds before deferring to server.

---

### When client data is newer

```kotlin
// Optimistic write protection window
private val OPTIMISTIC_WRITE_WINDOW = Duration.ofSeconds(5)

private fun shouldPreferLocal(
    localUpdatedAt: Instant,
    serverUpdatedAt: Instant,
    hasOptimisticWrite: Boolean,
): Boolean {
    if (!hasOptimisticWrite) return false
    val localIsRecent = Duration.between(localUpdatedAt, Instant.now()) < OPTIMISTIC_WRITE_WINDOW
    return localIsRecent && localUpdatedAt.isAfter(serverUpdatedAt)
}
```

---

### Handling duplicate server events

WebSocket reconnections may replay recent events. The state machine must be idempotent.

```kotlin
// In RideRepositoryImpl — deduplicate WebSocket events by sequence number or timestamp

private var lastProcessedEventId: String? = null

override fun observeRideEvents(): Flow<RideServerEvent> =
    rideWebSocketClient.observeEvents()
        .filter { dto ->
            // Deduplicate by event ID if server provides one
            if (dto.eventId != null) {
                val isDuplicate = dto.eventId == lastProcessedEventId
                if (!isDuplicate) lastProcessedEventId = dto.eventId
                !isDuplicate
            } else {
                true    // no ID — pass through (state machine handles idempotent transitions)
            }
        }
        .map { it.toDomain() }
        .catch { emit(RideServerEvent.Error(it.toDomainError())) }
```

---

## UI Behaviour While Offline

### Offline banner

The offline banner is a global component rendered at the root of the navigation graph.  
It appears on all screens when `isOnline = false`.

```kotlin
// :app/src/main/kotlin/.../ui/OfflineBanner.kt

@Composable
fun OfflineBanner(isOnline: Boolean) {
    AnimatedVisibility(
        visible = !isOnline,
        enter = slideInVertically() + fadeIn(),
        exit = slideOutVertically() + fadeOut(),
    ) {
        Surface(
            color = MaterialTheme.colorScheme.errorContainer,
            modifier = Modifier
                .fillMaxWidth()
                .semantics {
                    liveRegion = LiveRegionMode.Assertive
                    contentDescription = "No internet connection. Some features may be unavailable."
                },
        ) {
            Row(
                modifier = Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
                horizontalArrangement = Arrangement.spacedBy(8.dp),
                verticalAlignment = Alignment.CenterVertically,
            ) {
                Icon(
                    imageVector = Icons.Default.WifiOff,
                    contentDescription = null,
                    tint = MaterialTheme.colorScheme.onErrorContainer,
                )
                Text(
                    text = "No connection",
                    style = MaterialTheme.typography.labelLarge,
                    color = MaterialTheme.colorScheme.onErrorContainer,
                )
            }
        }
    }
}

// In AppNavHost or MainActivity
@Composable
fun AppScaffold(networkMonitor: NetworkMonitor) {
    val isOnline by networkMonitor.isOnline
        .collectAsStateWithLifecycle(initialValue = true)

    Column {
        OfflineBanner(isOnline = isOnline)
        // Nav host below
    }
}
```

---

### Degraded states per screen

| Screen | Offline indicator | Blocked actions | Allowed actions |
|---|---|---|---|
| Search | Banner + disabled search button + tooltip | Start ride search | View history, settings |
| Waiting | Banner + "ETA may be outdated" label | — | View driver info (cached) |
| InProgress | Banner + "Tracking continues" label | — | View route (cached polyline) |
| History | Banner + "Last updated X ago" | Refresh | View cached rides |
| Settings / Profile | No banner (fully local) | — | All settings |

---

### Error vs offline distinction

These are different conditions with different UI:

| Condition | Cause | UI | Recovery |
|---|---|---|---|
| `NetworkUnavailable` | No internet | Offline banner, degraded mode | Auto-recovers when online |
| `NetworkTimeout` | Request timed out (online but slow) | Inline retry button on the failed action | User taps retry |
| `ServerError(5xx)` | Server down | Inline error with retry | User taps retry |
| `ServerError(4xx)` | Client error | Specific error message | Depends on code |

```kotlin
// In ViewModel — map DomainError to specific UI treatment
private fun DomainError.toUiError(): UiError = when (this) {
    is DomainError.NetworkUnavailable ->
        UiError.Offline("No connection. Your ride data is saved.")
    is DomainError.NetworkTimeout ->
        UiError.Retryable("Request timed out. Tap to retry.")
    is DomainError.ServerError ->
        UiError.Retryable("Something went wrong. Tap to retry.")
    is DomainError.DataUnavailable ->
        UiError.Informational("This data is not available offline.")
    else ->
        UiError.Retryable("An error occurred. Tap to retry.")
}

sealed interface UiError {
    data class Offline(val message: String) : UiError         // don't offer retry — auto-recovers
    data class Retryable(val message: String) : UiError       // offer retry button
    data class Informational(val message: String) : UiError   // no action needed
}
```

---

## Location Tracking Offline

### FGS during network loss

The FGS must never stop because of network loss.  
It continues running, writing to the local buffer, until an explicit stop signal.

```
State: InProgress + NetworkLost → Error(NetworkLost, InProgress)

LocationTrackingService:
    - Continues requestLocationUpdates()
    - Continues writing to LocationBuffer
    - Does NOT attempt network calls
    - Logs location unavailability if GPS also lost (separate from network)

ViewModel:
    - Shows "Connection lost — tracking continues" banner
    - Does not emit StopLocationTrackingService SideEffect
    - Waits for NetworkRestored or RideCompleted/Cancelled
```

---

### Location buffer

```kotlin
// :core-data/src/main/kotlin/.../location/LocationBuffer.kt

@Singleton
class LocationBuffer @Inject constructor(
    private val locationHistoryDao: LocationHistoryDao,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,
) {
    // Thread-safe: all access via ioDispatcher
    private val buffer = ArrayDeque<LocationHistoryEntity>()

    companion object {
        const val FLUSH_THRESHOLD = 30       // points before auto-flush to Room
        const val MAX_SIZE = 500             // hard cap — ~41 minutes at 5s interval
        const val UPLOAD_BATCH_SIZE = 200    // max points per API call
    }

    suspend fun add(point: LocationHistoryEntity) = withContext(ioDispatcher) {
        if (buffer.size >= MAX_SIZE) {
            // Drop oldest — we prefer recent track over ancient history
            buffer.removeFirst()
            logger.w(TAG, "Location buffer full — dropping oldest point")
        }
        buffer.addLast(point)
        if (buffer.size >= FLUSH_THRESHOLD) flushToRoom()
    }

    /** Write buffer to Room. Safe to call multiple times. */
    suspend fun flushToRoom() = withContext(ioDispatcher) {
        if (buffer.isEmpty()) return@withContext
        val batch = buffer.toList()
        locationHistoryDao.insertAll(batch)
        buffer.clear()
        logger.d(TAG, "Flushed ${batch.size} points to Room")
    }

    /** Upload Room-persisted points to server. Clears on success. */
    suspend fun uploadToServer(rideId: String, apiService: RideApiService) =
        withContext(ioDispatcher) {
            // Flush anything still in memory first
            flushToRoom()

            val allPoints = locationHistoryDao.getTrackForRide(rideId)
            if (allPoints.isEmpty()) return@withContext

            // Upload in batches to avoid request size limits
            allPoints.chunked(UPLOAD_BATCH_SIZE).forEach { batch ->
                runCatching {
                    apiService.uploadTrackPoints(
                        rideId = rideId,
                        points = batch.map { it.toTrackPointDto() },
                    ).bodyOrThrow()
                }.onSuccess {
                    // Clear uploaded batch from Room
                    locationHistoryDao.deleteByIds(batch.map { it.id })
                }.onFailure { e ->
                    logger.e(TAG, "Track upload failed for batch — will retry", e)
                    return@withContext   // stop on first batch failure; retry next reconnect
                }
            }
            logger.d(TAG, "Uploaded ${allPoints.size} track points for ride $rideId")
        }

    fun pendingCount(): Int = buffer.size
}
```

---

### Track reconstruction on reconnect

```kotlin
// In LocationRepositoryImpl

suspend fun uploadPendingTrack(rideId: String): Result<Unit> =
    withContext(ioDispatcher) {
        runCatching {
            locationBuffer.uploadToServer(rideId, rideApiService)
        }.mapFailure { it.toDomainError() }
    }

// In SyncManager — called after ride state reconciliation
private suspend fun syncLocationTrack(rideId: String) {
    val pendingCount = locationHistoryDao.getCountForRide(rideId)
    if (pendingCount == 0) return

    logger.d(TAG, "Uploading $pendingCount pending track points for $rideId")
    locationRepository.uploadPendingTrack(rideId)
        .onFailure { logger.e(TAG, "Track upload failed — will retry on next reconnect") }
}
```

---

## Kotlin Implementation

### OfflineFirstRideRepository

Full repository wiring with offline-first reads and server reconciliation:

```kotlin
@Singleton
class RideRepositoryImpl @Inject constructor(
    private val rideApiService: RideApiService,
    private val rideWebSocketClient: RideWebSocketClient,
    private val rideDao: RideDao,
    private val networkMonitor: NetworkMonitor,
    private val syncMetadataStore: SyncMetadataStore,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,
    private val logger: Logger,
) : RideRepository {

    // Shared event channel for reconcile outcomes → ViewModel
    private val _reconcileEvents = MutableSharedFlow<ReconcileEvent>(extraBufferCapacity = 8)
    val reconcileEvents: Flow<ReconcileEvent> = _reconcileEvents

    override fun observeActiveRide(): Flow<Ride?> =
        rideDao.observeActiveRide()                     // Room auto-updates on any write
            .map { it?.toDomain() }
            .flowOn(ioDispatcher)

    override suspend fun startSearch(origin: LatLng): Result<Ride> {
        if (!networkMonitor.isCurrentlyOnline()) {
            return Result.failure(DomainError.NetworkUnavailable)
        }
        return withContext(ioDispatcher) {
            runCatching {
                val dto = rideApiService.startSearch(origin.toRequest()).bodyOrThrow()
                val domain = dto.toDomain()
                rideDao.upsert(domain.toEntity())       // cache immediately
                domain
            }.mapFailure { it.toDomainError() }
        }
    }

    // ... (cancelRide, observeRideEvents, etc. from DATA_LAYER.md)
}
```

---

### SyncManager

Already shown in [Reconnect & Sync Flow](#reconnect--sync-flow).

---

### DeferredWriteQueue

```kotlin
// :core-data/src/main/kotlin/.../sync/DeferredWriteQueue.kt

@Singleton
class DeferredWriteQueue @Inject constructor(
    private val workManager: WorkManager,
) {
    fun enqueue(write: DeferredWrite) {
        val request = when (write) {
            is DeferredWrite.RatingSubmission ->
                SubmitRatingWorker.buildRequest(write.rideId, write.rating)
        }
        workManager.enqueueUniqueWork(
            write.uniqueKey,
            ExistingWorkPolicy.REPLACE,     // idempotent — replace if same key
            request,
        )
    }
}

sealed interface DeferredWrite {
    val uniqueKey: String

    data class RatingSubmission(
        val rideId: String,
        val rating: Int,
    ) : DeferredWrite {
        override val uniqueKey = "rating_$rideId"
    }
}
```

---

### OfflineAwareUseCase pattern

```kotlin
// Base class for UseCases that need online access
abstract class NetworkRequiredUseCase<Params, Result>(
    private val networkMonitor: NetworkMonitor,
) {
    suspend operator fun invoke(params: Params): kotlin.Result<Result> {
        if (!networkMonitor.isCurrentlyOnline()) {
            return kotlin.Result.failure(DomainError.NetworkUnavailable)
        }
        return execute(params)
    }

    protected abstract suspend fun execute(params: Params): kotlin.Result<Result>
}

// Usage
class StartRideUseCase @Inject constructor(
    private val rideRepository: RideRepository,
    networkMonitor: NetworkMonitor,
) : NetworkRequiredUseCase<LatLng, Ride>(networkMonitor) {

    override suspend fun execute(params: LatLng): kotlin.Result<Ride> =
        rideRepository.startSearch(params)
}
```

---

## WorkManager — Deferred Operations

Operations queued for when connectivity is restored:

| Operation | Worker | Constraints | Retry | Max attempts |
|---|---|---|---|---|
| Submit rating | `SubmitRatingWorker` | `CONNECTED` | Exponential | 5 |
| Upload track points | `UploadTrackWorker` | `CONNECTED` | Exponential | 3 |
| Sync ride history | `SyncHistoryWorker` | `CONNECTED` | Linear | 2 |

```kotlin
// Enqueue on app start to catch any pending writes from previous sessions
class AppStartSyncWorker @AssistedInject constructor(...) : CoroutineWorker(...) {
    override suspend fun doWork(): Result {
        deferredWriteQueue.processPending()
        return Result.success()
    }
}

// In Application.onCreate()
WorkManager.getInstance(this).enqueueUniqueWork(
    "app_start_sync",
    ExistingWorkPolicy.KEEP,                // don't replace if already queued
    OneTimeWorkRequestBuilder<AppStartSyncWorker>()
        .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
        .build()
)
```

---

## Testing Offline Behaviour

### Unit tests

```kotlin
class OfflineFirstRideRepositoryTest {

    private val fakeNetworkMonitor = FakeNetworkMonitor(initialState = false)
    private val fakeRideDao = FakeRideDao()
    private val fakeApiService = FakeRideApiService()

    private val repository = RideRepositoryImpl(
        rideApiService = fakeApiService,
        rideWebSocketClient = FakeRideWebSocketClient(),
        rideDao = fakeRideDao,
        networkMonitor = fakeNetworkMonitor,
        syncMetadataStore = FakeSyncMetadataStore(),
        ioDispatcher = UnconfinedTestDispatcher(),
        logger = NoOpLogger(),
    )

    @Test
    fun `startSearch offline returns NetworkUnavailable`() = runTest {
        fakeNetworkMonitor.setOnline(false)
        val result = repository.startSearch(testLatLng())
        assertThat(result.isFailure).isTrue()
        assertThat(result.exceptionOrNull()).isInstanceOf(DomainError.NetworkUnavailable::class.java)
        assertThat(fakeRideDao.upsertedRides).isEmpty()  // nothing written offline
    }

    @Test
    fun `getActiveRide offline returns cached data`() = runTest {
        fakeNetworkMonitor.setOnline(false)
        fakeRideDao.seedRide(testRideEntity(status = "IN_PROGRESS"))
        val ride = repository.getActiveRide()
        assertThat(ride).isNotNull()
        assertThat(ride?.status).isEqualTo(RideStatus.IN_PROGRESS)
    }

    @Test
    fun `observeRideHistory emits cached data without network`() = runTest {
        fakeNetworkMonitor.setOnline(false)
        fakeRideDao.seedHistory(listOf(testRideEntity("r1"), testRideEntity("r2")))

        val result = repository.observeRideHistory().first()
        assertThat(result).hasSize(2)
        assertThat(fakeApiService.callCount).isEqualTo(0)   // no network calls made
    }

    @Test
    fun `reconcileWithServer server-wins on status conflict`() = runTest {
        fakeNetworkMonitor.setOnline(true)
        fakeRideDao.seedRide(testRideEntity(id = "r1", status = "WAITING"))
        fakeApiService.stubbedRide = testRideDto(id = "r1", status = "IN_PROGRESS")

        repository.reconcileWithServer()

        assertThat(fakeRideDao.getActiveRide()?.status).isEqualTo("IN_PROGRESS")
    }

    @Test
    fun `SyncManager triggers reconcile on network restored`() = runTest {
        val syncManager = SyncManager(
            rideRepository = repository,
            locationRepository = FakeLocationRepository(),
            networkMonitor = fakeNetworkMonitor,
            workManager = FakeWorkManager(),
            appScope = this,
            logger = NoOpLogger(),
        )

        fakeRideDao.seedRide(testRideEntity(status = "WAITING"))
        fakeApiService.stubbedRide = testRideDto(status = "IN_PROGRESS")

        fakeNetworkMonitor.setOnline(true)  // trigger sync
        advanceUntilIdle()

        assertThat(fakeRideDao.getActiveRide()?.status).isEqualTo("IN_PROGRESS")
    }
}
```

---

### Integration tests

```kotlin
@RunWith(AndroidJUnit4::class)
class OfflineIntegrationTest {

    @get:Rule
    val composeTestRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun activeRide_shownWhileOffline_afterKillingNetwork() {
        // Seed Room with an active InProgress ride
        seedActiveRideInRoom(status = "IN_PROGRESS")

        // Launch app with no network
        composeTestRule.onRoot().performClick()

        // App should show ride screen, not error screen
        composeTestRule
            .onNodeWithContentDescription("Ride in progress", substring = true)
            .assertIsDisplayed()

        // Offline banner should be visible
        composeTestRule
            .onNodeWithText("No connection")
            .assertIsDisplayed()
    }
}
```

---

### QA scenarios

| Scenario | Setup | Steps | Expected |
|---|---|---|---|
| Network loss while Waiting | Active ride in Waiting | Disable WiFi/mobile data → wait 5s | Offline banner. Driver card still visible with "ETA may be outdated". No crash. |
| Network loss while InProgress | Active ride InProgress | Disable network | Banner: "Connection lost — tracking continues". FGS notification still visible. |
| Restore network after InProgress offline | Above, then restore | Re-enable network | Banner disappears. State reconciles. ETA updates resume. |
| Process death recovery | Active InProgress ride | ADB `am kill` → reopen app | App navigates to InProgress screen. FGS restarts. |
| Offline history | Complete several rides → go offline | Open History tab | All completed rides visible. "Last updated X ago" shown. |
| Rating while offline | Complete ride → disable network → submit rating | Tap star rating → Done | Rating accepted locally. On reconnect: WorkManager submits to server. |
| No network on fresh install | New install, no network | Open app | Idle screen. Search disabled. Clear "No connection" explanation. No crash. |
| Captive portal (hotel WiFi) | Connect to portal WiFi | Open app | App correctly shows as offline (portal is not validated internet). |
| Force stop recovery | InProgress ride → Settings → Force Stop → reopen | Reopen app | InProgress screen shown from Room. Sync attempted. |
| Buffer overflow (very long offline) | InProgress, offline >41 min | Simulate 500+ GPS points | Oldest points dropped. Recent track preserved. No OOM. No crash. |

---

## Decision Log

| Decision | Rationale |
|---|---|
| Server wins on ride state conflicts | Client cannot be trusted as authority — process death, reconnects, and missed events make client state unreliable |
| FGS continues on network loss | Stopping FGS creates permanent gaps in the location track — unacceptable for a tracking app |
| Write-back for location (buffer + flush) | Writing to Room on every GPS fix (2-5s interval) would cause excessive disk IO and battery drain |
| WorkManager for rating, not immediate retry | Non-critical; using WorkManager means we get connectivity constraints and retry for free, without managing retry state ourselves |
| `SharingStarted.Eagerly` for NetworkMonitor | NetworkMonitor is app-critical infrastructure; it must be active before any screen loads, not on-demand |
| Optimistic write window (5 seconds) | Prevents the reconcile from overwriting a cancel request that is still in flight — without this, the UI would briefly show the ride as alive after the user cancelled it |
| Max buffer size 500 points | At 5s interval, 500 points ≈ 41 minutes of offline tracking. Beyond that, the ride data has other problems (driver contact, payment) that make GPS precision less critical |

---

*Last updated: <!-- date -->*  
*Owner: Android Tech Lead*  
*Review: Required before any change to sync strategy, cache policy, or conflict resolution*
