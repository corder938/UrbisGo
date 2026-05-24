# Data Layer

> **Owner:** Android Tech Lead  
> **Audience:** Android team  
> **Related:** `ARCHITECTURE.md` · `STATE_MACHINE.md` · `OFFLINE_FIRST.md` · `ERROR_HANDLING.md`  
> **Last reviewed against:** Room 2.6, DataStore 1.1, Kotlin Coroutines 1.8, Android 14

---

## Table of Contents

- [Purpose & Boundaries](#purpose--boundaries)
- [Layer Structure](#layer-structure)
- [Repository Pattern](#repository-pattern)
  - [Interface in domain](#interface-in-domain)
  - [Implementation in data](#implementation-in-data)
  - [Result wrapper](#result-wrapper)
- [Room — Local Database](#room--local-database)
  - [Schema overview](#schema-overview)
  - [Database setup](#database-setup)
  - [Entities](#entities)
  - [DAOs](#daos)
  - [Type converters](#type-converters)
  - [Migrations](#migrations)
- [DataStore — Preferences & State](#datastore--preferences--state)
  - [What goes in DataStore vs Room](#what-goes-in-datastore-vs-room)
  - [DataStore setup](#datastore-setup)
  - [Permission flow state](#permission-flow-state)
  - [User preferences](#user-preferences)
- [Network Layer](#network-layer)
  - [Retrofit setup](#retrofit-setup)
  - [OkHttp interceptors](#okhttp-interceptors)
  - [WebSocket client](#websocket-client)
  - [DTO mapping](#dto-mapping)
- [Offline-First Strategy](#offline-first-strategy)
  - [What is offline-first here](#what-is-offline-first-here)
  - [Cache policies by data type](#cache-policies-by-data-type)
  - [Network monitor](#network-monitor)
  - [Stale-while-revalidate](#stale-while-revalidate)
  - [Write-through vs write-back](#write-through-vs-write-back)
  - [Conflict resolution](#conflict-resolution)
- [Sync Strategy](#sync-strategy)
- [Data Flow Diagrams](#data-flow-diagrams)
  - [Read flow — active ride](#read-flow--active-ride)
  - [Write flow — cancel ride](#write-flow--cancel-ride)
  - [Reconnect flow](#reconnect-flow)
- [Dependency Injection — Data Module](#dependency-injection--data-module)
- [Testing the Data Layer](#testing-the-data-layer)
- [Common Mistakes](#common-mistakes)

---

## Purpose & Boundaries

The data layer owns everything related to **where data comes from and how it is stored**.

It does not:
- Know about UI state, ViewModels, or Compose
- Contain business logic (ride state machine lives in domain)
- Make product decisions (which data to show when offline — that's a UseCase decision)

It does:
- Implement `Repository` interfaces defined in the domain layer
- Manage Room database schema, DAOs, and migrations
- Manage DataStore for preferences and lightweight state
- Set up Retrofit, OkHttp, and WebSocket client
- Map DTOs (network models) ↔ Entities (DB models) ↔ Domain models
- Apply caching strategy to every data type
- Handle network errors, DB errors, and return typed `Result<T>`

**Rule:** nothing above the data layer imports `retrofit2`, `androidx.room`, or `okhttp3`.  
These are data-layer concerns only.

---

## Layer Structure

```
core-data/
├── db/
│   ├── AppDatabase.kt              Room database class
│   ├── dao/
│   │   ├── RideDao.kt
│   │   ├── DriverLocationDao.kt
│   │   └── LocationHistoryDao.kt
│   ├── entity/
│   │   ├── RideEntity.kt
│   │   ├── DriverLocationEntity.kt
│   │   └── LocationHistoryEntity.kt
│   ├── converter/
│   │   ├── InstantConverter.kt
│   │   └── LatLngConverter.kt
│   └── migration/
│       ├── Migration_1_2.kt
│       └── Migration_2_3.kt
├── datastore/
│   ├── PermissionStateDataStore.kt
│   └── UserPreferencesDataStore.kt
├── remote/
│   ├── api/
│   │   └── RideApiService.kt
│   ├── websocket/
│   │   └── RideWebSocketClient.kt
│   ├── dto/
│   │   ├── RideDto.kt
│   │   ├── DriverDto.kt
│   │   └── RideServerEventDto.kt
│   └── interceptor/
│       ├── AuthInterceptor.kt
│       └── LoggingInterceptor.kt
├── repository/
│   ├── RideRepositoryImpl.kt
│   └── LocationRepositoryImpl.kt
├── mapper/
│   ├── RideMapper.kt               DTO ↔ Entity ↔ Domain mappers
│   └── LocationMapper.kt
├── network/
│   └── NetworkMonitor.kt
└── di/
    └── DataModule.kt
```

---

## Repository Pattern

### Interface in domain

The domain layer defines **what** data operations are available.  
It knows nothing about how they are implemented.

```kotlin
// :core-domain/src/main/kotlin/.../repository/RideRepository.kt

interface RideRepository {
    /** Start a ride search. Returns the created Ride on success. */
    suspend fun startSearch(origin: LatLng): Result<Ride>

    /** Cancel an active ride. */
    suspend fun cancelRide(rideId: String, reason: CancellationReason): Result<Unit>

    /** Stream of real-time server events for the active ride. */
    fun observeRideEvents(): Flow<RideServerEvent>

    /** Returns the active ride from local cache. Null if no active ride. */
    suspend fun getActiveRide(): Ride?

    /** Persists ride state locally for process death recovery. */
    suspend fun saveRideState(ride: Ride)

    /** Returns ride history from local cache. */
    fun observeRideHistory(): Flow<List<Ride>>

    /** Sync ride history from server — call after reconnect. */
    suspend fun syncRideHistory(): Result<Unit>

    /** Mark active ride as interrupted (called from FGS onTaskRemoved). */
    suspend fun markRideInterrupted(): Result<Unit>
}
```

```kotlin
// :core-domain/src/main/kotlin/.../repository/LocationRepository.kt

interface LocationRepository {
    /** Hot flow of GPS updates from LocationTrackingService. */
    fun observeLocation(): Flow<LatLng>

    /** Observe GPS signal availability. */
    fun observeAvailability(): Flow<Boolean>

    /** Last known location — from Room if available, null otherwise. */
    suspend fun getLastKnownLocation(): LatLng?

    /** Called by LocationTrackingService on each GPS fix. */
    suspend fun emitLocation(latLng: LatLng)

    /** Persists to Room for process death recovery. */
    suspend fun persistLastKnown(latLng: LatLng)

    /** Emit GPS availability change. */
    suspend fun emitAvailabilityChange(available: Boolean)

    /** Flush in-memory buffer to Room. Called before FGS stops. */
    suspend fun flushBuffer()

    /** Returns buffered track points for a ride. Used for route drawing. */
    suspend fun getTrackForRide(rideId: String): List<LatLng>
}
```

---

### Implementation in data

```kotlin
// :core-data/src/main/kotlin/.../repository/RideRepositoryImpl.kt

@Singleton
class RideRepositoryImpl @Inject constructor(
    private val rideApiService: RideApiService,
    private val rideWebSocketClient: RideWebSocketClient,
    private val rideDao: RideDao,
    private val networkMonitor: NetworkMonitor,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,
) : RideRepository {

    override suspend fun startSearch(origin: LatLng): Result<Ride> =
        withContext(ioDispatcher) {
            runCatching {
                val response = rideApiService.startSearch(origin.toRequest())
                val dto = response.bodyOrThrow()                // throws on HTTP error
                val domain = dto.toDomain()
                rideDao.upsert(domain.toEntity())               // cache immediately
                domain
            }.mapFailure { it.toDomainError() }                 // map to typed errors
        }

    override suspend fun cancelRide(rideId: String, reason: CancellationReason): Result<Unit> =
        withContext(ioDispatcher) {
            runCatching {
                rideApiService.cancelRide(rideId, reason.toDto())
                rideDao.updateStatus(rideId, RideStatus.CANCELLED)
            }.mapFailure { it.toDomainError() }
        }

    override fun observeRideEvents(): Flow<RideServerEvent> =
        rideWebSocketClient.observeEvents()
            .map { it.toDomain() }
            .catch { e ->
                emit(RideServerEvent.Error(e.toDomainError()))
            }

    override suspend fun getActiveRide(): Ride? =
        withContext(ioDispatcher) {
            rideDao.getActiveRide()?.toDomain()
        }

    override suspend fun saveRideState(ride: Ride) =
        withContext(ioDispatcher) {
            rideDao.upsert(ride.toEntity())
        }

    override fun observeRideHistory(): Flow<List<Ride>> =
        rideDao.observeHistory()
            .map { entities -> entities.map { it.toDomain() } }
            .flowOn(ioDispatcher)

    override suspend fun syncRideHistory(): Result<Unit> =
        withContext(ioDispatcher) {
            runCatching {
                val rides = rideApiService.getRideHistory().bodyOrThrow()
                rideDao.replaceHistory(rides.map { it.toEntity() })
            }.mapFailure { it.toDomainError() }
        }

    override suspend fun markRideInterrupted(): Result<Unit> =
        withContext(ioDispatcher) {
            runCatching {
                val active = rideDao.getActiveRide() ?: return@runCatching
                rideApiService.markInterrupted(active.id)
            }.mapFailure { it.toDomainError() }
        }
}
```

**Rules for every `RepositoryImpl`:**

- Always `withContext(ioDispatcher)` — never perform IO on the calling coroutine's dispatcher
- Always wrap in `runCatching { }.mapFailure { }` — never let exceptions propagate raw
- Cache the result locally before returning it — network first, then Room
- Inject `CoroutineDispatcher` rather than hardcoding `Dispatchers.IO` — enables test replacement

---

### Result wrapper

We do not throw exceptions for expected error conditions.  
Every repository method that can fail returns `Result<T>`.

```kotlin
// :core-domain/src/main/kotlin/.../util/Result.kt
// Use kotlin.Result — no need for a custom wrapper.

// Extension to map the failure type
inline fun <T> Result<T>.mapFailure(
    transform: (Throwable) -> DomainError,
): Result<T> = this.fold(
    onSuccess = { Result.success(it) },
    onFailure = { Result.failure(transform(it)) },
)

// Extension for HTTP responses
fun <T> Response<T>.bodyOrThrow(): T {
    if (!isSuccessful) {
        throw HttpException(this)
    }
    return body() ?: throw EmptyBodyException()
}

// Mapping network/DB exceptions to typed domain errors
fun Throwable.toDomainError(): DomainError = when (this) {
    is UnknownHostException  -> DomainError.NetworkUnavailable
    is SocketTimeoutException -> DomainError.NetworkTimeout
    is HttpException          -> DomainError.ServerError(code(), message())
    is SQLiteException        -> DomainError.LocalStorageError(message ?: "DB error")
    else                      -> DomainError.Unknown(message ?: "Unknown error")
}
```

---

## Room — Local Database

### Schema overview

```
┌─────────────────┐     ┌──────────────────────┐     ┌───────────────────────┐
│      rides      │     │   driver_locations   │     │   location_history    │
├─────────────────┤     ├──────────────────────┤     ├───────────────────────┤
│ id (PK)         │     │ id (PK)              │     │ id (PK, autoincrement)│
│ status          │     │ ride_id (FK→rides)   │     │ ride_id               │
│ origin_lat      │     │ latitude             │     │ latitude              │
│ origin_lng      │     │ longitude            │     │ longitude             │
│ destination_lat │     │ heading              │     │ recorded_at           │
│ destination_lng │     │ recorded_at          │     │ accuracy_meters       │
│ driver_id       │     └──────────────────────┘     └───────────────────────┘
│ driver_name     │
│ driver_rating   │     ┌──────────────────────┐
│ vehicle_model   │     │  last_known_location │
│ vehicle_plate   │     ├──────────────────────┤
│ eta_seconds     │     │ id (always 1 — single│
│ started_at      │     │    row sentinel)     │
│ completed_at    │     │ latitude             │
│ fare_amount     │     │ longitude            │
│ fare_currency   │     │ recorded_at          │
│ cancellation_   │     └──────────────────────┘
│   reason        │
│ cancelled_by    │
│ route_polyline  │
│ created_at      │
│ updated_at      │
└─────────────────┘
```

**Design decisions:**

- `driver_*` columns are denormalised into `rides` — no separate `drivers` table.  
  Rationale: driver data is only meaningful in the context of a specific ride.  
  A separate table adds joins with no benefit.

- `last_known_location` uses a single-row sentinel (`id = 1`, `REPLACE` conflict strategy).  
  This avoids a growing table for a piece of data we only ever need the latest value of.

- `location_history` accumulates during InProgress. It is cleared after the ride ends  
  and the backend confirms receipt. Maximum 10,000 rows per ride.

- No foreign key enforcement between `rides` and `location_history`.  
  Rationale: FGS may write location history slightly before the ride entity is fully committed.  
  We accept eventual consistency here rather than adding FK constraint timing complexity.

---

### Database setup

```kotlin
// :core-data/src/main/kotlin/.../db/AppDatabase.kt

@Database(
    entities = [
        RideEntity::class,
        DriverLocationEntity::class,
        LocationHistoryEntity::class,
        LastKnownLocationEntity::class,
    ],
    version = 3,                        // increment on every schema change
    exportSchema = true,                // schema JSON exported to schemas/ dir
)
@TypeConverters(
    InstantConverter::class,
    LatLngConverter::class,
    RideStatusConverter::class,
    CancellationReasonConverter::class,
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun rideDao(): RideDao
    abstract fun driverLocationDao(): DriverLocationDao
    abstract fun locationHistoryDao(): LocationHistoryDao
    abstract fun lastKnownLocationDao(): LastKnownLocationDao
}
```

```kotlin
// :core-data/src/main/kotlin/.../di/DataModule.kt  (partial)

@Provides
@Singleton
fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
    Room.databaseBuilder(
        context,
        AppDatabase::class.java,
        "micromobility.db",
    )
        .addMigrations(
            Migration_1_2,
            Migration_2_3,
        )
        .fallbackToDestructiveMigrationOnDowngrade()  // dev only; remove before release
        .build()
```

**`exportSchema = true` is mandatory.**  
The exported schema JSON files go in `app/schemas/` and are committed to version control.  
This is how Room validates migrations and how we track schema history.

Add to `build.gradle.kts`:
```kotlin
android {
    defaultConfig {
        kapt {
            arguments {
                arg("room.schemaLocation", "$projectDir/schemas")
            }
        }
    }
}
```

---

### Entities

```kotlin
// :core-data/src/main/kotlin/.../db/entity/RideEntity.kt

@Entity(tableName = "rides")
data class RideEntity(
    @PrimaryKey val id: String,
    val status: String,                 // stored as string, converted via TypeConverter
    val originLat: Double,
    val originLng: Double,
    val destinationLat: Double?,
    val destinationLng: Double?,
    val driverId: String?,
    val driverName: String?,
    val driverRating: Float?,
    val vehicleModel: String?,
    val vehiclePlate: String?,
    val etaSeconds: Int?,
    val startedAt: Instant?,            // converted via InstantConverter
    val completedAt: Instant?,
    val fareAmount: Long?,              // stored as minor units (cents)
    val fareCurrency: String?,
    val cancellationReason: String?,
    val cancelledBy: String?,
    val routePolyline: String?,         // encoded polyline string
    val createdAt: Instant,
    val updatedAt: Instant,
)
```

```kotlin
// :core-data/src/main/kotlin/.../db/entity/LocationHistoryEntity.kt

@Entity(
    tableName = "location_history",
    indices = [Index(value = ["ride_id", "recorded_at"])],
)
data class LocationHistoryEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val rideId: String,
    val latitude: Double,
    val longitude: Double,
    val recordedAt: Instant,
    val accuracyMeters: Float?,
)
```

```kotlin
// :core-data/src/main/kotlin/.../db/entity/LastKnownLocationEntity.kt

@Entity(tableName = "last_known_location")
data class LastKnownLocationEntity(
    @PrimaryKey val id: Int = 1,       // sentinel — always a single row
    val latitude: Double,
    val longitude: Double,
    val recordedAt: Instant,
)
```

---

### DAOs

```kotlin
// :core-data/src/main/kotlin/.../db/dao/RideDao.kt

@Dao
interface RideDao {

    // ── Queries ──────────────────────────────────────────────────────────────

    @Query("""
        SELECT * FROM rides
        WHERE status NOT IN ('COMPLETED', 'CANCELLED')
        ORDER BY created_at DESC
        LIMIT 1
    """)
    suspend fun getActiveRide(): RideEntity?

    @Query("""
        SELECT * FROM rides
        WHERE status NOT IN ('COMPLETED', 'CANCELLED')
        ORDER BY created_at DESC
        LIMIT 1
    """)
    fun observeActiveRide(): Flow<RideEntity?>
    // Flow-returning DAOs auto-update when the underlying data changes.
    // Use for ViewModels that need to react to external writes (e.g. from FGS).

    @Query("SELECT * FROM rides ORDER BY created_at DESC")
    fun observeHistory(): Flow<List<RideEntity>>

    @Query("SELECT * FROM rides WHERE id = :rideId")
    suspend fun getById(rideId: String): RideEntity?

    // ── Mutations ─────────────────────────────────────────────────────────────

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsert(ride: RideEntity)

    @Query("UPDATE rides SET status = :status, updated_at = :updatedAt WHERE id = :rideId")
    suspend fun updateStatus(rideId: String, status: String, updatedAt: Instant = Instant.now())

    @Query("UPDATE rides SET eta_seconds = :etaSeconds, updated_at = :updatedAt WHERE id = :rideId")
    suspend fun updateEta(rideId: String, etaSeconds: Int, updatedAt: Instant = Instant.now())

    @Transaction
    suspend fun replaceHistory(rides: List<RideEntity>) {
        deleteHistory()
        insertAll(rides)
    }

    @Query("DELETE FROM rides WHERE status IN ('COMPLETED', 'CANCELLED')")
    suspend fun deleteHistory()

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(rides: List<RideEntity>)

    @Query("DELETE FROM rides WHERE id = :rideId")
    suspend fun deleteById(rideId: String)
}
```

```kotlin
// :core-data/src/main/kotlin/.../db/dao/LocationHistoryDao.kt

@Dao
interface LocationHistoryDao {

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insert(location: LocationHistoryEntity)

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertAll(locations: List<LocationHistoryEntity>)

    @Query("SELECT * FROM location_history WHERE ride_id = :rideId ORDER BY recorded_at ASC")
    suspend fun getTrackForRide(rideId: String): List<LocationHistoryEntity>

    @Query("SELECT COUNT(*) FROM location_history WHERE ride_id = :rideId")
    suspend fun getCountForRide(rideId: String): Int

    @Query("DELETE FROM location_history WHERE ride_id = :rideId")
    suspend fun deleteForRide(rideId: String)

    // Prune old history to prevent unbounded table growth
    @Query("""
        DELETE FROM location_history
        WHERE ride_id = :rideId
        AND id NOT IN (
            SELECT id FROM location_history
            WHERE ride_id = :rideId
            ORDER BY recorded_at DESC
            LIMIT :keepCount
        )
    """)
    suspend fun pruneOldestForRide(rideId: String, keepCount: Int = 10_000)
}
```

```kotlin
// :core-data/src/main/kotlin/.../db/dao/LastKnownLocationDao.kt

@Dao
interface LastKnownLocationDao {

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsert(entity: LastKnownLocationEntity)

    @Query("SELECT * FROM last_known_location WHERE id = 1")
    suspend fun get(): LastKnownLocationEntity?
}
```

---

### Type converters

```kotlin
// :core-data/src/main/kotlin/.../db/converter/InstantConverter.kt

class InstantConverter {
    @TypeConverter
    fun fromInstant(value: Instant?): Long? = value?.toEpochMilli()

    @TypeConverter
    fun toInstant(value: Long?): Instant? = value?.let { Instant.ofEpochMilli(it) }
}
```

```kotlin
// :core-data/src/main/kotlin/.../db/converter/LatLngConverter.kt
// We store LatLng as two separate Double columns in entities — no converter needed.
// This converter is for cases where LatLng appears as a nested field in a JSON blob.

class LatLngConverter {
    private val json = Json { ignoreUnknownKeys = true }

    @TypeConverter
    fun fromLatLng(value: LatLng?): String? =
        value?.let { json.encodeToString(LatLng.serializer(), it) }

    @TypeConverter
    fun toLatLng(value: String?): LatLng? =
        value?.let { json.decodeFromString(LatLng.serializer(), it) }
}
```

**Rule:** store monetary values as `Long` in minor units (cents/pence), never as `Double` or `String`.  
Floating-point arithmetic on money is wrong. `€12.50` = `1250L` with currency `"EUR"`.

---

### Migrations

```kotlin
// :core-data/src/main/kotlin/.../db/migration/Migration_2_3.kt

val Migration_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // Example: adding accuracy_meters column to location_history
        database.execSQL("""
            ALTER TABLE location_history
            ADD COLUMN accuracy_meters REAL
        """)
    }
}
```

**Migration rules:**

1. Never use `fallbackToDestructiveMigration()` in release builds. Data loss is unacceptable.
2. Every schema change → new `Migration_X_Y.kt` file.
3. Test every migration with `MigrationTestHelper`:

```kotlin
@RunWith(AndroidJUnit4::class)
class DatabaseMigrationTest {

    @get:Rule
    val helper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        AppDatabase::class.java,
    )

    @Test
    fun migrate2To3() {
        // Create version 2 database
        helper.createDatabase(TEST_DB, 2).apply {
            execSQL("INSERT INTO location_history (ride_id, latitude, longitude, recorded_at) VALUES ('r1', 52.5, 13.4, 1000000)")
            close()
        }

        // Migrate to version 3 — must not throw
        val db = helper.runMigrationsAndValidate(TEST_DB, 3, true, Migration_2_3)

        // Verify the migrated data is intact
        val cursor = db.query("SELECT * FROM location_history WHERE ride_id = 'r1'")
        assertThat(cursor.count).isEqualTo(1)
        cursor.moveToFirst()
        // New column should exist with null value (existing rows)
        val colIndex = cursor.getColumnIndex("accuracy_meters")
        assertThat(colIndex).isNotEqualTo(-1)
        db.close()
    }
}
```

---

## DataStore — Preferences & State

### What goes in DataStore vs Room

| Data | Storage | Reason |
|---|---|---|
| Active ride entity | Room | Relational, queryable, survives process death |
| Location history | Room | Large, needs indexed queries |
| Ride history list | Room | List, needs ORDER BY, filtering |
| Permission flow progress | DataStore | Single value, reactive, no queries |
| User preferences (map type, sound) | DataStore | Simple key-value, no relations |
| Last known location | Room | Needs to coexist with ride queries in transactions |
| Auth tokens | EncryptedSharedPreferences | Security — DataStore is not encrypted by default |
| Feature flags (local overrides) | DataStore | Simple boolean flags |
| App session state | In-memory only | Does not survive process death intentionally |

**Rule:** DataStore is not a database. Do not store lists, do not store relational data.  
If you need to query it, filter it, or join it — use Room.

---

### DataStore setup

```kotlin
// :core-data/src/main/kotlin/.../di/DataModule.kt  (partial)

private val Context.permissionStateDataStore by preferencesDataStore(
    name = "permission_state",
)

private val Context.userPreferencesDataStore by preferencesDataStore(
    name = "user_preferences",
)

@Provides
@Singleton
fun providePermissionStateDataStore(
    @ApplicationContext context: Context,
): DataStore<Preferences> = context.permissionStateDataStore

@Provides
@Singleton
fun provideUserPreferencesDataStore(
    @ApplicationContext context: Context,
): DataStore<Preferences> = context.userPreferencesDataStore
```

---

### Permission flow state

```kotlin
// :core-data/src/main/kotlin/.../datastore/PermissionStateDataStore.kt

@Singleton
class PermissionStateDataStore @Inject constructor(
    @PermissionPreferences private val dataStore: DataStore<Preferences>,
) {
    companion object {
        val KEY_FOREGROUND_REQUESTED = booleanPreferencesKey("foreground_requested")
        val KEY_BACKGROUND_RATIONALE_SHOWN = booleanPreferencesKey("background_rationale_shown")
        val KEY_PERMANENTLY_DENIED_DETECTED = booleanPreferencesKey("permanently_denied")
        val KEY_NOTIFICATION_REQUESTED = booleanPreferencesKey("notification_requested")
    }

    val permissionState: Flow<PermissionFlowState> = dataStore.data
        .catch { e ->
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs ->
            PermissionFlowState(
                hasForegroundBeenRequested = prefs[KEY_FOREGROUND_REQUESTED] ?: false,
                hasBackgroundRationaleBeenShown = prefs[KEY_BACKGROUND_RATIONALE_SHOWN] ?: false,
                permanentlyDeniedDetected = prefs[KEY_PERMANENTLY_DENIED_DETECTED] ?: false,
                hasNotificationBeenRequested = prefs[KEY_NOTIFICATION_REQUESTED] ?: false,
            )
        }

    suspend fun markForegroundRequested() = dataStore.edit { prefs ->
        prefs[KEY_FOREGROUND_REQUESTED] = true
    }

    suspend fun markBackgroundRationaleShown() = dataStore.edit { prefs ->
        prefs[KEY_BACKGROUND_RATIONALE_SHOWN] = true
    }

    suspend fun markPermanentlyDenied() = dataStore.edit { prefs ->
        prefs[KEY_PERMANENTLY_DENIED_DETECTED] = true
    }

    /** Called on fresh install detection — reset all flags. */
    suspend fun resetAll() = dataStore.edit { it.clear() }
}

data class PermissionFlowState(
    val hasForegroundBeenRequested: Boolean,
    val hasBackgroundRationaleBeenShown: Boolean,
    val permanentlyDeniedDetected: Boolean,
    val hasNotificationBeenRequested: Boolean,
)
```

---

### User preferences

```kotlin
// :core-data/src/main/kotlin/.../datastore/UserPreferencesDataStore.kt

@Singleton
class UserPreferencesDataStore @Inject constructor(
    @UserPreferences private val dataStore: DataStore<Preferences>,
) {
    companion object {
        val KEY_MAP_TYPE = intPreferencesKey("map_type")               // 0=Normal, 1=Satellite
        val KEY_NOTIFICATION_SOUND = booleanPreferencesKey("notif_sound")
        val KEY_DISTANCE_UNIT = stringPreferencesKey("distance_unit")  // "km" | "mi"
        val KEY_LAST_SEARCH_DESTINATION = stringPreferencesKey("last_destination")
    }

    val preferences: Flow<UserPreferences> = dataStore.data
        .catch { e ->
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs ->
            UserPreferences(
                mapType = MapType.fromInt(prefs[KEY_MAP_TYPE] ?: 0),
                notificationSoundEnabled = prefs[KEY_NOTIFICATION_SOUND] ?: true,
                distanceUnit = DistanceUnit.fromString(prefs[KEY_DISTANCE_UNIT] ?: "km"),
                lastSearchDestination = prefs[KEY_LAST_SEARCH_DESTINATION],
            )
        }

    suspend fun updateMapType(type: MapType) = dataStore.edit { prefs ->
        prefs[KEY_MAP_TYPE] = type.ordinal
    }

    suspend fun updateLastDestination(destination: String?) = dataStore.edit { prefs ->
        if (destination != null) prefs[KEY_LAST_SEARCH_DESTINATION] = destination
        else prefs.remove(KEY_LAST_SEARCH_DESTINATION)
    }
}
```

**Error handling in DataStore:**  
Always catch `IOException` in the `.catch {}` operator and emit `emptyPreferences()`.  
DataStore can throw `IOException` on first read if the file is corrupted or missing.  
Without this catch, the flow terminates and the app loses preferences reactivity permanently.

---

## Network Layer

### Retrofit setup

```kotlin
// :core-data/src/main/kotlin/.../di/DataModule.kt  (partial)

@Provides
@Singleton
fun provideOkHttpClient(
    authInterceptor: AuthInterceptor,
    loggingInterceptor: HttpLoggingInterceptor,
): OkHttpClient = OkHttpClient.Builder()
    .addInterceptor(authInterceptor)
    .addInterceptor(loggingInterceptor)
    .connectTimeout(30, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .build()

@Provides
@Singleton
fun provideRetrofit(
    okHttpClient: OkHttpClient,
    @BaseUrl baseUrl: String,
): Retrofit = Retrofit.Builder()
    .baseUrl(baseUrl)
    .client(okHttpClient)
    .addConverterFactory(
        Json { ignoreUnknownKeys = true }.asConverterFactory("application/json".toMediaType())
    )
    .build()

@Provides
@Singleton
fun provideRideApiService(retrofit: Retrofit): RideApiService =
    retrofit.create(RideApiService::class.java)
```

---

### OkHttp interceptors

```kotlin
// :core-data/src/main/kotlin/.../remote/interceptor/AuthInterceptor.kt

class AuthInterceptor @Inject constructor(
    private val tokenProvider: TokenProvider,   // injected, reads from EncryptedSharedPreferences
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenProvider.getAccessToken()
        val request = chain.request().newBuilder()
            .header("Authorization", "Bearer $token")
            .header("X-Platform", "android")
            .header("X-App-Version", BuildConfig.VERSION_NAME)
            .build()
        return chain.proceed(request)
    }
}
```

```kotlin
// Logging — debug only, never log in release builds

@Provides
@Singleton
fun provideLoggingInterceptor(): HttpLoggingInterceptor =
    HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG) {
            HttpLoggingInterceptor.Level.BODY
        } else {
            HttpLoggingInterceptor.Level.NONE   // never log request bodies in release
        }
    }
```

---

### WebSocket client

```kotlin
// :core-data/src/main/kotlin/.../remote/websocket/RideWebSocketClient.kt

@Singleton
class RideWebSocketClient @Inject constructor(
    private val okHttpClient: OkHttpClient,
    @WsBaseUrl private val wsBaseUrl: String,
    private val json: Json,
) {
    private var currentWebSocket: WebSocket? = null
    private val reconnectDelays = listOf(1_000L, 2_000L, 5_000L, 10_000L, 30_000L)

    fun observeEvents(): Flow<RideServerEventDto> = channelFlow {
        var attempt = 0

        while (isActive) {
            val request = Request.Builder()
                .url("$wsBaseUrl/rides/events")
                .build()

            val connected = CompletableDeferred<Boolean>()

            currentWebSocket = okHttpClient.newWebSocket(
                request,
                object : WebSocketListener() {
                    override fun onOpen(webSocket: WebSocket, response: Response) {
                        attempt = 0                             // reset backoff on success
                        connected.complete(true)
                    }

                    override fun onMessage(webSocket: WebSocket, text: String) {
                        val event = runCatching {
                            json.decodeFromString<RideServerEventDto>(text)
                        }.getOrNull() ?: return                 // skip malformed messages

                        trySend(event)
                    }

                    override fun onFailure(
                        webSocket: WebSocket,
                        t: Throwable,
                        response: Response?,
                    ) {
                        connected.completeExceptionally(t)
                    }

                    override fun onClosing(webSocket: WebSocket, code: Int, reason: String) {
                        webSocket.close(1000, null)
                    }
                }
            )

            // Wait for connection result
            val success = runCatching { connected.await() }.getOrDefault(false)

            if (!success) {
                // Exponential backoff before reconnect
                val delay = reconnectDelays.getOrLast(attempt++)
                delay(delay)
            } else {
                // Wait until the socket is closed (flow collector cancels or failure)
                awaitCancellation()
            }
        }
    }.shareIn(
        scope = CoroutineScope(SupervisorJob() + Dispatchers.IO),
        started = SharingStarted.WhileSubscribed(stopTimeoutMillis = 5_000),
        replay = 0,
    )

    fun disconnect() {
        currentWebSocket?.close(1000, "Client disconnected")
        currentWebSocket = null
    }
}

private fun <T> List<T>.getOrLast(index: Int): T = getOrElse(index) { last() }
```

**`shareIn` with `WhileSubscribed(5000)`:**  
The WebSocket connection stays open as long as there is at least one collector.  
If the last collector disappears (e.g. app backgrounded), the connection stays open  
for 5 more seconds — enough to handle a brief screen rotation — then closes.  
This prevents reconnecting every time the ViewModel is recreated.

---

### DTO mapping

All mapping between network DTOs, Room entities, and domain models is in dedicated mapper files.  
No mapping logic in Repository, DAO, or ViewModel.

```kotlin
// :core-data/src/main/kotlin/.../mapper/RideMapper.kt

// DTO → Domain
fun RideDto.toDomain(): Ride = Ride(
    id = id,
    status = status.toDomainStatus(),
    origin = LatLng(originLat, originLng),
    destination = destinationLat?.let { LatLng(it, destinationLng!!) },
    driver = driver?.toDomain(),
    etaSeconds = etaSeconds,
    startedAt = startedAt?.let { Instant.parse(it) },
    completedAt = completedAt?.let { Instant.parse(it) },
    summary = summary?.toDomain(),
)

// Domain → Entity
fun Ride.toEntity(): RideEntity = RideEntity(
    id = id,
    status = status.name,
    originLat = origin.latitude,
    originLng = origin.longitude,
    destinationLat = destination?.latitude,
    destinationLng = destination?.longitude,
    driverId = driver?.id,
    driverName = driver?.name,
    driverRating = driver?.rating,
    vehicleModel = driver?.vehicleModel,
    vehiclePlate = driver?.vehiclePlate,
    etaSeconds = etaSeconds,
    startedAt = startedAt,
    completedAt = completedAt,
    fareAmount = summary?.fareAmount?.amount,
    fareCurrency = summary?.fareAmount?.currency,
    createdAt = Instant.now(),
    updatedAt = Instant.now(),
)

// Entity → Domain
fun RideEntity.toDomain(): Ride = Ride(
    id = id,
    status = RideStatus.valueOf(status),
    origin = LatLng(originLat, originLng),
    destination = destinationLat?.let { LatLng(it, destinationLng!!) },
    driver = if (driverId != null) Driver(
        id = driverId,
        name = driverName ?: "",
        rating = driverRating ?: 0f,
        vehicleModel = vehicleModel ?: "",
        vehiclePlate = vehiclePlate ?: "",
    ) else null,
    etaSeconds = etaSeconds,
    startedAt = startedAt,
    completedAt = completedAt,
    summary = if (fareAmount != null) RideSummary(
        fareAmount = Money(fareAmount, fareCurrency ?: "EUR"),
    ) else null,
)
```

**Mapping rules:**
- Mappers are pure functions — no side effects, no IO
- DTOs are not used outside the data layer
- Entities are not used outside the data layer
- Domain models are the only models that cross the layer boundary upward

---

## Offline-First Strategy

### What is offline-first here

"Offline-first" does not mean "works completely offline forever."  
It means: **the user never sees a broken or empty screen just because the network is slow or absent.**

Concretely for this app:

| Scenario | Offline-first behaviour |
|---|---|
| App opens with no network | Shows last known ride state from Room. Shows "No connection" banner. |
| Network drops during Waiting | State persists in Room. ETA freezes. Banner shown. On reconnect: re-sync. |
| Network drops during InProgress | FGS continues. Location buffered. Route reconstructed from buffer on reconnect. |
| App killed, restarted with no network | Room restores last known state. Shows offline banner. |
| Ride history requested offline | Shows cached list from Room with staleness indicator. |

**What is NOT offline-first:**
- Starting a new ride search — requires network (cannot match with a driver offline)
- Payment — requires network
- Submitting a rating — queued for retry (see sync strategy)

---

### Cache policies by data type

| Data | Cache lifetime | Invalidation trigger | Staleness UI |
|---|---|---|---|
| Active ride state | Until ride ends | Server event / manual sync | None — always show latest |
| Driver location | Last 1 point in memory | New GPS update | "Location may be outdated" after 30s |
| Location history | Until ride ends, then archive | Ride completion | None |
| Ride history | Until next sync | `syncRideHistory()` on reconnect | "Last updated X ago" label |
| Last known location | 24 hours | New GPS fix | None |
| User preferences | Indefinite | User changes setting | None |
| Permission state | Indefinite | User grants/revokes | None |

---

### Network monitor

```kotlin
// :core-data/src/main/kotlin/.../network/NetworkMonitor.kt

interface NetworkMonitor {
    val isOnline: Flow<Boolean>
    suspend fun isCurrentlyOnline(): Boolean
}

@Singleton
class ConnectivityNetworkMonitor @Inject constructor(
    @ApplicationContext private val context: Context,
) : NetworkMonitor {

    override val isOnline: Flow<Boolean> = callbackFlow {
        val connectivityManager = context.getSystemService<ConnectivityManager>()
            ?: run { trySend(false); return@callbackFlow }

        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                trySend(true)
            }
            override fun onLost(network: Network) {
                trySend(false)
            }
            override fun onUnavailable() {
                trySend(false)
            }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .addCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
            .build()

        connectivityManager.registerNetworkCallback(request, callback)
        // Emit current state immediately
        trySend(connectivityManager.isCurrentlyConnected())

        awaitClose { connectivityManager.unregisterNetworkCallback(callback) }
    }
        .distinctUntilChanged()
        .shareIn(
            scope = CoroutineScope(SupervisorJob() + Dispatchers.IO),
            started = SharingStarted.WhileSubscribed(),
            replay = 1,
        )

    override suspend fun isCurrentlyOnline(): Boolean =
        isOnline.first()

    private fun ConnectivityManager.isCurrentlyConnected(): Boolean =
        activeNetwork
            ?.let { getNetworkCapabilities(it) }
            ?.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
            ?: false
}

// Fake for tests
class FakeNetworkMonitor(initialState: Boolean = true) : NetworkMonitor {
    private val _isOnline = MutableStateFlow(initialState)
    override val isOnline: Flow<Boolean> = _isOnline
    override suspend fun isCurrentlyOnline(): Boolean = _isOnline.value
    fun setOnline(online: Boolean) { _isOnline.value = online }
}
```

**`NET_CAPABILITY_VALIDATED`:** the network has confirmed internet access (not just connected to WiFi with a captive portal). Without this check, `isOnline` could be `true` while the network is actually unusable.

---

### Stale-while-revalidate

For ride history (non-critical, acceptable to show stale):

```kotlin
// In RideRepositoryImpl

fun observeRideHistory(): Flow<List<Ride>> = flow {
    // 1. Emit cached data immediately — user sees content fast
    val cached = rideDao.observeHistory().first()
    emit(cached.map { it.toDomain() })

    // 2. Try to refresh from network
    if (networkMonitor.isCurrentlyOnline()) {
        runCatching {
            val fresh = rideApiService.getRideHistory().bodyOrThrow()
            rideDao.replaceHistory(fresh.map { it.toEntity() })
        }
        // 3. Room Flow auto-emits the updated data — no manual re-emit needed
    }
}.flatMapLatest {
    rideDao.observeHistory().map { entities -> entities.map { it.toDomain() } }
}
```

---

### Write-through vs write-back

| Operation | Strategy | Reason |
|---|---|---|
| Ride status update (from server) | Write-through: update Room immediately | State must be consistent for process death recovery |
| Cancel ride (user action) | Write-through: call API, then update Room | Server is authority; optimistic update only if latency is visibly bad |
| Location history point | Write-back: buffer in memory, flush periodically | Prevents DB write on every GPS tick (every 2-5s) |
| User preference change | Write-through: DataStore immediately | No network call needed; local only |
| Rating submission | Write-back with retry queue | Non-critical; acceptable to retry later |

---

### Conflict resolution

When the client and server disagree on ride state (after reconnect, process death):

**Rule: server always wins for ride state.**

```kotlin
// In RideRepositoryImpl

suspend fun reconcileWithServer(): Result<Unit> = withContext(ioDispatcher) {
    runCatching {
        val serverRide = rideApiService.getActiveRide().bodyOrThrow()
            ?: run {
                // Server has no active ride — clear local state
                rideDao.clearActiveRide()
                return@runCatching
            }

        val localRide = rideDao.getActiveRide()

        if (localRide == null || localRide.id != serverRide.id) {
            // Server has a different ride than we thought — replace entirely
            rideDao.upsert(serverRide.toEntity())
        } else if (serverRide.updatedAt > localRide.updatedAt) {
            // Server version is newer — update
            rideDao.upsert(serverRide.toEntity())
        }
        // If local is newer (e.g. we wrote optimistically) — keep local,
        // server will catch up via WebSocket
    }.mapFailure { it.toDomainError() }
}
```

---

## Sync Strategy

Operations that must sync after reconnect, in priority order:

| Priority | Operation | When | Retry |
|---|---|---|---|
| 1 | Reconcile active ride state | On reconnect, always | 3× exponential backoff |
| 2 | Flush location history buffer | On reconnect if InProgress | 3× exponential backoff |
| 3 | Submit pending rating | On reconnect if pending | 5× with 24h max |
| 4 | Sync ride history | On reconnect, if >5 min stale | 1× |

```kotlin
// In RideRepositoryImpl — triggered by NetworkMonitor

init {
    observeScope.launch {
        networkMonitor.isOnline
            .filter { it }                          // only when coming online
            .collect { onNetworkRestored() }
    }
}

private suspend fun onNetworkRestored() {
    reconcileWithServer()                           // priority 1
    if (hasBufferedLocations()) flushLocationBuffer() // priority 2
    // Priority 3 and 4 handled by WorkManager (they can wait)
}
```

---

## Data Flow Diagrams

### Read flow — active ride

```
ViewModel
  └─► ObserveActiveRideUseCase()
         └─► rideRepository.observeActiveRide()          ← Flow<Ride?>
                └─► rideDao.observeActiveRide()           ← Room Flow (auto-updates)
                       └─► emits on every DB change

Meanwhile, WebSocket events arrive:
  WebSocket message
    └─► RideWebSocketClient emits RideServerEventDto
           └─► RideRepositoryImpl maps to domain event
                  └─► RideRepositoryImpl writes to Room via rideDao.updateStatus()
                         └─► Room notifies all active observers
                                └─► observeActiveRide() Flow emits updated ride
                                       └─► UseCase propagates
                                              └─► ViewModel receives
                                                     └─► reduce { state.copy(...) }
```

### Write flow — cancel ride

```
User taps "Cancel"
  └─► RideIntent.CancelRide
         └─► RideViewModel.handleCancelRide()
                └─► cancelRideUseCase(rideId, reason)
                       └─► rideRepository.cancelRide(rideId, reason)
                              ├─► rideApiService.cancelRide(...)   [network]
                              │      └─► Success → continue
                              │          Failure → Result.failure(DomainError)
                              │
                              └─► rideDao.updateStatus(CANCELLED)  [Room — always, even on API error]
                                     └─► Room notifies observeActiveRide() observers
                                            └─► ViewModel state updates to Cancelled
```

### Reconnect flow

```
NetworkMonitor detects: isOnline = true
  └─► RideRepositoryImpl.onNetworkRestored()
         ├─► reconcileWithServer()
         │      └─► GET /rides/active
         │             ├─► 200 → compare with local → upsert if newer
         │             └─► 404 → no active ride → clearActiveRide()
         │
         ├─► flushLocationBuffer()
         │      └─► POST /rides/{id}/track-points [batch]
         │
         └─► rideWebSocketClient reconnects (auto via channelFlow loop)
                └─► WebSocket events resume
                       └─► ViewModel receives updated state
```

---

## Dependency Injection — Data Module

```kotlin
// :core-data/src/main/kotlin/.../di/DataModule.kt

@Module
@InstallIn(SingletonComponent::class)
object DataModule {

    @Provides @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase = ...

    @Provides @Singleton
    fun provideRideDao(db: AppDatabase): RideDao = db.rideDao()

    @Provides @Singleton
    fun provideLocationHistoryDao(db: AppDatabase): LocationHistoryDao = db.locationHistoryDao()

    @Provides @Singleton
    fun provideLastKnownLocationDao(db: AppDatabase): LastKnownLocationDao =
        db.lastKnownLocationDao()

    @Provides @Singleton
    fun provideNetworkMonitor(
        @ApplicationContext context: Context,
    ): NetworkMonitor = ConnectivityNetworkMonitor(context)

    @Provides @Singleton
    fun provideJson(): Json = Json {
        ignoreUnknownKeys = true        // survive backend adding new fields
        isLenient = true
        encodeDefaults = false
    }

    // Named qualifiers for multiple DataStore instances
    @Provides @Singleton @PermissionPreferences
    fun providePermissionDataStore(@ApplicationContext context: Context): DataStore<Preferences> =
        context.permissionStateDataStore

    @Provides @Singleton @UserPreferences
    fun provideUserDataStore(@ApplicationContext context: Context): DataStore<Preferences> =
        context.userPreferencesDataStore
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds @Singleton
    abstract fun bindRideRepository(impl: RideRepositoryImpl): RideRepository

    @Binds @Singleton
    abstract fun bindLocationRepository(impl: LocationRepositoryImpl): LocationRepository

    @Binds @Singleton
    abstract fun bindNetworkMonitor(impl: ConnectivityNetworkMonitor): NetworkMonitor
}
```

---

## Testing the Data Layer

### Room DAO tests

```kotlin
// :core-data/src/androidTest/.../db/dao/RideDaoTest.kt

@RunWith(AndroidJUnit4::class)
class RideDaoTest {

    private lateinit var db: AppDatabase
    private lateinit var rideDao: RideDao

    @Before
    fun setup() {
        db = Room.inMemoryDatabaseBuilder(
            InstrumentationRegistry.getInstrumentation().targetContext,
            AppDatabase::class.java,
        )
            .allowMainThreadQueries()   // tests only — never in production
            .build()
        rideDao = db.rideDao()
    }

    @After
    fun teardown() { db.close() }

    @Test
    fun getActiveRide_returnsNonTerminalRide() = runTest {
        rideDao.upsert(testRideEntity(status = "SEARCHING"))
        rideDao.upsert(testRideEntity(id = "r2", status = "COMPLETED"))

        val active = rideDao.getActiveRide()

        assertThat(active?.id).isEqualTo("r1")
        assertThat(active?.status).isEqualTo("SEARCHING")
    }

    @Test
    fun observeActiveRide_emitsOnStatusUpdate() = runTest {
        rideDao.upsert(testRideEntity(status = "WAITING"))

        val emissions = mutableListOf<RideEntity?>()
        val job = launch {
            rideDao.observeActiveRide().take(2).toList(emissions)
        }

        rideDao.updateStatus("r1", "IN_PROGRESS")
        job.join()

        assertThat(emissions).hasSize(2)
        assertThat(emissions[0]?.status).isEqualTo("WAITING")
        assertThat(emissions[1]?.status).isEqualTo("IN_PROGRESS")
    }

    @Test
    fun upsert_replacesExistingRide() = runTest {
        rideDao.upsert(testRideEntity(etaSeconds = 120))
        rideDao.upsert(testRideEntity(etaSeconds = 60))

        val result = rideDao.getActiveRide()
        assertThat(result?.etaSeconds).isEqualTo(60)
    }
}
```

### Repository unit tests with fakes

```kotlin
// :core-data/src/test/.../repository/RideRepositoryImplTest.kt

class RideRepositoryImplTest {

    private val fakeApiService = FakeRideApiService()
    private val fakeRideDao = FakeRideDao()
    private val fakeNetworkMonitor = FakeNetworkMonitor(initialState = true)

    private val repository = RideRepositoryImpl(
        rideApiService = fakeApiService,
        rideWebSocketClient = FakeRideWebSocketClient(),
        rideDao = fakeRideDao,
        networkMonitor = fakeNetworkMonitor,
        ioDispatcher = UnconfinedTestDispatcher(),
    )

    @Test
    fun startSearch_onSuccess_cachesRideLocally() = runTest {
        fakeApiService.stubbedRide = testRideDto(id = "ride-1")

        val result = repository.startSearch(testLatLng())

        assertThat(result.isSuccess).isTrue()
        assertThat(fakeRideDao.upsertedRides).hasSize(1)
        assertThat(fakeRideDao.upsertedRides.first().id).isEqualTo("ride-1")
    }

    @Test
    fun startSearch_onNetworkFailure_returnsNetworkError() = runTest {
        fakeApiService.shouldThrow = UnknownHostException()

        val result = repository.startSearch(testLatLng())

        assertThat(result.isFailure).isTrue()
        assertThat(result.exceptionOrNull()).isInstanceOf(DomainError.NetworkUnavailable::class.java)
    }

    @Test
    fun getActiveRide_whenNoRideInDb_returnsNull() = runTest {
        val result = repository.getActiveRide()
        assertThat(result).isNull()
    }
}
```

### DataStore tests

```kotlin
@Test
fun permissionStateDataStore_markForegroundRequested_persistsFlag() = runTest {
    val dataStore = PreferenceDataStoreFactory.create(
        scope = this,
        produceFile = { testFile },
    )
    val store = PermissionStateDataStore(dataStore)

    assertThat(store.permissionState.first().hasForegroundBeenRequested).isFalse()

    store.markForegroundRequested()

    assertThat(store.permissionState.first().hasForegroundBeenRequested).isTrue()
}
```

---

## Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Performing IO on the default dispatcher | StrictMode violation, potential ANR | Always `withContext(ioDispatcher)` in Repository |
| Injecting `Dispatchers.IO` directly | Untestable — cannot replace in tests | Inject `CoroutineDispatcher` via Hilt `@Named` qualifier |
| Throwing from repository methods | UseCase/ViewModel must wrap in try-catch | Always return `Result<T>`, never throw |
| Storing monetary values as `Double` | Floating-point rounding errors | Use `Long` minor units + currency string |
| Forgetting `exportSchema = true` | Migration history lost, Room can't validate | Always export schema, commit to VCS |
| Using `fallbackToDestructiveMigration` in release | Silent data loss on upgrade | Use explicit `Migration` objects; `fallbackToDestructiveMigration` in debug only |
| Returning DTO or Entity types from Repository | Leaks data-layer types into domain | Always map to domain models before returning |
| Mapping in Repository rather than Mapper files | Logic scattered, hard to test | All mapping in dedicated `*Mapper.kt` files |
| DataStore without `.catch { emit(emptyPreferences()) }` | Flow terminates on IO error, app loses prefs | Always catch `IOException` in DataStore flows |
| Requesting `NET_CAPABILITY_INTERNET` without `VALIDATED` | Returns true for captive portal WiFi | Always check `NET_CAPABILITY_VALIDATED` |
| WebSocket reconnect without backoff | Hammers server on network issues | Exponential backoff list with max delay |
| Sharing mutable state between Service and ViewModel | Thread safety issues, unexpected updates | Communicate only via `SharedFlow` in Repository |

---

*Last updated: <!-- date -->*  
*Owner: Android Tech Lead*  
*Review: Required before any new entity, DAO, or Repository is added*
