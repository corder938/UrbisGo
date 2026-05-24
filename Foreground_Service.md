# Foreground Service — Location Tracking

> **Owner:** Android Tech Lead  
> **Audience:** Android team, QA  
> **Related:** `LOCATION_PERMISSIONS.md` · `STATE_MACHINE.md` · `ARCHITECTURE.md`  
> **Last reviewed against:** Android 14 (API 34), targetSdk 34

---

## Table of Contents

- [Purpose & Scope](#purpose--scope)
- [Why a Foreground Service](#why-a-foreground-service)
- [FGS Lifecycle Overview](#fgs-lifecycle-overview)
- [Lifecycle State Machine](#lifecycle-state-machine)
- [Manifest Declaration](#manifest-declaration)
- [Kotlin Implementation](#kotlin-implementation)
  - [LocationTrackingService](#locationtrackingservice)
  - [ServiceState sealed class](#servicestate-sealed-class)
  - [Starting and stopping from ViewModel](#starting-and-stopping-from-viewmodel)
  - [LocationRepository — the bridge](#locationrepository--the-bridge)
- [Notification Design](#notification-design)
  - [Notification spec](#notification-spec)
  - [Notification implementation](#notification-implementation)
  - [Updating notification during ride](#updating-notification-during-ride)
- [Kill Scenarios & Recovery](#kill-scenarios--recovery)
  - [Scenario matrix](#scenario-matrix)
  - [START_STICKY behaviour](#start_sticky-behaviour)
  - [onTaskRemoved handling](#ontaskremoved-handling)
  - [Process death recovery flow](#process-death-recovery-flow)
- [Android Version Restrictions](#android-version-restrictions)
  - [Background start restrictions (Android 12+)](#background-start-restrictions-android-12)
  - [Exemptions we rely on](#exemptions-we-rely-on)
  - [ForegroundServiceStartNotAllowedException](#foregroundservicestartnotallowedexception)
- [Battery & Doze Interaction](#battery--doze-interaction)
- [Testing](#testing)
- [Common Mistakes](#common-mistakes)

---

## Purpose & Scope

`LocationTrackingService` is the single Foreground Service in this application.

**It exists to solve one problem:**  
Android aggressively kills background processes. A ride that goes `InProgress` while the user switches apps — to answer a message, take a call — must continue tracking location. Without a Foreground Service, the OS suspends the process, GPS updates stop, and the ride track has gaps.

**Scope — what this service does:**
- Receives location updates from `FusedLocationProviderClient` while a ride is `InProgress`
- Writes updates to `LocationRepository` (shared `StateFlow` / buffered `Channel`)
- Maintains a persistent system notification so the OS knows the process is user-visible
- Persists the last known location to Room for process death recovery

**Scope — what this service does NOT do:**
- It does not know about ViewModels, UI state, or Compose
- It does not call the backend directly
- It does not start itself — it is always started via an explicit `Intent` from the Activity
- It does not make decisions about ride state — that belongs to `RideStateMachine`

---

## Why a Foreground Service

Alternatives considered and rejected:

| Alternative | Why rejected |
|---|---|
| `WorkManager` with periodic work | Min 15 min interval. Unacceptable for real-time ride tracking. |
| `WorkManager` with `EXPEDITED` work | Not designed for long-running tasks. Max runtime ~10 min, no continuous GPS. |
| Background coroutine in `ViewModel` | Process killed when app is backgrounded on Android 8+. No guarantee of continued execution. |
| Bound Service without FGS notification | System still kills it under memory pressure. No notification = no foreground privilege. |
| `JobScheduler` / `JobService` | Same constraints as WorkManager. Not suitable for continuous streaming work. |

**Foreground Service with `foregroundServiceType="location"` is the only correct solution**  
for continuous GPS tracking on Android 8+ (API 26+).

The mandatory persistent notification is a feature, not a bug — it is the OS's way of  
ensuring the user is aware the app is actively using their location.

---

## FGS Lifecycle Overview

```
Ride state: InProgress
        │
        │  SideEffect: StartLocationTrackingService
        ▼
Activity collects SideEffect
        │
        │  context.startForegroundService(
        │      Intent(context, LocationTrackingService::class.java)
        │  )
        ▼
LocationTrackingService.onStartCommand()
        │
        │  Within 5 seconds — mandatory:
        │  startForeground(NOTIFICATION_ID, buildNotification())
        │
        │  Then:
        │  fusedClient.requestLocationUpdates(request, callback, looper)
        ▼
Location updates stream
        │
        │  callback.onLocationResult(result)
        │  locationRepository.emitLocation(latLng)
        │  locationRepository.persistLastKnown(latLng)   ← Room
        ▼
ViewModel observes via ObserveDriverLocationUseCase
        │
        │  reduce { state.copy(driverLocation = latLng) }
        ▼
Compose re-renders map pin

...ride continues...

Ride state: Completed / Cancelled
        │
        │  SideEffect: StopLocationTrackingService
        ▼
Activity collects SideEffect
        │
        │  context.stopService(
        │      Intent(context, LocationTrackingService::class.java)
        │  )
        ▼
LocationTrackingService.onDestroy()
        │
        │  fusedClient.removeLocationUpdates(callback)
        │  stopForeground(STOP_FOREGROUND_REMOVE)
        │  (stopSelf() not needed — system calls onDestroy after stopService)
        ▼
Service destroyed. Notification removed.
Process no longer has foreground privilege.
```

**Critical timing constraint:**  
`startForeground()` must be called within **5 seconds** of `onStartCommand()`.  
On Android 12+ this window may be shorter under heavy load.  
If exceeded: `ForegroundServiceDidNotStartInTimeException` → app crash.  
See [implementation](#locationtrackingservice) — notification is built synchronously, before any async work.

---

## Lifecycle State Machine

The service itself is stateless — Android manages its lifecycle.  
But we model its external state for the ViewModel to reason about.

```kotlin
sealed interface FgsState {
    data object Stopped : FgsState
    data object Starting : FgsState          // startForegroundService() called, not yet onStartCommand
    data object Running : FgsState           // startForeground() called, updates flowing
    data class Error(val cause: Throwable) : FgsState
}
```

Transitions:

| From | Event | To | Notes |
|---|---|---|---|
| `Stopped` | `StartLocationTrackingService` SideEffect | `Starting` | |
| `Starting` | `onStartCommand` called | `Running` | After `startForeground()` |
| `Running` | `StopLocationTrackingService` SideEffect | `Stopped` | After `onDestroy` |
| `Running` | System kill (OOM / user swipe) | `Stopped` + `START_STICKY` reschedule | Service restarts if conditions allow |
| `Running` | `onTaskRemoved` (user swipes app from recents) | `Stopped` (intentional) | Stop self — ride is over from user intent |
| `Starting` / `Running` | `ForegroundServiceStartNotAllowedException` | `Error` | Log + recover via retry or notification |

---

## Manifest Declaration

```xml
<!-- AndroidManifest.xml -->

<!-- Permissions — must be declared; auto-granted at install time -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

<!-- Android 14+ (API 34): required for foregroundServiceType="location" -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />

<!-- Location permissions — runtime, see LOCATION_PERMISSIONS.md -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />

<!-- Android 13+ (API 33): to show the persistent notification -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<!-- Service declaration -->
<application ...>

    <service
        android:name=".location.LocationTrackingService"
        android:foregroundServiceType="location"
        android:exported="false"
        android:stopWithTask="false" />

    <!--
        stopWithTask="false" is intentional:
        We want the service to survive the user navigating away from the app.
        If the user swipes the app from recents, we handle it in onTaskRemoved()
        and stop the service manually — giving us control over the shutdown sequence.

        stopWithTask="true" would kill the service immediately when the task is removed,
        preventing clean shutdown (flush buffered locations, update Room, etc.).
    -->

</application>
```

---

## Kotlin Implementation

### LocationTrackingService

```kotlin
// :core-location/src/main/kotlin/.../service/LocationTrackingService.kt

@AndroidEntryPoint
class LocationTrackingService : Service() {

    @Inject lateinit var locationRepository: LocationRepository
    @Inject lateinit var rideRepository: RideRepository
    @Inject lateinit var logger: Logger
    @Inject lateinit var fusedLocationClient: FusedLocationProviderClient
    @Inject lateinit var notificationBuilder: RideNotificationBuilder

    private val serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private var locationCallback: LocationCallback? = null

    // ── Lifecycle ──────────────────────────────────────────────────────────────

    override fun onCreate() {
        super.onCreate()
        logger.d(TAG, "onCreate")
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        logger.d(TAG, "onStartCommand: action=${intent?.action}")

        // CRITICAL: startForeground() must be called here, synchronously,
        // before any suspend functions or coroutine launches.
        // Failure to do this within 5 seconds → ForegroundServiceDidNotStartInTimeException.
        val notification = notificationBuilder.buildInitialNotification()
        startForeground(NOTIFICATION_ID, notification)

        when (intent?.action) {
            ACTION_START -> handleStart()
            ACTION_STOP  -> handleStop(startId)
            null         -> handleStart()   // system restart after START_STICKY kill
        }

        return START_STICKY
        // START_STICKY: if the system kills the service due to memory pressure,
        // it will restart it with a null Intent when resources are available.
        // We handle null intent in onStartCommand (see above).
        // Alternative: START_NOT_STICKY — service is not restarted. Not suitable here
        // because a killed service during an active ride would mean lost tracking.
    }

    override fun onBind(intent: Intent?): IBinder? = null
    // This is a started service, not a bound service.
    // Returning null means no component can bind to it.

    override fun onTaskRemoved(rootIntent: Intent?) {
        // Called when the user swipes the app from the recents screen.
        // We treat this as an intentional termination of the current session.
        logger.d(TAG, "onTaskRemoved — user removed task, stopping service")
        serviceScope.launch {
            flushPendingLocationBuffer()    // write buffered points to Room
            rideRepository.markRideInterrupted()    // inform backend if possible
        }
        stopForeground(STOP_FOREGROUND_REMOVE)
        stopSelf()
        super.onTaskRemoved(rootIntent)
    }

    override fun onDestroy() {
        logger.d(TAG, "onDestroy")
        removeLocationUpdates()
        serviceScope.cancel()   // cancel all coroutines
        super.onDestroy()
    }

    // ── Internal ───────────────────────────────────────────────────────────────

    private fun handleStart() {
        logger.d(TAG, "Starting location updates")
        requestLocationUpdates()
        observeRideStateForNotificationUpdates()
    }

    private fun handleStop(startId: Int) {
        logger.d(TAG, "handleStop requested")
        removeLocationUpdates()
        stopForeground(STOP_FOREGROUND_REMOVE)
        stopSelf()
    }

    private fun requestLocationUpdates() {
        val request = buildLocationRequest()
        locationCallback = buildLocationCallback()

        try {
            fusedLocationClient.requestLocationUpdates(
                request,
                locationCallback!!,
                Looper.getMainLooper(),     // callbacks on main thread; we offload in callback
            )
        } catch (e: SecurityException) {
            // Permission was revoked between permission check and FGS start.
            // This should be caught by the ViewModel before starting the service,
            // but we handle it defensively here.
            logger.e(TAG, "SecurityException requesting location updates", e)
            stopSelf()
        }
    }

    private fun removeLocationUpdates() {
        locationCallback?.let { callback ->
            fusedLocationClient.removeLocationUpdates(callback)
            locationCallback = null
            logger.d(TAG, "Location updates removed")
        }
    }

    private fun buildLocationRequest(): LocationRequest =
        LocationRequest.Builder(
            Priority.PRIORITY_HIGH_ACCURACY,
            LOCATION_INTERVAL_MS,
        )
            .setMinUpdateIntervalMillis(LOCATION_MIN_UPDATE_MS)
            .setMaxUpdateDelayMillis(LOCATION_MAX_DELAY_MS)
            .setWaitForAccurateLocation(false)  // don't block; use best available immediately
            .build()

    private fun buildLocationCallback(): LocationCallback =
        object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                val location = result.lastLocation ?: return
                val latLng = LatLng(location.latitude, location.longitude)

                serviceScope.launch {
                    locationRepository.emitLocation(latLng)
                    locationRepository.persistLastKnown(latLng)
                }
            }

            override fun onLocationAvailability(availability: LocationAvailability) {
                if (!availability.isLocationAvailable) {
                    logger.w(TAG, "Location unavailable — GPS signal lost?")
                    // Do not stop the service. Signal may recover.
                    // ViewModel will show a degraded-signal UI state.
                    serviceScope.launch {
                        locationRepository.emitAvailabilityChange(available = false)
                    }
                }
            }
        }

    private fun observeRideStateForNotificationUpdates() {
        serviceScope.launch {
            rideRepository.observeActiveRide().collect { ride ->
                val updated = notificationBuilder.buildRideNotification(ride)
                notificationManager.notify(NOTIFICATION_ID, updated)
            }
        }
    }

    private suspend fun flushPendingLocationBuffer() {
        locationRepository.flushBuffer()
    }

    companion object {
        private const val TAG = "LocationTrackingService"
        const val NOTIFICATION_ID = 1001
        const val ACTION_START = "action.START_TRACKING"
        const val ACTION_STOP  = "action.STOP_TRACKING"

        // Location update intervals — tune per product requirements
        const val LOCATION_INTERVAL_MS      = 5_000L    // request update every 5s
        const val LOCATION_MIN_UPDATE_MS    = 2_000L    // but not faster than every 2s
        const val LOCATION_MAX_DELAY_MS     = 10_000L   // batch tolerance (battery saving)

        fun startIntent(context: Context): Intent =
            Intent(context, LocationTrackingService::class.java)
                .apply { action = ACTION_START }

        fun stopIntent(context: Context): Intent =
            Intent(context, LocationTrackingService::class.java)
                .apply { action = ACTION_STOP }
    }
}
```

---

### ServiceState sealed class

```kotlin
// :core-location/src/main/kotlin/.../service/FgsState.kt

sealed interface FgsState {
    data object Stopped  : FgsState
    data object Starting : FgsState
    data object Running  : FgsState
    data class  Error(val cause: Throwable) : FgsState
}
```

---

### Starting and stopping from ViewModel

The service is **never started directly from a composable**.  
The composable collects `SideEffect`s and delegates to `context.startForegroundService()`.

```kotlin
// :feature-ride/src/main/kotlin/.../RideScreen.kt

@Composable
fun RideScreen(viewModel: RideViewModel = hiltViewModel(), ...) {
    val context = LocalContext.current

    viewModel.collectSideEffect { effect ->
        when (effect) {
            RideSideEffect.StartLocationTrackingService -> {
                try {
                    ContextCompat.startForegroundService(
                        context,
                        LocationTrackingService.startIntent(context),
                    )
                } catch (e: ForegroundServiceStartNotAllowedException) {
                    // Android 12+: cannot start FGS from background.
                    // This should not happen in our flow — we only start from InProgress
                    // which requires an active foreground Activity.
                    // If it does happen, log it and notify the ViewModel.
                    viewModel.onIntent(RideIntent.FgsStartFailed(e))
                }
            }

            RideSideEffect.StopLocationTrackingService ->
                context.stopService(LocationTrackingService.stopIntent(context))

            // ... other effects
        }
    }
}
```

**Why `ContextCompat.startForegroundService()` rather than `startForegroundService()`:**  
On API < 26 it falls back to `startService()`. On API 26+ it calls `startForegroundService()`.  
Using the compat version means we don't need API version checks at the call site.

---

### LocationRepository — the bridge

The service and ViewModel communicate **only through `LocationRepository`**.  
No direct references between service and ViewModel in either direction.

```kotlin
// :core-domain/src/main/kotlin/.../repository/LocationRepository.kt

interface LocationRepository {
    /** Hot flow of location updates from the FGS. Emits when GPS updates arrive. */
    fun observeLocation(): Flow<LatLng>

    /** Last known location — may be from a previous session (Room). */
    suspend fun getLastKnownLocation(): LatLng?

    /** Called by LocationTrackingService on each GPS update. */
    suspend fun emitLocation(latLng: LatLng)

    /** Persists to Room for process death recovery. */
    suspend fun persistLastKnown(latLng: LatLng)

    /** Signals GPS signal availability changes to ViewModel. */
    suspend fun emitAvailabilityChange(available: Boolean)

    /** Flushes in-memory buffer to Room — called before service stops. */
    suspend fun flushBuffer()
}

// :core-data/src/main/kotlin/.../repository/LocationRepositoryImpl.kt

@Singleton
class LocationRepositoryImpl @Inject constructor(
    private val locationDao: LocationDao,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,
) : LocationRepository {

    // Shared across Service and ViewModel — Singleton scope ensures same instance.
    private val _locationFlow = MutableSharedFlow<LatLng>(
        replay = 1,
        onBufferOverflow = BufferOverflow.DROP_OLDEST,
    )
    override fun observeLocation(): Flow<LatLng> = _locationFlow.asSharedFlow()

    // In-memory buffer for network-loss scenarios
    private val locationBuffer = ArrayDeque<LatLng>(maxSize = 50)

    override suspend fun emitLocation(latLng: LatLng) {
        _locationFlow.emit(latLng)
        locationBuffer.addLast(latLng)
        if (locationBuffer.size > 50) locationBuffer.removeFirst()
    }

    override suspend fun persistLastKnown(latLng: LatLng) =
        withContext(ioDispatcher) {
            locationDao.upsertLastKnown(latLng.toEntity())
        }

    override suspend fun flushBuffer() =
        withContext(ioDispatcher) {
            if (locationBuffer.isNotEmpty()) {
                locationDao.insertAll(locationBuffer.map { it.toEntity() })
                locationBuffer.clear()
            }
        }

    override suspend fun emitAvailabilityChange(available: Boolean) {
        _availabilityFlow.emit(available)
    }

    private val _availabilityFlow = MutableSharedFlow<Boolean>(replay = 1)
}
```

---

## Notification Design

### Notification spec

The persistent notification is **mandatory** — it is the OS signal that the service is  
legitimately running and the user is aware of it. Design it accordingly:

| Property | Value | Reason |
|---|---|---|
| Channel ID | `"location_tracking"` | Separate channel → user can customise independently |
| Channel importance | `IMPORTANCE_LOW` | No sound/vibration; doesn't interrupt the user |
| Priority (pre-Oreo) | `PRIORITY_LOW` | Same intent |
| Small icon | App icon monochrome | Required; shown in status bar |
| Content title | "Ride in progress" | Clear, actionable |
| Content text | "Tap to view your ride" or dynamic ETA | Context-aware |
| Content intent | Deep link to `RideScreen` | User taps → returns to ride |
| Ongoing | `true` | Cannot be swiped away — protects the service |
| Category | `CATEGORY_SERVICE` | Correct semantic category for OS |
| Colour | Brand primary colour | Visual identity in notification shade |

**Do NOT use `IMPORTANCE_HIGH` or `IMPORTANCE_DEFAULT`.**  
These add sound and heads-up display — unacceptable for a persistent service notification  
that the user will see for the entire duration of a ride.

**Do NOT make the notification dismissible (`ongoing = false`).**  
If the user can dismiss it, they dismiss the service, killing location tracking mid-ride.

---

### Notification implementation

```kotlin
// :core-location/src/main/kotlin/.../notification/RideNotificationBuilder.kt

@Singleton
class RideNotificationBuilder @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    private val notificationManager =
        context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager

    init {
        createNotificationChannel()
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                context.getString(R.string.notification_channel_tracking_name),
                NotificationManager.IMPORTANCE_LOW,
            ).apply {
                description = context.getString(R.string.notification_channel_tracking_desc)
                setShowBadge(false)             // no app icon badge
                enableLights(false)
                enableVibration(false)
                setSound(null, null)            // explicitly silence
            }
            notificationManager.createNotificationChannel(channel)
        }
    }

    /**
     * Called synchronously in onStartCommand() before any async work.
     * Must be fast — no IO, no network calls.
     */
    fun buildInitialNotification(): Notification =
        buildNotification(
            title = context.getString(R.string.notification_tracking_title),
            text = context.getString(R.string.notification_tracking_initial_text),
        )

    fun buildRideNotification(ride: Ride?): Notification {
        val text = when {
            ride == null -> context.getString(R.string.notification_tracking_initial_text)
            ride.etaSeconds != null -> context.getString(
                R.string.notification_tracking_eta_text,
                formatEta(ride.etaSeconds),
            )
            else -> context.getString(R.string.notification_tracking_in_progress_text)
        }
        return buildNotification(
            title = context.getString(R.string.notification_tracking_title),
            text = text,
        )
    }

    private fun buildNotification(title: String, text: String): Notification {
        val tapIntent = PendingIntent.getActivity(
            context,
            0,
            Intent(context, MainActivity::class.java).apply {
                action = Intent.ACTION_MAIN
                addCategory(Intent.CATEGORY_LAUNCHER)
                flags = Intent.FLAG_ACTIVITY_SINGLE_TOP
            },
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE,
        )

        return NotificationCompat.Builder(context, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_notification_location)
            .setContentTitle(title)
            .setContentText(text)
            .setContentIntent(tapIntent)
            .setOngoing(true)                       // cannot be dismissed
            .setCategory(NotificationCompat.CATEGORY_SERVICE)
            .setForegroundServiceBehavior(
                NotificationCompat.FOREGROUND_SERVICE_IMMEDIATE
            )                                       // show immediately, no 10s delay
            .setColor(ContextCompat.getColor(context, R.color.brand_primary))
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .setVisibility(NotificationCompat.VISIBILITY_PUBLIC)  // show on lock screen
            .build()
    }

    private fun formatEta(seconds: Int): String {
        val minutes = (seconds / 60).coerceAtLeast(1)
        return context.resources.getQuantityString(
            R.plurals.eta_minutes, minutes, minutes
        )
    }

    companion object {
        const val CHANNEL_ID = "location_tracking"
    }
}
```

**`FOREGROUND_SERVICE_IMMEDIATE`:**  
Without this flag, Android 12+ delays showing the FGS notification by up to 10 seconds  
for services that start and stop quickly. For our use case (ride tracking), this is wrong —  
the notification should appear immediately when tracking starts.

---

### Updating notification during ride

The notification content should update when the ride state changes  
(e.g. ETA countdown while waiting, "Ride in progress" when started).

```kotlin
// In LocationTrackingService

private fun observeRideStateForNotificationUpdates() {
    serviceScope.launch {
        rideRepository.observeActiveRide()
            .distinctUntilChanged()
            .collect { ride ->
                val updated = notificationBuilder.buildRideNotification(ride)
                // notificationManager.notify() updates an existing notification in place.
                // No flicker. The notification ID must match the one in startForeground().
                notificationManager.notify(NOTIFICATION_ID, updated)
            }
    }
}
```

---

## Kill Scenarios & Recovery

### Scenario matrix

| Scenario | What happens | Recovery |
|---|---|---|
| User switches to another app | Service continues running (that's the whole point of FGS) | Nothing to do |
| User locks screen | Service continues running | Nothing to do |
| System low-memory kill | `onDestroy()` called. `START_STICKY` → service restarted with null intent | Service restarts; `onStartCommand(null)` calls `handleStart()` again |
| User swipes app from recents | `onTaskRemoved()` called | Flush buffer, stop service intentionally |
| User force-stops app in Settings | Service killed immediately, `onDestroy()` may not be called | `START_STICKY` does NOT help here — force stop prevents restart. Accept data loss for buffered points. Room has last persisted location. |
| Battery optimisation kills process | Same as force stop — Doze can kill the process | Mitigation: request `IGNORE_BATTERY_OPTIMIZATIONS` exemption (see [Battery & Doze](#battery--doze-interaction)) |
| Process death (OOM killer) | `onDestroy()` may not be called | Room has last known location. On next launch, `rideRepository.getActiveRide()` restores state. |
| App crash (unhandled exception) | `onDestroy()` called | Service also dies. `START_STICKY` restarts it. But ride may be in inconsistent state — server is authority. |

### START_STICKY behaviour

```kotlin
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    // ...
    return START_STICKY
}
```

`START_STICKY` means:
- If the system kills the service, it will restart it automatically
- `onStartCommand` is called again with `intent = null`
- We handle this: `null` action → `handleStart()` → resumes location updates
- The service does NOT restart if the user force-stops the app

Alternative values and why we don't use them:

| Return value | Behaviour | Why rejected |
|---|---|---|
| `START_NOT_STICKY` | Not restarted after system kill | Gaps in tracking during low-memory situations |
| `START_REDELIVER_INTENT` | Restarted with the original intent re-delivered | Intent may be stale; we prefer to re-initialise from repository state |

### onTaskRemoved handling

```kotlin
override fun onTaskRemoved(rootIntent: Intent?) {
    logger.d(TAG, "onTaskRemoved — user removed task")

    // Launch a coroutine for cleanup — onTaskRemoved is on main thread
    // and we have a brief window before the system kills us
    serviceScope.launch {
        // 1. Flush buffered location points to Room
        flushPendingLocationBuffer()

        // 2. Attempt to notify backend the ride was interrupted
        // Best-effort — network may already be gone
        runCatching { rideRepository.markRideInterrupted() }
    }

    // 3. Remove notification and stop the service
    // Do this synchronously — coroutines above may not complete in time
    stopForeground(STOP_FOREGROUND_REMOVE)
    stopSelf()

    super.onTaskRemoved(rootIntent)
}
```

**Why flush buffer synchronously via `launch` but stop the service synchronously?**  
`stopSelf()` initiates the stop — it does not kill the process immediately.  
The coroutines in `serviceScope` have a brief window to complete.  
`SupervisorJob()` ensures one coroutine failure doesn't cancel the others.  
We accept that the backend notification may not complete before the process dies.

### Process death recovery flow

```kotlin
// :app/src/main/kotlin/.../AppViewModel.kt

@HiltViewModel
class AppViewModel @Inject constructor(
    private val rideRepository: RideRepository,
    private val locationPermissionManager: LocationPermissionManager,
) : ViewModel() {

    init {
        restoreStateAfterProcessDeath()
    }

    private fun restoreStateAfterProcessDeath() = viewModelScope.launch {
        val activeRide = rideRepository.getActiveRide() ?: return@launch

        // An active ride found in Room — process died during a ride.
        // Dispatch a RideSynced event to restore state.
        _restoredRideEvent.emit(activeRide)

        // If the ride was InProgress, restart the FGS
        if (activeRide.status == RideStatus.IN_PROGRESS) {
            _shouldRestartFgs.emit(true)
        }
    }
}
```

```kotlin
// :app/src/main/kotlin/.../MainActivity.kt

class MainActivity : ComponentActivity() {

    @Inject lateinit var appViewModel: AppViewModel

    override fun onCreate(...) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            appViewModel.shouldRestartFgs.collect { shouldRestart ->
                if (shouldRestart &&
                    locationPermissionManager.currentState() != LocationPermissionState.NotGranted
                ) {
                    ContextCompat.startForegroundService(
                        this@MainActivity,
                        LocationTrackingService.startIntent(this@MainActivity),
                    )
                }
            }
        }
    }
}
```

---

## Android Version Restrictions

### Background start restrictions (Android 12+)

Starting with Android 12 (API 31), apps **cannot start a Foreground Service from the background**.

"Background" means: no Activity is visible, no other FGS is running, no high-priority notification is being shown, and various other exemptions do not apply.

**Why this does not affect our flow:**  
We only start `LocationTrackingService` when the ride transitions to `InProgress`.  
This transition happens via a WebSocket event — but the user is actively in the app  
(waiting for the driver), so the Activity is in the foreground.

The `SideEffect` is collected in the composable, which requires the Activity to be running.  
An active, resumed Activity is one of the exemptions from background start restrictions.

**The dangerous edge case:**  
App is backgrounded while in `Waiting` state. WebSocket delivers `RideStarted` event.  
ViewModel processes it and emits `StartLocationTrackingService` SideEffect.  
If the Activity is not in the foreground — the SideEffect collection is suspended  
(Compose lifecycle stops collecting in `STOPPED` state by default with `collectAsStateWithLifecycle`).

**Solution:** when the Activity comes back to foreground (`ON_RESUME`), re-check ride state  
and start FGS if we are `InProgress` but FGS is not running.

```kotlin
// In RideScreen or RideViewModel

LifecycleEventEffect(Lifecycle.Event.ON_RESUME) {
    viewModel.onIntent(RideIntent.ResumedFromBackground)
}

// In RideViewModel
fun handleResumedFromBackground() = intent {
    val currentRideState = state.rideState
    val fgsRunning = locationTrackingServiceChecker.isRunning()
    if (currentRideState is RideState.InProgress && !fgsRunning) {
        postSideEffect(RideSideEffect.StartLocationTrackingService)
    }
}
```

### Exemptions we rely on

| Exemption | How we qualify |
|---|---|
| App has a visible Activity | Primary case — user is in the app |
| App received a high-priority FCM message | Not relied on in current implementation |
| `START_STICKY` restart | System-initiated restart — exempt from background start restriction |

### ForegroundServiceStartNotAllowedException

```kotlin
// Thrown on Android 12+ when attempting to start FGS from background

try {
    ContextCompat.startForegroundService(context, intent)
} catch (e: ForegroundServiceStartNotAllowedException) {
    // This is a programming error in our flow — investigate why we are
    // attempting to start FGS from background.
    logger.e(TAG, "FGS start not allowed — app is in background", e)
    crashReporter.recordNonFatal(e)

    // Notify ViewModel so it can show an appropriate error state
    // and attempt recovery on next foreground resume.
    viewModel.onIntent(RideIntent.FgsStartFailed(e))
}
```

---

## Battery & Doze Interaction

Doze mode (introduced in Android 6, tightened in every subsequent release) can  
defer or block background work. FGS with `foregroundServiceType="location"` has  
specific exemptions, but they are not absolute.

| Scenario | Doze impact on FGS | Our mitigation |
|---|---|---|
| Screen off, charging | Minimal — Doze light mode only | No action needed |
| Screen off, on battery, stationary | Doze may enter deep mode. FGS with `foregroundServiceType="location"` continues receiving location updates but at reduced frequency. | Accept reduced frequency. UI shows last known. |
| Screen off, on battery, moving | Motion detected — Doze stays in light mode. Normal location updates. | No action needed. |
| Battery Saver mode enabled | Location updates limited to `PRIORITY_BALANCED_POWER_ACCURACY` effectively. | Log degraded accuracy. Do not show error — this is expected. |

**Battery optimisation whitelist:**  
For European logistics/courier apps where continuous tracking is business-critical,  
consider requesting `IGNORE_BATTERY_OPTIMIZATIONS` permission.

This is a sensitive permission — Google reviews it for Play Store and may reject it  
if the use case is not sufficiently justified. For a courier/driver app: justified.  
For a passenger app: likely not required.

```kotlin
// Check if we need to request the exemption
fun isIgnoringBatteryOptimizations(context: Context): Boolean {
    val pm = context.getSystemService(Context.POWER_SERVICE) as PowerManager
    return pm.isIgnoringBatteryOptimizations(context.packageName)
}

// Direct user to the system dialog
fun requestIgnoreBatteryOptimizations(context: Context) {
    val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
        data = Uri.parse("package:${context.packageName}")
    }
    context.startActivity(intent)
}
```

**Manifest declaration required if using this:**
```xml
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
```

This also requires a Play Store justification in the Permissions declaration form.

---

## Testing

### Unit tests — service logic in isolation

```kotlin
// :core-location/src/test/.../service/LocationTrackingServiceTest.kt

// Test the service's internal logic without starting the real Android service.
// Use a fake LocationRepository and verify emissions.

class LocationRepositoryFake : LocationRepository {
    val emittedLocations = mutableListOf<LatLng>()
    val persistedLocations = mutableListOf<LatLng>()
    private val _flow = MutableSharedFlow<LatLng>(replay = 1)

    override fun observeLocation() = _flow.asSharedFlow()
    override suspend fun emitLocation(latLng: LatLng) {
        emittedLocations += latLng
        _flow.emit(latLng)
    }
    override suspend fun persistLastKnown(latLng: LatLng) { persistedLocations += latLng }
    override suspend fun flushBuffer() {}
    override suspend fun emitAvailabilityChange(available: Boolean) {}
    override suspend fun getLastKnownLocation() = persistedLocations.lastOrNull()
}

@Test
fun `emitLocation is called for each GPS update`() = runTest {
    val fakeRepo = LocationRepositoryFake()
    // Use a real LocationCallback and simulate calls to it
    val callback = buildLocationCallback(fakeRepo)

    callback.onLocationResult(buildFakeLocationResult(lat = 52.52, lng = 13.40))
    callback.onLocationResult(buildFakeLocationResult(lat = 52.53, lng = 13.41))

    assertThat(fakeRepo.emittedLocations).hasSize(2)
    assertThat(fakeRepo.emittedLocations[0]).isEqualTo(LatLng(52.52, 13.40))
}
```

### Integration tests — service lifecycle

```kotlin
// :core-location/src/androidTest/.../service/LocationTrackingServiceIntegrationTest.kt

@RunWith(AndroidJUnit4::class)
class LocationTrackingServiceIntegrationTest {

    @get:Rule
    val serviceRule = ServiceTestRule()

    @Test
    fun `service starts and calls startForeground within timeout`() {
        val intent = LocationTrackingService.startIntent(
            InstrumentationRegistry.getInstrumentation().targetContext
        )
        // ServiceTestRule.startService() verifies the service starts successfully
        serviceRule.startService(intent)
        // If startForeground() is not called in time, the test times out
    }

    @Test
    fun `service stops cleanly on stop intent`() {
        val context = InstrumentationRegistry.getInstrumentation().targetContext
        serviceRule.startService(LocationTrackingService.startIntent(context))
        serviceRule.startService(LocationTrackingService.stopIntent(context))
        // Verify notification is removed — check NotificationManager
    }
}
```

### FGS notification tests

```kotlin
// Verify notification content is correct

@Test
fun `buildInitialNotification has correct channel and priority`() {
    val builder = RideNotificationBuilder(context)
    val notification = builder.buildInitialNotification()

    assertThat(notification.channelId).isEqualTo("location_tracking")
    assertThat(notification.priority).isEqualTo(NotificationCompat.PRIORITY_LOW)
    assertThat(notification.flags and Notification.FLAG_ONGOING_EVENT).isNotEqualTo(0)
}

@Test
fun `buildRideNotification shows ETA when available`() {
    val builder = RideNotificationBuilder(context)
    val ride = testRide(etaSeconds = 180)
    val notification = builder.buildRideNotification(ride)

    assertThat(notification.extras.getString(Notification.EXTRA_TEXT))
        .contains("3")  // 180 seconds = 3 minutes
}
```

### QA test cases

| Test | Steps | Expected |
|---|---|---|
| FGS starts on ride InProgress | Start ride, accept driver, ride begins | Persistent notification visible in shade. Status bar icon visible. |
| FGS continues when app backgrounded | Ride active → press Home → wait 60s | Notification still visible. Location still updating (verify via backend or log). |
| FGS stops on ride Complete | Complete a ride | Notification disappears within 1s of completion. |
| FGS stops on ride Cancel | Cancel an active ride | Same as above. |
| User swipes app from recents | Active ride → swipe from recents | Notification disappears. Service stopped. No crash. |
| Process death recovery | Active ride → ADB kill process → reopen app | App shows InProgress screen. FGS restarts. Notification reappears. |
| Notification tap | Tap notification during ride | App opens to active ride screen. |
| Battery optimisation (if implemented) | Enable battery saver → active ride | Degraded frequency banner shown. Service continues. No crash. |

---

## Common Mistakes

| Mistake | Impact | Correct approach |
|---|---|---|
| Calling async code before `startForeground()` | `ForegroundServiceDidNotStartInTimeException` crash | Call `startForeground()` synchronously at the top of `onStartCommand()`, before any coroutine launch |
| Using `IMPORTANCE_HIGH` for FGS notification channel | Notification makes sound / pops up heads-up on every update | Use `IMPORTANCE_LOW` + `PRIORITY_LOW` |
| `ongoing = false` on notification | User can dismiss notification, killing the service mid-ride | Always `setOngoing(true)` for FGS notifications |
| Starting FGS from a background coroutine | `ForegroundServiceStartNotAllowedException` on Android 12+ | Always start from Activity foreground via SideEffect collection |
| Holding a reference to Activity or ViewModel in the Service | Memory leak, stale reference crashes | Service communicates only via Repository layer |
| `stopSelf()` without `stopForeground()` first | Notification may linger even after service stops | Always call `stopForeground(STOP_FOREGROUND_REMOVE)` before `stopSelf()` |
| Not cancelling `serviceScope` in `onDestroy()` | Coroutines leak after service is destroyed | `serviceScope.cancel()` in `onDestroy()` |
| `START_NOT_STICKY` return value | Service not restarted after system kill — gaps in tracking | Use `START_STICKY` |
| Not handling `intent == null` in `onStartCommand` | Crash on `START_STICKY` restart (intent is null) | Always handle null intent — treat as `ACTION_START` |
| Updating notification with `IMPORTANCE_HIGH` channel | Can't downgrade channel importance after creation | Create the channel once with the correct importance; uninstall + reinstall to change during dev |

---

*Last updated: <!-- date -->*  
*Owner: Android Tech Lead*  
*Review: Required before any change to `LocationTrackingService` or notification channel*
