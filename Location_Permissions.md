# Location Permissions

> **Owner:** Android Tech Lead  
> **Audience:** Android team, QA, Product Manager  
> **Related:** `FOREGROUND_SERVICE.md` · `ACCESSIBILITY.md` · `STATE_MACHINE.md`  
> **Last reviewed against:** Android 14 (API 34), Play Store policy May 2024

---

## Table of Contents

- [Why This Document Exists](#why-this-document-exists)
- [Permission Overview](#permission-overview)
- [Android Version Behaviour Matrix](#android-version-behaviour-matrix)
- [Step-by-Step Permission Flow](#step-by-step-permission-flow)
  - [Step 0 — Check current state](#step-0--check-current-state)
  - [Step 1 — Request foreground location](#step-1--request-foreground-location)
  - [Step 2 — Request background location](#step-2--request-background-location)
  - [Step 3 — Permanent denial recovery](#step-3--permanent-denial-recovery)
- [Permission State Machine](#permission-state-machine)
- [UI Rationale Strings](#ui-rationale-strings)
- [Kotlin Implementation](#kotlin-implementation)
  - [LocationPermissionState](#locationpermissionstate)
  - [LocationPermissionManager](#locationpermissionmanager)
  - [PermissionViewModel integration](#permissionviewmodel-integration)
  - [Composable screens](#composable-screens)
- [Manifest declarations](#manifest-declarations)
- [Play Store compliance](#play-store-compliance)
- [QA test cases](#qa-test-cases)
- [Common mistakes](#common-mistakes)

---

## Why This Document Exists

Location permissions on Android are the single most common cause of:

- **Play Store rejections** — background location requires explicit policy justification
- **User trust loss** — asking for background location too early, or without explanation
- **ANR / crash** — starting a Foreground Service before foreground permission is granted
- **Broken tracking** — assuming background permission when the user only granted foreground

The rules changed significantly across Android 10 → 11 → 12 → 13 → 14.  
This document defines the canonical flow for this app and must be followed exactly.  
Do not implement permission logic outside `LocationPermissionManager`.

**Assumption:** this app requires background location only during an active ride (`InProgress` state).  
It does not track location when the app is idle or between rides.  
This assumption directly shapes the rationale UI and Play Store justification.

---

## Permission Overview

| Permission | API level introduced | What it grants | How requested |
|---|---|---|---|
| `ACCESS_COARSE_LOCATION` | 1 | ~100m accuracy (cell/WiFi) | Runtime dialog |
| `ACCESS_FINE_LOCATION` | 1 | GPS-level accuracy | Runtime dialog |
| `ACCESS_BACKGROUND_LOCATION` | 29 (Android 10) | Location when app not in foreground | Runtime dialog (10) → Settings (11+) |
| `FOREGROUND_SERVICE` | 28 | Allows `startForegroundService()` | Manifest only, auto-granted |
| `FOREGROUND_SERVICE_LOCATION` | 34 (Android 14) | Foreground Service with location type | Manifest only, auto-granted |
| `POST_NOTIFICATIONS` | 33 (Android 13) | Show the FGS persistent notification | Runtime dialog |

**This app declares and uses:** `ACCESS_FINE_LOCATION` + `ACCESS_BACKGROUND_LOCATION`  
`ACCESS_COARSE_LOCATION` is declared as a fallback but the app degrades gracefully —  
coarse-only means no accurate pickup pin, not a broken app.

---

## Android Version Behaviour Matrix

| Android version | API | Foreground location | Background location | Notes |
|---|---|---|---|---|
| 9 (Pie) | 28 | Single dialog, always allowed | Not a separate permission — foreground = background | Minimum supported SDK |
| 10 (Q) | 29 | Standard runtime dialog | Separate `ACCESS_BACKGROUND_LOCATION` permission, shown in same dialog | "Allow all the time" option in dialog |
| 11 (R) | 30 | Standard runtime dialog | **Cannot be requested in dialog** — must direct user to Settings | Requesting background in same dialog is silently ignored |
| 12 (S) | 31–32 | Standard runtime dialog | Settings only. `shouldShowRequestPermissionRationale` still works | New approximate location option |
| 12L (Sv2) | 32 | Same as 12 | Same as 12 | |
| 13 (T) | 33 | Standard runtime dialog | Settings only | `POST_NOTIFICATIONS` required for FGS notification |
| 14 (U) | 34 | Standard runtime dialog | Settings only. Google Play **requires** declaration of background location use case | `FOREGROUND_SERVICE_LOCATION` required in manifest. Stricter Play review. |

**Key rule for Android 11+:** you cannot show a system dialog for background location.  
Calling `requestPermissions(ACCESS_BACKGROUND_LOCATION)` does nothing.  
You must explain why, then call `startActivity(Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS))`.

---

## Step-by-Step Permission Flow

This is the canonical flow. Every deviation must be discussed with the tech lead and documented as an ADR.

```
App launch / ride search initiated
           │
           ▼
   ┌───────────────────────────────────┐
   │  Step 0: Check current state      │
   │  LocationPermissionManager        │
   │  .currentState()                  │
   └───────────────┬───────────────────┘
                   │
       ┌───────────┴────────────┐
       │                        │
       ▼                        ▼
  GRANTED_ALL            NOT_GRANTED / FOREGROUND_ONLY / DENIED
  (foreground +               │
   background)                ▼
       │               ┌──────────────────────────────┐
       │               │  Step 1: Foreground location  │
       ▼               └──────────────────────────────┘
  Proceed to                   │
  start FGS          ┌─────────┴──────────┐
                     │                    │
                     ▼                    ▼
              shouldShowRationale    shouldShowRationale
                  == true                == false
                     │                    │
                     ▼                    ▼
            Show in-app            Request system
            rationale UI           dialog directly
            (RationaleDialog)           │
                     │                  │
                     ▼                  │
            User reads rationale        │
            taps "Continue"    ◄────────┘
                     │
                     ▼
          ┌─────────────────────┐
          │  System permission  │
          │  dialog appears     │
          └──────┬──────────────┘
                 │
     ┌───────────┼───────────────┐
     │           │               │
     ▼           ▼               ▼
  GRANTED    DENIED           DENIED
  (fine)    (first time)   (permanently /
     │           │           "Don't ask again")
     │           ▼               │
     │    Show gentle        Show settings
     │    nudge, allow       deeplink dialog
     │    search with            │
     │    reduced accuracy       ▼
     │                   openAppSettings()
     │                   User manually grants
     │                           │
     ▼                           ▼
Step 2: Background location  (loop back to Step 0)
(only during InProgress, not at search time)
     │
     ▼
Show background rationale screen
(full screen, not a dialog)
     │
     ▼
User taps "Grant in Settings"
     │
     ▼
startActivity(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
     │
     ▼
User sets "Allow all the time"
App resumes via onResume / LifecycleObserver
     │
     ▼
Re-check permission state
     │
   ┌─┴───────────┐
   ▼             ▼
GRANTED_ALL   Still FOREGROUND_ONLY
   │               │
   ▼               ▼
Start FGS     Inform user: tracking
              will pause when
              app is backgrounded.
              Allow ride to continue
              with degraded tracking.
```

---

### Step 0 — Check current state

Always check before doing anything. Never assume the state from the previous check.

```kotlin
val state = locationPermissionManager.currentState()
when (state) {
    LocationPermissionState.GrantedAll          -> proceedWithFullTracking()
    LocationPermissionState.ForegroundOnly      -> proceedWithForegroundOnly()
    LocationPermissionState.NotGranted          -> requestForeground()
    LocationPermissionState.PermanentlyDenied   -> showSettingsDeeplink()
    LocationPermissionState.Unknown             -> requestForeground()
}
```

**When to check:**
- On `SearchStarted` intent received by ViewModel
- In `onResume` / `ON_RESUME` lifecycle event after returning from Settings
- Before starting `LocationTrackingService`
- After the system permission dialog callback

---

### Step 1 — Request foreground location

**Trigger:** user taps "Request ride" for the first time, or state is `NotGranted`.

**Rule:** always check `shouldShowRequestPermissionRationale` before launching the system dialog.  
If true — show your own rationale UI first. If false — request directly.

`shouldShowRequestPermissionRationale` returns:
- `false` on first ever request (never asked before) → request directly
- `true` after first denial (user said "Deny" once) → show rationale
- `false` after permanent denial ("Don't ask again") → go to settings

This is the only reliable signal the OS gives you to distinguish "first ask" from "user denied once" from "permanently denied". Do not try to track this yourself with DataStore — the OS tracks it.

**Rationale dialog content:** see [UI Rationale Strings](#ui-rationale-strings) — `FOREGROUND_RATIONALE`.

After the system dialog, you receive the result in `ActivityResultLauncher<Array<String>>`.

---

### Step 2 — Request background location

**Trigger:** ride transitions to `InProgress` for the first time, foreground is already granted.

**Critical rule:** do NOT request background location at app launch or at search time.  
Google's policy explicitly requires that background location be requested only in context —  
when the user understands why it is needed. For this app: when a ride is active.

Requesting background location early is a Play Store rejection reason.

**On Android 10 (API 29) only:**  
You can include `ACCESS_BACKGROUND_LOCATION` in the same `requestPermissions` call.  
The dialog will show an "Allow all the time" option alongside "While using the app".

**On Android 11+ (API 30+):**  
Show a full-screen rationale screen (not a dialog — it needs room to explain).  
The screen has a single CTA: "Open Settings".  
Call `startActivity(Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS))`.  
Listen for the result via `LifecycleEventObserver` on `ON_RESUME`.

**Degraded mode:** if the user declines background location:
- Foreground-only tracking still works while the app is visible
- When the app is backgrounded during a ride, tracking pauses
- Show a persistent banner: "Location tracking paused. Keep the app open for accurate tracking."
- Do NOT block the ride — allow it to continue with degraded accuracy
- Log the event for telemetry (`analytics.track("background_location_declined")`)

---

### Step 3 — Permanent denial recovery

**Detection:** `checkSelfPermission` returns `PERMISSION_DENIED` AND `shouldShowRequestPermissionRationale` returns `false`.  
This means the user chose "Don't ask again" or denied the permission twice on some OEMs.

**What to do:**

1. Show a non-blocking dialog explaining the impact
2. Offer two options: "Open Settings" and "Continue without full tracking"
3. If "Open Settings" → `Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)` with the app's package URI
4. On `ON_RESUME` — re-check the permission state
5. If still denied — respect it. Do not ask again in this session.

**What NOT to do:**
- Do not call `requestPermissions` again — it will be silently ignored
- Do not show a dialog on every screen the user visits
- Do not block core app functionality (searching for a ride) behind background location

---

## Permission State Machine

```kotlin
sealed interface LocationPermissionState {
    /** Never asked, or state is unknown (e.g. first install). */
    data object Unknown : LocationPermissionState

    /** ACCESS_FINE_LOCATION not granted. */
    data object NotGranted : LocationPermissionState

    /**
     * ACCESS_FINE_LOCATION granted.
     * ACCESS_BACKGROUND_LOCATION not granted (or API < 29 where it doesn't exist).
     * Tracking works while app is in foreground only.
     */
    data object ForegroundOnly : LocationPermissionState

    /**
     * Both ACCESS_FINE_LOCATION and ACCESS_BACKGROUND_LOCATION granted.
     * Full background tracking available. FGS can run uninterrupted.
     */
    data object GrantedAll : LocationPermissionState

    /**
     * User selected "Don't ask again" or denied twice (OEM behaviour varies).
     * Cannot request via dialog — must redirect to Settings.
     */
    data object PermanentlyDenied : LocationPermissionState
}
```

Transitions:

| From | Event | To |
|---|---|---|
| `Unknown` | First check | `NotGranted` or `GrantedAll` (restored install) |
| `NotGranted` | Permission granted | `ForegroundOnly` |
| `NotGranted` | Denied with "don't ask again" | `PermanentlyDenied` |
| `ForegroundOnly` | Background granted via Settings | `GrantedAll` |
| `ForegroundOnly` | App revokes via Settings | `NotGranted` |
| `PermanentlyDenied` | User grants via Settings | `ForegroundOnly` or `GrantedAll` |
| `GrantedAll` | User revokes in Settings | `ForegroundOnly` or `NotGranted` |

**Note:** `GrantedAll` → `NotGranted` is possible at any time — the user can revoke permissions  
in Settings while the app is backgrounded. Always re-check in `onResume`.

---

## UI Rationale Strings

These are the canonical English strings. Translations go through the standard localisation process.  
Do not modify rationale copy without product sign-off — these directly affect Play Store review.

### `FOREGROUND_RATIONALE`
**When:** before showing the foreground location system dialog (after first denial).  
**Format:** modal bottom sheet or dialog.

> **"Location access needed"**
>
> To find drivers near you and show your position on the map, this app needs access to your location while you're using it.
>
> Your location is only used during an active ride and is never shared without your consent.
>
> [**Continue**]  [Not now]

---

### `BACKGROUND_RATIONALE`
**When:** before directing to Settings for background location (during `InProgress`).  
**Format:** full-screen dedicated screen — not a dialog. Users need space to read this.

> **"Keep tracking while the app is in the background"**
>
> Your ride is active. To accurately track your journey and keep you safe, the app needs to access your location even when you switch to another app or lock your screen.
>
> **What we do with your location:**
> - Track your route for safety and support
> - Update your ETA in real time
> - Detect when your ride is complete
>
> **What we don't do:**
> - Track your location outside of active rides
> - Share your location with third parties without consent
>
> You'll be taken to your phone's Settings. Under "Location", select **"Allow all the time"**.
>
> [**Open Settings**]  [Continue with limited tracking]

---

### `PERMANENT_DENIAL_RATIONALE`
**When:** permanent denial detected.  
**Format:** dialog or bottom sheet.

> **"Location access is blocked"**
>
> You've previously denied location access. To enable full ride tracking, please open Settings and set location to "Allow all the time".
>
> Without location access, we can't show your position on the map or accurately track your route.
>
> [**Open Settings**]  [Continue anyway]

---

### `DEGRADED_TRACKING_BANNER`
**When:** `ForegroundOnly` state during an active ride, app is visible.  
**Format:** persistent banner at top of ride screen.

> ⚠️ Location tracking paused when app is in background. Keep the app open for accurate tracking.

---

## Kotlin Implementation

### LocationPermissionState

```kotlin
// :core-location/src/main/kotlin/.../permission/LocationPermissionState.kt

sealed interface LocationPermissionState {
    data object Unknown : LocationPermissionState
    data object NotGranted : LocationPermissionState
    data object ForegroundOnly : LocationPermissionState
    data object GrantedAll : LocationPermissionState
    data object PermanentlyDenied : LocationPermissionState
}
```

---

### LocationPermissionManager

```kotlin
// :core-location/src/main/kotlin/.../permission/LocationPermissionManager.kt

import android.Manifest
import android.content.Context
import android.content.pm.PackageManager
import android.os.Build
import androidx.core.content.ContextCompat
import dagger.hilt.android.qualifiers.ApplicationContext
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class LocationPermissionManager @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    /**
     * Returns the current permission state by checking the OS at call time.
     * Never cache this — the user can revoke permissions from Settings at any moment.
     */
    fun currentState(): LocationPermissionState {
        val fineGranted = isGranted(Manifest.permission.ACCESS_FINE_LOCATION)
        val coarseGranted = isGranted(Manifest.permission.ACCESS_COARSE_LOCATION)

        if (!fineGranted && !coarseGranted) return LocationPermissionState.NotGranted

        // On API < 29 there is no background permission — foreground is sufficient
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) {
            return LocationPermissionState.GrantedAll
        }

        val backgroundGranted = isGranted(Manifest.permission.ACCESS_BACKGROUND_LOCATION)
        return if (backgroundGranted) {
            LocationPermissionState.GrantedAll
        } else {
            LocationPermissionState.ForegroundOnly
        }
    }

    /**
     * Returns true if we should explain why we need the permission before
     * showing the system dialog.
     *
     * True only when the user has denied once but not permanently.
     * This is the OS-managed signal — do not replicate it with DataStore.
     */
    fun shouldShowForegroundRationale(activity: androidx.activity.ComponentActivity): Boolean =
        activity.shouldShowRequestPermissionRationale(Manifest.permission.ACCESS_FINE_LOCATION)

    /**
     * Permanent denial: permission denied AND the OS says we should NOT show rationale.
     * Means the user chose "Don't ask again" or denied twice on some OEMs.
     *
     * Only meaningful AFTER we have already attempted a request in this session.
     * Do not call this on first launch — it will return true before any request.
     *
     * [hasRequestedInThisSession] must be tracked by the caller (ViewModel / DataStore).
     */
    fun isPermanentlyDenied(
        activity: androidx.activity.ComponentActivity,
        hasRequestedInThisSession: Boolean,
    ): Boolean {
        if (!hasRequestedInThisSession) return false
        val denied = !isGranted(Manifest.permission.ACCESS_FINE_LOCATION)
        val noRationale = !activity.shouldShowRequestPermissionRationale(
            Manifest.permission.ACCESS_FINE_LOCATION
        )
        return denied && noRationale
    }

    private fun isGranted(permission: String): Boolean =
        ContextCompat.checkSelfPermission(context, permission) ==
            PackageManager.PERMISSION_GRANTED
}
```

---

### PermissionViewModel integration

```kotlin
// :feature-ride/src/main/kotlin/.../permission/LocationPermissionViewModel.kt

@HiltViewModel
class LocationPermissionViewModel @Inject constructor(
    private val permissionManager: LocationPermissionManager,
    private val permissionStateStore: PermissionStateStore,  // DataStore wrapper
) : ViewModel(), ContainerHost<LocationPermissionUiState, LocationPermissionSideEffect> {

    override val container = container<LocationPermissionUiState, LocationPermissionSideEffect>(
        LocationPermissionUiState()
    )

    fun onResume(activity: ComponentActivity) = intent {
        val state = permissionManager.currentState()
        val hasRequested = permissionStateStore.hasRequestedForeground()
        val permanentlyDenied = permissionManager.isPermanentlyDenied(activity, hasRequested)

        reduce {
            state.copy(
                permissionState = if (permanentlyDenied) {
                    LocationPermissionState.PermanentlyDenied
                } else {
                    state
                },
            )
        }
    }

    fun onRequestForeground(activity: ComponentActivity) = intent {
        val shouldShowRationale = permissionManager.shouldShowForegroundRationale(activity)
        if (shouldShowRationale) {
            postSideEffect(LocationPermissionSideEffect.ShowForegroundRationale)
        } else {
            postSideEffect(LocationPermissionSideEffect.RequestForegroundPermission)
        }
    }

    fun onForegroundPermissionResult(granted: Boolean, activity: ComponentActivity) = intent {
        permissionStateStore.markForegroundRequested()

        if (granted) {
            reduce { state.copy(permissionState = LocationPermissionState.ForegroundOnly) }
            postSideEffect(LocationPermissionSideEffect.PermissionGranted)
        } else {
            val permanentlyDenied = permissionManager.isPermanentlyDenied(
                activity, hasRequestedInThisSession = true
            )
            reduce {
                state.copy(
                    permissionState = if (permanentlyDenied) {
                        LocationPermissionState.PermanentlyDenied
                    } else {
                        LocationPermissionState.NotGranted
                    }
                )
            }
        }
    }

    fun onRequestBackground() = intent {
        // Only valid on API 29. On API 30+ we always go to settings.
        if (Build.VERSION.SDK_INT == Build.VERSION_CODES.Q) {
            postSideEffect(LocationPermissionSideEffect.RequestBackgroundPermissionApi29)
        } else {
            postSideEffect(LocationPermissionSideEffect.ShowBackgroundRationale)
        }
    }

    fun onOpenSettings() = intent {
        postSideEffect(LocationPermissionSideEffect.OpenAppSettings)
    }
}

// ── Side effects ──────────────────────────────────────────────────────────────

sealed interface LocationPermissionSideEffect {
    data object ShowForegroundRationale : LocationPermissionSideEffect
    data object RequestForegroundPermission : LocationPermissionSideEffect
    data object ShowBackgroundRationale : LocationPermissionSideEffect
    data object RequestBackgroundPermissionApi29 : LocationPermissionSideEffect
    data object OpenAppSettings : LocationPermissionSideEffect
    data object PermissionGranted : LocationPermissionSideEffect
}
```

---

### Composable screens

```kotlin
// :feature-ride/src/main/kotlin/.../permission/LocationPermissionScreen.kt

@Composable
fun LocationPermissionScreen(
    viewModel: LocationPermissionViewModel = hiltViewModel(),
    onPermissionGranted: () -> Unit,
) {
    val activity = LocalContext.current as ComponentActivity
    val state by viewModel.collectAsState()

    // Re-check every time we come back from Settings
    LifecycleEventEffect(Lifecycle.Event.ON_RESUME) {
        viewModel.onResume(activity)
    }

    // Foreground permission launcher
    val foregroundLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { results ->
        val granted = results[Manifest.permission.ACCESS_FINE_LOCATION] == true
        viewModel.onForegroundPermissionResult(granted, activity)
    }

    // Background permission launcher (API 29 only)
    val backgroundLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        // Result handled in onResume re-check
    }

    viewModel.collectSideEffect { effect ->
        when (effect) {
            LocationPermissionSideEffect.RequestForegroundPermission ->
                foregroundLauncher.launch(
                    arrayOf(
                        Manifest.permission.ACCESS_FINE_LOCATION,
                        Manifest.permission.ACCESS_COARSE_LOCATION,
                    )
                )

            LocationPermissionSideEffect.RequestBackgroundPermissionApi29 ->
                backgroundLauncher.launch(Manifest.permission.ACCESS_BACKGROUND_LOCATION)

            LocationPermissionSideEffect.OpenAppSettings ->
                activity.openAppSettings()

            LocationPermissionSideEffect.PermissionGranted ->
                onPermissionGranted()

            else -> Unit // dialogs handled by state rendering below
        }
    }

    when (state.permissionState) {
        LocationPermissionState.NotGranted ->
            ForegroundPermissionExplainerScreen(
                onContinue = { viewModel.onRequestForeground(activity) }
            )

        LocationPermissionState.ForegroundOnly ->
            BackgroundPermissionRationaleScreen(
                onOpenSettings = { viewModel.onOpenSettings() },
                onSkip = onPermissionGranted,  // degraded tracking
            )

        LocationPermissionState.PermanentlyDenied ->
            PermanentDenialScreen(
                onOpenSettings = { viewModel.onOpenSettings() },
                onContinue = onPermissionGranted,
            )

        LocationPermissionState.GrantedAll ->
            LaunchedEffect(Unit) { onPermissionGranted() }

        else -> Unit
    }
}

// ── BackgroundPermissionRationaleScreen ────────────────────────────────────────

@Composable
private fun BackgroundPermissionRationaleScreen(
    onOpenSettings: () -> Unit,
    onSkip: () -> Unit,
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(horizontal = 24.dp, vertical = 32.dp)
            .semantics { contentDescription = "Background location permission explanation" },
        verticalArrangement = Arrangement.SpaceBetween,
    ) {
        Column(verticalArrangement = Arrangement.spacedBy(16.dp)) {
            Text(
                text = "Keep tracking while the app is in the background",
                style = MaterialTheme.typography.headlineMedium,
                modifier = Modifier.semantics { heading() },
            )
            Text(
                text = "Your ride is active. To accurately track your journey and keep you safe, " +
                    "the app needs location access even when you switch apps or lock your screen.",
                style = MaterialTheme.typography.bodyLarge,
            )
            // Bullet points with semantic grouping
            Column(
                modifier = Modifier.semantics(mergeDescendants = true) {
                    contentDescription = "What we do with your location: " +
                        "Track your route for safety. Update your ETA. Detect when your ride is complete."
                }
            ) {
                RationalePoint("Track your route for safety and support")
                RationalePoint("Update your ETA in real time")
                RationalePoint("Detect when your ride is complete")
            }
            Text(
                text = "In Settings, under "Location", select Allow all the time.",
                style = MaterialTheme.typography.bodyMedium,
                modifier = Modifier.semantics {
                    stateDescription = "Instruction: set location to Allow all the time in Settings"
                },
            )
        }

        Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
            Button(
                onClick = onOpenSettings,
                modifier = Modifier
                    .fillMaxWidth()
                    .minimumInteractiveComponentSize()
                    .semantics {
                        role = Role.Button
                        contentDescription = "Open Settings to grant background location access"
                    },
            ) {
                Text("Open Settings")
            }
            TextButton(
                onClick = onSkip,
                modifier = Modifier
                    .fillMaxWidth()
                    .semantics {
                        role = Role.Button
                        contentDescription = "Continue with limited tracking. Location pauses when app is backgrounded."
                    },
            ) {
                Text("Continue with limited tracking")
            }
        }
    }
}

// ── Helper extension ───────────────────────────────────────────────────────────

fun Context.openAppSettings() {
    startActivity(
        Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
            data = Uri.fromParts("package", packageName, null)
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        }
    )
}
```

---

## Manifest Declarations

```xml
<!-- AndroidManifest.xml -->

<!-- Always declare both fine and coarse -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- Required for background tracking (API 29+).
     Must be accompanied by a Play Store declaration. -->
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />

<!-- Required to call startForegroundService() -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

<!-- Required on Android 14+ for Foreground Services with location type -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />

<!-- Required on Android 13+ to show the FGS persistent notification -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<!-- LocationTrackingService declaration -->
<service
    android:name=".location.LocationTrackingService"
    android:foregroundServiceType="location"
    android:exported="false" />
```

**`android:exported="false"` is mandatory.**  
The service must not be callable from outside the app.

---

## Play Store Compliance

### Background location declaration

Google requires apps that use `ACCESS_BACKGROUND_LOCATION` to:

1. Submit a **Privacy Policy** URL in Play Console
2. Complete the **"Permissions declaration form"** in Play Console → App Content → Sensitive permissions
3. Explain the **specific feature** that requires background location

**Our justification text for the form** (adapt if product changes):

> This app uses background location to track the user's ride route while an active trip is in progress. Location access is initiated only after the user starts a ride and is automatically terminated when the ride completes or is cancelled. Location data is used solely for route tracking, ETA updates, and safety purposes. It is not used for advertising or shared with third parties without explicit consent.

### What triggers Play Store rejection

| Cause | How we avoid it |
|---|---|
| Requesting background location at app launch | We request only on `InProgress` state |
| No foreground use before background request | We always gate on foreground-first |
| No in-app rationale screen before Settings redirect | We show `BackgroundPermissionRationaleScreen` |
| Background permission requested without active feature | State machine ensures request only in `InProgress` |
| Privacy Policy absent or doesn't mention location | Legal to handle — must reference this doc |

### Annual re-review reminder

Play Store reviews background location apps periodically.  
When Google sends a review notification, verify:
- The in-app rationale copy matches the Play Store justification
- The permission is not requested outside `InProgress` state
- The FGS stops when the ride ends

---

## QA Test Cases

### Foreground permission

| Test | Steps | Expected |
|---|---|---|
| First install, grant fine location | Install → open → tap Search → system dialog appears → Allow | App proceeds to search |
| First install, deny once | Install → open → tap Search → Deny | Rationale screen shown on next attempt |
| Deny twice (or "Don't ask again") | Deny twice | Settings deeplink screen shown |
| Grant coarse only (Android 12+) | System dialog → "Use approximate location" | App shows degraded accuracy banner |
| Revoke in Settings while app running | Background → Settings → revoke → return | `onResume` re-check detects, shows permission screen |

### Background permission

| Test | Steps | Expected |
|---|---|---|
| First ride, foreground only | Complete search → match found → rationale screen shown | Rationale screen appears |
| Grant "Allow all the time" | Rationale screen → Open Settings → set "Allow all the time" → return | `GrantedAll` state, FGS starts |
| Decline "Allow all the time" | Rationale screen → Continue with limited tracking | Degraded tracking banner shown on ride screen |
| Background revoked mid-ride | Active ride → Settings → revoke background → return | `onResume` re-check, degraded tracking banner shown, ride continues |
| API 29 specific | Test on Android 10 device/emulator | Background permission dialog in system dialog, not Settings |

### FGS notification

| Test | Steps | Expected |
|---|---|---|
| Android 13+, notification not granted | Cold install, first ride | `POST_NOTIFICATIONS` dialog shown before FGS starts |
| Notification denied | Deny notification permission | FGS can still start (notification permission is for the visible notification, not the service itself), but no notification visible. Log warning. |

---

## Common Mistakes

| Mistake | Impact | Correct approach |
|---|---|---|
| Requesting `ACCESS_BACKGROUND_LOCATION` at the same time as foreground on API 30+ | Silently ignored by OS, user never grants background | Always check `Build.VERSION.SDK_INT` and redirect to Settings on API 30+ |
| Calling `requestPermissions` after "Don't ask again" | Silently ignored, no dialog shown, user confused | Check `isPermanentlyDenied` first, show settings deeplink |
| Checking permission state once at launch and caching it | Misses permission revocations in Settings | Always call `permissionManager.currentState()` in `onResume` |
| Blocking ride search behind background location | Play Store rejection, bad UX | Background is required only for FGS during `InProgress`, not for search |
| Starting FGS without foreground permission | `SecurityException` crash on Android 12+ | State machine ensures FGS only starts from `InProgress` which requires prior permission |
| Using `shouldShowRequestPermissionRationale` before any request | Returns false — looks like permanent denial | Gate `isPermanentlyDenied` behind `hasRequestedInThisSession` flag |
| Showing background rationale as a dialog | Too small, insufficient explanation, Play Store may flag | Use a full-screen rationale screen |
| Not handling `POST_NOTIFICATIONS` on Android 13+ | FGS notification invisible, user can't see active ride | Request notification permission before starting first FGS |

---

*Last updated: <!-- date -->*  
*Owner: Android Tech Lead*  
*Play Store policy reviewed: May 2024 — re-review if policy update notification received*
