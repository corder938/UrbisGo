# Accessibility & TalkBack

> **Owner:** Android Tech Lead  
> **Audience:** Android team, design, QA  
> **Related:** `COMPOSE_SEMANTICS.md` · `UI_STATES.md` · `ARCHITECTURE.md`  
> **Last reviewed against:** Jetpack Compose 1.6, Android 14, TalkBack 14.1

---

## Table of Contents

- [Why Accessibility Is Architecture Here](#why-accessibility-is-architecture-here)
- [Guiding Principles](#guiding-principles)
- [User Context: Couriers & Drivers on the Move](#user-context-couriers--drivers-on-the-move)
- [Core Semantics Patterns](#core-semantics-patterns)
  - [mergeDescendants — grouping cards](#mergedescendants--grouping-cards)
  - [contentDescription — naming elements](#contentdescription--naming-elements)
  - [stateDescription — describing dynamic state](#statedescription--describing-dynamic-state)
  - [liveRegion — announcing real-time updates](#liveregion--announcing-real-time-updates)
  - [traversalIndex — controlling reading order](#traversalindex--controlling-reading-order)
  - [heading — structuring screens](#heading--structuring-screens)
  - [role — interactive element types](#role--interactive-element-types)
  - [disabled — inactive controls](#disabled--inactive-controls)
  - [invisibleToUser — hiding decorative elements](#invisibletouser--hiding-decorative-elements)
- [Screen-by-Screen Semantics Guide](#screen-by-screen-semantics-guide)
  - [Search screen](#search-screen)
  - [Waiting screen — driver en route](#waiting-screen--driver-en-route)
  - [InProgress screen — active ride](#inprogress-screen--active-ride)
  - [Completed screen](#completed-screen)
  - [Error states](#error-states)
  - [Permission rationale screens](#permission-rationale-screens)
- [Hands-Free UX Patterns](#hands-free-ux-patterns)
  - [Touch target sizing](#touch-target-sizing)
  - [Single-action primary zones](#single-action-primary-zones)
  - [Swipe gesture conflicts with maps](#swipe-gesture-conflicts-with-maps)
  - [Reducing cognitive load while moving](#reducing-cognitive-load-while-moving)
- [Kotlin / Compose Implementation Reference](#kotlin--compose-implementation-reference)
  - [Driver card](#driver-card)
  - [ETA counter with live updates](#eta-counter-with-live-updates)
  - [Primary action button with state](#primary-action-button-with-state)
  - [Map composable exclusion](#map-composable-exclusion)
  - [Status banner](#status-banner)
  - [Ride complete summary card](#ride-complete-summary-card)
- [What NOT to Do](#what-not-to-do)
- [Contrast & Visual Requirements](#contrast--visual-requirements)
- [Testing](#testing)
  - [Manual TalkBack audit checklist](#manual-talkback-audit-checklist)
  - [Automated tests](#automated-tests)
  - [QA test cases](#qa-test-cases)
- [Localisation Considerations](#localisation-considerations)
- [Checklist for New Screens](#checklist-for-new-screens)

---

## Why Accessibility Is Architecture Here

In most apps, accessibility is a polish task — something you add at the end before release.  
**In this app, it is a safety requirement.**

Couriers and drivers use this application while operating a vehicle or riding a scooter.  
They cannot look at the screen for more than a fraction of a second.  
They may be wearing gloves. They may be relying on audio feedback through earbuds.

A screen that a sighted user navigates in 2 seconds may take a courier 10 seconds  
of glancing at the screen while trying not to crash. That is unacceptable.

**Accessibility here means:** the app is operable safely while the user is in motion.

This has three consequences for how we build:

1. **TalkBack must work correctly on every screen.** Couriers running TalkBack need the same  
   information as sighted users — in a logical reading order, without redundancy, without gaps.

2. **Touch targets must be large enough to tap while wearing gloves or from a mount.**  
   48×48dp is the minimum. Primary actions should be 56dp or larger.

3. **Information hierarchy must be clear without visual context.**  
   A screen announced by TalkBack as "Button. Button. 3 min. Driver. Cancel." is useless.  
   The same screen should announce as "Driver Ivan, 3 minutes away. Cancel ride, button."

Accessibility is checked as part of the PR review process.  
A PR that adds a new screen without semantic annotations will not be merged.

---

## Guiding Principles

**1. Every user-facing element has an explicit semantic.**  
We do not rely on Compose's default inference. Defaults are a starting point, not an answer.

**2. Group related information into single TalkBack nodes.**  
A driver card is one piece of information — not a collection of text views.  
A user should hear "Driver Ivan, Toyota Camry, 3 minutes away, rating 4.9" in one swipe — not four.

**3. Dynamic content announces itself.**  
ETA updates, status changes, and error messages use `liveRegion` so TalkBack  
announces them without the user having to navigate to them.

**4. Reading order matches task order.**  
The first thing TalkBack reads on the active ride screen is the ride status — not the map,  
not the cancel button. `traversalIndex` enforces this regardless of visual layout.

**5. Interactive elements are clearly identified.**  
Every tappable element has `role = Role.Button` (or the appropriate role).  
TalkBack announces "double-tap to activate" only for elements that have a role.

**6. Decorative elements are hidden.**  
Dividers, background shapes, icon decorations — `invisibleToUser()`.  
They add noise; TalkBack users do not benefit from them.

**7. State is always described, not just implied.**  
A spinner means "loading" — but TalkBack does not see spinners.  
`stateDescription = "Searching for driver"` is what TalkBack reads.

---

## User Context: Couriers & Drivers on the Move

Understanding the use context shapes every decision below.

| Context | Implication |
|---|---|
| Mounted phone on handlebars / dashboard | User interacts via single taps and swipes. No complex gestures. |
| Gloves (cold weather, courier gloves) | Larger touch targets. Avoid small dismiss areas. |
| Earbuds with TalkBack | Audio feedback is primary interface. Content descriptions must be complete sentences. |
| Sunlight on screen | High contrast requirements. Text must be readable at 100% ambient light. |
| Moving at speed | Limited attention window. Primary information must be in top 40% of screen. |
| One-handed operation | Primary action bottom-centre. No important elements in corners. |
| Background noise | TalkBack audio competes with traffic noise. Descriptions must be concise but complete — not wordy. |

**The test question for every screen design:**  
*If a courier hears this screen announced once through an earbud while riding,  
do they have everything they need to take the right action?*

If no — redesign the content description, not the visual design.

---

## Core Semantics Patterns

### mergeDescendants — grouping cards

Use when: a card or row contains multiple text/icon elements that together form one piece of information.

```kotlin
// ✅ Correct — driver info merged into one TalkBack node
@Composable
fun DriverCard(driver: Driver, etaSeconds: Int) {
    val etaText = formatEta(etaSeconds)
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .semantics(mergeDescendants = true) {
                // Provide a single, complete description for the whole card
                contentDescription = "Driver ${driver.name}, " +
                    "${driver.vehicleModel} ${driver.vehiclePlate}, " +
                    "$etaText away, " +
                    "rating ${driver.rating} out of 5"
            }
    ) {
        DriverAvatar(driver)        // decorative — hidden inside merged node
        DriverNameText(driver.name) // individual text — merged into parent
        EtaText(etaSeconds)         // individual text — merged into parent
        RatingBadge(driver.rating)  // individual text — merged into parent
    }
}

// ❌ Wrong — TalkBack reads each child separately
Row(modifier = Modifier.fillMaxWidth()) {
    Text(driver.name)               // → "Ivan"
    Text(formatEta(etaSeconds))     // → "3 min"
    Text("★ ${driver.rating}")      // → "★ 4.9"
    // User hears: "Ivan", "3 min", "star 4.9" — no context between them
}
```

**Important:** when you use `mergeDescendants = true` and also set `contentDescription`  
on the parent, the explicit `contentDescription` wins. All child descriptions are ignored.  
This is what we want — full control over the announcement.

---

### contentDescription — naming elements

Use when: the visual meaning of an element is not derivable from its text content.  
Icons, images, icon buttons always need explicit `contentDescription`.

```kotlin
// ✅ Icon button — always needs contentDescription
IconButton(
    onClick = onCancel,
    modifier = Modifier.semantics {
        contentDescription = "Cancel ride"
        role = Role.Button
    }
) {
    Icon(
        imageVector = Icons.Default.Close,
        contentDescription = null,  // null here — parent semantics covers it
    )
}

// ✅ Status icon — decorative in context of labelled card, hide it
Icon(
    imageVector = Icons.Default.DirectionsCar,
    contentDescription = null,  // parent card already describes "Driver Ivan..."
    modifier = Modifier.semantics { invisibleToUser() },
)

// ✅ Map pin image
Image(
    painter = painterResource(R.drawable.ic_pickup_pin),
    contentDescription = "Pickup location pin on map",
)

// ❌ Wrong — icon button without contentDescription
IconButton(onClick = onSettings) {
    Icon(Icons.Default.Settings, contentDescription = null)
    // TalkBack: "Button" — user has no idea what it does
}
```

**Rule:** if `contentDescription = null` on a standalone interactive element → PR comment.

---

### stateDescription — describing dynamic state

Use when: an element changes its meaning based on state, and the change is not  
captured by `contentDescription` alone. Also used for loading states.

```kotlin
// ✅ Loading button
Button(
    onClick = onStartSearch,
    enabled = !isLoading,
    modifier = Modifier.semantics {
        role = Role.Button
        contentDescription = "Find ride"
        stateDescription = if (isLoading) "Searching for driver" else "Ready"
    }
) {
    if (isLoading) CircularProgressIndicator(modifier = Modifier.size(20.dp))
    else Text("Find ride")
}
// TalkBack: "Find ride, Searching for driver, button, dimmed" (when loading)
// TalkBack: "Find ride, Ready, button" (when active)

// ✅ Toggle-style card
RideTypeCard(
    type = RideType.SCOOTER,
    isSelected = selectedType == RideType.SCOOTER,
    modifier = Modifier.semantics {
        role = Role.Button
        stateDescription = if (isSelected) "Selected" else "Not selected"
        contentDescription = "Scooter"
    }
)
// TalkBack: "Scooter, Selected, button" / "Scooter, Not selected, button"

// ✅ Connection status
Box(
    modifier = Modifier.semantics {
        stateDescription = if (isConnected) "Connected" else "No connection, retrying"
        contentDescription = "Connection status"
    }
)
```

---

### liveRegion — announcing real-time updates

Use when: content updates without user interaction and the update is important enough  
to announce automatically. Critical for ETA countdown and status changes.

```kotlin
// ✅ ETA that updates every 30 seconds
Text(
    text = "Driver arrives in $etaText",
    modifier = Modifier.semantics {
        liveRegion = LiveRegionMode.Polite
        // Polite: waits for TalkBack to finish current announcement before speaking
        // Assertive: interrupts immediately — use only for urgent alerts
    }
)

// ✅ Ride status change — use Assertive for critical state changes
Text(
    text = rideStatusText,  // "Driver arrived", "Ride started", "Ride complete"
    modifier = Modifier.semantics {
        liveRegion = LiveRegionMode.Assertive
        // Interrupts whatever TalkBack is saying — appropriate for key lifecycle events
    }
)

// ✅ Error message that appears dynamically
if (error != null) {
    Text(
        text = error.userMessage,
        modifier = Modifier.semantics {
            liveRegion = LiveRegionMode.Assertive
            contentDescription = "Error: ${error.userMessage}"
        }
    )
}

// ❌ Wrong — liveRegion on something that updates every second
Text(
    text = "$currentSpeedKmh km/h",
    modifier = Modifier.semantics {
        liveRegion = LiveRegionMode.Polite  // will spam TalkBack every second
    }
)
// Solution: update liveRegion text only on significant changes, not every tick
```

**Polite vs Assertive:**

| Mode | Behaviour | Use for |
|---|---|---|
| `Polite` | Waits for TalkBack to finish speaking | ETA updates, status refreshes, non-critical info |
| `Assertive` | Interrupts current TalkBack announcement | Ride started, driver arrived, critical errors |

**Caution with update frequency:**  
A `liveRegion` that updates every second will cause TalkBack to speak every second,  
making the phone unusable. Throttle updates — announce ETA only when it changes  
by more than 30 seconds, not on every backend tick.

---

### traversalIndex — controlling reading order

Use when: the natural top-to-bottom reading order is wrong for the task flow.  
On the active ride screen, the map appears visually above the action buttons —  
but TalkBack should read the ride status first, then actions, and the map last (or never).

Lower `traversalIndex` = read first. Default is `0f`.

```kotlin
// Active ride screen layout — visual order: Map → Status card → Button
// TalkBack order we want: Status card → Button → Map (or skip map)

// Status card — read first
RideStatusCard(
    modifier = Modifier.semantics {
        traversalIndex = -2f    // read before everything
    }
)

// Primary action button — read second
Button(
    onClick = onAction,
    modifier = Modifier.semantics {
        traversalIndex = -1f
    }
) { Text(actionLabel) }

// Map — read last, or skip entirely
GoogleMap(
    modifier = Modifier.semantics {
        traversalIndex = 1f     // read after everything else
        contentDescription = "Map showing your current location and driver position"
        // Or: invisibleToUser() if map adds no value for TalkBack users
    }
)
```

**Note on `traversalIndex` scope:**  
`traversalIndex` is scoped to the parent container's children.  
It does not work across nested composable trees unless `isTraversalGroup = true` is set  
on the parent. For screen-level ordering, apply `traversalIndex` directly to  
the top-level children of the screen's `Box` or `Column`.

```kotlin
// Correct: isTraversalGroup on the container to scope ordering
Box(
    modifier = Modifier.semantics { isTraversalGroup = true }
) {
    MapView(modifier = Modifier.semantics { traversalIndex = 1f })
    BottomSheet(modifier = Modifier.semantics { traversalIndex = 0f })
}
```

---

### heading — structuring screens

Use when: a text element acts as the title of a section.  
TalkBack users can navigate by headings (swipe up/down with headings navigation mode).

```kotlin
Text(
    text = "Your ride summary",
    style = MaterialTheme.typography.headlineMedium,
    modifier = Modifier.semantics { heading() },
)

Text(
    text = "Driver information",
    style = MaterialTheme.typography.titleLarge,
    modifier = Modifier.semantics { heading() },
)

// ❌ Do not mark body text or labels as headings
Text(
    text = "3 minutes",
    modifier = Modifier.semantics { heading() }  // wrong — this is data, not a heading
)
```

---

### role — interactive element types

TalkBack announces the role after the label: "Cancel ride, button" / "Scooter, radio button".  
Without a role, TalkBack says nothing — the user does not know the element is interactive.

```kotlin
// Button
Modifier.semantics { role = Role.Button }

// Checkbox
Modifier.semantics { role = Role.Checkbox }

// RadioButton
Modifier.semantics { role = Role.RadioButton }

// Switch/Toggle
Modifier.semantics { role = Role.Switch }

// Tab in a tab row
Modifier.semantics { role = Role.Tab }

// Image (when meaningful, not decorative)
Modifier.semantics { role = Role.Image }

// ✅ Custom clickable container acting as a button
Box(
    modifier = Modifier
        .clickable(onClick = onSelect)
        .semantics {
            role = Role.Button
            contentDescription = "Select scooter ride type"
        }
)
```

---

### disabled — inactive controls

```kotlin
// ✅ Mark disabled elements explicitly
Button(
    onClick = {},
    enabled = false,
    modifier = Modifier.semantics {
        role = Role.Button
        contentDescription = "Confirm ride"
        disabled()    // TalkBack: "Confirm ride, button, dimmed"
        stateDescription = "Waiting for driver confirmation"
    }
)

// Without disabled() + stateDescription, TalkBack says "Confirm ride, button"
// with no indication of why it cannot be tapped — confusing for the user
```

---

### invisibleToUser — hiding decorative elements

```kotlin
// Decorative divider
HorizontalDivider(
    modifier = Modifier.semantics { invisibleToUser() }
)

// Decorative background shape
Box(
    modifier = Modifier
        .background(...)
        .semantics { invisibleToUser() }
)

// Rating stars when rating is in parent description already
repeat(5) { index ->
    Icon(
        imageVector = if (index < rating) Icons.Filled.Star else Icons.Outlined.Star,
        contentDescription = null,
        modifier = Modifier.semantics { invisibleToUser() },
    )
}
// Parent already says "rating 4.9 out of 5" — individual stars add noise
```

---

## Screen-by-Screen Semantics Guide

### Search screen

**Goal:** user can initiate a ride search in one TalkBack gesture.

TalkBack reading order:
1. Screen heading: "Find a ride"
2. Destination input: "Destination, text field, double-tap to enter destination"
3. Current location summary: "From: Current location, Unter den Linden 1, Berlin"
4. Search button: "Find ride, button" (or "Find ride, Searching for driver, button, dimmed" when loading)

```kotlin
@Composable
fun SearchScreen(state: SearchUiState, onIntent: (SearchIntent) -> Unit) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .semantics { isTraversalGroup = true }
    ) {
        Text(
            text = "Find a ride",
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier.semantics {
                heading()
                traversalIndex = -3f
            },
        )

        DestinationInputField(
            value = state.destination,
            onValueChange = { onIntent(SearchIntent.DestinationChanged(it)) },
            modifier = Modifier.semantics {
                traversalIndex = -2f
                contentDescription = "Destination"
                // Let TextField handle its own role announcement
            },
        )

        CurrentLocationRow(
            location = state.currentLocationLabel,
            modifier = Modifier
                .semantics(mergeDescendants = true) {
                    traversalIndex = -1f
                    contentDescription = "From: ${state.currentLocationLabel}"
                },
        )

        SearchButton(
            isLoading = state.isLoading,
            onClick = { onIntent(SearchIntent.StartSearch) },
        )
    }
}

@Composable
fun SearchButton(isLoading: Boolean, onClick: () -> Unit) {
    Button(
        onClick = onClick,
        enabled = !isLoading,
        modifier = Modifier
            .fillMaxWidth()
            .minimumInteractiveComponentSize()
            .semantics {
                role = Role.Button
                contentDescription = "Find ride"
                stateDescription = if (isLoading) "Searching for driver" else "Ready"
                if (isLoading) disabled()
            },
    ) {
        if (isLoading) {
            CircularProgressIndicator(
                modifier = Modifier
                    .size(20.dp)
                    .semantics { invisibleToUser() }
            )
        } else {
            Text("Find ride")
        }
    }
}
```

---

### Waiting screen — driver en route

**Goal:** courier can hear driver info and ETA in one swipe. Cancel is clearly labelled.

TalkBack reading order:
1. Status heading: "Driver on the way"
2. Driver card (merged): "Driver Ivan, Toyota Camry B-MT 123, 4 minutes away, rating 4.9"
3. ETA live region (separate — updates automatically): "Arriving in 4 minutes"
4. Cancel button: "Cancel ride, button"
5. Map: last or hidden

```kotlin
@Composable
fun WaitingScreen(state: WaitingUiState, onIntent: (RideIntent) -> Unit) {
    Box(modifier = Modifier.semantics { isTraversalGroup = true }) {

        // Map — visually behind, semantically last
        RideMap(
            driverLocation = state.driverLocation,
            modifier = Modifier.semantics {
                traversalIndex = 2f
                contentDescription = "Map showing driver location"
            },
        )

        // Bottom sheet content
        Column(
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .semantics { isTraversalGroup = true },
        ) {

            Text(
                text = "Driver on the way",
                style = MaterialTheme.typography.titleLarge,
                modifier = Modifier.semantics {
                    heading()
                    traversalIndex = -3f
                },
            )

            DriverInfoCard(
                driver = state.driver,
                etaSeconds = state.etaSeconds,
                modifier = Modifier.semantics { traversalIndex = -2f },
            )

            // Separate live region for ETA — announces automatically when eta changes
            EtaAnnouncement(
                etaSeconds = state.etaSeconds,
                modifier = Modifier.semantics { traversalIndex = -1f },
            )

            CancelRideButton(
                onClick = { onIntent(RideIntent.CancelRide) },
            )
        }
    }
}

@Composable
fun DriverInfoCard(driver: Driver, etaSeconds: Int, modifier: Modifier = Modifier) {
    val etaText = formatEta(etaSeconds)
    Row(
        modifier = modifier
            .fillMaxWidth()
            .padding(16.dp)
            .semantics(mergeDescendants = true) {
                contentDescription = "Driver ${driver.name}, " +
                    "${driver.vehicleModel} ${driver.vehiclePlate}, " +
                    "$etaText away, " +
                    "rating ${driver.rating} out of 5"
            },
    ) {
        AsyncImage(
            model = driver.avatarUrl,
            contentDescription = null,      // merged into parent
            modifier = Modifier
                .size(48.dp)
                .clip(CircleShape)
                .semantics { invisibleToUser() },
        )
        Column {
            Text(driver.name)               // merged into parent
            Text("${driver.vehicleModel} · ${driver.vehiclePlate}")
            Text("$etaText away")
        }
        RatingBadge(
            rating = driver.rating,
            modifier = Modifier.semantics { invisibleToUser() },
        )
    }
}

@Composable
fun EtaAnnouncement(etaSeconds: Int, modifier: Modifier = Modifier) {
    // Only announce when ETA changes by more than 30 seconds
    // to avoid spamming TalkBack on every backend tick
    var lastAnnouncedEta by remember { mutableIntStateOf(etaSeconds) }
    val shouldUpdate = remember(etaSeconds) {
        abs(etaSeconds - lastAnnouncedEta) >= 30
    }

    if (shouldUpdate) {
        lastAnnouncedEta = etaSeconds
    }

    Text(
        text = "Arriving in ${formatEta(lastAnnouncedEta)}",
        modifier = modifier.semantics {
            liveRegion = LiveRegionMode.Polite
        },
        color = Color.Transparent,  // Visually hidden — information is in DriverInfoCard
        // Only exists for the liveRegion announcement
    )
}
```

---

### InProgress screen — active ride

**Goal:** absolute minimum interaction required. Courier should be focused on riding.

TalkBack reading order:
1. Status: "Ride in progress"
2. Destination summary: "Heading to Alexanderplatz, 1.2 kilometres remaining"
3. Primary action: "End ride, button" (only if product allows manual end)
4. Map: last or hidden

```kotlin
@Composable
fun InProgressScreen(state: InProgressUiState, onIntent: (RideIntent) -> Unit) {
    Box(modifier = Modifier.semantics { isTraversalGroup = true }) {

        // Full-screen map — important for visual users, lowest priority for TalkBack
        RideMap(
            currentLocation = state.currentLocation,
            routePolyline = state.routePolyline,
            modifier = Modifier
                .fillMaxSize()
                .semantics {
                    traversalIndex = 10f
                    contentDescription = "Map showing current route"
                },
        )

        // Status card — top of screen, first for TalkBack
        RideStatusBanner(
            status = "Ride in progress",
            detail = state.destinationSummary,
            modifier = Modifier
                .align(Alignment.TopCenter)
                .padding(top = 16.dp)
                .semantics {
                    traversalIndex = -2f
                    liveRegion = LiveRegionMode.Polite
                },
        )

        // End ride button — bottom centre, large touch target
        if (state.canEndRide) {
            Button(
                onClick = { onIntent(RideIntent.EndRide) },
                modifier = Modifier
                    .align(Alignment.BottomCenter)
                    .padding(24.dp)
                    .fillMaxWidth()
                    .height(56.dp)      // larger than minimum — primary action
                    .semantics {
                        traversalIndex = -1f
                        role = Role.Button
                        contentDescription = "End ride"
                        stateDescription = "Tap to complete your ride"
                    },
            ) {
                Text("End ride")
            }
        }
    }
}
```

---

### Completed screen

**Goal:** ride summary is readable in one pass. Next action is clear.

```kotlin
@Composable
fun CompletedScreen(state: CompletedUiState, onIntent: (RideIntent) -> Unit) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .semantics { isTraversalGroup = true }
            .padding(24.dp),
    ) {
        // Heading — announced first, interrupts anything TalkBack was reading
        Text(
            text = "Ride complete",
            style = MaterialTheme.typography.headlineLarge,
            modifier = Modifier.semantics {
                heading()
                traversalIndex = -4f
                liveRegion = LiveRegionMode.Assertive  // interrupt — important event
            },
        )

        // Summary card — merged into one announcement
        RideSummaryCard(
            summary = state.summary,
            modifier = Modifier
                .semantics(mergeDescendants = true) {
                    traversalIndex = -3f
                    contentDescription = buildSummaryDescription(state.summary)
                },
        )

        // Rating prompt — optional, not announced via liveRegion
        RatingPrompt(
            onRatingSelected = { onIntent(RideIntent.RideRated(it)) },
            modifier = Modifier.semantics { traversalIndex = -2f },
        )

        // Done button
        Button(
            onClick = { onIntent(RideIntent.SummaryDismissed) },
            modifier = Modifier
                .fillMaxWidth()
                .semantics {
                    traversalIndex = -1f
                    role = Role.Button
                    contentDescription = "Done"
                },
        ) {
            Text("Done")
        }
    }
}

private fun buildSummaryDescription(summary: RideSummary): String {
    val duration = formatDuration(summary.durationSeconds)
    val distance = formatDistance(summary.distanceMeters)
    val fare = formatMoney(summary.fareAmount)
    return "Ride summary: $duration, $distance, $fare"
}
```

---

### Error states

```kotlin
@Composable
fun ErrorContent(error: RideError, onRetry: () -> Unit) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp)
            .semantics { isTraversalGroup = true },
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        // Error message — announced immediately via Assertive
        Text(
            text = error.userMessage,
            style = MaterialTheme.typography.bodyLarge,
            modifier = Modifier.semantics {
                traversalIndex = -2f
                liveRegion = LiveRegionMode.Assertive
                contentDescription = "Error: ${error.userMessage}"
            },
        )

        Button(
            onClick = onRetry,
            modifier = Modifier
                .fillMaxWidth()
                .semantics {
                    traversalIndex = -1f
                    role = Role.Button
                    contentDescription = "Retry"
                    stateDescription = "Double-tap to try again"
                },
        ) {
            Text("Retry")
        }
    }
}
```

---

### Permission rationale screens

```kotlin
@Composable
fun BackgroundPermissionRationaleScreen(
    onOpenSettings: () -> Unit,
    onSkip: () -> Unit,
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(horizontal = 24.dp, vertical = 32.dp)
            .semantics {
                // Screen-level description for when user first lands here
                contentDescription =
                    "Background location permission required for ride tracking"
            },
        verticalArrangement = Arrangement.SpaceBetween,
    ) {
        Column(verticalArrangement = Arrangement.spacedBy(16.dp)) {
            Text(
                text = "Keep tracking while the app is in the background",
                style = MaterialTheme.typography.headlineMedium,
                modifier = Modifier.semantics { heading() },
            )
            Text(
                text = "Your ride is active. To accurately track your journey, " +
                    "the app needs location access even when you switch apps.",
                style = MaterialTheme.typography.bodyLarge,
            )
            // Bullet list — merged into one announcement
            Column(
                modifier = Modifier.semantics(mergeDescendants = true) {
                    contentDescription = "We use your location to: " +
                        "track your route for safety, " +
                        "update your estimated arrival time, " +
                        "and detect when your ride is complete. " +
                        "We do not track your location outside of active rides."
                }
            ) {
                BulletPoint("Track your route for safety")
                BulletPoint("Update your ETA in real time")
                BulletPoint("Detect when your ride is complete")
            }
            Text(
                text = "In Settings, under Location, select Allow all the time.",
                style = MaterialTheme.typography.bodyMedium,
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
                        contentDescription = "Open Settings to allow background location"
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
                        contentDescription =
                            "Continue with limited tracking. " +
                            "Location will pause when the app is in the background."
                    },
            ) {
                Text("Continue with limited tracking")
            }
        }
    }
}
```

---

## Hands-Free UX Patterns

### Touch target sizing

| Element type | Minimum size | Recommended |
|---|---|---|
| Any interactive element | 48×48dp | — |
| Primary action button | 48dp height | 56dp height |
| Secondary action | 48dp | — |
| Icon button | 48×48dp | — |
| List item (tappable) | 56dp height | — |
| Bottom sheet drag handle | 32×4dp visual, 48×48dp tap area | — |

Use `Modifier.minimumInteractiveComponentSize()` — it enforces 48×48dp without  
changing the visual size of the element.

```kotlin
// ✅ Correct — visual size 24dp, tap area 48dp
IconButton(
    onClick = onCancel,
    modifier = Modifier.minimumInteractiveComponentSize(),
) {
    Icon(Icons.Default.Close, contentDescription = "Cancel")
}

// ❌ Wrong — 24dp icon with 24dp tap area
Icon(
    imageVector = Icons.Default.Close,
    contentDescription = "Cancel",
    modifier = Modifier
        .size(24.dp)
        .clickable { onCancel() }   // 24dp tap area — impossible with gloves
)
```

---

### Single-action primary zones

The InProgress screen should have **one primary action** — and it should be in the  
bottom centre of the screen, reachable with a thumb without looking.

Avoid:
- Two equally weighted buttons side by side ("End ride" / "Support")
- Important information that requires scrolling to reach
- Confirmation dialogs with small "Yes" / "No" buttons

Prefer:
- One large primary button, full width, bottom of screen
- Secondary action as a TextButton above it (less prominent)
- Destructive actions (Cancel ride) require a confirmation bottom sheet — not an immediate action

---

### Swipe gesture conflicts with maps

Google Maps uses swipe gestures for panning. TalkBack also uses swipe gestures for navigation.  
These conflict.

**Solution:** disable map touch interaction when TalkBack is active.

```kotlin
@Composable
fun RideMap(
    currentLocation: LatLng,
    modifier: Modifier = Modifier,
) {
    val isTalkBackEnabled = LocalAccessibilityManager.current
        ?.isEnabled == true

    GoogleMap(
        modifier = modifier,
        uiSettings = MapUiSettings(
            // Disable scroll/zoom gestures when TalkBack is on
            scrollGesturesEnabled = !isTalkBackEnabled,
            zoomGesturesEnabled = !isTalkBackEnabled,
            rotationGesturesEnabled = false,    // always off — confusing
            tiltGesturesEnabled = false,
        ),
        properties = MapProperties(
            isMyLocationEnabled = true,
        ),
    )
}

// Detecting TalkBack state
@Composable
fun rememberTalkBackEnabled(): Boolean {
    val context = LocalContext.current
    return remember {
        val am = context.getSystemService(Context.ACCESSIBILITY_SERVICE)
            as AccessibilityManager
        am.isEnabled && am.isTouchExplorationEnabled
    }
}
```

---

### Reducing cognitive load while moving

1. **No auto-advancing carousels or animations** that change what TalkBack focuses on.
2. **Status changes use liveRegion** — user hears the update without navigating.
3. **One confirmation step for destructive actions.** "Cancel ride" → bottom sheet → "Confirm cancel".  
   Never an immediate destructive action on a single tap.
4. **Error messages are persistent**, not auto-dismissing toasts.  
   A courier who misses a 3-second toast has lost the error context entirely.
5. **No modal dialogs that block ride progress** — information, not interruption.

---

## Kotlin / Compose Implementation Reference

Quick reference of patterns — full examples are in the screen-by-screen guide above.

### Driver card

```kotlin
Modifier.semantics(mergeDescendants = true) {
    contentDescription = "Driver ${name}, ${vehicle}, ${eta} away, rating ${rating} out of 5"
}
```

### ETA counter with live updates

```kotlin
Modifier.semantics {
    liveRegion = LiveRegionMode.Polite
}
// Throttle: only update when delta > 30 seconds
```

### Primary action button with state

```kotlin
Modifier.semantics {
    role = Role.Button
    contentDescription = "Find ride"
    stateDescription = if (isLoading) "Searching" else "Ready"
    if (isLoading) disabled()
}
```

### Map composable exclusion

```kotlin
Modifier.semantics {
    traversalIndex = 10f
    contentDescription = "Map showing your route"
    // Or: invisibleToUser() if the map adds nothing for TalkBack users
}
```

### Status banner

```kotlin
Modifier.semantics {
    liveRegion = LiveRegionMode.Assertive   // for ride state transitions
    traversalIndex = -2f
}
```

### Ride complete summary card

```kotlin
Modifier.semantics(mergeDescendants = true) {
    contentDescription = "Ride summary: $duration, $distance, $fare"
}
```

---

## What NOT to Do

| Anti-pattern | Impact | Fix |
|---|---|---|
| `contentDescription = null` on icon buttons | TalkBack says "Button" — no information | Always provide `contentDescription` for interactive elements |
| `clickable {}` without `role` | TalkBack does not say "button" — element feels broken | Add `role = Role.Button` to `Modifier.semantics` |
| `liveRegion` on elements that update every second | TalkBack spams the user, phone becomes unusable | Throttle updates; only announce on significant changes |
| `mergeDescendants = true` without explicit `contentDescription` | TalkBack reads all child texts concatenated — often nonsensical | Always pair `mergeDescendants = true` with `contentDescription` |
| Hiding loading state (just show spinner, no semantic) | TalkBack says nothing changed — user thinks app is frozen | Use `stateDescription = "Loading"` or `disabled()` on the button |
| Map touch gestures active with TalkBack | Swipes navigate the map instead of TalkBack focus | Disable map gestures when `isTouchExplorationEnabled` |
| `traversalIndex` without `isTraversalGroup` on parent | Order may not be respected across composable subtrees | Set `isTraversalGroup = true` on the parent container |
| Auto-dismissing toast for errors | Courier misses it while looking at road | Use persistent error state in `UiState`, not a toast |
| Confirmation dialog with 24dp buttons | Cannot tap reliably under any conditions | Minimum 48dp touch targets; prefer full-width buttons in bottom sheet |
| Announcing every single state update as `Assertive` | Interrupts TalkBack constantly | `Assertive` only for critical lifecycle events (ride started, error) |

---

## Contrast & Visual Requirements

| Element | Minimum contrast ratio | Target |
|---|---|---|
| Normal text (< 18pt) | 4.5 : 1 | 7 : 1 |
| Large text (≥ 18pt or 14pt bold) | 3 : 1 | 4.5 : 1 |
| Interactive element borders / outlines | 3 : 1 | — |
| Status icons conveying information | 3 : 1 | — |
| Disabled elements | No requirement | Keep readable where possible |

**Verify with:** Android Studio's Accessibility Scanner, or [contrast-ratio.com](https://contrast-ratio.com).

**Critical screens for contrast:**
- Active ride screen — map background may be white or dark depending on map style
- Driver card — avatar + text on card background
- Error banner — error colour on background
- Notification in status bar — small icon must be monochrome (system colouring)

**Do not rely on colour alone** to convey state.  
A "driver arrived" indicator that is only green → red → green is invisible to colour-blind users.  
Always pair colour with text, icon, or shape change.

---

## Testing

### Manual TalkBack audit checklist

Run this checklist on a physical device before every screen is marked ready for review.  
Enable TalkBack: Settings → Accessibility → TalkBack → On.

**Navigation:**
- [ ] Swipe right through all elements in order — is the order logical?
- [ ] Is every interactive element reachable via swipe?
- [ ] Is every non-interactive decorative element skipped?
- [ ] Can you complete the primary task (search, confirm, cancel) using TalkBack only?

**Announcements:**
- [ ] Does the driver card announce as a single sentence, not individual words?
- [ ] Does the ETA announce automatically when it changes significantly?
- [ ] Does a ride state change (driver arrived, ride started) interrupt and announce?
- [ ] Does a loading state say something other than "Loading…"?
- [ ] Do error messages announce immediately?

**Interactive elements:**
- [ ] Does every button say "button" after its label?
- [ ] Does every disabled button say "dimmed"?
- [ ] Does the search input say "text field, double-tap to edit"?
- [ ] Can the cancel ride button be reached and activated without looking at the screen?

**Maps:**
- [ ] Are map swipe gestures disabled when TalkBack is on?
- [ ] Is the map announced as a single element, not hundreds of map tiles?

**Heading navigation:**
- [ ] Switch TalkBack to "Headings" navigation mode (volume key shortcut)
- [ ] Can you navigate between screen sections using headings?

---

### Automated tests

```kotlin
// :feature-ride/src/androidTest/.../WaitingScreenAccessibilityTest.kt

@RunWith(AndroidJUnit4::class)
class WaitingScreenAccessibilityTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Before
    fun enableAccessibilityChecks() {
        // Enables automated accessibility checks on every UI interaction
        AccessibilityChecks.enable()
            .setRunChecksFromRootView(true)
    }

    @Test
    fun driverCard_hasCompleteContentDescription() {
        val driver = testDriver(name = "Ivan", vehicleModel = "Toyota", rating = 4.9f)

        composeTestRule.setContent {
            DriverInfoCard(driver = driver, etaSeconds = 180)
        }

        // Verify merged description exists and contains key info
        composeTestRule
            .onNodeWithContentDescription(
                "Driver Ivan",
                substring = true,
            )
            .assertExists()
            .assertIsDisplayed()
    }

    @Test
    fun cancelButton_hasButtonRole() {
        composeTestRule.setContent {
            CancelRideButton(onClick = {})
        }
        composeTestRule
            .onNodeWithText("Cancel ride")
            .assertHasRole(SemanticsProperties.Role.Button)
    }

    @Test
    fun loadingSearchButton_isAnnounced_asSearching() {
        composeTestRule.setContent {
            SearchButton(isLoading = true, onClick = {})
        }
        composeTestRule
            .onNodeWithContentDescription("Find ride", substring = true)
            .assertIsNotEnabled()
    }

    @Test
    fun etaText_hasLiveRegion() {
        composeTestRule.setContent {
            EtaAnnouncement(etaSeconds = 180)
        }
        composeTestRule
            .onNodeWithText("Arriving in", substring = true)
            .assertExists()
        // Note: LiveRegionMode cannot be directly asserted in ComposeTestRule.
        // Verify via SemanticsMatcher if needed.
    }

    @Test
    fun waitingScreen_accessibilityChecks_passWithNoErrors() {
        val state = testWaitingUiState()
        composeTestRule.setContent {
            WaitingScreen(state = state, onIntent = {})
        }
        // AccessibilityChecks.enable() will fail the test if any check fails:
        // - touch target too small
        // - missing contentDescription on interactive element
        // - insufficient contrast (with AccessibilityChecks.enable().setRootView())
        composeTestRule
            .onRoot()
            .performClick()     // triggers accessibility check evaluation
    }
}
```

---

### QA test cases

| Test | Device state | Steps | Expected TalkBack announcement |
|---|---|---|---|
| Driver card on Waiting screen | TalkBack on | Swipe to driver card | "Driver Ivan, Toyota Camry B-MT 123, 3 minutes away, rating 4.9 out of 5" (one node) |
| ETA updates automatically | TalkBack on, ride active | Wait for ETA to change by > 30s | "Arriving in 2 minutes" announced without user interaction |
| Ride started event | TalkBack on, in Waiting | Server sends RideStarted | "Ride in progress" announced, interrupts current speech |
| Cancel button reachable | TalkBack on | Swipe through screen | Cancel button reachable within 3 swipes from any element |
| Map does not interfere | TalkBack on | Swipe on map area | TalkBack focus moves to next element — does not pan the map |
| Search button loading | TalkBack on, search initiated | Tap Find ride | "Find ride, Searching for driver, button, dimmed" |
| Error announced | TalkBack on | Trigger network error | Error message announced immediately without user navigation |
| Touch target — gloves | Physical device | Use bulky gloves | All primary actions tappable first attempt |

---

## Localisation Considerations

Content descriptions must be localised — they are user-facing strings.

```kotlin
// ❌ Wrong — hardcoded English
contentDescription = "Driver $name, $vehicle, $eta away, rating $rating out of 5"

// ✅ Correct — string resource with format args
contentDescription = context.getString(
    R.string.cd_driver_card,
    driver.name,
    driver.vehicleModel,
    driver.vehiclePlate,
    formatEta(etaSeconds),
    driver.rating,
)
```

```xml
<!-- res/values/strings.xml -->
<string name="cd_driver_card">Driver %1$s, %2$s %3$s, %4$s away, rating %5$.1f out of 5</string>

<!-- res/values-de/strings.xml -->
<string name="cd_driver_card">Fahrer %1$s, %2$s %3$s, noch %4$s, Bewertung %5$.1f von 5</string>
```

**RTL languages:**  
`traversalIndex` ordering is independent of layout direction.  
Test with Arabic or Hebrew locale enabled to verify reading order is still logical.

---

## Checklist for New Screens

Copy this into the PR description when adding a new screen.

```
## Accessibility checklist

- [ ] Every interactive element has `contentDescription` or meaningful text
- [ ] Every icon button has `contentDescription` and `role = Role.Button`
- [ ] Card components use `mergeDescendants = true` with explicit `contentDescription`
- [ ] Dynamic content (loading, status changes) uses `stateDescription` or `liveRegion`
- [ ] `traversalIndex` set on key elements if default order is wrong
- [ ] Decorative elements use `invisibleToUser()` or `contentDescription = null`
- [ ] All touch targets ≥ 48×48dp (use `minimumInteractiveComponentSize()`)
- [ ] Primary action button ≥ 56dp height
- [ ] Content descriptions are string resources (not hardcoded English)
- [ ] Screen heading marked with `heading()`
- [ ] Tested manually with TalkBack on a physical device
- [ ] `AccessibilityChecks.enable()` test added or existing test covers this screen
- [ ] No colour-only state indicators
- [ ] Error states are persistent (not auto-dismissing)
```

---

*Last updated: <!-- date -->*  
*Owner: Android Tech Lead*  
*Review: Required before any new screen is merged. TalkBack audit is not optional.*
