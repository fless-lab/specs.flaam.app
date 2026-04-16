# Flaam — Spécification Architecture Mobile (Kotlin / Jetpack Compose)

> Version 1.0 — Avril 2026
> Companion du document `flaam-backend-spec.md`
> Target : Android 8+ (API 26+), Kotlin 2.0, Jetpack Compose

---

## 1. Stack technique

```kotlin
// build.gradle.kts (version catalog)

/**
 * Choix de chaque dépendance et POURQUOI :
 *
 * Jetpack Compose  → UI déclarative, moins de code, recomposition intelligente
 * Hilt             → DI standard Android, intégration Compose/ViewModel native
 * Retrofit + OkHttp → HTTP client le plus mature, interceptors, retry
 * Room             → SQLite abstrait, offline-first, Flow support
 * DataStore        → Remplacement SharedPreferences, async, type-safe
 * Coil             → Image loading Kotlin-first, Compose natif, léger
 * Navigation Compose → Navigation type-safe, deep linking intégré
 * WorkManager      → Background tasks fiable, survit aux process kills
 * Ktor WebSocket   → WebSocket client léger, coroutines-native
 *                    (on utilise Ktor UNIQUEMENT pour le WS, pas pour le HTTP)
 * Accompanist      → Permissions, system UI controller
 * CameraX          → Selfie vérification, photo capture
 * Timber           → Logging structuré
 * LeakCanary       → Détection memory leaks (debug only)
 * Turbine          → Test des Flows
 */

[versions]
kotlin = "2.0.0"
compose-bom = "2024.12.01"
hilt = "2.51"
retrofit = "2.11.0"
okhttp = "4.12.0"
room = "2.6.1"
datastore = "1.1.1"
coil = "2.7.0"
navigation = "2.8.4"
workmanager = "2.10.0"
ktor = "2.3.12"       # WebSocket uniquement
camerax = "1.4.0"
accompanist = "0.36.0"
coroutines = "1.8.1"
timber = "5.0.1"

[libraries]
# ── Compose ──
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
compose-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }

# ── Architecture ──
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
hilt-navigation = { group = "androidx.hilt", name = "hilt-navigation-compose", version = "1.2.0" }

# ── Network ──
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-moshi = { group = "com.squareup.retrofit2", name = "converter-moshi", version.ref = "retrofit" }
okhttp = { group = "com.squareup.okhttp3", name = "okhttp", version.ref = "okhttp" }
okhttp-logging = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }
ktor-websocket = { group = "io.ktor", name = "ktor-client-websockets", version.ref = "ktor" }
ktor-okhttp = { group = "io.ktor", name = "ktor-client-okhttp", version.ref = "ktor" }

# ── Local Storage ──
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
datastore = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }

# ── Image ──
coil = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }

# ── Navigation ──
navigation = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation" }

# ── Background ──
workmanager = { group = "androidx.work", name = "work-runtime-ktx", version.ref = "workmanager" }

# ── Camera ──
camerax-core = { group = "androidx.camera", name = "camera-core", version.ref = "camerax" }
camerax-lifecycle = { group = "androidx.camera", name = "camera-lifecycle", version.ref = "camerax" }
camerax-view = { group = "androidx.camera", name = "camera-view", version.ref = "camerax" }

# ── Utils ──
accompanist-permissions = { group = "com.google.accompanist", name = "accompanist-permissions", version.ref = "accompanist" }
timber = { group = "com.jakewharton.timber", name = "timber", version.ref = "timber" }
```

---

## 2. Structure du projet (Multi-module)

```
flaam-android/
├── app/                          # Module principal, point d'entrée
│   ├── src/main/
│   │   ├── FlaamApp.kt           # Application class (Hilt)
│   │   ├── MainActivity.kt       # Single Activity
│   │   └── FlaamNavGraph.kt      # Graphe de navigation root
│   └── build.gradle.kts
│
├── core/                         # Modules partagés (pas de UI)
│   ├── network/                  # Retrofit, OkHttp, interceptors
│   │   ├── api/                  # Interfaces Retrofit (AuthApi, FeedApi, ChatApi...)
│   │   ├── interceptor/          # AuthInterceptor, IdempotencyInterceptor
│   │   ├── websocket/            # WebSocket client, reconnection, message queue
│   │   └── dto/                  # Data Transfer Objects (réponses API)
│   │
│   ├── database/                 # Room database, DAOs, entities
│   │   ├── entity/               # ProfileEntity, MessageEntity, SpotEntity...
│   │   ├── dao/                  # ProfileDao, MessageDao, SpotDao...
│   │   ├── FlaamDatabase.kt      # Room database class
│   │   └── Converters.kt         # Type converters (UUID, DateTime, JSON)
│   │
│   ├── datastore/                # DataStore preferences
│   │   ├── AuthDataStore.kt      # JWT tokens, refresh token
│   │   ├── UserPrefsDataStore.kt # Préférences utilisateur (data saver, notifs...)
│   │   └── OnboardingDataStore.kt # État de l'onboarding
│   │
│   ├── common/                   # Utilitaires partagés
│   │   ├── result/               # Result<T> wrapper, error handling
│   │   ├── extension/            # Extensions Kotlin
│   │   ├── constants/            # FlaamConstants (URLs, timeouts, limites)
│   │   └── connectivity/         # NetworkMonitor (online/slow/offline)
│   │
│   └── security/                 # Sécurité
│       ├── EncryptedStorage.kt   # EncryptedSharedPreferences pour tokens
│       ├── DeviceFingerprint.kt  # Génération du device fingerprint
│       └── CertificatePinner.kt  # Certificate pinning config
│
├── domain/                       # Logique métier pure (pas de dépendance Android)
│   ├── model/                    # Modèles métier (Profile, Match, Message, Spot...)
│   ├── repository/               # Interfaces repository (contrats)
│   └── usecase/                  # Cas d'usage
│       ├── auth/                 # LoginUseCase, VerifyOtpUseCase
│       ├── feed/                 # GetDailyFeedUseCase, LikeProfileUseCase
│       ├── chat/                 # SendMessageUseCase, GetConversationUseCase
│       ├── spots/                # CheckInUseCase, GetSpotsUseCase
│       ├── events/               # GetEventsUseCase, RegisterEventUseCase
│       ├── profile/              # UpdateProfileUseCase, UploadPhotoUseCase
│       └── safety/               # ShareDateUseCase, TriggerEmergencyUseCase
│
├── data/                         # Implémentation des repositories
│   ├── repository/               # Implémentations concrètes
│   │   ├── AuthRepositoryImpl.kt
│   │   ├── FeedRepositoryImpl.kt
│   │   ├── ChatRepositoryImpl.kt
│   │   ├── SpotRepositoryImpl.kt
│   │   ├── EventRepositoryImpl.kt
│   │   ├── ProfileRepositoryImpl.kt
│   │   └── SafetyRepositoryImpl.kt
│   │
│   ├── sync/                     # Synchronisation offline-first
│   │   ├── SyncQueue.kt          # Queue d'actions en attente
│   │   ├── SyncWorker.kt         # WorkManager worker pour la sync
│   │   └── ConflictResolver.kt   # Résolution de conflits serveur/local
│   │
│   └── mapper/                   # Mappers DTO → Entity → Domain Model
│       ├── ProfileMapper.kt
│       ├── MessageMapper.kt
│       └── SpotMapper.kt
│
├── feature/                      # Modules feature (UI)
│   ├── onboarding/               # Écrans 01-14
│   │   ├── navigation/           # OnboardingNavGraph.kt
│   │   ├── screen/               # WelcomeScreen, PhoneScreen, OtpScreen...
│   │   ├── viewmodel/            # OnboardingViewModel
│   │   └── component/            # Composables réutilisables onboarding
│   │
│   ├── feed/                     # Écrans 15-16
│   │   ├── navigation/
│   │   ├── screen/               # FeedScreen, ProfileDetailScreen
│   │   ├── viewmodel/            # FeedViewModel
│   │   └── component/            # ProfileCard, ActionButtons, GeoScore
│   │
│   ├── spots/                    # Écrans 17-18
│   │   ├── navigation/
│   │   ├── screen/               # SpotsScreen, SpotDetailScreen
│   │   ├── viewmodel/            # SpotsViewModel
│   │   └── component/            # SpotCard, CheckInButton, SpotMap
│   │
│   ├── chat/                     # Écrans 19-20
│   │   ├── navigation/
│   │   ├── screen/               # ChatListScreen, ConversationScreen
│   │   ├── viewmodel/            # ChatListViewModel, ConversationViewModel
│   │   └── component/            # MessageBubble, VoiceRecorder, IceBreaker
│   │
│   ├── events/                   # Écrans 21-22
│   │   ├── navigation/
│   │   ├── screen/               # EventsScreen, EventDetailScreen
│   │   ├── viewmodel/            # EventsViewModel
│   │   └── component/            # EventCard, AttendeeList
│   │
│   ├── profile/                  # Écrans 23-28
│   │   ├── navigation/
│   │   ├── screen/               # MyProfileScreen, EditProfileScreen,
│   │   │                         # SettingsScreen, DiscoveryPrefsScreen,
│   │   │                         # SubscriptionScreen, NotifPrefsScreen
│   │   ├── viewmodel/            # ProfileViewModel, SettingsViewModel
│   │   └── component/            # PhotoGrid, PromptEditor, TagSelector
│   │
│   └── safety/                   # Écrans 37-39
│       ├── navigation/
│       ├── screen/               # ShareDateScreen, TimerScreen, EmergencyScreen
│       ├── viewmodel/            # SafetyViewModel
│       └── component/            # CountdownTimer, EmergencyButton
│
├── ui/                           # Design system partagé
│   ├── theme/
│   │   ├── FlaamTheme.kt         # MaterialTheme wrapper
│   │   ├── Color.kt              # Palette complète (Terracotta, Teal, Cream...)
│   │   ├── Type.kt               # Typographie (Instrument Serif + DM Sans)
│   │   ├── Shape.kt              # Formes (6px → full)
│   │   └── Spacing.kt            # Spacing system (4/8/12/16/24/32)
│   │
│   ├── component/                # Composables réutilisables
│   │   ├── FlaamButton.kt        # Primary, Secondary, Ghost, Icon buttons
│   │   ├── FlaamCard.kt          # Cards avec différents styles
│   │   ├── FlaamTextField.kt     # Input fields stylés
│   │   ├── FlaamBadge.kt         # Badges (vérifié, fidélité, premium)
│   │   ├── FlaamPill.kt          # Tags, pills
│   │   ├── FlaamBottomBar.kt     # Tab bar 5 onglets
│   │   ├── FlaamTopBar.kt        # Headers
│   │   ├── FlaamDialog.kt        # Modals (block, report, skip...)
│   │   ├── FlaamEmptyState.kt    # États vides
│   │   ├── FlaamLoadingSkeleton.kt # Skeleton screens
│   │   ├── FlaamNetworkBanner.kt # Bannière offline/slow
│   │   ├── FlaamProgressBar.kt   # Barre de progression onboarding
│   │   └── FlaamPhotoViewer.kt   # Viewer photos avec pagination
│   │
│   └── animation/                # Animations
│       ├── MatchAnimation.kt     # Animation de match (cœurs, photos overlap)
│       ├── LikeAnimation.kt      # Heart spring animation sur le bouton like
│       ├── CheckInAnimation.kt   # Confirmation check-in
│       └── TransitionSpecs.kt    # Ease-out, ease-in, spring configs
│
└── build-logic/                  # Convention plugins Gradle
    └── convention/
```

---

## 3. Architecture pattern — Clean Architecture + MVVM

```
┌──────────────────────────────────────────────────────────────┐
│                     feature/:screen/                         │
│  ┌─────────────┐   ┌───────────────┐   ┌──────────────────┐ │
│  │  Composable  │──→│  ViewModel    │──→│  UiState         │ │
│  │  (Screen)    │←──│  (StateFlow)  │   │  (data class)    │ │
│  └─────────────┘   └───────┬───────┘   └──────────────────┘ │
│                            │                                  │
│         ┌──────────────────┼──────────────────┐               │
│         │  collectAsState  │  UiEvent sealed   │               │
│         │                  │  class            │               │
└─────────┼──────────────────┼──────────────────┼───────────────┘
          │                  │                  │
┌─────────┼──────────────────┼──────────────────┼───────────────┐
│         │    domain/usecase/                   │               │
│         │  ┌───────────────┴──────────────┐   │               │
│         │  │  UseCase                     │   │               │
│         │  │  (suspend operator fun invoke)│   │               │
│         │  └───────────────┬──────────────┘   │               │
│         │                  │                  │               │
│         │  ┌───────────────┴──────────────┐   │               │
│         │  │  Repository (interface)      │   │               │
│         │  └───────────────┬──────────────┘   │               │
└─────────┼──────────────────┼──────────────────┼───────────────┘
          │                  │                  │
┌─────────┼──────────────────┼──────────────────┼───────────────┐
│         │    data/repository/                  │               │
│         │  ┌───────────────┴──────────────┐   │               │
│         │  │  RepositoryImpl              │   │               │
│         │  │  (single source of truth)    │   │               │
│         │  └──┬──────────┬───────────┬────┘   │               │
│         │     │          │           │        │               │
│     ┌───┴──┐ ┌┴────────┐ ┌┴────────┐         │               │
│     │ Room │ │ Retrofit │ │DataStore│         │               │
│     │ (DB) │ │ (API)    │ │ (prefs) │         │               │
│     └──────┘ └──────────┘ └─────────┘         │               │
└───────────────────────────────────────────────────────────────┘
```

```kotlin
// ── Pattern : chaque écran a exactement ces 3 composants ──

// 1. UiState — l'état complet de l'écran (immutable data class)
data class FeedUiState(
    val profiles: List<ProfileUi> = emptyList(),
    val currentIndex: Int = 0,
    val likesRemaining: Int = 5,
    val isLoading: Boolean = true,
    val error: String? = null,
    val isOffline: Boolean = false,
    val matchAnimation: MatchAnimationState? = null,
)

// 2. UiEvent — les actions de l'utilisateur (sealed interface)
sealed interface FeedUiEvent {
    data class Like(val profileId: String) : FeedUiEvent
    data class Skip(val profileId: String, val reason: String?) : FeedUiEvent
    data object NextProfile : FeedUiEvent
    data object RefreshFeed : FeedUiEvent
    data object DismissMatch : FeedUiEvent
}

// 3. ViewModel — transforme les events en state updates
@HiltViewModel
class FeedViewModel @Inject constructor(
    private val getDailyFeed: GetDailyFeedUseCase,
    private val likeProfile: LikeProfileUseCase,
    private val skipProfile: SkipProfileUseCase,
    private val networkMonitor: NetworkMonitor,
) : ViewModel() {

    private val _state = MutableStateFlow(FeedUiState())
    val state: StateFlow<FeedUiState> = _state.asStateFlow()

    init {
        loadFeed()
        observeNetwork()
    }

    fun onEvent(event: FeedUiEvent) {
        when (event) {
            is FeedUiEvent.Like -> handleLike(event.profileId)
            is FeedUiEvent.Skip -> handleSkip(event.profileId, event.reason)
            is FeedUiEvent.NextProfile -> nextProfile()
            is FeedUiEvent.RefreshFeed -> loadFeed()
            is FeedUiEvent.DismissMatch -> dismissMatch()
        }
    }

    private fun loadFeed() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            getDailyFeed()
                .onSuccess { profiles ->
                    _state.update { it.copy(
                        profiles = profiles.map { p -> p.toUi() },
                        isLoading = false,
                        currentIndex = 0,
                    )}
                }
                .onFailure { error ->
                    _state.update { it.copy(
                        isLoading = false,
                        error = error.toUserMessage(),
                    )}
                }
        }
    }

    private fun observeNetwork() {
        viewModelScope.launch {
            networkMonitor.status.collect { status ->
                _state.update { it.copy(isOffline = status == NetworkStatus.OFFLINE) }
            }
        }
    }

    // ... handleLike, handleSkip, nextProfile, dismissMatch
}
```

---

## 4. Dependency Injection (Hilt)

```kotlin
// ── Module réseau ──
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor,
        idempotencyInterceptor: IdempotencyInterceptor,
        connectivityInterceptor: ConnectivityInterceptor,
        certificatePinner: CertificatePinner,
    ): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(authInterceptor)         // Ajoute le JWT à chaque requête
        .addInterceptor(idempotencyInterceptor)  // Ajoute X-Idempotency-Key
        .addInterceptor(connectivityInterceptor) // Vérifie le réseau avant d'envoyer
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) Level.BODY else Level.NONE
        })
        .certificatePinner(certificatePinner.build())
        .connectTimeout(15, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .retryOnConnectionFailure(true)
        .build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit = Retrofit.Builder()
        .baseUrl(FlaamConstants.API_BASE_URL)
        .client(client)
        .addConverterFactory(MoshiConverterFactory.create(
            Moshi.Builder()
                .add(KotlinJsonAdapterFactory())
                .add(UuidAdapter())
                .add(InstantAdapter())
                .build()
        ))
        .build()

    @Provides
    @Singleton
    fun provideAuthApi(retrofit: Retrofit): AuthApi =
        retrofit.create(AuthApi::class.java)

    @Provides
    @Singleton
    fun provideFeedApi(retrofit: Retrofit): FeedApi =
        retrofit.create(FeedApi::class.java)

    // ... ChatApi, SpotApi, EventApi, ProfileApi, SafetyApi, SubscriptionApi
}


// ── Module base de données locale ──
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): FlaamDatabase =
        Room.databaseBuilder(context, FlaamDatabase::class.java, "flaam.db")
            .addMigrations(*FlaamMigrations.ALL)
            .fallbackToDestructiveMigration()  // En dev seulement
            .build()

    @Provides
    fun provideProfileDao(db: FlaamDatabase): ProfileDao = db.profileDao()

    @Provides
    fun provideMessageDao(db: FlaamDatabase): MessageDao = db.messageDao()

    @Provides
    fun provideSpotDao(db: FlaamDatabase): SpotDao = db.spotDao()

    // ... MatchDao, EventDao, SyncQueueDao
}


// ── Module repositories ──
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository

    @Binds
    @Singleton
    abstract fun bindFeedRepository(impl: FeedRepositoryImpl): FeedRepository

    @Binds
    @Singleton
    abstract fun bindChatRepository(impl: ChatRepositoryImpl): ChatRepository

    // ... SpotRepository, EventRepository, ProfileRepository, SafetyRepository
}
```

---

## 5. Data Layer — Room Database

```kotlin
// ── Entities Room (ce qui est stocké localement) ──

@Entity(tableName = "profiles")
data class ProfileEntity(
    @PrimaryKey val id: String,
    val firstName: String,
    val age: Int,
    val sector: String,
    val intention: String,
    val geoScore: Float,              // Score géo calculé par le backend
    val commonSpotsCount: Int,
    val isVerified: Boolean,
    val photoUrls: List<String>,      // Converties en JSON
    val thumbnailUrl: String,         // Première photo en thumbnail
    val dominantColor: String,        // Pour le placeholder
    val prompts: List<PromptResponse>, // JSON
    val tags: List<String>,            // JSON
    val quartierName: String,
    val languages: List<String>,       // JSON
    val feedDate: String,              // "2026-04-14" — le jour où ce profil a été recommandé
    val position: Int,                 // Position dans le feed du jour
    val viewedAt: Long? = null,        // Quand l'utilisateur a vu ce profil
    val likedAt: Long? = null,         // Quand l'utilisateur a liké
    val skippedAt: Long? = null,       // Quand l'utilisateur a skippé
    val cachedAt: Long = System.currentTimeMillis(),
)

@Entity(tableName = "messages")
data class MessageEntity(
    @PrimaryKey val id: String,
    val matchId: String,
    val senderId: String,
    val content: String,
    val type: String,                  // "text", "voice", "meetup_proposal"
    val clientMessageId: String,       // Pour déduplication
    val status: String,                // "sending", "sent", "delivered", "read"
    val createdAt: Long,
    val serverTimestamp: Long? = null,
)

@Entity(tableName = "matches")
data class MatchEntity(
    @PrimaryKey val id: String,
    val otherUserId: String,
    val otherUserName: String,
    val otherUserPhotoUrl: String,
    val commonSpots: List<String>,     // JSON
    val matchedAt: Long,
    val expiresAt: Long,
    val lastMessageContent: String? = null,
    val lastMessageAt: Long? = null,
    val unreadCount: Int = 0,
    val isOnline: Boolean = false,
)

@Entity(tableName = "spots")
data class SpotEntity(
    @PrimaryKey val id: String,
    val name: String,
    val category: String,
    val quartierName: String,
    val iconEmoji: String,
    val userCount: Int,
    val userLevel: String?,            // "habitue", "regulier", "confirme", "declare"
    val checkInCount: Int = 0,
    val lastCheckInAt: Long? = null,
)

@Entity(tableName = "sync_queue")
data class SyncQueueEntity(
    @PrimaryKey val actionId: String,  // UUID généré côté client
    val type: String,                  // "like", "skip", "message", "check_in", "read_receipt"
    val payload: String,               // JSON sérialisé
    val createdAt: Long,
    val retryCount: Int = 0,
    val lastRetryAt: Long? = null,
    val status: String = "pending",    // "pending", "syncing", "synced", "failed"
)


// ── DAOs ──

@Dao
interface ProfileDao {
    @Query("SELECT * FROM profiles WHERE feedDate = :date ORDER BY position ASC")
    fun getFeedForDate(date: String): Flow<List<ProfileEntity>>

    @Query("SELECT * FROM profiles WHERE id = :id")
    suspend fun getById(id: String): ProfileEntity?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(profiles: List<ProfileEntity>)

    @Query("UPDATE profiles SET likedAt = :timestamp WHERE id = :id")
    suspend fun markLiked(id: String, timestamp: Long)

    @Query("UPDATE profiles SET skippedAt = :timestamp WHERE id = :id")
    suspend fun markSkipped(id: String, timestamp: Long)

    @Query("DELETE FROM profiles WHERE feedDate < :date")
    suspend fun deleteOlderThan(date: String)

    @Query("SELECT COUNT(*) FROM profiles WHERE feedDate = :date AND likedAt IS NOT NULL")
    fun getLikeCountForDate(date: String): Flow<Int>
}

@Dao
interface MessageDao {
    @Query("""
        SELECT * FROM messages WHERE matchId = :matchId
        ORDER BY createdAt ASC LIMIT :limit OFFSET :offset
    """)
    fun getMessages(matchId: String, limit: Int = 50, offset: Int = 0): Flow<List<MessageEntity>>

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insert(message: MessageEntity)

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertAll(messages: List<MessageEntity>)

    @Query("UPDATE messages SET status = :status WHERE id = :id")
    suspend fun updateStatus(id: String, status: String)

    @Query("UPDATE messages SET status = :status WHERE clientMessageId = :clientId")
    suspend fun updateStatusByClientId(clientId: String, status: String)

    @Query("SELECT * FROM messages WHERE matchId = :matchId ORDER BY createdAt DESC LIMIT 1")
    suspend fun getLastMessage(matchId: String): MessageEntity?
}

@Dao
interface SyncQueueDao {
    @Query("SELECT * FROM sync_queue WHERE status = 'pending' ORDER BY createdAt ASC")
    suspend fun getPendingActions(): List<SyncQueueEntity>

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun enqueue(action: SyncQueueEntity)

    @Query("UPDATE sync_queue SET status = 'synced' WHERE actionId = :id")
    suspend fun markSynced(id: String)

    @Query("UPDATE sync_queue SET status = 'syncing', retryCount = retryCount + 1, lastRetryAt = :now WHERE actionId = :id")
    suspend fun markSyncing(id: String, now: Long = System.currentTimeMillis())

    @Query("DELETE FROM sync_queue WHERE status = 'synced'")
    suspend fun clearSynced()

    @Query("SELECT COUNT(*) FROM sync_queue WHERE status = 'pending'")
    fun getPendingCount(): Flow<Int>
}


// ── Room Database ──

@Database(
    entities = [
        ProfileEntity::class,
        MessageEntity::class,
        MatchEntity::class,
        SpotEntity::class,
        SyncQueueEntity::class,
    ],
    version = 1,
    exportSchema = true,
)
@TypeConverters(FlaamConverters::class)
abstract class FlaamDatabase : RoomDatabase() {
    abstract fun profileDao(): ProfileDao
    abstract fun messageDao(): MessageDao
    abstract fun matchDao(): MatchDao
    abstract fun spotDao(): SpotDao
    abstract fun syncQueueDao(): SyncQueueDao
}
```

---

## 6. Data Layer — API Retrofit

```kotlin
// ── Auth API ──
interface AuthApi {
    @POST("auth/otp/request")
    suspend fun requestOtp(@Body body: OtpRequestBody): Response<OtpResponse>

    @POST("auth/otp/verify")
    suspend fun verifyOtp(@Body body: OtpVerifyBody): Response<AuthTokenResponse>

    @POST("auth/refresh")
    suspend fun refreshToken(@Body body: RefreshTokenBody): Response<AuthTokenResponse>
}

// ── Feed API ──
interface FeedApi {
    @GET("feed/daily")
    suspend fun getDailyFeed(): Response<DailyFeedResponse>

    @POST("feed/{profileId}/like")
    suspend fun likeProfile(
        @Path("profileId") profileId: String,
        @Header("X-Idempotency-Key") idempotencyKey: String,
    ): Response<LikeResponse>

    @POST("feed/{profileId}/skip")
    suspend fun skipProfile(
        @Path("profileId") profileId: String,
        @Body body: SkipBody?,
        @Header("X-Idempotency-Key") idempotencyKey: String,
    ): Response<Unit>
}

// ── Chat API (REST, pour l'historique — le temps réel est via WebSocket) ──
interface ChatApi {
    @GET("matches")
    suspend fun getMatches(): Response<MatchesResponse>

    @GET("matches/{matchId}/messages")
    suspend fun getMessages(
        @Path("matchId") matchId: String,
        @Query("before") before: String? = null,
        @Query("limit") limit: Int = 50,
    ): Response<MessagesResponse>

    @POST("matches/{matchId}/messages")
    suspend fun sendMessage(
        @Path("matchId") matchId: String,
        @Body body: SendMessageBody,
        @Header("X-Idempotency-Key") idempotencyKey: String,
    ): Response<MessageResponse>

    @POST("matches/{matchId}/meetup")
    suspend fun proposeMeetup(
        @Path("matchId") matchId: String,
        @Body body: MeetupProposalBody,
    ): Response<MeetupResponse>
}

// ── Spots API ──
interface SpotApi {
    @GET("spots/mine")
    suspend fun getMySpots(): Response<MySpotsResponse>

    @GET("spots/{spotId}")
    suspend fun getSpotDetail(@Path("spotId") spotId: String): Response<SpotDetailResponse>

    @POST("spots/{spotId}/checkin")
    suspend fun checkIn(
        @Path("spotId") spotId: String,
        @Header("X-Idempotency-Key") idempotencyKey: String,
    ): Response<CheckInResponse>

    @GET("spots/search")
    suspend fun searchSpots(@Query("q") query: String): Response<SpotSearchResponse>
}

// ── Profile API ──
interface ProfileApi {
    @GET("profiles/me")
    suspend fun getMyProfile(): Response<MyProfileResponse>

    @PATCH("profiles/me")
    suspend fun updateProfile(@Body body: UpdateProfileBody): Response<MyProfileResponse>

    @Multipart
    @POST("profiles/me/photos")
    suspend fun uploadPhoto(
        @Part photo: MultipartBody.Part,
        @Part("position") position: RequestBody,
    ): Response<PhotoResponse>

    @DELETE("profiles/me/photos/{photoId}")
    suspend fun deletePhoto(@Path("photoId") photoId: String): Response<Unit>
}

// ── Events API ──
interface EventApi {
    @GET("events")
    suspend fun getEvents(
        @Query("city_id") cityId: String,
        @Query("week") week: String? = null,
    ): Response<EventsResponse>

    @POST("events/{eventId}/register")
    suspend fun registerEvent(
        @Path("eventId") eventId: String,
    ): Response<EventRegistrationResponse>
}

// ── Subscription API ──
interface SubscriptionApi {
    @GET("subscriptions/me")
    suspend fun getSubscription(): Response<SubscriptionResponse>

    @POST("subscriptions/initialize")
    suspend fun initializePayment(
        @Body body: InitPaymentBody,
    ): Response<PaymentInitResponse>
}

// ── Safety API ──
interface SafetyApi {
    @POST("safety/share-date")
    suspend fun shareDate(@Body body: ShareDateBody): Response<Unit>

    @POST("safety/emergency")
    suspend fun triggerEmergency(@Body body: EmergencyBody): Response<Unit>

    @POST("safety/timer/start")
    suspend fun startTimer(@Body body: TimerBody): Response<Unit>

    @POST("safety/timer/cancel")
    suspend fun cancelTimer(): Response<Unit>
}
```

---

## 7. Data Layer — Offline-first Repository Pattern

```kotlin
/**
 * PRINCIPE : Le repository est le "single source of truth".
 *
 * Pour chaque donnée, la stratégie est :
 * 1. Retourner les données locales (Room) immédiatement
 * 2. Fetch les données fraîches du serveur en background
 * 3. Mettre à jour Room avec les données fraîches
 * 4. Le Flow Room émet automatiquement la mise à jour
 *
 * Pour les ACTIONS (like, message, check-in) :
 * 1. Écrire dans Room immédiatement (optimistic update)
 * 2. Enqueue dans la SyncQueue
 * 3. Tenter d'envoyer au serveur
 * 4. Si succès → marquer synced
 * 5. Si échec réseau → restera dans la queue, SyncWorker réessaiera
 */

class FeedRepositoryImpl @Inject constructor(
    private val feedApi: FeedApi,
    private val profileDao: ProfileDao,
    private val syncQueueDao: SyncQueueDao,
    private val networkMonitor: NetworkMonitor,
) : FeedRepository {

    override fun getDailyFeed(): Flow<List<Profile>> {
        val today = LocalDate.now().toString()

        return profileDao.getFeedForDate(today)
            .map { entities -> entities.map { it.toDomain() } }
            .onStart {
                // En parallèle, fetch le serveur si online
                if (networkMonitor.isOnline()) {
                    try {
                        val response = feedApi.getDailyFeed()
                        if (response.isSuccessful) {
                            val profiles = response.body()!!.profiles
                                .mapIndexed { index, dto -> dto.toEntity(today, index) }
                            profileDao.insertAll(profiles)
                            // Le Flow Room émet automatiquement
                        }
                    } catch (e: Exception) {
                        Timber.w(e, "Failed to fetch feed from server, using cache")
                    }
                }
            }
    }

    override suspend fun likeProfile(profileId: String): Result<LikeResult> {
        // 1. Optimistic update local
        profileDao.markLiked(profileId, System.currentTimeMillis())

        // 2. Enqueue pour sync
        val actionId = UUID.randomUUID().toString()
        syncQueueDao.enqueue(SyncQueueEntity(
            actionId = actionId,
            type = "like",
            payload = """{"profile_id": "$profileId"}""",
            createdAt = System.currentTimeMillis(),
        ))

        // 3. Tenter d'envoyer maintenant
        return try {
            val response = feedApi.likeProfile(profileId, idempotencyKey = actionId)
            if (response.isSuccessful) {
                syncQueueDao.markSynced(actionId)
                val body = response.body()!!
                if (body.isMatch) {
                    Result.success(LikeResult.Match(body.matchId!!, body.iceBreaker))
                } else {
                    Result.success(LikeResult.Liked)
                }
            } else {
                Result.failure(ApiException(response.code(), response.message()))
            }
        } catch (e: Exception) {
            // Pas de réseau → le like est en queue, sera sync plus tard
            Timber.w(e, "Like queued for sync")
            Result.success(LikeResult.Queued)
        }
    }
}


// ── SyncWorker : synchronise la queue quand le réseau revient ──

@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val syncQueueDao: SyncQueueDao,
    private val feedApi: FeedApi,
    private val chatApi: ChatApi,
    private val spotApi: SpotApi,
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val pending = syncQueueDao.getPendingActions()

        if (pending.isEmpty()) return Result.success()

        var allSynced = true

        for (action in pending) {
            if (action.retryCount >= 5) {
                // Trop de retries → abandonner
                syncQueueDao.markSynced(action.actionId)
                continue
            }

            syncQueueDao.markSyncing(action.actionId)

            val success = when (action.type) {
                "like" -> syncLike(action)
                "skip" -> syncSkip(action)
                "message" -> syncMessage(action)
                "check_in" -> syncCheckIn(action)
                "read_receipt" -> syncReadReceipt(action)
                else -> {
                    Timber.w("Unknown sync action type: ${action.type}")
                    true // Ignorer les types inconnus
                }
            }

            if (success) {
                syncQueueDao.markSynced(action.actionId)
            } else {
                allSynced = false
            }
        }

        // Nettoyer les actions synced
        syncQueueDao.clearSynced()

        return if (allSynced) Result.success() else Result.retry()
    }

    private suspend fun syncLike(action: SyncQueueEntity): Boolean {
        return try {
            val payload = Moshi.parse<LikePayload>(action.payload)
            val response = feedApi.likeProfile(
                payload.profileId,
                idempotencyKey = action.actionId,
            )
            response.isSuccessful || response.code() == 409 // 409 = déjà traité
        } catch (e: Exception) {
            false
        }
    }

    // ... syncSkip, syncMessage, syncCheckIn, syncReadReceipt
}


// ── Enregistrement du SyncWorker ──

@Module
@InstallIn(SingletonComponent::class)
object WorkerModule {

    /**
     * Le SyncWorker est déclenché quand :
     * 1. Le réseau revient (NetworkType.CONNECTED constraint)
     * 2. Toutes les 15 minutes en background (periodic)
     * 3. Manuellement quand on enqueue une action (one-time)
     */
    fun scheduleSyncWorker(context: Context) {
        // Periodic : toutes les 15 min quand le réseau est dispo
        val periodicRequest = PeriodicWorkRequestBuilder<SyncWorker>(
            15, TimeUnit.MINUTES,
        ).setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        ).build()

        WorkManager.getInstance(context).enqueueUniquePeriodicWork(
            "flaam_sync",
            ExistingPeriodicWorkPolicy.KEEP,
            periodicRequest,
        )
    }

    fun triggerImmediateSync(context: Context) {
        val oneTimeRequest = OneTimeWorkRequestBuilder<SyncWorker>()
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            )
            .build()

        WorkManager.getInstance(context).enqueue(oneTimeRequest)
    }
}
```

---

## 8. Core — Network Monitor

```kotlin
/**
 * Détecte 3 états réseau : ONLINE, SLOW, OFFLINE
 *
 * Utilisé par :
 * - Les repositories (décider si on fetch le serveur)
 * - L'UI (afficher la bannière offline/slow)
 * - Le WebSocket (déclencher la reconnexion)
 */

@Singleton
class NetworkMonitor @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    private val connectivityManager =
        context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager

    private val _status = MutableStateFlow(NetworkStatus.ONLINE)
    val status: StateFlow<NetworkStatus> = _status.asStateFlow()

    fun isOnline(): Boolean = _status.value != NetworkStatus.OFFLINE

    init {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                _status.value = NetworkStatus.ONLINE
            }

            override fun onLost(network: Network) {
                _status.value = NetworkStatus.OFFLINE
            }

            override fun onCapabilitiesChanged(
                network: Network,
                capabilities: NetworkCapabilities,
            ) {
                val downKbps = capabilities.linkDownstreamBandwidthKbps
                _status.value = when {
                    downKbps < 50 -> NetworkStatus.SLOW   // < 50 KB/s = lent
                    else -> NetworkStatus.ONLINE
                }
            }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()

        connectivityManager.registerNetworkCallback(request, callback)
    }
}

enum class NetworkStatus { ONLINE, SLOW, OFFLINE }
```

---

## 9. Core — WebSocket Client (Chat temps réel)

```kotlin
/**
 * Client WebSocket avec reconnexion automatique.
 * Protocole défini dans backend-spec section 37.
 *
 * Responsabilités :
 * - Maintenir la connexion WS
 * - Reconnexion avec backoff exponentiel
 * - Sync des messages manqués au retour
 * - Déduplication par clientMessageId
 * - Heartbeat ping/pong toutes les 30s
 */

@Singleton
class FlaamWebSocket @Inject constructor(
    private val authDataStore: AuthDataStore,
    private val messageDao: MessageDao,
    private val matchDao: MatchDao,
    private val networkMonitor: NetworkMonitor,
) {
    private var client: HttpClient? = null
    private var session: DefaultClientWebSocketSession? = null
    private var reconnectJob: Job? = null

    private val _incomingMessages = MutableSharedFlow<WsEvent>(extraBufferCapacity = 64)
    val incomingMessages: SharedFlow<WsEvent> = _incomingMessages.asSharedFlow()

    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    // ── Backoff config ──
    private var retryDelay = 1000L
    private val maxDelay = 30_000L
    private val backoffFactor = 2.0
    private val jitter = 500L

    suspend fun connect() {
        val token = authDataStore.getAccessToken() ?: return

        client = HttpClient(OkHttp) {
            install(WebSockets)
        }

        try {
            client!!.webSocket(
                urlString = "${FlaamConstants.WS_BASE_URL}/ws/chat?token=$token"
            ) {
                session = this
                retryDelay = 1000L // Reset backoff on success

                // Sync messages manqués
                syncMissedMessages()

                // Heartbeat
                val heartbeat = launch {
                    while (isActive) {
                        delay(30_000)
                        send(Frame.Text("""{"type":"ping"}"""))
                    }
                }

                // Écoute des messages entrants
                for (frame in incoming) {
                    if (frame is Frame.Text) {
                        handleIncomingFrame(frame.readText())
                    }
                }

                heartbeat.cancel()
            }
        } catch (e: Exception) {
            Timber.w(e, "WebSocket connection failed")
        }

        // Connexion perdue → reconnexion
        scheduleReconnect()
    }

    fun disconnect() {
        reconnectJob?.cancel()
        scope.launch {
            session?.close()
            client?.close()
        }
    }

    suspend fun sendMessage(matchId: String, content: String, clientMessageId: String) {
        val payload = buildJsonObject {
            put("type", "message")
            put("match_id", matchId)
            put("content", content)
            put("client_message_id", clientMessageId)
        }

        try {
            session?.send(Frame.Text(payload.toString()))
        } catch (e: Exception) {
            Timber.w(e, "Failed to send via WS, will sync via REST")
        }
    }

    suspend fun sendReadReceipt(matchId: String, lastReadId: String) {
        val payload = buildJsonObject {
            put("type", "read")
            put("match_id", matchId)
            put("last_read_id", lastReadId)
        }
        try {
            session?.send(Frame.Text(payload.toString()))
        } catch (_: Exception) { }
    }

    private suspend fun syncMissedMessages() {
        // Récupérer le dernier message reçu localement
        // et demander au serveur tout ce qui est arrivé après
        val lastMsg = messageDao.getLastReceivedMessage()
        val syncPayload = buildJsonObject {
            put("type", "sync")
            put("last_message_id", lastMsg?.id ?: "")
        }
        session?.send(Frame.Text(syncPayload.toString()))
    }

    private suspend fun handleIncomingFrame(text: String) {
        val json = Json.parseToJsonElement(text).jsonObject
        val type = json["type"]?.jsonPrimitive?.content ?: return

        when (type) {
            "message" -> {
                val msg = Json.decodeFromJsonElement<WsMessage>(json)
                // Déduplication
                val existing = messageDao.getByClientId(msg.clientMessageId)
                if (existing == null) {
                    messageDao.insert(msg.toEntity())
                    _incomingMessages.emit(WsEvent.NewMessage(msg))
                }
            }
            "ack" -> {
                val msgId = json["message_id"]?.jsonPrimitive?.content ?: return
                val status = json["status"]?.jsonPrimitive?.content ?: return
                messageDao.updateStatus(msgId, status)
                _incomingMessages.emit(WsEvent.Ack(msgId, status))
            }
            "delivered" -> {
                val msgId = json["message_id"]?.jsonPrimitive?.content ?: return
                messageDao.updateStatus(msgId, "delivered")
                _incomingMessages.emit(WsEvent.Delivered(msgId))
            }
            "read" -> {
                val matchId = json["match_id"]?.jsonPrimitive?.content ?: return
                val lastReadId = json["last_read_id"]?.jsonPrimitive?.content ?: return
                _incomingMessages.emit(WsEvent.Read(matchId, lastReadId))
            }
            "sync_response" -> {
                val messages = Json.decodeFromJsonElement<List<WsMessage>>(json["messages"]!!)
                messageDao.insertAll(messages.map { it.toEntity() })
                _incomingMessages.emit(WsEvent.SyncComplete(messages.size))
            }
            "pong" -> { /* Heartbeat OK */ }
        }
    }

    private fun scheduleReconnect() {
        reconnectJob?.cancel()
        reconnectJob = scope.launch {
            delay(retryDelay + (-jitter..jitter).random())
            retryDelay = (retryDelay * backoffFactor).toLong().coerceAtMost(maxDelay)

            // Attendre que le réseau soit dispo
            networkMonitor.status.first { it != NetworkStatus.OFFLINE }
            connect()
        }
    }
}

sealed interface WsEvent {
    data class NewMessage(val message: WsMessage) : WsEvent
    data class Ack(val messageId: String, val status: String) : WsEvent
    data class Delivered(val messageId: String) : WsEvent
    data class Read(val matchId: String, val lastReadId: String) : WsEvent
    data class SyncComplete(val count: Int) : WsEvent
}
```

---

## 10. Navigation Graph

```kotlin
/**
 * Graphe de navigation complet de l'app.
 * Single Activity (MainActivity) + Jetpack Navigation Compose.
 *
 * Structure :
 * - AuthGraph     → Welcome, Phone, OTP
 * - OnboardingGraph → City, Info, Selfie, Photos, Quartiers, Intention,
 *                     Sector, Prompts, Tags, Spots, Done
 * - MainGraph     → 5 onglets (Feed, Spots, Profile, Chat, Events)
 *                   + écrans de détail (ProfileDetail, SpotDetail, EventDetail,
 *                     Conversation, EditProfile, Settings, etc.)
 * - SafetyGraph   → ShareDate, Timer, Emergency (accessible depuis Chat)
 */

// ── Routes (type-safe) ──

sealed class Route(val path: String) {
    // Auth
    data object Welcome : Route("welcome")
    data object Phone : Route("phone")
    data object Otp : Route("otp/{phone}") {
        fun create(phone: String) = "otp/$phone"
    }

    // Onboarding
    data object CitySelection : Route("onboarding/city")
    data object BasicInfo : Route("onboarding/info")
    data object SelfieVerification : Route("onboarding/selfie")
    data object PhotoUpload : Route("onboarding/photos")
    data object QuartierSelection : Route("onboarding/quartiers")
    data object IntentionSelection : Route("onboarding/intention")
    data object SectorSelection : Route("onboarding/sector")
    data object PromptAnswers : Route("onboarding/prompts")
    data object TagSelection : Route("onboarding/tags")
    data object SpotSelection : Route("onboarding/spots")
    data object OnboardingDone : Route("onboarding/done")

    // Main tabs
    data object Feed : Route("main/feed")
    data object Spots : Route("main/spots")
    data object MyProfile : Route("main/profile")
    data object ChatList : Route("main/chat")
    data object Events : Route("main/events")

    // Detail screens
    data object ProfileDetail : Route("profile/{profileId}") {
        fun create(id: String) = "profile/$id"
    }
    data object SpotDetail : Route("spot/{spotId}") {
        fun create(id: String) = "spot/$id"
    }
    data object EventDetail : Route("event/{eventId}") {
        fun create(id: String) = "event/$id"
    }
    data object Conversation : Route("chat/{matchId}") {
        fun create(id: String) = "chat/$id"
    }

    // Settings
    data object EditProfile : Route("settings/edit-profile")
    data object Settings : Route("settings")
    data object DiscoveryPrefs : Route("settings/discovery")
    data object Subscription : Route("settings/subscription")
    data object NotifPrefs : Route("settings/notifications")
    data object ContactBlacklist : Route("settings/contacts")

    // Safety
    data object ShareDate : Route("safety/share/{matchId}") {
        fun create(id: String) = "safety/share/$id"
    }
    data object SafetyTimer : Route("safety/timer")
    data object Emergency : Route("safety/emergency")

    // Overlays (dialogs)
    data object PremiumUpsell : Route("premium")
    data object AccountDeletion : Route("settings/delete-account")
}


// ── Deep linking scheme ──

/**
 * flaam://feed              → Feed principal
 * flaam://chat/{matchId}    → Conversation spécifique
 * flaam://profile/{userId}  → Profil détaillé
 * flaam://event/{eventId}   → Détail d'un event
 * flaam://spot/{spotId}     → Détail d'un spot
 * flaam://premium           → Upsell premium
 * flaam://safety/timer      → Timer actif
 *
 * Utilisé par :
 * - Push notifications (tap → deep link)
 * - Liens partagés
 * - La bannière "paiement confirmé"
 */

val flaamDeepLinks = listOf(
    navDeepLink { uriPattern = "flaam://feed" },
    navDeepLink { uriPattern = "flaam://chat/{matchId}" },
    navDeepLink { uriPattern = "flaam://profile/{profileId}" },
    navDeepLink { uriPattern = "flaam://event/{eventId}" },
    navDeepLink { uriPattern = "flaam://spot/{spotId}" },
    navDeepLink { uriPattern = "flaam://premium" },
    navDeepLink { uriPattern = "flaam://safety/timer" },
)
```

---

## 11. UI — Design System en Compose

```kotlin
// ── Couleurs ──

object FlaamColors {
    // Primary
    val Terracotta = Color(0xFFD85A30)
    val CoralLight = Color(0xFFE8784A)
    val Brick = Color(0xFFB8462A)
    val Peach = Color(0xFFFFF3E0)

    // Secondary
    val Teal = Color(0xFF0F6E56)
    val Green = Color(0xFF1D9E75)
    val Mint = Color(0xFFE8F5E9)

    // Neutral (warm, pas de gris froid)
    val Cream = Color(0xFFFAFAF8)
    val Sand = Color(0xFFF5F0E8)
    val Stone = Color(0xFFEDEBE4)
    val Pebble = Color(0xFFD3D1C7)
    val Ash = Color(0xFF8A8A80)
    val Charcoal = Color(0xFF5A5A50)
    val Night = Color(0xFF1A1A1A)

    // Semantic
    val Success = Color(0xFF2E7D32)
    val Warning = Color(0xFFEF9F27)
    val Danger = Color(0xFFD32F2F)
    val Info = Color(0xFF1565C0)

    // Dark mode variants
    val NightSurface = Color(0xFF121212)
    val NightCard = Color(0xFF1E1E1E)
    val NightBorder = Color(0xFF333333)
}


// ── Typographie ──

val InstrumentSerif = FontFamily(
    Font(R.font.instrument_serif_regular, FontWeight.Normal),
    Font(R.font.instrument_serif_italic, FontWeight.Normal, FontStyle.Italic),
)

val DmSans = FontFamily(
    Font(R.font.dm_sans_regular, FontWeight.Normal),
    Font(R.font.dm_sans_medium, FontWeight.Medium),
    Font(R.font.dm_sans_semibold, FontWeight.SemiBold),
    Font(R.font.dm_sans_bold, FontWeight.Bold),
)

val FlaamTypography = Typography(
    // Display — Instrument Serif, uniquement pour le logo et les headers d'onboarding
    displayLarge = TextStyle(
        fontFamily = InstrumentSerif,
        fontSize = 56.sp,
        letterSpacing = (-2).sp,
    ),
    displayMedium = TextStyle(
        fontFamily = InstrumentSerif,
        fontSize = 32.sp,
        letterSpacing = (-0.5).sp,
    ),
    displaySmall = TextStyle(
        fontFamily = InstrumentSerif,
        fontSize = 26.sp,
    ),

    // H1-H3 — DM Sans
    headlineLarge = TextStyle(
        fontFamily = DmSans,
        fontWeight = FontWeight.Bold,
        fontSize = 28.sp,
    ),
    headlineMedium = TextStyle(
        fontFamily = DmSans,
        fontWeight = FontWeight.SemiBold,
        fontSize = 22.sp,
    ),
    headlineSmall = TextStyle(
        fontFamily = DmSans,
        fontWeight = FontWeight.SemiBold,
        fontSize = 17.sp,
    ),

    // Body
    bodyLarge = TextStyle(
        fontFamily = DmSans,
        fontWeight = FontWeight.Normal,
        fontSize = 15.sp,
    ),
    bodyMedium = TextStyle(
        fontFamily = DmSans,
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
    ),
    bodySmall = TextStyle(
        fontFamily = DmSans,
        fontWeight = FontWeight.Normal,
        fontSize = 12.sp,
    ),

    // Labels
    labelLarge = TextStyle(
        fontFamily = DmSans,
        fontWeight = FontWeight.SemiBold,
        fontSize = 14.sp,
    ),
    labelMedium = TextStyle(
        fontFamily = DmSans,
        fontWeight = FontWeight.Medium,
        fontSize = 12.sp,
    ),
    labelSmall = TextStyle(
        fontFamily = DmSans,
        fontWeight = FontWeight.SemiBold,
        fontSize = 10.sp,
        letterSpacing = 0.8.sp,
    ),
)


// ── Spacing ──

object FlaamSpacing {
    val xs = 4.dp
    val sm = 8.dp
    val md = 12.dp
    val lg = 16.dp
    val xl = 24.dp
    val xxl = 32.dp
}


// ── Shapes ──

val FlaamShapes = Shapes(
    small = RoundedCornerShape(6.dp),
    medium = RoundedCornerShape(12.dp),
    large = RoundedCornerShape(16.dp),
    extraLarge = RoundedCornerShape(24.dp),
)


// ── Theme ──

@Composable
fun FlaamTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit,
) {
    val colorScheme = if (darkTheme) {
        darkColorScheme(
            primary = FlaamColors.Terracotta,
            onPrimary = Color.White,
            secondary = FlaamColors.Teal,
            background = FlaamColors.NightSurface,
            surface = FlaamColors.NightCard,
            onBackground = Color.White,
            onSurface = Color.White,
            outline = FlaamColors.NightBorder,
        )
    } else {
        lightColorScheme(
            primary = FlaamColors.Terracotta,
            onPrimary = Color.White,
            secondary = FlaamColors.Teal,
            background = FlaamColors.Cream,
            surface = Color.White,
            onBackground = FlaamColors.Night,
            onSurface = FlaamColors.Night,
            outline = FlaamColors.Stone,
            surfaceVariant = FlaamColors.Sand,
        )
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = FlaamTypography,
        shapes = FlaamShapes,
        content = content,
    )
}
```

---

## 12. Core — Auth & Security

```kotlin
// ── Auth Interceptor : injecte le JWT dans chaque requête ──

@Singleton
class AuthInterceptor @Inject constructor(
    private val authDataStore: AuthDataStore,
    private val authApi: Lazy<AuthApi>,  // Lazy pour éviter la dépendance circulaire
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): okhttp3.Response {
        val originalRequest = chain.request()

        // Skip pour les endpoints auth (login, OTP, refresh)
        if (originalRequest.url.pathSegments.firstOrNull() == "auth") {
            return chain.proceed(originalRequest)
        }

        val token = runBlocking { authDataStore.getAccessToken() }

        val request = originalRequest.newBuilder()
            .header("Authorization", "Bearer $token")
            .build()

        val response = chain.proceed(request)

        // Si 401 → tenter un refresh
        if (response.code == 401) {
            response.close()
            val newToken = runBlocking { refreshToken() }
            if (newToken != null) {
                val retryRequest = originalRequest.newBuilder()
                    .header("Authorization", "Bearer $newToken")
                    .build()
                return chain.proceed(retryRequest)
            }
            // Refresh failed → forcer le logout
            runBlocking { authDataStore.clear() }
        }

        return response
    }

    private suspend fun refreshToken(): String? {
        val refreshToken = authDataStore.getRefreshToken() ?: return null
        return try {
            val response = authApi.get().refreshToken(RefreshTokenBody(refreshToken))
            if (response.isSuccessful) {
                val body = response.body()!!
                authDataStore.saveTokens(body.accessToken, body.refreshToken)
                body.accessToken
            } else null
        } catch (e: Exception) {
            null
        }
    }
}


// ── Idempotency Interceptor ──

class IdempotencyInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): okhttp3.Response {
        val request = chain.request()

        // Ajouter X-Idempotency-Key uniquement sur POST/PUT/PATCH
        if (request.method in listOf("POST", "PUT", "PATCH")) {
            // Si pas déjà présent (les actions queue-ées ont leur propre key)
            if (request.header("X-Idempotency-Key") == null) {
                val key = UUID.randomUUID().toString()
                val newRequest = request.newBuilder()
                    .header("X-Idempotency-Key", key)
                    .build()
                return chain.proceed(newRequest)
            }
        }

        return chain.proceed(request)
    }
}


// ── Encrypted Storage pour les tokens ──

@Singleton
class AuthDataStore @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    private val prefs = EncryptedSharedPreferences.create(
        context,
        "flaam_auth",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM,
    )

    suspend fun saveTokens(accessToken: String, refreshToken: String) {
        withContext(Dispatchers.IO) {
            prefs.edit()
                .putString("access_token", accessToken)
                .putString("refresh_token", refreshToken)
                .apply()
        }
    }

    suspend fun getAccessToken(): String? = withContext(Dispatchers.IO) {
        prefs.getString("access_token", null)
    }

    suspend fun getRefreshToken(): String? = withContext(Dispatchers.IO) {
        prefs.getString("refresh_token", null)
    }

    suspend fun clear() = withContext(Dispatchers.IO) {
        prefs.edit().clear().apply()
    }
}


// ── Device Fingerprint ──

@Singleton
class DeviceFingerprint @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    /**
     * Génère un fingerprint stable basé sur :
     * - Android ID (reset uniquement au factory reset)
     * - Modèle du device (ex: "Tecno Spark 10")
     * - Résolution écran
     *
     * Hashé en SHA-256 pour l'envoyer au serveur.
     */
    fun generate(): String {
        val androidId = Settings.Secure.getString(
            context.contentResolver,
            Settings.Secure.ANDROID_ID,
        )
        val model = "${Build.MANUFACTURER}_${Build.MODEL}"
        val screen = context.resources.displayMetrics.let {
            "${it.widthPixels}x${it.heightPixels}"
        }

        val raw = "$androidId|$model|$screen"
        return raw.sha256()
    }

    private fun String.sha256(): String {
        val digest = MessageDigest.getInstance("SHA-256")
        return digest.digest(toByteArray())
            .joinToString("") { "%02x".format(it) }
    }
}


// ── Certificate Pinning ──

object FlaamCertificatePinner {
    fun build(): CertificatePinner = CertificatePinner.Builder()
        .add(
            "api.flaam.app",
            "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=", // Pin principal
            "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=", // Pin backup
        )
        .build()
    // Les pins réels seront générés lors du déploiement du certificat TLS
}
```

---

## 13. Push Notifications (FCM)

```kotlin
/**
 * Firebase Cloud Messaging pour les push notifications.
 *
 * Types de notifications :
 * - new_match        → "Ama s'intéresse à toi !"     → deep link flaam://feed
 * - new_message      → "Ama : Hey, on se voit ?"     → deep link flaam://chat/{matchId}
 * - daily_feed       → "Tes profils du jour"          → deep link flaam://feed
 * - match_expiring   → "Ton match expire demain"      → deep link flaam://chat/{matchId}
 * - event_reminder   → "Afterwork dans 2h"            → deep link flaam://event/{eventId}
 * - payment_confirmed → "Premium activé !"            → deep link flaam://settings/subscription
 * - safety_alert     → "Timer expiré — tout va bien?" → deep link flaam://safety/timer
 */

class FlaamMessagingService : FirebaseMessagingService() {

    @Inject lateinit var authDataStore: AuthDataStore

    override fun onNewToken(token: String) {
        // Envoyer le nouveau FCM token au serveur
        CoroutineScope(Dispatchers.IO).launch {
            try {
                // POST /notifications/fcm-token
                // Le serveur stocke ce token pour envoyer les push
            } catch (e: Exception) {
                Timber.e(e, "Failed to update FCM token")
            }
        }
    }

    override fun onMessageReceived(message: RemoteMessage) {
        val data = message.data
        val type = data["type"] ?: return
        val deepLink = data["deep_link"] ?: return

        // Construire la notification Android
        val channelId = when (type) {
            "new_match", "new_message" -> "flaam_messages"
            "daily_feed" -> "flaam_daily"
            "match_expiring" -> "flaam_reminders"
            "event_reminder" -> "flaam_events"
            "payment_confirmed" -> "flaam_account"
            "safety_alert" -> "flaam_safety"
            else -> "flaam_default"
        }

        val intent = Intent(Intent.ACTION_VIEW, Uri.parse(deepLink)).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
        }

        val pendingIntent = PendingIntent.getActivity(
            this, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE,
        )

        val notification = NotificationCompat.Builder(this, channelId)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(data["title"] ?: "Flaam")
            .setContentText(data["body"] ?: "")
            .setAutoCancel(true)
            .setContentIntent(pendingIntent)
            .setPriority(
                if (type == "safety_alert") NotificationCompat.PRIORITY_MAX
                else NotificationCompat.PRIORITY_HIGH
            )
            .build()

        val manager = getSystemService(NOTIFICATION_SERVICE) as NotificationManager
        manager.notify(type.hashCode(), notification)
    }
}


// ── Notification Channels (créés au démarrage de l'app) ──

fun createNotificationChannels(context: Context) {
    val manager = context.getSystemService(NOTIFICATION_SERVICE) as NotificationManager

    val channels = listOf(
        NotificationChannel("flaam_messages", "Messages", NotificationManager.IMPORTANCE_HIGH),
        NotificationChannel("flaam_daily", "Feed du jour", NotificationManager.IMPORTANCE_DEFAULT),
        NotificationChannel("flaam_reminders", "Rappels", NotificationManager.IMPORTANCE_DEFAULT),
        NotificationChannel("flaam_events", "Events", NotificationManager.IMPORTANCE_DEFAULT),
        NotificationChannel("flaam_account", "Compte", NotificationManager.IMPORTANCE_LOW),
        NotificationChannel("flaam_safety", "Sécurité", NotificationManager.IMPORTANCE_MAX).apply {
            enableVibration(true)
            vibrationPattern = longArrayOf(0, 500, 200, 500)
        },
    )

    channels.forEach { manager.createNotificationChannel(it) }
}
```

---

## 14. Performance — Image Loading Strategy

```kotlin
/**
 * Stratégie de chargement d'images adaptée au réseau africain.
 *
 * 3 tailles d'images côté serveur :
 * - thumbnail : 150px, ~5-10KB  → feed, liste de matchs
 * - medium    : 600px, ~30-50KB → profil détaillé, conversation
 * - original  : pleine taille   → jamais chargé en-app
 *
 * Le champ dominant_color dans le modèle Photo sert de placeholder
 * pendant le chargement (rectangle coloré → image).
 */

@Composable
fun FlaamPhoto(
    url: String,
    dominantColor: Color = FlaamColors.Sand,
    contentDescription: String?,
    modifier: Modifier = Modifier,
    size: FlaamPhotoSize = FlaamPhotoSize.THUMBNAIL,
) {
    val imageUrl = when (size) {
        FlaamPhotoSize.THUMBNAIL -> url.toThumbnailUrl()
        FlaamPhotoSize.MEDIUM -> url.toMediumUrl()
    }

    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .crossfade(300)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .build(),
        contentDescription = contentDescription,
        contentScale = ContentScale.Crop,
        placeholder = ColorPainter(dominantColor),
        error = ColorPainter(dominantColor),
        modifier = modifier,
    )
}

enum class FlaamPhotoSize { THUMBNAIL, MEDIUM }

// Coil cache config dans FlaamApp.kt
fun configureCoilImageLoader(context: Context): ImageLoader {
    return ImageLoader.Builder(context)
        .memoryCache {
            MemoryCache.Builder(context)
                .maxSizePercent(0.20)  // 20% de la RAM disponible
                .build()
        }
        .diskCache {
            DiskCache.Builder()
                .directory(context.cacheDir.resolve("flaam_images"))
                .maxSizeBytes(50 * 1024 * 1024) // 50 MB max
                .build()
        }
        .respectCacheHeaders(true)
        .build()
}
```

---

## 15. CI/CD Mobile

```yaml
# .github/workflows/android.yml

name: Flaam Android CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - name: Run unit tests
        run: ./gradlew testDebugUnitTest
      - name: Run lint
        run: ./gradlew lintDebug
      - name: Upload test results
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: app/build/reports/tests/

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: Decode keystore
        run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > app/flaam-release.jks

      - name: Build release AAB
        run: ./gradlew bundleRelease
        env:
          FLAAM_KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          FLAAM_KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          FLAAM_KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}

      - name: Upload to Play Store (internal track)
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT }}
          packageName: app.flaam.android
          releaseFiles: app/build/outputs/bundle/release/app-release.aab
          track: internal
          # internal → closed testing → open testing → production

      - name: Notify Telegram
        run: |
          curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendMessage" \
            -d "chat_id=${{ secrets.TELEGRAM_CHAT_ID }}" \
            -d "text=✅ Flaam Android v$(cat app/version.txt) deployed to internal track"
```

---

## 16. Permissions Android

```xml
<!-- AndroidManifest.xml -->
<manifest>
    <!-- Réseau -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <!-- Localisation (check-in uniquement, PAS de tracking continu) -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <!-- PAS de ACCESS_BACKGROUND_LOCATION — on ne track pas en arrière-plan -->

    <!-- Caméra (selfie vérification + photo profil) -->
    <uses-permission android:name="android.permission.CAMERA" />

    <!-- Contacts (import blacklist, optionnel) -->
    <uses-permission android:name="android.permission.READ_CONTACTS" />

    <!-- Notifications -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <!-- Micro (messages vocaux) -->
    <uses-permission android:name="android.permission.RECORD_AUDIO" />

    <!-- Vibration (notifications safety) -->
    <uses-permission android:name="android.permission.VIBRATE" />
</manifest>
```


---

## 17. Onboarding State Machine (côté client)

```kotlin
/**
 * L'onboarding a 12 étapes. L'utilisateur peut quitter et revenir.
 * Le client doit reprendre EXACTEMENT où il s'est arrêté.
 *
 * Correspondance avec backend spec section 13.
 * L'état est stocké localement (DataStore) ET synchronisé serveur.
 */

enum class OnboardingStep(val index: Int, val route: String, val skippable: Boolean) {
    PHONE(0, Route.Phone.path, false),
    OTP(1, Route.Otp.path, false),
    CITY(2, Route.CitySelection.path, false),
    BASIC_INFO(3, Route.BasicInfo.path, false),
    SELFIE(4, Route.SelfieVerification.path, false),
    PHOTOS(5, Route.PhotoUpload.path, false),         // Min 3 photos
    QUARTIERS(6, Route.QuartierSelection.path, false), // Min 1 quartier
    INTENTION(7, Route.IntentionSelection.path, false),
    SECTOR(8, Route.SectorSelection.path, false),
    PROMPTS(9, Route.PromptAnswers.path, true),        // Skippable
    TAGS(10, Route.TagSelection.path, true),            // Skippable
    SPOTS(11, Route.SpotSelection.path, true),          // Skippable
    DONE(12, Route.OnboardingDone.path, false),
}

@Singleton
class OnboardingManager @Inject constructor(
    private val onboardingDataStore: OnboardingDataStore,
) {
    suspend fun getCurrentStep(): OnboardingStep {
        val index = onboardingDataStore.getCurrentStepIndex()
        return OnboardingStep.entries.getOrElse(index) { OnboardingStep.PHONE }
    }

    suspend fun completeStep(step: OnboardingStep) {
        onboardingDataStore.saveStepIndex(step.index + 1)
    }

    suspend fun skipStep(step: OnboardingStep) {
        if (step.skippable) {
            onboardingDataStore.saveStepIndex(step.index + 1)
        }
    }

    suspend fun isOnboardingComplete(): Boolean {
        return onboardingDataStore.getCurrentStepIndex() >= OnboardingStep.DONE.index
    }

    fun getProgressFraction(step: OnboardingStep): Float {
        return step.index.toFloat() / OnboardingStep.DONE.index
    }
}
```

---

## 18. Photo Upload & Compression

```kotlin
/**
 * Les photos sont le poste data #1. 
 * Sur réseau africain, un upload de 5MB = 30-60 secondes.
 * On compresse AVANT l'upload.
 *
 * Pipeline :
 * 1. L'utilisateur choisit/prend une photo
 * 2. Compression côté client (max 800px, JPEG 80%)
 * 3. Upload multipart vers POST /profiles/me/photos
 * 4. Le serveur génère thumbnail (150px) + medium (600px)
 * 5. Le serveur lance la modération async
 * 6. Le client affiche un badge "en vérification" 
 * 7. Le backend envoie une push quand la modération est terminée
 */

@Singleton
class PhotoCompressor @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    /**
     * Compresse une image à max 800px de large et 80% JPEG quality.
     * Résultat typique : 2-5MB → 80-200KB
     */
    suspend fun compress(uri: Uri): File = withContext(Dispatchers.IO) {
        val inputStream = context.contentResolver.openInputStream(uri)
            ?: throw IllegalStateException("Cannot open URI")

        val original = BitmapFactory.decodeStream(inputStream)
        inputStream.close()

        // Redimensionner si > 800px
        val maxDim = 800
        val scaled = if (original.width > maxDim || original.height > maxDim) {
            val ratio = minOf(
                maxDim.toFloat() / original.width,
                maxDim.toFloat() / original.height,
            )
            Bitmap.createScaledBitmap(
                original,
                (original.width * ratio).toInt(),
                (original.height * ratio).toInt(),
                true,
            )
        } else original

        // Compresser en JPEG
        val outputFile = File(context.cacheDir, "upload_${UUID.randomUUID()}.jpg")
        FileOutputStream(outputFile).use { out ->
            scaled.compress(Bitmap.CompressFormat.JPEG, 80, out)
        }

        if (scaled !== original) scaled.recycle()
        original.recycle()

        outputFile
    }

    /**
     * Extrait la couleur dominante d'une image.
     * Envoyé avec l'upload pour le placeholder côté client.
     */
    fun extractDominantColor(bitmap: Bitmap): Color {
        val palette = Palette.from(bitmap).generate()
        val swatch = palette.dominantSwatch
            ?: palette.vibrantSwatch
            ?: palette.mutedSwatch
        return swatch?.let { Color(it.rgb) } ?: FlaamColors.Sand
    }
}

// ── Photo moderation status tracking ──

data class PhotoUiState(
    val id: String,
    val url: String,
    val position: Int,
    val status: PhotoStatus,
    val rejectionReason: String? = null,
)

enum class PhotoStatus {
    UPLOADING,      // En cours d'upload
    PENDING_REVIEW, // Uploadé, en attente de modération
    APPROVED,       // Approuvé par la modération
    REJECTED,       // Rejeté (raison fournie)
}
```

---

## 19. Voice Messages

```kotlin
/**
 * Messages vocaux dans le chat.
 * Max 60 secondes. Compressé en AAC avant envoi.
 *
 * UX :
 * - Maintenir le bouton micro pour enregistrer
 * - Glisser vers le haut pour annuler
 * - Relâcher pour envoyer
 * - Waveform affiché pendant l'enregistrement
 * - En mode data saver : "Télécharger (34 KB)" au lieu d'auto-play
 */

class VoiceRecorder @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    private var recorder: MediaRecorder? = null
    private var outputFile: File? = null
    private var startTime: Long = 0

    private val _amplitude = MutableStateFlow(0)
    val amplitude: StateFlow<Int> = _amplitude.asStateFlow()

    private val _duration = MutableStateFlow(0L)
    val duration: StateFlow<Long> = _duration.asStateFlow()

    private var amplitudeJob: Job? = null

    fun startRecording(): File {
        outputFile = File(context.cacheDir, "voice_${UUID.randomUUID()}.m4a")

        recorder = MediaRecorder().apply {
            setAudioSource(MediaRecorder.AudioSource.MIC)
            setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
            setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
            setAudioSamplingRate(44100)
            setAudioEncodingBitRate(64000) // 64kbps = bonne qualité, fichier léger
            setMaxDuration(60_000) // 60 secondes max
            setOutputFile(outputFile!!.absolutePath)
            prepare()
            start()
        }

        startTime = System.currentTimeMillis()

        // Échantillonner l'amplitude pour la waveform
        amplitudeJob = CoroutineScope(Dispatchers.IO).launch {
            while (isActive) {
                delay(100)
                _amplitude.value = recorder?.maxAmplitude ?: 0
                _duration.value = System.currentTimeMillis() - startTime
            }
        }

        return outputFile!!
    }

    fun stopRecording(): File? {
        amplitudeJob?.cancel()
        return try {
            recorder?.stop()
            recorder?.release()
            recorder = null
            outputFile
        } catch (e: Exception) {
            recorder?.release()
            recorder = null
            null
        }
    }

    fun cancelRecording() {
        amplitudeJob?.cancel()
        recorder?.release()
        recorder = null
        outputFile?.delete()
        outputFile = null
    }
}
```

---

## 20. Payment Flow (Mobile Money)

```kotlin
/**
 * Flow de paiement mobile money.
 * Correspondance avec backend spec section 36.
 *
 * Le flow côté client :
 * 1. L'utilisateur choisit un plan (mensuel/hebdo)
 * 2. POST /subscriptions/initialize → reçoit une URL Paystack
 * 3. Ouvrir l'URL dans un WebView ou le navigateur externe
 * 4. L'utilisateur complète le paiement sur son MoMo/TMoney
 * 5. Afficher l'écran "Paiement en cours..." (max 30 min)
 * 6. Polling GET /subscriptions/me toutes les 30 secondes
 * 7. Quand status = "active" → afficher "Premium activé !"
 * 8. OU push notification "payment_confirmed" arrive
 */

@HiltViewModel
class PaymentViewModel @Inject constructor(
    private val subscriptionApi: SubscriptionApi,
) : ViewModel() {

    data class PaymentUiState(
        val plans: List<PlanUi> = emptyList(),
        val selectedPlan: PlanUi? = null,
        val paymentStatus: PaymentStatus = PaymentStatus.IDLE,
        val paymentReference: String? = null,
        val error: String? = null,
    )

    enum class PaymentStatus {
        IDLE,           // Pas de paiement en cours
        INITIALIZING,   // POST /subscriptions/initialize
        AWAITING_USSD,  // L'utilisateur doit confirmer sur son téléphone
        POLLING,        // En attente de confirmation webhook
        SUCCESS,        // Premium activé
        FAILED,         // Échec
        TIMEOUT,        // 30 min sans réponse
    }

    private val _state = MutableStateFlow(PaymentUiState())
    val state = _state.asStateFlow()

    private var pollingJob: Job? = null

    fun initializePayment(planId: String, method: String) {
        viewModelScope.launch {
            _state.update { it.copy(paymentStatus = PaymentStatus.INITIALIZING) }

            try {
                val response = subscriptionApi.initializePayment(
                    InitPaymentBody(planId = planId, method = method)
                )
                if (response.isSuccessful) {
                    val body = response.body()!!
                    _state.update { it.copy(
                        paymentStatus = PaymentStatus.AWAITING_USSD,
                        paymentReference = body.reference,
                    )}
                    // Démarrer le polling
                    startPolling(body.reference)
                } else {
                    _state.update { it.copy(
                        paymentStatus = PaymentStatus.FAILED,
                        error = "Erreur d'initialisation",
                    )}
                }
            } catch (e: Exception) {
                _state.update { it.copy(
                    paymentStatus = PaymentStatus.FAILED,
                    error = "Vérifie ta connexion et réessaie",
                )}
            }
        }
    }

    private fun startPolling(reference: String) {
        pollingJob?.cancel()
        pollingJob = viewModelScope.launch {
            _state.update { it.copy(paymentStatus = PaymentStatus.POLLING) }
            val startTime = System.currentTimeMillis()
            val timeout = 30 * 60 * 1000L // 30 minutes

            while (isActive) {
                delay(30_000) // Poll toutes les 30 secondes

                // Timeout check
                if (System.currentTimeMillis() - startTime > timeout) {
                    _state.update { it.copy(paymentStatus = PaymentStatus.TIMEOUT) }
                    break
                }

                try {
                    val response = subscriptionApi.getSubscription()
                    if (response.isSuccessful) {
                        val sub = response.body()!!
                        if (sub.status == "active") {
                            _state.update { it.copy(paymentStatus = PaymentStatus.SUCCESS) }
                            break
                        }
                    }
                } catch (_: Exception) {
                    // Ignorer les erreurs de polling, on réessaie
                }
            }
        }
    }
}
```

---

## 21. Safety Features (côté client)

```kotlin
/**
 * Fonctionnalités de sécurité.
 * Correspondance avec backend spec sections 37-39 (écrans) + modèles safety.
 *
 * 3 features :
 * 1. Partager un date (SMS au contact de confiance)
 * 2. Timer de sécurité (countdown, alerte si pas désactivé)
 * 3. Bouton d'urgence (envoie la position GPS)
 */

@HiltViewModel
class SafetyViewModel @Inject constructor(
    private val safetyApi: SafetyApi,
    private val locationProvider: FusedLocationProviderClient,
) : ViewModel() {

    data class SafetyUiState(
        val timerActive: Boolean = false,
        val timerEndTime: Long = 0,
        val timerRemainingSeconds: Long = 0,
        val trustedContact: TrustedContact? = null,
        val currentDateInfo: DateInfo? = null,
    )

    private val _state = MutableStateFlow(SafetyUiState())
    val state = _state.asStateFlow()

    private var countdownJob: Job? = null

    fun shareDate(matchId: String, spotName: String, dateTime: String, durationHours: Int) {
        viewModelScope.launch {
            try {
                safetyApi.shareDate(ShareDateBody(
                    matchId = matchId,
                    spotName = spotName,
                    dateTime = dateTime,
                ))
                // Démarrer le timer
                startTimer(durationHours)
            } catch (e: Exception) {
                Timber.e(e, "Failed to share date")
            }
        }
    }

    fun startTimer(hours: Int) {
        val endTime = System.currentTimeMillis() + hours * 3600 * 1000L
        _state.update { it.copy(timerActive = true, timerEndTime = endTime) }

        viewModelScope.launch {
            safetyApi.startTimer(TimerBody(durationMinutes = hours * 60))
        }

        // Countdown local
        countdownJob?.cancel()
        countdownJob = viewModelScope.launch {
            while (isActive) {
                val remaining = (endTime - System.currentTimeMillis()) / 1000
                _state.update { it.copy(timerRemainingSeconds = remaining) }
                if (remaining <= 0) {
                    // Timer expiré → le backend envoie l'alerte automatiquement
                    break
                }
                delay(1000)
            }
        }
    }

    fun cancelTimer() {
        countdownJob?.cancel()
        _state.update { it.copy(timerActive = false) }
        viewModelScope.launch {
            safetyApi.cancelTimer()
        }
    }

    @SuppressLint("MissingPermission")
    fun triggerEmergency() {
        viewModelScope.launch {
            // Récupérer la position GPS actuelle
            try {
                val location = locationProvider.getCurrentLocation(
                    Priority.PRIORITY_HIGH_ACCURACY, null
                ).await()

                safetyApi.triggerEmergency(EmergencyBody(
                    latitude = location?.latitude,
                    longitude = location?.longitude,
                ))
            } catch (e: Exception) {
                // Envoyer même sans GPS
                safetyApi.triggerEmergency(EmergencyBody())
            }
        }
    }
}
```

---

## 22. Contact Import & Blacklist

```kotlin
/**
 * Import des contacts pour la blacklist.
 * Correspondance avec écran 39 (Import contacts).
 *
 * L'utilisateur peut masquer des numéros (boss, ex, famille)
 * pour ne jamais apparaître dans leur feed et vice versa.
 *
 * IMPORTANT : On hash les numéros côté client avant de les envoyer.
 * Le serveur ne voit JAMAIS les numéros en clair des contacts.
 */

class ContactImporter @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    /**
     * Lit les contacts du téléphone et retourne une liste de ContactInfo.
     * Requiert READ_CONTACTS permission.
     */
    suspend fun loadContacts(): List<ContactInfo> = withContext(Dispatchers.IO) {
        val contacts = mutableListOf<ContactInfo>()

        val cursor = context.contentResolver.query(
            ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
            arrayOf(
                ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME,
                ContactsContract.CommonDataKinds.Phone.NUMBER,
            ),
            null, null,
            ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME + " ASC",
        )

        cursor?.use {
            while (it.moveToNext()) {
                val name = it.getString(0) ?: continue
                val number = it.getString(1) ?: continue
                val normalized = normalizePhone(number)
                contacts.add(ContactInfo(
                    name = name,
                    phoneDisplay = maskPhone(number),
                    phoneHash = normalized.sha256(),
                ))
            }
        }

        contacts.distinctBy { it.phoneHash }
    }

    /**
     * Normalise un numéro : supprime espaces, tirets, parenthèses.
     * Ajoute l'indicatif pays si absent.
     */
    private fun normalizePhone(raw: String): String {
        val cleaned = raw.replace(Regex("[\\s\\-().]"), "")
        return if (cleaned.startsWith("+")) cleaned
        else "+228$cleaned" // Default Togo, ajuster selon la ville
    }

    private fun maskPhone(phone: String): String {
        if (phone.length < 6) return phone
        return phone.take(4) + " ** ** " + phone.takeLast(2)
    }
}

data class ContactInfo(
    val name: String,
    val phoneDisplay: String, // Masqué pour l'UI
    val phoneHash: String,    // SHA-256 du numéro normalisé
)
```

---

## 23. Behavior Logging (signaux implicites pour l'algo)

```kotlin
/**
 * Le backend a besoin de signaux comportementaux pour améliorer le matching.
 * Correspondance avec backend spec section 8 (L4 behavior multiplier).
 *
 * Signaux collectés :
 * - Temps passé sur un profil (secondes)
 * - Nombre de photos scrollées
 * - Si l'utilisateur a tapé "voir plus" sur un prompt
 * - Temps de réponse aux matchs (heures entre match et premier message)
 * - Ratio scroll complet vs skip rapide
 *
 * Ces signaux sont envoyés en batch (pas en temps réel)
 * via POST /behavior/log toutes les 5 minutes ou quand l'app passe en background.
 */

@Singleton
class BehaviorTracker @Inject constructor() {

    private val events = mutableListOf<BehaviorEvent>()
    private var currentProfileId: String? = null
    private var profileViewStart: Long = 0

    fun onProfileViewed(profileId: String) {
        // Finaliser l'ancien profil
        finalizeCurrentProfile()
        // Commencer le nouveau
        currentProfileId = profileId
        profileViewStart = System.currentTimeMillis()
    }

    fun onPhotoScrolled(profileId: String, photoIndex: Int) {
        events.add(BehaviorEvent(
            type = "photo_scrolled",
            profileId = profileId,
            data = mapOf("photo_index" to photoIndex),
            timestamp = System.currentTimeMillis(),
        ))
    }

    fun onPromptExpanded(profileId: String, promptId: String) {
        events.add(BehaviorEvent(
            type = "prompt_expanded",
            profileId = profileId,
            data = mapOf("prompt_id" to promptId),
            timestamp = System.currentTimeMillis(),
        ))
    }

    fun onProfileAction(profileId: String, action: String) {
        finalizeCurrentProfile()
        events.add(BehaviorEvent(
            type = "profile_action",
            profileId = profileId,
            data = mapOf("action" to action), // "like" ou "skip"
            timestamp = System.currentTimeMillis(),
        ))
    }

    private fun finalizeCurrentProfile() {
        currentProfileId?.let { id ->
            val viewDuration = System.currentTimeMillis() - profileViewStart
            events.add(BehaviorEvent(
                type = "profile_view_duration",
                profileId = id,
                data = mapOf("duration_ms" to viewDuration),
                timestamp = System.currentTimeMillis(),
            ))
        }
        currentProfileId = null
    }

    fun drainEvents(): List<BehaviorEvent> {
        val drained = events.toList()
        events.clear()
        return drained
    }
}

data class BehaviorEvent(
    val type: String,
    val profileId: String,
    val data: Map<String, Any>,
    val timestamp: Long,
)

// Worker pour envoyer les events en batch
@HiltWorker
class BehaviorLogWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val behaviorTracker: BehaviorTracker,
    private val behaviorApi: BehaviorApi,
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val events = behaviorTracker.drainEvents()
        if (events.isEmpty()) return Result.success()

        return try {
            behaviorApi.logBatch(BehaviorBatchBody(events))
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}
```

---

## 24. Rate Limiting Handling

```kotlin
/**
 * Quand le serveur retourne 429 Too Many Requests,
 * le client doit gérer ça proprement.
 *
 * Correspondance avec backend spec section 15 (rate limiting).
 */

class RateLimitInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): okhttp3.Response {
        val response = chain.proceed(chain.request())

        if (response.code == 429) {
            val retryAfter = response.header("Retry-After")?.toLongOrNull() ?: 60
            // Le ViewModel recevra une RateLimitException
            // et affichera un message adapté
            throw RateLimitException(retryAfterSeconds = retryAfter)
        }

        // Lire les headers de rate limit pour l'UI
        val remaining = response.header("X-RateLimit-Remaining")?.toIntOrNull()
        val limit = response.header("X-RateLimit-Limit")?.toIntOrNull()

        // Si c'est l'endpoint /feed/{id}/like, propager le compteur
        if (remaining != null && chain.request().url.encodedPath.contains("/like")) {
            RateLimitState.likesRemaining.value = remaining
        }

        return response
    }
}

class RateLimitException(val retryAfterSeconds: Long) : IOException(
    "Rate limited. Retry after ${retryAfterSeconds}s"
)

// State global pour les compteurs
object RateLimitState {
    val likesRemaining = MutableStateFlow(5)
}
```

---

## 25. Force Update & Feature Flags

```kotlin
/**
 * Vérification de version au démarrage.
 * Correspondance avec backend spec section 27 (versioning) et 28 (feature flags).
 */

@Singleton
class AppVersionChecker @Inject constructor(
    private val configApi: ConfigApi,
) {
    data class VersionCheck(
        val forceUpdate: Boolean,
        val suggestUpdate: Boolean,
        val updateUrl: String,
        val message: String?,
    )

    suspend fun check(): VersionCheck {
        return try {
            val response = configApi.getMinVersion()
            if (response.isSuccessful) {
                val config = response.body()!!
                val currentVersion = BuildConfig.VERSION_CODE
                VersionCheck(
                    forceUpdate = currentVersion < config.minVersion,
                    suggestUpdate = currentVersion < config.latestVersion,
                    updateUrl = config.updateUrl,
                    message = config.updateMessage,
                )
            } else {
                VersionCheck(false, false, "", null)
            }
        } catch (e: Exception) {
            // Pas de réseau → pas de vérification, laisser passer
            VersionCheck(false, false, "", null)
        }
    }
}


// ── Feature Flags ──

@Singleton
class FeatureFlags @Inject constructor(
    private val configApi: ConfigApi,
    private val dataStore: DataStore<Preferences>,
) {
    private val _flags = MutableStateFlow<Map<String, Boolean>>(emptyMap())
    val flags: StateFlow<Map<String, Boolean>> = _flags.asStateFlow()

    suspend fun refresh() {
        try {
            val response = configApi.getFeatureFlags()
            if (response.isSuccessful) {
                _flags.value = response.body()!!.flags
            }
        } catch (_: Exception) {
            // Utiliser les flags cachés
        }
    }

    fun isEnabled(flag: String): Boolean = _flags.value[flag] ?: false

    // Flags connus
    companion object {
        const val VOICE_MESSAGES = "voice_messages"
        const val EVENTS_TAB = "events_tab"
        const val DATA_SAVER_DEFAULT = "data_saver_default"
        const val MEETUP_PROPOSALS = "meetup_proposals"
        const val INTERESTED_QUARTIERS = "interested_quartiers"
    }
}
```

---

## 26. Error Handling & User-Facing Messages

```kotlin
/**
 * Catalogue d'erreurs côté client.
 * Correspondance avec backend spec section 21 (error catalogue).
 *
 * Chaque erreur API est traduite en message user-friendly FR/EN.
 */

sealed class FlaamError(
    val code: String,
    val userMessageFr: String,
    val userMessageEn: String,
) {
    // Network
    data object NoNetwork : FlaamError(
        "NETWORK_ERROR",
        "Pas de connexion. Tes actions seront envoyées quand le réseau revient.",
        "No connection. Your actions will sync when you're back online.",
    )
    data object Timeout : FlaamError(
        "TIMEOUT",
        "La connexion est lente. Réessaie dans quelques secondes.",
        "Connection is slow. Try again in a few seconds.",
    )

    // Auth
    data object OtpExpired : FlaamError(
        "OTP_EXPIRED",
        "Le code a expiré. Renvoie un nouveau code.",
        "Code expired. Request a new one.",
    )
    data object OtpInvalid : FlaamError(
        "OTP_INVALID",
        "Code incorrect. Vérifie et réessaie.",
        "Invalid code. Check and try again.",
    )
    data object AccountBlocked : FlaamError(
        "ACCOUNT_BLOCKED",
        "Ton compte est suspendu. Contacte support@flaam.app",
        "Your account is suspended. Contact support@flaam.app",
    )

    // Feed
    data object NoMoreLikes : FlaamError(
        "LIKES_EXHAUSTED",
        "Tu as utilisé tous tes likes du jour. Reviens demain !",
        "You've used all your daily likes. Come back tomorrow!",
    )
    data object FeedEmpty : FlaamError(
        "FEED_EMPTY",
        "Plus de profils pour aujourd'hui. Reviens demain à 9h.",
        "No more profiles today. Come back tomorrow at 9am.",
    )

    // Chat
    data object MatchExpired : FlaamError(
        "MATCH_EXPIRED",
        "Ce match a expiré. Pas d'inquiétude, d'autres arrivent !",
        "This match has expired. Don't worry, more are coming!",
    )

    // Payment
    data object PaymentFailed : FlaamError(
        "PAYMENT_FAILED",
        "Le paiement a échoué. Vérifie ton solde et réessaie.",
        "Payment failed. Check your balance and try again.",
    )
    data object PaymentPending : FlaamError(
        "PAYMENT_PENDING",
        "Paiement en cours. Ça peut prendre jusqu'à 30 min avec mobile money.",
        "Payment processing. It can take up to 30 min with mobile money.",
    )

    // Rate limit
    data class RateLimited(val retryAfter: Long) : FlaamError(
        "RATE_LIMITED",
        "Doucement ! Réessaie dans $retryAfter secondes.",
        "Slow down! Try again in $retryAfter seconds.",
    )

    // Generic
    data object Unknown : FlaamError(
        "UNKNOWN",
        "Quelque chose s'est mal passé. Réessaie.",
        "Something went wrong. Please try again.",
    )
}


// Extension pour convertir une exception en FlaamError
fun Throwable.toFlaamError(): FlaamError = when (this) {
    is RateLimitException -> FlaamError.RateLimited(retryAfterSeconds)
    is UnknownHostException -> FlaamError.NoNetwork
    is SocketTimeoutException -> FlaamError.Timeout
    is ApiException -> when (code) {
        401 -> FlaamError.AccountBlocked
        429 -> FlaamError.RateLimited(60)
        else -> FlaamError.Unknown
    }
    else -> FlaamError.Unknown
}

// Extension pour convertir en message selon la locale
fun FlaamError.toUserMessage(locale: Locale = Locale.getDefault()): String {
    return if (locale.language == "fr") userMessageFr else userMessageEn
}
```

---

## 27. Localization (FR/EN)

```kotlin
/**
 * L'app supporte FR et EN nativement.
 * Les langues locales (Ewe, Mina, Kabyè) sont prévues en V2.
 *
 * On utilise le système Android standard (res/values-fr, res/values-en).
 * Les textes dynamiques du backend arrivent déjà traduits
 * (le serveur connaît la langue préférée de l'utilisateur).
 */

// res/values/strings.xml (EN - default)
// res/values-fr/strings.xml (FR)

// Exemples de strings critiques :

// ── Onboarding ──
// en: "Your number"
// fr: "Ton numéro"

// en: "Where do you live?"
// fr: "Où tu vis ?"

// en: "What are you looking for?"
// fr: "Tu cherches quoi ?"

// ── Feed ──
// en: "%d likes remaining"
// fr: "%d likes restants"

// en: "common spots"  
// fr: "spots en commun"

// en: "Skip" / "Like"
// fr: "Passer" / "Liker"

// ── Chat ──  
// en: "Let's meet up?"
// fr: "On se retrouve ?"

// en: "Type a message..."
// fr: "Message..."

// en: "Voice message (%s)"
// fr: "Message vocal (%s)"

// ── Safety ──
// en: "Share your date"
// fr: "Partager ton date"

// en: "Timer active"
// fr: "Timer actif"

// en: "ALERT"
// fr: "ALERTER"

// ── Errors ──
// Gérés via FlaamError.toUserMessage() (section 26)
```

---

## 28. Build Variants & ProGuard

```kotlin
// build.gradle.kts (app)

android {
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            isDebuggable = true
            // API pointe vers le serveur de dev
            buildConfigField("String", "API_BASE_URL", "\"https://api-dev.flaam.app\"")
            buildConfigField("String", "WS_BASE_URL", "\"wss://api-dev.flaam.app\"")
        }

        create("staging") {
            applicationIdSuffix = ".staging"
            isDebuggable = false
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            buildConfigField("String", "API_BASE_URL", "\"https://api-staging.flaam.app\"")
            buildConfigField("String", "WS_BASE_URL", "\"wss://api-staging.flaam.app\"")
        }

        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            buildConfigField("String", "API_BASE_URL", "\"https://api.flaam.app\"")
            buildConfigField("String", "WS_BASE_URL", "\"wss://api.flaam.app\"")

            signingConfig = signingConfigs.getByName("release")
        }
    }
}


// ── proguard-rules.pro ──

// Retrofit
-keepattributes Signature
-keepattributes *Annotation*
-keep class retrofit2.** { *; }
-keepclassmembers,allowshrinking,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}

// Moshi
-keep class com.squareup.moshi.** { *; }
-keep @com.squareup.moshi.JsonClass class * { *; }

// Room
-keep class * extends androidx.room.RoomDatabase { *; }

// Flaam DTOs (ne pas obfusquer les noms de champs JSON)
-keep class app.flaam.core.network.dto.** { *; }
-keep class app.flaam.domain.model.** { *; }

// Compose
-dontwarn androidx.compose.**
```

---

## 29. Testing Strategy

```kotlin
/**
 * 3 niveaux de tests :
 * 1. Unit tests    — ViewModels, UseCases, Repositories, Mappers
 * 2. Integration   — Room DAOs, API avec MockWebServer
 * 3. UI tests      — Compose screens avec Compose Testing
 *
 * Ratio cible : 70% unit, 20% integration, 10% UI
 */

// ── Unit test exemple : FeedViewModel ──

@OptIn(ExperimentalCoroutinesApi::class)
class FeedViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var viewModel: FeedViewModel
    private val getDailyFeed = mockk<GetDailyFeedUseCase>()
    private val likeProfile = mockk<LikeProfileUseCase>()
    private val skipProfile = mockk<SkipProfileUseCase>()
    private val networkMonitor = mockk<NetworkMonitor>()

    @Before
    fun setup() {
        every { networkMonitor.status } returns MutableStateFlow(NetworkStatus.ONLINE)
        coEvery { getDailyFeed() } returns Result.success(testProfiles)

        viewModel = FeedViewModel(getDailyFeed, likeProfile, skipProfile, networkMonitor)
    }

    @Test
    fun `initial load populates feed`() = runTest {
        viewModel.state.test {
            val state = awaitItem() // Loading
            val loaded = awaitItem() // Loaded
            assertThat(loaded.profiles).hasSize(10)
            assertThat(loaded.isLoading).isFalse()
        }
    }

    @Test
    fun `like decrements counter`() = runTest {
        coEvery { likeProfile(any()) } returns Result.success(LikeResult.Liked)

        viewModel.state.test {
            skipItems(2) // Skip loading states

            viewModel.onEvent(FeedUiEvent.Like("profile-1"))

            val updated = awaitItem()
            assertThat(updated.likesRemaining).isEqualTo(4)
        }
    }

    @Test
    fun `offline state shown when network lost`() = runTest {
        val networkFlow = MutableStateFlow(NetworkStatus.ONLINE)
        every { networkMonitor.status } returns networkFlow

        viewModel = FeedViewModel(getDailyFeed, likeProfile, skipProfile, networkMonitor)

        viewModel.state.test {
            skipItems(2)

            networkFlow.value = NetworkStatus.OFFLINE
            val offlineState = awaitItem()
            assertThat(offlineState.isOffline).isTrue()
        }
    }
}


// ── Integration test : Room DAO ──

@RunWith(AndroidJUnit4::class)
class ProfileDaoTest {

    private lateinit var db: FlaamDatabase
    private lateinit var dao: ProfileDao

    @Before
    fun setup() {
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            FlaamDatabase::class.java,
        ).build()
        dao = db.profileDao()
    }

    @After
    fun teardown() { db.close() }

    @Test
    fun insertAndRetrieveFeed() = runTest {
        val profiles = (1..10).map { createTestProfile(it, "2026-04-14") }
        dao.insertAll(profiles)

        dao.getFeedForDate("2026-04-14").test {
            val result = awaitItem()
            assertThat(result).hasSize(10)
            assertThat(result.first().position).isEqualTo(0)
        }
    }

    @Test
    fun likeCountUpdates() = runTest {
        dao.insertAll(listOf(createTestProfile(1, "2026-04-14")))
        dao.markLiked("profile-1", System.currentTimeMillis())

        dao.getLikeCountForDate("2026-04-14").test {
            assertThat(awaitItem()).isEqualTo(1)
        }
    }
}


// ── UI test : FeedScreen ──

class FeedScreenTest {

    @get:Rule
    val composeRule = createComposeRule()

    @Test
    fun feedDisplaysProfileCard() {
        composeRule.setContent {
            FlaamTheme {
                FeedScreen(
                    state = FeedUiState(
                        profiles = listOf(testProfileUi),
                        isLoading = false,
                        likesRemaining = 3,
                    ),
                    onEvent = {},
                )
            }
        }

        composeRule.onNodeWithText("Ama").assertIsDisplayed()
        composeRule.onNodeWithText("26").assertIsDisplayed()
        composeRule.onNodeWithText("3 restants").assertIsDisplayed()
    }
}
```

---

## 30. App Startup & Routing

```kotlin
/**
 * Au démarrage de l'app, on doit décider où envoyer l'utilisateur :
 *
 * 1. Vérifier la version → force update ?
 * 2. Token existant ? → NON → Welcome screen (auth)
 * 3. Token valide ? → NON → Refresh → NON → Welcome screen
 * 4. Onboarding complet ? → NON → Reprendre l'onboarding
 * 5. Ville en teaser ? → OUI → Waitlist screen
 * 6. Tout OK → Feed principal
 */

@HiltViewModel
class MainViewModel @Inject constructor(
    private val authDataStore: AuthDataStore,
    private val onboardingManager: OnboardingManager,
    private val versionChecker: AppVersionChecker,
    private val featureFlags: FeatureFlags,
    private val networkMonitor: NetworkMonitor,
) : ViewModel() {

    sealed interface StartDestination {
        data object Loading : StartDestination
        data object ForceUpdate : StartDestination
        data object Auth : StartDestination
        data class Onboarding(val step: OnboardingStep) : StartDestination
        data object Waitlist : StartDestination
        data object Main : StartDestination
    }

    private val _destination = MutableStateFlow<StartDestination>(StartDestination.Loading)
    val destination: StateFlow<StartDestination> = _destination.asStateFlow()

    init {
        viewModelScope.launch {
            // 1. Version check
            val version = versionChecker.check()
            if (version.forceUpdate) {
                _destination.value = StartDestination.ForceUpdate
                return@launch
            }

            // 2. Auth check
            val token = authDataStore.getAccessToken()
            if (token == null) {
                _destination.value = StartDestination.Auth
                return@launch
            }

            // 3. Refresh feature flags
            featureFlags.refresh()

            // 4. Onboarding check
            if (!onboardingManager.isOnboardingComplete()) {
                val step = onboardingManager.getCurrentStep()
                _destination.value = StartDestination.Onboarding(step)
                return@launch
            }

            // 5. City check (teaser mode)
            // TODO: vérifier si la ville de l'utilisateur est en mode teaser

            // 6. Main app
            _destination.value = StartDestination.Main
        }
    }
}


// ── MainActivity ──

@AndroidEntryPoint
class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Splash screen (Android 12+)
        installSplashScreen()

        // Notification channels
        createNotificationChannels(this)

        // Sync worker
        WorkerModule.scheduleSyncWorker(this)

        setContent {
            FlaamTheme {
                val viewModel: MainViewModel = hiltViewModel()
                val destination by viewModel.destination.collectAsState()

                when (destination) {
                    StartDestination.Loading -> {
                        FlaamSplash() // Logo animé
                    }
                    StartDestination.ForceUpdate -> {
                        ForceUpdateScreen()
                    }
                    StartDestination.Auth -> {
                        AuthNavGraph()
                    }
                    is StartDestination.Onboarding -> {
                        OnboardingNavGraph(
                            startStep = (destination as StartDestination.Onboarding).step,
                        )
                    }
                    StartDestination.Waitlist -> {
                        WaitlistScreen()
                    }
                    StartDestination.Main -> {
                        MainNavGraph()
                    }
                }
            }
        }
    }
}
```

---

## 31. Data Saver Mode

```kotlin
/**
 * Mode économie de données. Activable dans les paramètres.
 * Critique pour les utilisateurs avec des forfaits limités (500 FCFA/semaine).
 *
 * Quand activé :
 * - Feed : charge uniquement la photo 1 de chaque profil (pas les autres)
 * - Profil détaillé : les photos 2-6 se chargent au scroll
 * - Messages vocaux : "Télécharger (34 KB)" au lieu d'auto-play
 * - Images dans le chat : thumbnail uniquement, tap pour charger
 * - Pas de prefetch des profils suivants dans le feed
 * - Coil disk cache réduit à 20 MB (au lieu de 50)
 */

@Singleton
class DataSaverManager @Inject constructor(
    private val prefs: UserPrefsDataStore,
) {
    val isEnabled: StateFlow<Boolean> = prefs.dataSaverEnabled

    fun shouldLoadPhoto(photoIndex: Int): Boolean {
        if (!isEnabled.value) return true
        return photoIndex == 0 // Uniquement la première photo
    }

    fun shouldAutoPlayVoice(): Boolean = !isEnabled.value

    fun shouldPrefetch(): Boolean = !isEnabled.value

    fun getImageCacheSize(): Long {
        return if (isEnabled.value) 20L * 1024 * 1024 else 50L * 1024 * 1024
    }
}
```

---

## 32. Analytics

```kotlin
/**
 * Analytics events envoyés pour suivre les KPIs.
 * Correspondance avec backend spec section 29 (KPIs).
 *
 * On utilise un système simple : les events sont bufferisés
 * et envoyés en batch au backend, pas à un tiers (Firebase Analytics, Mixpanel).
 * Raison : maîtrise totale des données, pas de dépendance externe,
 * conformité RGPD simplifiée.
 */

@Singleton
class FlaamAnalytics @Inject constructor() {

    private val events = mutableListOf<AnalyticsEvent>()

    fun track(name: String, params: Map<String, Any> = emptyMap()) {
        events.add(AnalyticsEvent(
            name = name,
            params = params,
            timestamp = System.currentTimeMillis(),
        ))
    }

    fun drainEvents(): List<AnalyticsEvent> {
        val drained = events.toList()
        events.clear()
        return drained
    }

    // ── Events standardisés ──

    fun onboardingStepCompleted(step: String) =
        track("onboarding_step", mapOf("step" to step))

    fun feedViewed(profileCount: Int) =
        track("feed_viewed", mapOf("profile_count" to profileCount))

    fun profileLiked(profileId: String, geoScore: Float) =
        track("profile_liked", mapOf("profile_id" to profileId, "geo_score" to geoScore))

    fun profileSkipped(profileId: String, reason: String?) =
        track("profile_skipped", mapOf("profile_id" to profileId, "reason" to (reason ?: "none")))

    fun matchCreated(matchId: String) =
        track("match_created", mapOf("match_id" to matchId))

    fun messageSent(matchId: String, type: String) =
        track("message_sent", mapOf("match_id" to matchId, "type" to type))

    fun checkInCompleted(spotId: String) =
        track("checkin", mapOf("spot_id" to spotId))

    fun eventRegistered(eventId: String) =
        track("event_registered", mapOf("event_id" to eventId))

    fun premiumPurchased(plan: String, method: String) =
        track("premium_purchased", mapOf("plan" to plan, "method" to method))

    fun emergencyTriggered() =
        track("emergency_triggered")
}
```

---

## 33. Accessibility

```kotlin
/**
 * Règles d'accessibilité pour Compose.
 *
 * Minimum :
 * - Tous les éléments interactifs ont un contentDescription
 * - Les images ont un contentDescription (nom + âge de la personne)
 * - Les boutons ont un label sémantique
 * - Les contrastes respectent WCAG AA (4.5:1 pour le texte)
 * - TalkBack fonctionne sur tous les écrans
 * - Les touch targets font minimum 48dp
 */

// Exemples dans les composables :

@Composable
fun LikeButton(onClick: () -> Unit) {
    IconButton(
        onClick = onClick,
        modifier = Modifier
            .size(60.dp)
            .semantics { contentDescription = "Liker ce profil" }
    ) {
        Icon(
            painter = painterResource(R.drawable.ic_heart),
            contentDescription = null, // Le parent a la description
            tint = Color.White,
        )
    }
}

@Composable
fun ProfilePhoto(
    url: String,
    name: String,
    age: Int,
    modifier: Modifier = Modifier,
) {
    FlaamPhoto(
        url = url,
        contentDescription = "$name, $age ans",
        modifier = modifier,
    )
}

// Les skeletons sont marqués comme "En chargement"
@Composable
fun SkeletonCard(modifier: Modifier = Modifier) {
    Box(
        modifier = modifier
            .semantics { contentDescription = "Chargement en cours" }
            .background(FlaamColors.Sand, RoundedCornerShape(16.dp))
    )
}
```


---

## 34. API Retrofit manquantes

```kotlin
// ── Report & Block API ──
interface ModerationApi {
    @POST("reports")
    suspend fun reportUser(
        @Body body: ReportBody,
    ): Response<Unit>
    // body: { reported_user_id, reason: "fake_profile"|"inappropriate"|
    //         "harassment"|"scam"|"underage", details?: string }

    @POST("blocks")
    suspend fun blockUser(
        @Body body: BlockBody,
    ): Response<Unit>
    // body: { blocked_user_id }

    @DELETE("blocks/{userId}")
    suspend fun unblockUser(
        @Path("userId") userId: String,
    ): Response<Unit>

    @GET("blocks")
    suspend fun getBlockedUsers(): Response<BlockedUsersResponse>
}


// ── Quartier API ──
interface QuartierApi {
    @GET("cities/{cityId}/quartiers")
    suspend fun getQuartiers(
        @Path("cityId") cityId: String,
    ): Response<QuartiersResponse>
    // Retourne la liste des quartiers avec leur type (résidentiel, commercial, etc.)

    @PUT("profiles/me/quartiers")
    suspend fun updateMyQuartiers(
        @Body body: UpdateQuartiersBody,
    ): Response<Unit>
    // body: {
    //   lives_in: ["quartier_id"],        // 1 seul obligatoire
    //   hangs_in: ["quartier_id", ...],    // 0-5
    //   interested_in: ["quartier_id", ...]// 0-3 free, 0-6 premium
    // }

    @GET("profiles/me/quartiers")
    suspend fun getMyQuartiers(): Response<MyQuartiersResponse>
}


// ── Account Deletion API ──
// Ajouté à AuthApi existant
interface AuthApi {
    // ... (existants : requestOtp, verifyOtp, refreshToken)

    @HTTP(method = "DELETE", path = "auth/account", hasBody = true)
    suspend fun deleteAccount(
        @Body body: DeleteAccountBody,
    ): Response<Unit>
    // body: { reason: "found_someone"|"app_not_for_me"|"safety"|"other",
    //         feedback?: string }
    // Réponse 200 → compte en phase 1 (anonymisé immédiatement)
    // Le serveur gère les phases 2 (J+7) et 3 (J+30)

    @POST("auth/logout")
    suspend fun logout(): Response<Unit>
}


// ── City Launch / Waitlist API ──
interface CityApi {
    @GET("cities")
    suspend fun getCities(): Response<CitiesResponse>
    // Retourne les villes avec leur phase: "teaser"|"launch"|"growth"|"stable"

    @GET("cities/{cityId}/launch-status")
    suspend fun getLaunchStatus(
        @Path("cityId") cityId: String,
    ): Response<CityLaunchStatusResponse>
    // Retourne: { phase, total_registered, threshold, your_position,
    //             invite_count, invite_needed_for_premium }

    @POST("cities/{cityId}/invite")
    suspend fun sendInvite(
        @Path("cityId") cityId: String,
        @Body body: InviteBody,
    ): Response<InviteResponse>
    // body: { phone_hash } → envoie un SMS d'invitation
}


// ── Config API (mentionné dans §25 mais pas défini) ──
interface ConfigApi {
    @GET("config/version")
    suspend fun getMinVersion(): Response<VersionConfigResponse>
    // { min_version: 10, latest_version: 15, update_url, update_message }

    @GET("config/feature-flags")
    suspend fun getFeatureFlags(): Response<FeatureFlagsResponse>
    // { flags: { "voice_messages": true, "events_tab": true, ... } }
}


// ── Behavior API (mentionné dans §23 mais pas défini) ──
interface BehaviorApi {
    @POST("behavior/log")
    suspend fun logBatch(
        @Body body: BehaviorBatchBody,
    ): Response<Unit>
}
```

---

## 35. Selfie Vérification — CameraX + Liveness

```kotlin
/**
 * Flow de vérification selfie :
 * 1. Ouvrir la caméra frontale
 * 2. Détecter un visage (ML Kit Face Detection)
 * 3. Demander une pose aléatoire ("tourne la tête à gauche", "souris", "ferme les yeux")
 * 4. Vérifier que la pose est correcte
 * 5. Capturer la photo
 * 6. Upload au serveur pour vérification
 *
 * La pose aléatoire empêche l'utilisation d'une photo statique.
 * On n'utilise PAS de solution tierce (Onfido, FaceTec) en MVP.
 * ML Kit est gratuit et fonctionne offline.
 */

// Dépendances additionnelles :
// implementation("com.google.mlkit:face-detection:16.1.6")

enum class LivenessChallenge(val instructionFr: String, val instructionEn: String) {
    TURN_LEFT("Tourne la tête à gauche", "Turn your head left"),
    TURN_RIGHT("Tourne la tête à droite", "Turn your head right"),
    SMILE("Souris !", "Smile!"),
    BLINK("Cligne des yeux", "Blink your eyes"),
}

@HiltViewModel
class SelfieVerificationViewModel @Inject constructor(
    private val profileApi: ProfileApi,
    private val photoCompressor: PhotoCompressor,
) : ViewModel() {

    data class SelfieUiState(
        val challenge: LivenessChallenge = LivenessChallenge.entries.random(),
        val faceDetected: Boolean = false,
        val challengePassed: Boolean = false,
        val isCapturing: Boolean = false,
        val isUploading: Boolean = false,
        val error: String? = null,
    )

    private val _state = MutableStateFlow(SelfieUiState())
    val state = _state.asStateFlow()

    private val faceDetector = FaceDetection.getClient(
        FaceDetectorOptions.Builder()
            .setPerformanceMode(FaceDetectorOptions.PERFORMANCE_MODE_FAST)
            .setClassificationMode(FaceDetectorOptions.CLASSIFICATION_MODE_ALL)
            .enableTracking()
            .build()
    )

    /**
     * Appelé à chaque frame de la caméra (via CameraX ImageAnalysis).
     * Vérifie la détection de visage et si le challenge est réussi.
     */
    fun analyzeFace(imageProxy: ImageProxy) {
        val mediaImage = imageProxy.image ?: run {
            imageProxy.close()
            return
        }
        val inputImage = InputImage.fromMediaImage(
            mediaImage, imageProxy.imageInfo.rotationDegrees
        )

        faceDetector.process(inputImage)
            .addOnSuccessListener { faces ->
                if (faces.isEmpty()) {
                    _state.update { it.copy(faceDetected = false, challengePassed = false) }
                    return@addOnSuccessListener
                }

                val face = faces.first()
                _state.update { it.copy(faceDetected = true) }

                // Vérifier le challenge
                val passed = when (_state.value.challenge) {
                    LivenessChallenge.TURN_LEFT ->
                        face.headEulerAngleY > 25f // Tourné à gauche
                    LivenessChallenge.TURN_RIGHT ->
                        face.headEulerAngleY < -25f // Tourné à droite
                    LivenessChallenge.SMILE ->
                        (face.smilingProbability ?: 0f) > 0.7f
                    LivenessChallenge.BLINK ->
                        (face.leftEyeOpenProbability ?: 1f) < 0.3f &&
                        (face.rightEyeOpenProbability ?: 1f) < 0.3f
                }

                if (passed) {
                    _state.update { it.copy(challengePassed = true) }
                }
            }
            .addOnCompleteListener { imageProxy.close() }
    }

    /**
     * Appelé quand le challenge est passé et l'utilisateur tape le bouton capture.
     */
    fun captureAndUpload(imageFile: File) {
        viewModelScope.launch {
            _state.update { it.copy(isUploading = true) }

            try {
                val compressed = photoCompressor.compress(imageFile.toUri())
                val part = MultipartBody.Part.createFormData(
                    "selfie", "selfie.jpg",
                    compressed.asRequestBody("image/jpeg".toMediaType()),
                )

                val response = profileApi.uploadSelfie(part)
                if (response.isSuccessful) {
                    _state.update { it.copy(isUploading = false) }
                    // Navigation vers l'étape suivante gérée par l'OnboardingNavGraph
                } else {
                    _state.update { it.copy(
                        isUploading = false,
                        error = "Vérification échouée. Réessaie.",
                    )}
                }
            } catch (e: Exception) {
                _state.update { it.copy(
                    isUploading = false,
                    error = "Erreur de connexion. Réessaie.",
                )}
            }
        }
    }
}

// Ajout à ProfileApi :
// @Multipart
// @POST("profiles/me/selfie")
// suspend fun uploadSelfie(@Part selfie: MultipartBody.Part): Response<SelfieResponse>
```

---

## 36. Pagination (Paging 3)

```kotlin
/**
 * Pagination pour les listes longues :
 * - Messages dans une conversation (50 par page, scroll vers le haut pour charger plus)
 * - Events (20 par page)
 * - Résultats de recherche de spots
 *
 * On utilise Jetpack Paging 3 avec un RemoteMediator
 * qui gère le cache Room + le fetch API.
 */

// ── Messages Paging Source ──

class MessagesPagingSource(
    private val chatApi: ChatApi,
    private val matchId: String,
) : PagingSource<String, MessageDto>() {

    override fun getRefreshKey(state: PagingState<String, MessageDto>): String? = null

    override suspend fun load(params: LoadParams<String>): LoadResult<String, MessageDto> {
        return try {
            val before = params.key // cursor = ID du message le plus ancien chargé
            val response = chatApi.getMessages(
                matchId = matchId,
                before = before,
                limit = params.loadSize,
            )
            if (response.isSuccessful) {
                val messages = response.body()!!.messages
                LoadResult.Page(
                    data = messages,
                    prevKey = messages.lastOrNull()?.id,  // Messages plus anciens
                    nextKey = null, // Pas de chargement vers le bas (c'est un chat)
                )
            } else {
                LoadResult.Error(ApiException(response.code(), response.message()))
            }
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}


// ── Dans le ConversationViewModel ──

@HiltViewModel
class ConversationViewModel @Inject constructor(
    private val chatApi: ChatApi,
    private val messageDao: MessageDao,
    private val webSocket: FlaamWebSocket,
    savedStateHandle: SavedStateHandle,
) : ViewModel() {

    private val matchId: String = savedStateHandle["matchId"]!!

    // Messages paginés (les anciens) depuis le serveur
    val pagedMessages: Flow<PagingData<MessageUi>> = Pager(
        config = PagingConfig(
            pageSize = 50,
            enablePlaceholders = false,
        ),
        pagingSourceFactory = { MessagesPagingSource(chatApi, matchId) },
    ).flow
        .map { pagingData -> pagingData.map { it.toUi() } }
        .cachedIn(viewModelScope)

    // Messages temps réel (les nouveaux) via WebSocket
    val liveMessages: Flow<MessageUi> = webSocket.incomingMessages
        .filterIsInstance<WsEvent.NewMessage>()
        .filter { it.message.matchId == matchId }
        .map { it.message.toUi() }
}
```

---

## 37. OTP Flow — WhatsApp Fallback Logic

```kotlin
/**
 * Correspondance avec backend spec section 35 (SMS fallback).
 *
 * Flow côté client :
 * 1. POST /auth/otp/request → serveur tente SMS
 * 2. Réponse : { channel: "sms", expires_in: 600, retry_after: 60 }
 * 3. Afficher le countdown "Renvoyer dans 0:47"
 * 4. Si l'utilisateur attend 60s sans recevoir → afficher le bouton WhatsApp
 * 5. Tap WhatsApp → POST /auth/otp/request?channel=whatsapp
 * 6. Réponse : { channel: "whatsapp", ... }
 * 7. OU si le serveur a déjà échoué en SMS, la première réponse dit :
 *    { channel: "whatsapp", ... } → le client affiche directement "Code envoyé sur WhatsApp"
 */

@HiltViewModel
class OtpViewModel @Inject constructor(
    private val authApi: AuthApi,
    private val authDataStore: AuthDataStore,
    private val onboardingManager: OnboardingManager,
    private val deviceFingerprint: DeviceFingerprint,
) : ViewModel() {

    data class OtpUiState(
        val phone: String = "",
        val digits: List<Char?> = List(6) { null },
        val channel: String = "sms",        // "sms" ou "whatsapp"
        val isVerifying: Boolean = false,
        val error: String? = null,
        val retryCountdown: Int = 60,       // Secondes avant de pouvoir renvoyer
        val canRetry: Boolean = false,
        val showWhatsAppFallback: Boolean = false,
        val otpSent: Boolean = false,
    )

    private val _state = MutableStateFlow(OtpUiState())
    val state = _state.asStateFlow()

    private var countdownJob: Job? = null

    fun requestOtp(phone: String) {
        _state.update { it.copy(phone = phone, otpSent = false) }

        viewModelScope.launch {
            try {
                val response = authApi.requestOtp(OtpRequestBody(phone = phone))
                if (response.isSuccessful) {
                    val body = response.body()!!
                    _state.update { it.copy(
                        channel = body.channel, // "sms" ou "whatsapp" (si SMS a échoué)
                        otpSent = true,
                        retryCountdown = body.retryAfter,
                    )}
                    startCountdown(body.retryAfter)
                } else if (response.code() == 429) {
                    val retryAfter = response.headers()["Retry-After"]?.toIntOrNull() ?: 60
                    _state.update { it.copy(
                        error = "Trop de tentatives. Réessaie dans ${retryAfter}s",
                        retryCountdown = retryAfter,
                    )}
                    startCountdown(retryAfter)
                }
            } catch (e: Exception) {
                _state.update { it.copy(error = "Erreur réseau. Réessaie.") }
            }
        }
    }

    fun requestOtpViaWhatsApp() {
        viewModelScope.launch {
            try {
                val response = authApi.requestOtp(
                    OtpRequestBody(phone = _state.value.phone, channel = "whatsapp")
                )
                if (response.isSuccessful) {
                    _state.update { it.copy(
                        channel = "whatsapp",
                        showWhatsAppFallback = false,
                        retryCountdown = 60,
                    )}
                    startCountdown(60)
                }
            } catch (e: Exception) {
                _state.update { it.copy(error = "Erreur. Réessaie.") }
            }
        }
    }

    fun enterDigit(index: Int, digit: Char) {
        val newDigits = _state.value.digits.toMutableList()
        newDigits[index] = digit
        _state.update { it.copy(digits = newDigits, error = null) }

        // Auto-vérifier quand les 6 digits sont remplis
        if (newDigits.all { it != null }) {
            verifyOtp(newDigits.joinToString(""))
        }
    }

    private fun verifyOtp(code: String) {
        viewModelScope.launch {
            _state.update { it.copy(isVerifying = true) }
            try {
                val response = authApi.verifyOtp(OtpVerifyBody(
                    phone = _state.value.phone,
                    code = code,
                    deviceFingerprint = deviceFingerprint.generate(),
                ))
                if (response.isSuccessful) {
                    val body = response.body()!!
                    authDataStore.saveTokens(body.accessToken, body.refreshToken)

                    if (body.isNewUser) {
                        onboardingManager.completeStep(OnboardingStep.OTP)
                    }

                    _state.update { it.copy(isVerifying = false) }
                    // Navigation gérée par le NavGraph
                } else if (response.code() == 400) {
                    _state.update { it.copy(
                        isVerifying = false,
                        error = "Code incorrect",
                        digits = List(6) { null }, // Reset
                    )}
                } else if (response.code() == 410) {
                    _state.update { it.copy(
                        isVerifying = false,
                        error = "Code expiré. Renvoie un nouveau code.",
                        digits = List(6) { null },
                    )}
                } else if (response.code() == 403) {
                    // Compte bloqué (anti-abuse section 30 backend)
                    _state.update { it.copy(
                        isVerifying = false,
                        error = "Compte suspendu. Contacte support@flaam.app",
                    )}
                }
            } catch (e: Exception) {
                _state.update { it.copy(
                    isVerifying = false,
                    error = "Erreur réseau. Réessaie.",
                )}
            }
        }
    }

    private fun startCountdown(seconds: Int) {
        countdownJob?.cancel()
        countdownJob = viewModelScope.launch {
            for (i in seconds downTo 0) {
                _state.update { it.copy(
                    retryCountdown = i,
                    canRetry = i == 0,
                    // Montrer WhatsApp fallback après 60s si le channel est SMS
                    showWhatsAppFallback = i == 0 && _state.value.channel == "sms",
                )}
                delay(1000)
            }
        }
    }
}
```

---

## 38. Crash Reporting & City Waitlist

```kotlin
// ── Crash Reporting (Sentry) ──

/**
 * On utilise Sentry plutôt que Firebase Crashlytics pour :
 * - Pas de dépendance Google Play Services (certains devices en Afrique n'ont pas GMS)
 * - Self-hostable si besoin (data sovereignty)
 * - Meilleure intégration avec le backend Python (même outil partout)
 */

// build.gradle.kts
// implementation("io.sentry:sentry-android:7.14.0")
// implementation("io.sentry:sentry-android-timber:7.14.0")

// FlaamApp.kt
class FlaamApp : Application() {
    override fun onCreate() {
        super.onCreate()

        SentryAndroid.init(this) { options ->
            options.dsn = BuildConfig.SENTRY_DSN
            options.environment = BuildConfig.BUILD_TYPE // debug, staging, release
            options.isAttachScreenshot = true
            options.sampleRate = if (BuildConfig.DEBUG) 1.0 else 0.2 // 20% en prod
            options.tracesSampleRate = 0.1 // 10% des transactions
            options.setBeforeSend { event, _ ->
                // Ne pas envoyer les crashes en debug
                if (BuildConfig.DEBUG) null else event
            }
        }

        // Timber → Sentry bridge
        if (!BuildConfig.DEBUG) {
            Timber.plant(SentryTimberTree())
        } else {
            Timber.plant(Timber.DebugTree())
        }
    }
}


// ── City Waitlist (détaillé) ──

/**
 * Quand la ville est en phase "teaser" (backend spec section 31),
 * l'utilisateur voit l'écran waitlist avec :
 * - Un compteur "247/500"
 * - Sa position dans la file
 * - Le système d'invitations (3 invitations = 1 semaine premium gratuite)
 * - Un polling toutes les 30 minutes pour mettre à jour le compteur
 */

@HiltViewModel
class WaitlistViewModel @Inject constructor(
    private val cityApi: CityApi,
    private val userPrefs: UserPrefsDataStore,
) : ViewModel() {

    data class WaitlistUiState(
        val cityName: String = "",
        val totalRegistered: Int = 0,
        val threshold: Int = 500,
        val userPosition: Int = 0,
        val inviteSent: Int = 0,
        val inviteNeeded: Int = 3,
        val isLaunched: Boolean = false, // Passe à true quand le seuil est atteint
        val estimatedLaunch: String? = null,
    )

    private val _state = MutableStateFlow(WaitlistUiState())
    val state = _state.asStateFlow()

    private var pollingJob: Job? = null

    init {
        loadStatus()
        startPolling()
    }

    private fun loadStatus() {
        viewModelScope.launch {
            try {
                val cityId = userPrefs.getCityId()
                val response = cityApi.getLaunchStatus(cityId)
                if (response.isSuccessful) {
                    val body = response.body()!!
                    _state.update { it.copy(
                        cityName = body.cityName,
                        totalRegistered = body.totalRegistered,
                        threshold = body.threshold,
                        userPosition = body.yourPosition,
                        inviteSent = body.inviteCount,
                        inviteNeeded = body.inviteNeededForPremium,
                        isLaunched = body.phase != "teaser",
                        estimatedLaunch = body.estimatedLaunch,
                    )}
                }
            } catch (_: Exception) { }
        }
    }

    private fun startPolling() {
        pollingJob = viewModelScope.launch {
            while (isActive) {
                delay(30 * 60 * 1000) // 30 minutes
                loadStatus()
            }
        }
    }

    fun sendInvite(contactPhoneHash: String) {
        viewModelScope.launch {
            try {
                val cityId = userPrefs.getCityId()
                val response = cityApi.sendInvite(cityId, InviteBody(contactPhoneHash))
                if (response.isSuccessful) {
                    _state.update { it.copy(inviteSent = it.inviteSent + 1) }
                }
            } catch (_: Exception) { }
        }
    }

    override fun onCleared() {
        pollingJob?.cancel()
        super.onCleared()
    }
}
```

---

# MISE À JOUR — Villes/Pays + Auth sans mot de passe + Email

> À intégrer dans flaam-backend-spec.md et flaam-mobile-spec.md

---

## BACKEND — Mise à jour du modèle City

```python
# app/models/city.py (MIS À JOUR)
"""
Chaque ville a un status qui détermine ce que l'utilisateur voit.
L'écran de sélection de ville affiche TOUTES les villes du pays
sauf celles en status "hidden".
"""

class CityPhase(str, Enum):
    HIDDEN = "hidden"     # Pas affichée du tout (pas dans les plans immédiats)
    TEASER = "teaser"     # Affichée grisée, waitlist active, compteur visible
    LAUNCH = "launch"     # Active, matching activé, boost agressif
    GROWTH = "growth"     # Active, marketing en cours
    STABLE = "stable"     # Active, maturité atteinte


class City(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "cities"

    name: Mapped[str] = mapped_column(String(100), nullable=False)
    country_code: Mapped[str] = mapped_column(String(2), nullable=False)
    # ISO 3166-1 alpha-2 : "TG" = Togo, "CI" = Côte d'Ivoire, etc.

    country_name: Mapped[str] = mapped_column(String(100), nullable=False)
    # "Togo", "Côte d'Ivoire" — pour affichage

    country_flag: Mapped[str] = mapped_column(String(10), nullable=False)
    # Emoji drapeau : "🇹🇬", "🇨🇮"

    phone_prefix: Mapped[str] = mapped_column(String(5), nullable=False)
    # "+228", "+225" — pour auto-détection du pays

    timezone: Mapped[str] = mapped_column(String(50), nullable=False)
    # "Africa/Lome", "Africa/Abidjan"

    phase: Mapped[str] = mapped_column(String(20), default="hidden")
    # hidden | teaser | launch | growth | stable

    display_order: Mapped[int] = mapped_column(Integer, default=0)
    # Ordre d'affichage dans le picker (villes actives en premier)

    # ── Waitlist (pour les villes en teaser) ──
    waitlist_threshold: Mapped[int] = mapped_column(Integer, default=500)
    # Nombre d'inscrits nécessaires pour passer en launch


# ── API : liste des villes par pays ──

# GET /cities?country=TG
# Retourne TOUTES les villes du Togo SAUF celles en status "hidden"
# Triées par display_order (actives d'abord, teaser ensuite)

# Response :
{
    "country_code": "TG",
    "country_name": "Togo",
    "country_flag": "🇹🇬",
    "cities": [
        {
            "id": "uuid-lome",
            "name": "Lomé",
            "phase": "launch",        # ← active, sélectionnable
            "selectable": True,
        },
        {
            "id": "uuid-kara",
            "name": "Kara",
            "phase": "teaser",        # ← grisée, waitlist
            "selectable": False,
            "waitlist": {
                "total_registered": 127,
                "threshold": 500,
                "remaining": 373,
            },
        },
        {
            "id": "uuid-sokode",
            "name": "Sokodé",
            "phase": "teaser",
            "selectable": False,
            "waitlist": {
                "total_registered": 43,
                "threshold": 500,
                "remaining": 457,
            },
        },
    ]
}


# GET /cities (sans filtre pays)
# Retourne TOUS les pays qui ont au moins 1 ville non-hidden
# Utilisé quand quelqu'un tape "Pas au Togo ?" pour changer de pays

# Response :
{
    "countries": [
        {
            "country_code": "TG",
            "country_name": "Togo",
            "country_flag": "🇹🇬",
            "phone_prefix": "+228",
            "active_cities_count": 1,   # Villes en launch/growth/stable
            "teaser_cities_count": 2,   # Villes en teaser
        },
        {
            "country_code": "CI",
            "country_name": "Côte d'Ivoire",
            "country_flag": "🇨🇮",
            "phone_prefix": "+225",
            "active_cities_count": 1,
            "teaser_cities_count": 1,
        },
        # ...
    ]
}


# POST /cities/{cityId}/waitlist/join
# Quand quelqu'un sélectionne une ville en teaser
# L'utilisateur est inscrit en waitlist pour cette ville
# Il peut continuer l'onboarding normalement avec cette ville
# mais son feed sera vide (mode waitlist) tant que la ville n'est pas active
```

---

## BACKEND — Auth sans mot de passe + Email recovery

```python
# app/models/user.py (CHAMPS AJOUTÉS)
"""
Modèle d'authentification : PASSWORDLESS (comme WhatsApp)
- Auth principale : OTP par SMS/WhatsApp
- Pas de mot de passe. Jamais.
- Email optionnel mais fortement encouragé (recovery + notifications)
- MFA optionnel : PIN à 6 chiffres

Pourquoi pas de mot de passe :
1. Cible Afrique de l'Ouest → beaucoup d'utilisateurs pas habitués aux mots de passe
2. Moins de friction à l'inscription (pas de "8 caractères dont 1 majuscule...")
3. Pas de "mot de passe oublié" (le problème #1 du support)
4. Le téléphone EST le facteur d'authentification (comme WhatsApp, Telegram)
"""

class User(Base, UUIDMixin, TimestampMixin):
    # ... (champs existants)

    # ── Email (optionnel mais encouragé) ──
    email: Mapped[str | None] = mapped_column(String(255), nullable=True, unique=True)
    is_email_verified: Mapped[bool] = mapped_column(Boolean, default=False)
    email_verified_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    # ── MFA optionnel ──
    mfa_enabled: Mapped[bool] = mapped_column(Boolean, default=False)
    mfa_pin_hash: Mapped[str | None] = mapped_column(String(128), nullable=True)
    # PIN à 6 chiffres, hashé bcrypt. Demandé à chaque login si activé.
    # PAS un mot de passe — c'est un PIN court pour empêcher quelqu'un
    # qui a accès au téléphone de se connecter sans le code.

    # ── Recovery ──
    recovery_email: Mapped[str | None] = mapped_column(String(255), nullable=True)
    # Peut être différent de l'email principal
    # Utilisé UNIQUEMENT pour la récupération de compte


# ── Scénarios d'authentification ──

AUTH_SCENARIOS = {

    "inscription_normale": {
        # 1. Entrer le numéro
        # 2. Recevoir l'OTP (SMS ou WhatsApp)
        # 3. Vérifier l'OTP
        # 4. Compte créé → JWT émis
        # 5. Plus tard dans l'onboarding ou les settings :
        #    "Ajoute ton email pour sécuriser ton compte" (encouragé mais pas bloquant)
    },

    "connexion_normale": {
        # 1. Entrer le numéro
        # 2. OTP
        # 3. Si MFA activé → demander le PIN à 6 chiffres
        # 4. JWT émis
    },

    "changement_de_telephone": {
        # L'utilisateur a un nouveau numéro mais le même téléphone (ou un nouveau)
        #
        # Option A : depuis l'app (connecté)
        # Settings > Compte > Changer de numéro
        # 1. Vérifier l'ancien numéro (OTP)
        # 2. Entrer le nouveau numéro
        # 3. Vérifier le nouveau numéro (OTP)
        # 4. Numéro mis à jour
        #
        # Option B : perdu l'accès au téléphone
        # → voir "recovery" ci-dessous
    },

    "recovery_numero_perdu": {
        # L'utilisateur a perdu son téléphone ET changé de numéro.
        # Il ne peut plus recevoir d'OTP sur l'ancien numéro.
        #
        # PRÉREQUIS : avoir ajouté un email de récupération
        #
        # Flow :
        # 1. Écran login → "J'ai perdu mon numéro"
        # 2. Entrer l'email de récupération
        # 3. Recevoir un lien de récupération par email (expire en 1h)
        # 4. Cliquer le lien → ouvre l'app
        # 5. Entrer le nouveau numéro
        # 6. Vérifier le nouveau numéro (OTP)
        # 7. Ancien numéro remplacé, JWT émis
        #
        # SANS email de récupération :
        # → Support manuel (support@flaam.app)
        # → Vérification d'identité (selfie + pièce d'identité)
        # → Délai 24-48h
    },

    "mfa_pin": {
        # Optionnel. Activable dans Settings > Sécurité.
        # PIN à 6 chiffres demandé à chaque nouveau login.
        # Pas demandé si le JWT est encore valide (l'app est déjà ouverte).
        #
        # Cas d'usage : empêcher un collègue/ami qui a accès au téléphone
        # de se connecter à Flaam avec un OTP intercepté.
        #
        # PIN oublié → recovery via email
    },
}


# ── API endpoints ──

# POST /auth/email/add
# { "email": "raouf@example.com" }
# → Envoie un email de vérification avec un lien
# → Le lien contient un token (expire en 24h)

# POST /auth/email/verify
# { "token": "abc123..." }
# → Marque l'email comme vérifié

# POST /auth/recovery/request
# { "email": "raouf@example.com" }
# → Envoie un lien de récupération (expire en 1h)
# → Retourne 200 même si l'email n'existe pas (sécurité)

# POST /auth/recovery/confirm
# { "recovery_token": "abc123...", "new_phone": "+22890123456" }
# → Déclenche un OTP vers le nouveau numéro

# POST /auth/recovery/complete
# { "recovery_token": "abc123...", "otp": "482917" }
# → Ancien numéro remplacé, JWT émis

# POST /auth/mfa/enable
# { "pin": "142857" }
# → Hash le PIN, active le MFA

# POST /auth/mfa/verify
# { "pin": "142857" }
# → Appelé après l'OTP si MFA est activé
# → Si OK → JWT émis

# POST /auth/mfa/disable
# { "pin": "142857" }  (doit fournir le PIN actuel pour désactiver)


# ── Utilisation de l'email (au-delà de la recovery) ──

EMAIL_USES = {
    "account_recovery": True,       # Toujours
    "payment_receipts": True,       # Reçu après chaque paiement premium
    "security_alerts": True,        # "Nouvelle connexion depuis un device inconnu"
    "weekly_digest": "opt_in",      # "Tu as 3 nouveaux likes cette semaine"
    "event_invitations": "opt_in",  # "Afterwork vendredi à Tokoin"
    "marketing": "opt_in",          # "Nouvelle feature : messages vocaux !"
    # Les opt_in sont gérés dans NotificationPreference
    # L'utilisateur choisit dans Settings > Notifications > Email
}
```

---

## MOBILE — Mise à jour écran Ville

```kotlin
/**
 * L'écran ville apparaît TOUJOURS dans l'onboarding.
 * Il montre toutes les villes du pays (détecté via l'indicatif).
 *
 * 3 états visuels :
 * - Active (vert) → tappable, continue l'onboarding normalement
 * - Teaser (grisé + badge "Bientôt") → tappable, inscrit en waitlist
 * - Hidden → pas affichée
 */

@HiltViewModel
class CitySelectionViewModel @Inject constructor(
    private val cityApi: CityApi,
    private val userPrefs: UserPrefsDataStore,
) : ViewModel() {

    data class CityUiState(
        val countryCode: String = "TG",
        val countryName: String = "Togo",
        val countryFlag: String = "🇹🇬",
        val cities: List<CityUi> = emptyList(),
        val selectedCityId: String? = null,
        val isLoading: Boolean = true,
        val showCountryPicker: Boolean = false,
    )

    data class CityUi(
        val id: String,
        val name: String,
        val phase: String,          // "launch", "growth", "stable", "teaser"
        val selectable: Boolean,    // true si active
        val waitlistCount: Int? = null,
        val waitlistThreshold: Int? = null,
    )

    private val _state = MutableStateFlow(CityUiState())
    val state = _state.asStateFlow()

    fun detectCountryFromPhone(phonePrefix: String) {
        // "+228" → "TG", "+225" → "CI", etc.
        val countryCode = PhonePrefixMap.getCountry(phonePrefix)
        loadCities(countryCode)
    }

    private fun loadCities(countryCode: String) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val response = cityApi.getCitiesByCountry(countryCode)
                if (response.isSuccessful) {
                    val body = response.body()!!
                    _state.update { it.copy(
                        countryCode = body.countryCode,
                        countryName = body.countryName,
                        countryFlag = body.countryFlag,
                        cities = body.cities.map { c -> CityUi(
                            id = c.id,
                            name = c.name,
                            phase = c.phase,
                            selectable = c.selectable,
                            waitlistCount = c.waitlist?.totalRegistered,
                            waitlistThreshold = c.waitlist?.threshold,
                        )},
                        isLoading = false,
                    )}
                }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false) }
            }
        }
    }

    fun selectCity(cityId: String) {
        val city = _state.value.cities.find { it.id == cityId } ?: return

        if (city.selectable) {
            // Ville active → continuer l'onboarding
            _state.update { it.copy(selectedCityId = cityId) }
            viewModelScope.launch { userPrefs.saveCityId(cityId) }
        } else {
            // Ville en teaser → inscrire en waitlist
            viewModelScope.launch {
                try {
                    cityApi.joinWaitlist(cityId)
                    _state.update { it.copy(selectedCityId = cityId) }
                    userPrefs.saveCityId(cityId)
                    // L'utilisateur continue l'onboarding normalement
                    // mais son feed sera en mode waitlist
                } catch (_: Exception) { }
            }
        }
    }

    fun changeCountry() {
        _state.update { it.copy(showCountryPicker = true) }
    }
}
```

---

## MOBILE — Mise à jour onboarding : ajout de l'écran Email

```kotlin
/**
 * Après l'onboarding principal (profil complet), on propose
 * d'ajouter un email. C'est OPTIONNEL mais fortement encouragé.
 *
 * Cet écran apparaît dans l'écran "C'est prêt !" (écran 14)
 * comme un encart, PAS comme un écran bloquant séparé.
 *
 * Le texte dit :
 * "Ajoute ton email pour sécuriser ton compte.
 *  Si tu perds ton numéro, c'est le seul moyen de récupérer ton compte."
 */

// Modifié dans OnboardingStep : PAS de nouvelle étape
// L'email est proposé dans l'écran Done, pas un écran séparé

// Settings > Sécurité ajoute ces options :
SECURITY_SETTINGS = {
    "email_recovery": {
        // "Email de récupération" → tap → écran pour ajouter/changer l'email
        // Si pas encore ajouté : badge orange "Recommandé"
        // Si ajouté et vérifié : badge vert "Actif"
    },
    "mfa_pin": {
        // "Code PIN" → tap → activer/désactiver le PIN 6 chiffres
        // Si activé : toggle vert
        // Si désactivé : texte "Ajoute un PIN pour protéger ton compte"
    },
    "change_phone": {
        // "Changer de numéro" → tap → flow vérification ancien + nouveau
    },
}


// ── Écran de login : cas "numéro perdu" ──

// Sur l'écran Phone (écran 02), ajouter un lien discret en bas :
// "J'ai perdu mon numéro" → ouvre un écran :
//
// 1. "Entre l'email associé à ton compte Flaam"
// 2. Si l'email existe → "Un lien de récupération a été envoyé"
// 3. Si l'email n'existe pas → même message (sécurité)
//    + "Si tu n'as pas d'email associé, contacte support@flaam.app"
// 4. L'utilisateur clique le lien dans son email
// 5. Deep link : flaam://recovery?token=abc123
// 6. L'app s'ouvre sur l'écran "Nouveau numéro"
// 7. Il entre son nouveau numéro → OTP → connecté
```

---

## MOBILE — Mise à jour Retrofit : nouveaux endpoints

```kotlin
// Ajoutés à AuthApi

interface AuthApi {
    // ... (existants)

    // ── Email ──
    @POST("auth/email/add")
    suspend fun addEmail(@Body body: AddEmailBody): Response<Unit>
    // { "email": "raouf@example.com" }

    @POST("auth/email/verify")
    suspend fun verifyEmail(@Body body: VerifyEmailBody): Response<Unit>
    // { "token": "abc123..." }

    // ── Recovery ──
    @POST("auth/recovery/request")
    suspend fun requestRecovery(@Body body: RecoveryRequestBody): Response<Unit>
    // { "email": "raouf@example.com" }

    @POST("auth/recovery/confirm")
    suspend fun confirmRecovery(@Body body: RecoveryConfirmBody): Response<Unit>
    // { "recovery_token": "abc123...", "new_phone": "+22890123456" }

    @POST("auth/recovery/complete")
    suspend fun completeRecovery(@Body body: RecoveryCompleteBody): Response<AuthTokenResponse>
    // { "recovery_token": "abc123...", "otp": "482917" }

    // ── MFA ──
    @POST("auth/mfa/enable")
    suspend fun enableMfa(@Body body: MfaPinBody): Response<Unit>
    // { "pin": "142857" }

    @POST("auth/mfa/verify")
    suspend fun verifyMfa(@Body body: MfaPinBody): Response<AuthTokenResponse>
    // { "pin": "142857" }

    @POST("auth/mfa/disable")
    suspend fun disableMfa(@Body body: MfaPinBody): Response<Unit>

    // ── Phone change ──
    @POST("auth/phone/change/verify-old")
    suspend fun verifyOldPhone(@Body body: OtpVerifyBody): Response<PhoneChangeTokenResponse>

    @POST("auth/phone/change/set-new")
    suspend fun setNewPhone(@Body body: SetNewPhoneBody): Response<AuthTokenResponse>
    // { "change_token": "...", "new_phone": "+228...", "otp": "..." }
}


// Ajouté à CityApi

interface CityApi {
    // ... (existants)

    @GET("cities")
    suspend fun getCitiesByCountry(
        @Query("country") countryCode: String,
    ): Response<CitiesByCountryResponse>

    @GET("cities/countries")
    suspend fun getAvailableCountries(): Response<CountriesResponse>

    @POST("cities/{cityId}/waitlist/join")
    suspend fun joinWaitlist(
        @Path("cityId") cityId: String,
    ): Response<WaitlistJoinResponse>
}
```


---

# MISE À JOUR 2 — Email hors onboarding + Limites spots

---

## CORRECTION : Email retiré de l'onboarding

```python
# AVANT (FAUX) :
# L'email était proposé dans l'écran "C'est prêt !" (écran 14)
#
# APRÈS (CORRECT) :
# L'email n'apparaît NULLE PART dans l'onboarding.
# Il vit uniquement dans Settings > Sécurité > Email de récupération
#
# L'onboarding reste :
# Phone → OTP → Ville → Infos → Selfie → Photos → Quartiers →
# Intention → Secteur → Prompts → Tags → Spots → C'est prêt
#
# 0 mention d'email. 0 mention de PIN. 0 friction supplémentaire.
#
# Dans Settings > Sécurité, l'utilisateur voit :
# - "Email de récupération" avec badge orange "Recommandé" si pas encore ajouté
# - "Code PIN" avec toggle désactivé par défaut
# - "Changer de numéro"
# - "Vérification selfie" (statut)
#
# On peut envoyer une notification push 7 jours après l'inscription :
# "Sécurise ton compte ! Ajoute un email de récupération au cas où tu perds ton numéro."
# Ça pousse vers Settings > Sécurité sans polluer l'onboarding.

SECURITY_SETTINGS_SCREEN = {
    "sections": [
        {
            "title": "Protection du compte",
            "items": [
                {
                    "label": "Email de récupération",
                    "type": "navigation",
                    "badge": "Recommandé",          # Orange si pas encore ajouté
                    "badge_type": "warning",
                    "value_if_set": "r***f@example.com",  # Masqué
                    "value_if_not_set": "Non configuré",
                    "description": "Si tu perds ton numéro, c'est le seul moyen de récupérer ton compte.",
                },
                {
                    "label": "Code PIN",
                    "type": "toggle",
                    "default": False,
                    "description": "Un code à 6 chiffres demandé à chaque connexion.",
                },
                {
                    "label": "Changer de numéro",
                    "type": "navigation",
                    "value": "+228 90 ** 56",
                },
            ],
        },
        {
            "title": "Vérification",
            "items": [
                {
                    "label": "Selfie vérifié",
                    "type": "status",
                    "value": "Vérifié",             # Vert
                    "badge_type": "success",
                },
            ],
        },
        {
            "title": "Contact de confiance",
            "items": [
                {
                    "label": "Contact pour les dates",
                    "type": "navigation",
                    "value_if_set": "Maman",
                    "value_if_not_set": "Non configuré",
                    "description": "Reçoit un SMS quand tu partages un date + alerte si le timer expire.",
                },
            ],
        },
        {
            "title": "Données",
            "items": [
                {
                    "label": "Contacts masqués",
                    "type": "navigation",
                    "value": "3 contacts",
                },
                {
                    "label": "Exporter mes données",
                    "type": "navigation",
                },
                {
                    "label": "Supprimer mon compte",
                    "type": "navigation",
                    "destructive": True,
                },
            ],
        },
    ],
}
```

---

## AJOUT : Limites des spots

```python
# app/constants.py (AJOUT)

SPOT_LIMITS = {
    "free": {
        "max_spots": 5,
        # L'utilisateur free peut déclarer jusqu'à 5 spots sur son profil
        # Au-delà → message "Passe à Premium pour ajouter plus de spots"
    },
    "premium": {
        "max_spots": 12,
        # Premium peut avoir jusqu'à 12 spots
        # Suffisant pour couvrir ses habitudes sans diluer le signal
    },

    # Pourquoi limiter :
    # 1. Signal géo : si quelqu'un a 50 spots, le score géo "spots en commun"
    #    perd sa valeur (tout le monde a des spots en commun avec lui)
    # 2. Levier premium : "ajoute plus de spots" est un upsell naturel
    # 3. Anti-gaming : empêche de tagger tous les spots de la ville pour
    #    maximiser artificiellement la visibilité
}

# Le check-in n'est pas limité — tu peux faire autant de check-ins
# que tu veux dans tes spots existants. C'est le NOMBRE de spots
# sur ton profil qui est limité, pas l'activité.


# ── Backend : vérification à l'ajout d'un spot ──

async def add_spot_to_profile(user_id, spot_id, db_session):
    user = await get_user(user_id)
    current_spots = await count_user_spots(user_id, db_session)

    max_spots = SPOT_LIMITS["premium"]["max_spots"] if user.is_premium \
        else SPOT_LIMITS["free"]["max_spots"]

    if current_spots >= max_spots:
        if user.is_premium:
            raise SpotLimitReached("Tu as atteint la limite de 12 spots.")
        else:
            raise SpotLimitReached(
                "Tu as atteint la limite de 5 spots. "
                "Passe à Premium pour en ajouter jusqu'à 12."
            )

    # Ajouter le spot
    # ...


# ── Mobile : ce que voit l'utilisateur ──

# Écran Spots (tab) :
# En bas de la liste des spots, le compteur :
# "3 / 5 spots" (free) ou "7 / 12 spots" (premium)
#
# Quand le max est atteint et qu'il tape "+ Ajouter" :
# Free → bottom sheet upsell premium
# Premium → message "Tu as atteint la limite de 12 spots.
#           Retire un spot existant pour en ajouter un nouveau."

# Écran Onboarding spots (écran 13) :
# Même limite appliquée. Le compteur est visible.
# Mais pendant l'onboarding on affiche pas l'upsell premium
# (trop tôt, l'utilisateur ne comprend pas encore la valeur).
```


---

# MISE À JOUR 4 — Algo L3+L4 manquants + Mobile events

---

## BACKEND — Section 6.3b : L3 — Score lifestyle (MANQUAIT)

```python
# app/services/matching_engine/lifestyle_scorer.py
"""
L3 : Score de compatibilité lifestyle.

Composantes :
1. Tags TF-IDF (Jaccard pondéré)
2. Matrice compatibilité intentions (4×4)
3. Similitude rythme de vie
4. Bonus langues communes

Score final normalisé entre 0.0 et 1.0.
Poids configurables via MatchingConfig.
"""

from app.core.constants import INTENTION_COMPATIBILITY_MATRIX


async def compute_lifestyle_scores(
    user,
    candidate_profiles: list,
    config: dict,
) -> dict[str, float]:
    """
    Retourne {candidate_id: lifestyle_score} pour chaque candidat.
    """
    scores = {}

    for candidate in candidate_profiles:
        score = 0.0

        # ── 1. Tags Jaccard pondéré (50% du score lifestyle) ──
        # Jaccard = |A ∩ B| / |A ∪ B|
        # Pondéré : les tags rares valent plus que les tags communs
        # Un tag que 5% des users ont vaut plus qu'un tag que 80% ont
        user_tags = set(user.profile.tags or [])
        candidate_tags = set(candidate.tags or [])

        if user_tags and candidate_tags:
            intersection = user_tags & candidate_tags
            union = user_tags | candidate_tags
            if union:
                # Jaccard simple
                jaccard = len(intersection) / len(union)
                # Pondération par rareté (TF-IDF simplifié)
                # Les tags rares en commun comptent plus
                # tag_weight = 1 / log(1 + frequency_in_city)
                # Pour le MVP : Jaccard simple suffit
                score += jaccard * config.get("lifestyle_w_tags", 0.50)

        # ── 2. Matrice intentions (25% du score lifestyle) ──
        # 4 intentions : serious, getting_to_know, friendship, open
        # Matrice de compatibilité :
        #                serious  getting_to_know  friendship  open
        # serious          1.0        0.5            0.1       0.7
        # getting_to_know  0.5        1.0            0.6       0.8
        # friendship       0.1        0.6            1.0       0.5
        # open             0.7        0.8            0.5       1.0
        user_intention = user.profile.intention
        candidate_intention = candidate.intention

        intention_score = INTENTION_COMPATIBILITY_MATRIX.get(
            user_intention, {}
        ).get(candidate_intention, 0.5)

        score += intention_score * config.get("lifestyle_w_intention", 0.25)

        # ── 3. Rythme de vie (15% du score lifestyle) ──
        # "early_bird" vs "night_owl" vs "flexible"
        # Même rythme = 1.0, flexible + anything = 0.7, opposés = 0.3
        user_rhythm = user.profile.rhythm or "flexible"
        candidate_rhythm = candidate.rhythm or "flexible"

        if user_rhythm == candidate_rhythm:
            rhythm_score = 1.0
        elif "flexible" in (user_rhythm, candidate_rhythm):
            rhythm_score = 0.7
        else:
            rhythm_score = 0.3  # early_bird vs night_owl

        score += rhythm_score * config.get("lifestyle_w_rhythm", 0.15)

        # ── 4. Langues communes (10% du score lifestyle) ──
        # Bonus si on partage au moins une langue
        # Plus de langues en commun = meilleur score
        user_langs = set(user.profile.languages or [])
        candidate_langs = set(candidate.languages or [])

        if user_langs and candidate_langs:
            common_langs = user_langs & candidate_langs
            if common_langs:
                # Au moins 1 langue commune → score proportionnel
                lang_score = min(1.0, len(common_langs) / 2)
                # 1 langue = 0.5, 2+ langues = 1.0
            else:
                lang_score = 0.0  # Pas de langue commune = problème
        else:
            lang_score = 0.5  # Données manquantes → neutre

        score += lang_score * config.get("lifestyle_w_languages", 0.10)

        # ── Normalisation ──
        scores[str(candidate.user_id)] = min(1.0, max(0.0, score))

    return scores


# ── Constante : matrice d'intentions ──

INTENTION_COMPATIBILITY_MATRIX = {
    "serious": {
        "serious": 1.0,
        "getting_to_know": 0.5,
        "friendship": 0.1,
        "open": 0.7,
    },
    "getting_to_know": {
        "serious": 0.5,
        "getting_to_know": 1.0,
        "friendship": 0.6,
        "open": 0.8,
    },
    "friendship": {
        "serious": 0.1,
        "getting_to_know": 0.6,
        "friendship": 1.0,
        "open": 0.5,
    },
    "open": {
        "serious": 0.7,
        "getting_to_know": 0.8,
        "friendship": 0.5,
        "open": 1.0,
    },
}
```

---

## BACKEND — Section 6.3c : L4 — Multiplicateur comportemental (MANQUAIT)

```python
# app/services/matching_engine/behavior_scorer.py
"""
L4 : Multiplicateur comportemental.

Pas un "score" mais un MULTIPLICATEUR appliqué au score final.
Récompense les utilisateurs qui utilisent l'app de manière saine.
Pénalise ceux qui spamment ou ne répondent jamais.

4 composantes, chacune produit un facteur entre min et max.
Le multiplicateur final = produit des 4 facteurs.
Résultat typique : 0.6 → 1.4 (borné par config).

Stocké dans Redis (behavior:{user_id}) pour accès rapide.
Persisté en DB (profiles.behavior_multiplier) par cron horaire.
"""


async def get_behavior_multipliers(
    candidate_ids: list[str],
    redis_client,
    db_session,
) -> dict[str, float]:
    """
    Retourne {candidate_id: multiplier} pour chaque candidat.
    Source : Redis d'abord, fallback DB.
    """
    multipliers = {}

    # Batch Redis GET
    pipe = redis_client.pipeline()
    for cid in candidate_ids:
        pipe.get(f"behavior:{cid}")
    results = await pipe.execute()

    for cid, cached in zip(candidate_ids, results):
        if cached is not None:
            multipliers[cid] = float(cached)
        else:
            # Fallback DB
            profile = await get_profile(cid, db_session)
            multipliers[cid] = profile.behavior_multiplier if profile else 1.0

    return multipliers


async def update_behavior_on_action(
    user_id: str,
    action_type: str,
    metadata: dict,
    redis_client,
    db_session,
    config: dict,
):
    """
    Appelé à chaque action utilisateur (like, skip, message envoyé,
    message reçu, match, etc.) pour mettre à jour le behavior score.

    Les 4 composantes :

    1. RESPONSE QUALITY (répond-il à ses matchs ?)
       - Compteur : messages_received / messages_sent_first
       - Bon = répond vite et régulièrement
       - Mauvais = ne répond jamais, ghost ses matchs
       - Range : config["behavior_response_min"] (0.6) → config["behavior_response_max"] (1.4)

    2. SELECTIVITY (est-il sélectif dans ses likes ?)
       - Compteur : likes / total_profiles_seen
       - Bon = like 20-40% des profils (sélectif mais actif)
       - Mauvais = like 100% (spam) ou 0% (inactif)
       - Range : config["behavior_selectivity_min"] (0.7) → config["behavior_selectivity_max"] (1.3)

    3. PROFILE RICHNESS (profil complet ?)
       - Score de complétion : photos, prompts, tags, spots
       - Bon = profil complet avec 3+ prompts, 5+ spots
       - Mauvais = profil minimal (3 photos, 0 prompts, 0 spots)
       - Range : config["behavior_richness_min"] (0.8) → config["behavior_richness_max"] (1.2)

    4. CONVERSATION DEPTH (qualité des conversations ?)
       - Moyenne : messages par match actif
       - Bon = conversations longues avec échanges réels
       - Mauvais = "salut" et plus rien
       - Range : config["behavior_depth_min"] (0.8) → config["behavior_depth_max"] (1.3)
    """

    # Lire les compteurs actuels depuis Redis
    stats_key = f"behavior_stats:{user_id}"
    stats = await redis_client.hgetall(stats_key)

    # Mettre à jour selon l'action
    if action_type == "like":
        await redis_client.hincrby(stats_key, "total_likes", 1)
    elif action_type == "skip":
        await redis_client.hincrby(stats_key, "total_skips", 1)
    elif action_type == "message_sent":
        await redis_client.hincrby(stats_key, "messages_sent", 1)
    elif action_type == "message_received":
        await redis_client.hincrby(stats_key, "messages_received", 1)
    elif action_type == "match_response":
        await redis_client.hincrby(stats_key, "matches_responded", 1)
    elif action_type == "profile_viewed":
        await redis_client.hincrby(stats_key, "profiles_viewed", 1)

    # Recalculer les 4 composantes
    total_likes = int(stats.get("total_likes", 0)) + (1 if action_type == "like" else 0)
    total_profiles = int(stats.get("profiles_viewed", 0)) + (1 if action_type == "profile_viewed" else 0)
    messages_sent = int(stats.get("messages_sent", 0))
    messages_received = int(stats.get("messages_received", 0))
    matches_responded = int(stats.get("matches_responded", 0))
    total_matches = int(stats.get("total_matches", 0))

    # 1. Response quality
    if total_matches > 0:
        response_rate = matches_responded / total_matches
        response_factor = lerp(
            config["behavior_response_min"],
            config["behavior_response_max"],
            min(1.0, response_rate),
        )
    else:
        response_factor = 1.0  # Pas encore de matchs → neutre

    # 2. Selectivity
    if total_profiles > 10:  # Minimum 10 profils vus pour évaluer
        like_rate = total_likes / total_profiles
        # Sweet spot = 20-40%. Trop haut ou trop bas = pénalité
        if 0.15 <= like_rate <= 0.45:
            selectivity_factor = config["behavior_selectivity_max"]
        elif like_rate > 0.80:  # Like tout = spam
            selectivity_factor = config["behavior_selectivity_min"]
        elif like_rate < 0.05:  # Like personne = inactif
            selectivity_factor = config["behavior_selectivity_min"]
        else:
            selectivity_factor = 1.0
    else:
        selectivity_factor = 1.0

    # 3. Richness (lire depuis le profil en DB, c'est statique)
    profile = await get_profile(user_id, db_session)
    completeness = calculate_completeness(profile)
    richness_factor = lerp(
        config["behavior_richness_min"],
        config["behavior_richness_max"],
        completeness,
    )

    # 4. Conversation depth
    if messages_sent > 0 and total_matches > 0:
        avg_messages_per_match = messages_sent / max(1, total_matches)
        # 5+ messages par match = bon, 1 message = mauvais
        depth_factor = lerp(
            config["behavior_depth_min"],
            config["behavior_depth_max"],
            min(1.0, avg_messages_per_match / 5),
        )
    else:
        depth_factor = 1.0

    # Multiplicateur final = produit des 4 facteurs
    multiplier = response_factor * selectivity_factor * richness_factor * depth_factor

    # Borner entre 0.6 et 1.4 (sécurité)
    multiplier = max(0.6, min(1.4, multiplier))

    # Stocker dans Redis
    await redis_client.set(f"behavior:{user_id}", str(round(multiplier, 3)))

    return multiplier


def lerp(min_val: float, max_val: float, t: float) -> float:
    """Interpolation linéaire entre min et max."""
    return min_val + (max_val - min_val) * max(0.0, min(1.0, t))
```

---

## MOBILE — Mise à jour Events

```kotlin
/**
 * Le mobile doit connaître les catégories et status d'events
 * pour l'affichage correct dans l'UI.
 */

// ── Enums côté client ──

enum class EventCategory(val labelFr: String, val labelEn: String) {
    AFTERWORK("Afterwork", "Afterwork"),
    SPORT("Sport", "Sport"),
    BRUNCH("Brunch", "Brunch"),
    CULTURAL("Culture", "Cultural"),
    NETWORKING("Networking", "Networking"),
    WORKSHOP("Atelier", "Workshop"),
    OUTDOOR("Plein air", "Outdoor"),
}

enum class EventStatus {
    PUBLISHED,   // Visible, inscription possible
    FULL,        // Visible, inscription fermée (capacité max)
    ONGOING,     // En cours (le jour J)
    COMPLETED,   // Terminé (disparaît du feed)
    CANCELLED,   // Annulé
}


// ── EventsViewModel (complété) ──

@HiltViewModel
class EventsViewModel @Inject constructor(
    private val eventApi: EventApi,
    private val userPrefs: UserPrefsDataStore,
) : ViewModel() {

    data class EventsUiState(
        val events: List<EventUi> = emptyList(),
        val selectedCategory: EventCategory? = null, // null = "Tout"
        val isLoading: Boolean = true,
        val error: String? = null,
    )

    data class EventUi(
        val id: String,
        val title: String,
        val description: String,
        val category: EventCategory,
        val venueName: String,
        val startsAt: Instant,
        val endsAt: Instant,
        val maxCapacity: Int?,
        val currentRegistrations: Int,
        val isFull: Boolean,
        val isSponsored: Boolean,
        val sponsorName: String?,
        val coverImageUrl: String?,
        val coverColor: String,
        val potentialMatches: Int?, // null si pas encore chargé
        val isRegistered: Boolean,
        val dayLabel: String, // "Vendredi", "Samedi", "Dimanche"
        val timeLabel: String, // "18h — 21h"
    )

    private val _state = MutableStateFlow(EventsUiState())
    val state = _state.asStateFlow()

    init { loadEvents() }

    private fun loadEvents() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val cityId = userPrefs.getCityId()
                val response = eventApi.getEvents(cityId = cityId)
                if (response.isSuccessful) {
                    _state.update { it.copy(
                        events = response.body()!!.events.map { e -> e.toUi() },
                        isLoading = false,
                    )}
                }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, error = e.toFlaamError().toUserMessage()) }
            }
        }
    }

    fun filterByCategory(category: EventCategory?) {
        _state.update { it.copy(selectedCategory = category) }
    }

    fun register(eventId: String) {
        viewModelScope.launch {
            try {
                val response = eventApi.registerEvent(eventId)
                if (response.isSuccessful) {
                    _state.update { state ->
                        state.copy(events = state.events.map { e ->
                            if (e.id == eventId) e.copy(
                                isRegistered = true,
                                currentRegistrations = e.currentRegistrations + 1,
                                isFull = e.maxCapacity != null &&
                                         e.currentRegistrations + 1 >= e.maxCapacity,
                            ) else e
                        })
                    }
                }
            } catch (_: Exception) { }
        }
    }

    fun unregister(eventId: String) {
        viewModelScope.launch {
            try {
                eventApi.unregisterEvent(eventId)
                _state.update { state ->
                    state.copy(events = state.events.map { e ->
                        if (e.id == eventId) e.copy(
                            isRegistered = false,
                            currentRegistrations = e.currentRegistrations - 1,
                            isFull = false,
                        ) else e
                    })
                }
            } catch (_: Exception) { }
        }
    }
}
```

---

# MISE À JOUR 5 — API Retrofit manquantes + corrections de paths

---

## Corrections de paths (alignement avec le catalogue des endpoints)

```kotlin
/**
 * IMPORTANT : Les paths dans les interfaces Retrofit doivent correspondre
 * EXACTEMENT au catalogue des endpoints (flaam-api-endpoints.md).
 * 
 * Corrections appliquées :
 * - Photos : /profiles/me/photos → /photos (le catalogue fait foi)
 * - Contacts : /contacts/blacklist (pas /safety/contacts/import)
 */

// ── ProfileApi (CORRIGÉ) ──
// Les méthodes photos sont déplacées dans PhotoApi séparé
// Ajout des méthodes onboarding manquantes

interface ProfileApi {
    @GET("profiles/me")
    suspend fun getMyProfile(): Response<MyProfileResponse>

    @PUT("profiles/me")
    suspend fun updateProfile(@Body body: UpdateProfileBody): Response<MyProfileResponse>

    @GET("profiles/{userId}")
    suspend fun getProfile(@Path("userId") userId: String): Response<ProfileResponse>

    @Multipart
    @POST("profiles/me/selfie")
    suspend fun uploadSelfie(@Part selfie: MultipartBody.Part): Response<SelfieResponse>

    @GET("profiles/me/completeness")
    suspend fun getCompleteness(): Response<CompletenessResponse>

    @PATCH("profiles/me/visibility")
    suspend fun toggleVisibility(@Body body: VisibilityBody): Response<Unit>

    // ── Onboarding (MANQUAIT) ──
    @GET("profiles/me/onboarding")
    suspend fun getOnboardingState(): Response<OnboardingStateResponse>

    @POST("profiles/me/onboarding/skip")
    suspend fun skipOnboardingStep(): Response<OnboardingStateResponse>

    // ── RGPD export (MANQUAIT) ──
    @GET("profiles/me/export")
    @Streaming  // Télécharge un ZIP, pas de buffering en mémoire
    suspend fun exportMyData(): Response<ResponseBody>
}


// ── PhotoApi (NOUVEAU — séparé de ProfileApi pour clarté) ──

interface PhotoApi {
    @Multipart
    @POST("photos")
    suspend fun uploadPhoto(
        @Part photo: MultipartBody.Part,
        @Part("position") position: RequestBody,
    ): Response<PhotoResponse>

    @DELETE("photos/{photoId}")
    suspend fun deletePhoto(@Path("photoId") photoId: String): Response<Unit>

    @PATCH("photos/reorder")
    suspend fun reorderPhotos(@Body body: ReorderPhotosBody): Response<Unit>
    // body: { "order": ["photo_id_1", "photo_id_3", "photo_id_2"] }
}


// ── EventApi (COMPLÉTÉ — il manquait 3 méthodes) ──

interface EventApi {
    @GET("events")
    suspend fun getEvents(
        @Query("city_id") cityId: String,
        @Query("from") from: String? = null,
        @Query("to") to: String? = null,
    ): Response<EventsResponse>

    @GET("events/{eventId}")
    suspend fun getEventDetail(
        @Path("eventId") eventId: String,
    ): Response<EventDetailResponse>

    @POST("events/{eventId}/register")
    suspend fun registerEvent(
        @Path("eventId") eventId: String,
    ): Response<EventRegistrationResponse>

    @DELETE("events/{eventId}/register")
    suspend fun unregisterEvent(
        @Path("eventId") eventId: String,
    ): Response<Unit>

    @GET("events/{eventId}/matches-preview")
    suspend fun getMatchesPreview(
        @Path("eventId") eventId: String,
    ): Response<MatchesPreviewResponse>
    // { "potential_matches": 3, "total_attendees": 12 }
}


// ── NotificationsApi (NOUVEAU — manquait entièrement) ──

interface NotificationsApi {
    @GET("notifications/preferences")
    suspend fun getPreferences(): Response<NotificationPreferencesResponse>

    @PUT("notifications/preferences")
    suspend fun updatePreferences(
        @Body body: UpdateNotificationPreferencesBody,
    ): Response<NotificationPreferencesResponse>
    // body: { "new_match": true, "new_message": true, "daily_feed": true,
    //         "match_expiring": true, "event_reminder": true,
    //         "email_digest": false, "email_marketing": false,
    //         "quiet_hours_start": "23:00", "quiet_hours_end": "07:00" }

    @POST("notifications/fcm-token")
    suspend fun registerFcmToken(
        @Body body: FcmTokenBody,
    ): Response<Unit>
    // body: { "token": "fcm_token_string", "device_id": "fingerprint_hash" }
}


// ── ContactsApi (NOUVEAU — manquait entièrement) ──

interface ContactsApi {
    @POST("contacts/blacklist")
    suspend fun importBlacklist(
        @Body body: BlacklistImportBody,
    ): Response<BlacklistImportResponse>
    // body: { "phone_hashes": ["sha256_hash_1", "sha256_hash_2", ...] }
    // IMPORTANT : le client hash les numéros en SHA-256 AVANT l'envoi
    // Le serveur ne reçoit JAMAIS les numéros en clair

    @GET("contacts/blacklist")
    suspend fun getBlacklist(): Response<BlacklistResponse>
    // Retourne la liste des phone_hashes masqués

    @DELETE("contacts/blacklist/{phoneHash}")
    suspend fun removeFromBlacklist(
        @Path("phoneHash") phoneHash: String,
    ): Response<Unit>
}


// ── MatchesApi (NOUVEAU — séparé de ChatApi pour clarté) ──
// ChatApi gère les messages. MatchesApi gère les matchs.

interface MatchesApi {
    @GET("matches")
    suspend fun getMatches(): Response<MatchesResponse>
    // Triés par dernière activité (dernier message)

    @GET("matches/{matchId}")
    suspend fun getMatchDetail(
        @Path("matchId") matchId: String,
    ): Response<MatchDetailResponse>
    // Inclut le profil complet + l'ice-breaker

    @DELETE("matches/{matchId}")
    suspend fun unmatch(
        @Path("matchId") matchId: String,
    ): Response<Unit>
    // Irréversible. Supprime le match et les messages.

    @GET("matches/likes-received")
    suspend fun getLikesReceived(): Response<LikesReceivedResponse>
    // Premium only. Retourne 403 si user free.
}
```

---

## Hilt DI — ajout des nouveaux providers

```kotlin
// Ajouts dans NetworkModule

@Provides @Singleton
fun providePhotoApi(retrofit: Retrofit): PhotoApi =
    retrofit.create(PhotoApi::class.java)

@Provides @Singleton
fun provideNotificationsApi(retrofit: Retrofit): NotificationsApi =
    retrofit.create(NotificationsApi::class.java)

@Provides @Singleton
fun provideContactsApi(retrofit: Retrofit): ContactsApi =
    retrofit.create(ContactsApi::class.java)

@Provides @Singleton
fun provideMatchesApi(retrofit: Retrofit): MatchesApi =
    retrofit.create(MatchesApi::class.java)
```

---

## ContactBlacklistViewModel (MANQUAIT)

```kotlin
/**
 * ViewModel pour Settings > Contacts masqués.
 * 
 * Flow complet :
 * 1. L'utilisateur ouvre Settings > Contacts masqués
 * 2. On affiche la liste des contacts déjà masqués
 * 3. Bouton "Importer depuis le carnet" → demande READ_CONTACTS
 * 4. Affiche la liste des contacts du téléphone avec checkboxes
 * 5. L'utilisateur sélectionne les contacts à masquer
 * 6. Le client hash les numéros en SHA-256 (ContactImporter, section 22)
 * 7. POST /contacts/blacklist avec les hashes
 * 8. Effet bidirectionnel ET invisible : les deux personnes ne se voient plus
 * 
 * Bouton "Ajouter manuellement" → champ téléphone → hash → POST
 * Bouton ✕ sur un contact masqué → DELETE /contacts/blacklist/{hash}
 */

@HiltViewModel
class ContactBlacklistViewModel @Inject constructor(
    private val contactsApi: ContactsApi,
    private val contactImporter: ContactImporter,
) : ViewModel() {

    data class BlacklistUiState(
        val maskedContacts: List<MaskedContactUi> = emptyList(),
        val phoneContacts: List<PhoneContactUi> = emptyList(),
        val isLoadingMasked: Boolean = true,
        val isLoadingPhone: Boolean = false,
        val isImporting: Boolean = false,
        val showPhoneList: Boolean = false,
        val showManualEntry: Boolean = false,
        val selectedCount: Int = 0,
        val error: String? = null,
    )

    data class MaskedContactUi(
        val phoneHash: String,
        val displayName: String?,  // Stocké LOCALEMENT, pas sur le serveur
        val maskedNumber: String?, // "+228 90 ** 56" — local aussi
    )

    data class PhoneContactUi(
        val name: String,
        val maskedNumber: String,
        val phoneHash: String,
        val isSelected: Boolean = false,
        val isAlreadyMasked: Boolean = false,
    )

    private val _state = MutableStateFlow(BlacklistUiState())
    val state = _state.asStateFlow()

    init { loadMaskedContacts() }

    private fun loadMaskedContacts() {
        viewModelScope.launch {
            try {
                val response = contactsApi.getBlacklist()
                if (response.isSuccessful) {
                    val hashes = response.body()!!.phoneHashes
                    // Enrichir avec les noms locaux si disponibles
                    _state.update { it.copy(
                        maskedContacts = hashes.map { h -> MaskedContactUi(
                            phoneHash = h,
                            displayName = null, // TODO: lookup local
                            maskedNumber = null,
                        )},
                        isLoadingMasked = false,
                    )}
                }
            } catch (e: Exception) {
                _state.update { it.copy(isLoadingMasked = false, error = e.toFlaamError().toUserMessage()) }
            }
        }
    }

    fun loadPhoneContacts() {
        viewModelScope.launch {
            _state.update { it.copy(isLoadingPhone = true, showPhoneList = true) }
            try {
                val contacts = contactImporter.loadContacts()
                val maskedHashes = _state.value.maskedContacts.map { it.phoneHash }.toSet()
                _state.update { it.copy(
                    phoneContacts = contacts.map { c -> PhoneContactUi(
                        name = c.name,
                        maskedNumber = c.phoneDisplay,
                        phoneHash = c.phoneHash,
                        isAlreadyMasked = c.phoneHash in maskedHashes,
                    )},
                    isLoadingPhone = false,
                )}
            } catch (e: Exception) {
                _state.update { it.copy(isLoadingPhone = false, error = "Erreur de lecture des contacts") }
            }
        }
    }

    fun toggleContact(phoneHash: String) {
        _state.update { state ->
            state.copy(phoneContacts = state.phoneContacts.map { c ->
                if (c.phoneHash == phoneHash && !c.isAlreadyMasked) c.copy(isSelected = !c.isSelected)
                else c
            }, selectedCount = state.phoneContacts.count { it.isSelected })
        }
    }

    fun importSelected() {
        val selected = _state.value.phoneContacts.filter { it.isSelected }
        if (selected.isEmpty()) return

        viewModelScope.launch {
            _state.update { it.copy(isImporting = true) }
            try {
                val response = contactsApi.importBlacklist(
                    BlacklistImportBody(phoneHashes = selected.map { it.phoneHash })
                )
                if (response.isSuccessful) {
                    _state.update { it.copy(isImporting = false, showPhoneList = false) }
                    loadMaskedContacts() // Refresh
                }
            } catch (e: Exception) {
                _state.update { it.copy(isImporting = false, error = e.toFlaamError().toUserMessage()) }
            }
        }
    }

    fun removeMasked(phoneHash: String) {
        viewModelScope.launch {
            try {
                contactsApi.removeFromBlacklist(phoneHash)
                _state.update { it.copy(
                    maskedContacts = it.maskedContacts.filter { c -> c.phoneHash != phoneHash }
                )}
            } catch (_: Exception) { }
        }
    }

    fun addManually(phoneNumber: String) {
        viewModelScope.launch {
            val normalized = contactImporter.normalizeAndHash(phoneNumber)
            try {
                contactsApi.importBlacklist(BlacklistImportBody(listOf(normalized)))
                _state.update { it.copy(showManualEntry = false) }
                loadMaskedContacts()
            } catch (_: Exception) { }
        }
    }
}
```

---

# MISE À JOUR 6 — Sanitisation des signaux + préférences implicites

---

## BehaviorTracker — protection contre les faux signaux

```kotlin
/**
 * MISE À JOUR CRITIQUE du BehaviorTracker.
 * 
 * Problème : si l'utilisateur ouvre un profil, pose son téléphone 
 * et va travailler, on enregistre un faux signal de 5 minutes.
 * 
 * 3 protections :
 * 
 * 1. onPause() coupe le chrono immédiatement
 *    L'app passe en arrière-plan → timer stoppé.
 * 
 * 2. Retour après > 30s = NOUVEAU signal
 *    Si l'utilisateur revient après 2 minutes, c'est un nouveau "view",
 *    pas la suite de l'ancien. Empêche de cumuler du temps mort.
 *    Si < 30s (répondu à une notif, vérifié l'heure) → on reprend.
 * 
 * 3. Flag "corroborated" envoyé au serveur
 *    true = l'utilisateur a interagi (scroll photo, tap prompt, scroll profil)
 *    false = le temps a juste passé sans interaction
 *    Le serveur JETTE les signaux temps-seul sans corroboration.
 * 
 * Le cas "téléphone posé" :
 * → ouvre profil → 12s → pose téléphone → onPause() → chrono stoppé à 12s
 * → 0 photo, 0 scroll → corroborated = false → serveur jette → 0 impact
 * 
 * Le cas "quitte et revient" (intérêt RÉEL) :
 * → ouvre profil → 8s → appel → onPause() → signal 1 jeté (pas corroboré)
 * → revient 10min plus tard → return_visit (poids 3.0x !)
 * → re-consulte, scrolle photos → signal 2 corroboré → fort impact positif
 */

@Singleton
class BehaviorTracker @Inject constructor(
    private val behaviorApi: BehaviorApi,
    @ApplicationScope private val scope: CoroutineScope,
) {
    private var currentProfileId: String? = null
    private var profileViewStart: Long = 0
    private var hasInteracted: Boolean = false
    private var lastPauseTime: Long = 0
    private var photoScrollCount: Int = 0
    private var maxScrollDepth: Float = 0f
    
    private val pendingEvents = mutableListOf<BehaviorEvent>()
    
    companion object {
        const val TIME_CAP_SECONDS = 60f
        const val MIN_TIME_THRESHOLD = 8f
        const val RESUME_NEW_SIGNAL_THRESHOLD_MS = 30_000L
        const val BATCH_UPLOAD_INTERVAL_MS = 60_000L
        const val BATCH_MAX_EVENTS = 50
    }

    // ── Événements de profil ──

    fun onProfileViewed(profileId: String) {
        finalizeCurrentProfile()
        currentProfileId = profileId
        profileViewStart = SystemClock.elapsedRealtime()
        hasInteracted = false
        photoScrollCount = 0
        maxScrollDepth = 0f
    }

    fun onPhotoScrolled(profileId: String, photoIndex: Int) {
        if (profileId == currentProfileId) {
            hasInteracted = true
            photoScrollCount++
        }
        addEvent("photo_scrolled", profileId, mapOf("index" to photoIndex))
    }

    fun onPromptExpanded(profileId: String, promptId: String) {
        if (profileId == currentProfileId) {
            hasInteracted = true
        }
        addEvent("prompt_expanded", profileId, mapOf("prompt_id" to promptId))
    }

    fun onScrollDepthChanged(profileId: String, depth: Float) {
        if (profileId == currentProfileId) {
            maxScrollDepth = maxOf(maxScrollDepth, depth)
            if (depth > 0.30f) {
                hasInteracted = true
            }
        }
    }

    fun onProfileAction(profileId: String, action: String, data: Map<String, Any> = emptyMap()) {
        finalizeCurrentProfile()
        addEvent("profile_action", profileId, data + mapOf("action" to action))
    }

    // ── Lifecycle (CRITIQUE) ──

    fun onAppPaused() {
        lastPauseTime = SystemClock.elapsedRealtime()
        finalizeCurrentProfile()
    }

    fun onAppResumed() {
        if (lastPauseTime > 0) {
            val pauseDuration = SystemClock.elapsedRealtime() - lastPauseTime
            if (pauseDuration > RESUME_NEW_SIGNAL_THRESHOLD_MS) {
                // > 30s d'absence → contexte cassé, ne pas reprendre
                currentProfileId = null
                hasInteracted = false
            }
            // < 30s → on reprend normalement (notif rapide, check heure)
        }
    }

    // ── Finalisation ──

    private fun finalizeCurrentProfile() {
        val id = currentProfileId ?: return
        val durationMs = SystemClock.elapsedRealtime() - profileViewStart
        val durationSec = durationMs / 1000f
        val cappedSec = minOf(durationSec, TIME_CAP_SECONDS)

        addEvent("profile_view_duration", id, mapOf(
            "duration_ms" to durationMs,
            "duration_capped_s" to cappedSec,
            "corroborated" to hasInteracted,
            "photo_scroll_count" to photoScrollCount,
            "max_scroll_depth" to maxScrollDepth,
        ))

        currentProfileId = null
        hasInteracted = false
        photoScrollCount = 0
        maxScrollDepth = 0f
    }

    // ── Batch upload ──

    private fun addEvent(type: String, profileId: String, data: Map<String, Any>) {
        pendingEvents.add(BehaviorEvent(
            type = type,
            profileId = profileId,
            data = data,
            timestamp = System.currentTimeMillis(),
        ))
        if (pendingEvents.size >= BATCH_MAX_EVENTS) {
            flushEvents()
        }
    }

    fun flushEvents() {
        if (pendingEvents.isEmpty()) return
        val batch = pendingEvents.toList()
        pendingEvents.clear()
        scope.launch {
            try {
                behaviorApi.logBehavior(BehaviorBatchBody(events = batch))
            } catch (_: Exception) {
                // Re-queue en cas d'échec réseau (offline-first)
                pendingEvents.addAll(0, batch)
            }
        }
    }
}

data class BehaviorEvent(
    val type: String,
    val profileId: String,
    val data: Map<String, Any>,
    val timestamp: Long,
)
```

---

## Intégration dans FeedViewModel

```kotlin
// Dans FeedViewModel.kt — hooks lifecycle

@HiltViewModel
class FeedViewModel @Inject constructor(
    private val feedApi: FeedApi,
    private val behaviorTracker: BehaviorTracker,
) : ViewModel() {

    // Appelé par l'Activity dans onPause()
    fun onPause() {
        behaviorTracker.onAppPaused()
        behaviorTracker.flushEvents()  // Upload immédiat avant background
    }

    // Appelé par l'Activity dans onResume()  
    fun onResume() {
        behaviorTracker.onAppResumed()
    }

    // Quand l'utilisateur voit un nouveau profil dans le feed
    fun onProfileVisible(profileId: String) {
        behaviorTracker.onProfileViewed(profileId)
    }

    // Quand l'utilisateur scrolle les photos
    fun onPhotoSwiped(profileId: String, photoIndex: Int) {
        behaviorTracker.onPhotoScrolled(profileId, photoIndex)
    }

    // Etc. pour chaque interaction
}
```

---

## Return visit — détection côté mobile

```kotlin
/**
 * Le return_visit est détecté quand :
 * 1. L'utilisateur a vu le profil X une première fois
 * 2. Il a quitté (vu d'autres profils ou fermé l'app)
 * 3. Il revient sur le profil X (même session ou session suivante)
 * 
 * C'est le signal le plus fort (poids 3.0x) parce qu'il est
 * impossible à simuler : personne ne se dit "je vais revenir sur
 * ce profil pour booster mon score implicite".
 */

// Dans BehaviorTracker, maintenir un Set des profils déjà vus
private val viewedProfileIds = mutableSetOf<String>()

fun onProfileViewed(profileId: String) {
    finalizeCurrentProfile()
    
    if (profileId in viewedProfileIds) {
        // C'est un RETOUR → signal return_visit
        addEvent("return_visit", profileId, emptyMap())
    }
    viewedProfileIds.add(profileId)
    
    currentProfileId = profileId
    profileViewStart = SystemClock.elapsedRealtime()
    hasInteracted = false
    photoScrollCount = 0
    maxScrollDepth = 0f
}

// Reset le Set au changement de jour (nouveau feed = nouveau contexte)
fun onNewFeedLoaded() {
    viewedProfileIds.clear()
}
```

---

# MISE À JOUR 7 — Invitations + waitlist genrée + ambassadrices

---

## Retrofit — InviteApi (NOUVEAU)

```kotlin
interface InviteApi {
    @POST("invites/generate")
    suspend fun generateCodes(): Response<InviteCodesResponse>
    // Génère les codes d'invitation de l'utilisateur

    @GET("invites/me")
    suspend fun getMyCodes(): Response<InviteCodesResponse>
    // Liste mes codes : actifs, utilisés, expirés

    @POST("invites/validate")
    suspend fun validateCode(
        @Body body: ValidateCodeBody,
    ): Response<ValidateCodeResponse>
    // body: { "code": "FLAAM-A3K9M2P1" }
    // Réponse: { "valid": true, "city": "Lomé", "creator_name": "Ama" }

    @POST("invites/redeem")
    suspend fun redeemCode(
        @Body body: RedeemCodeBody,
    ): Response<RedeemCodeResponse>
    // Utiliser un code → skip waitlist + créditer l'inviter
}

// Hilt
@Provides @Singleton
fun provideInviteApi(retrofit: Retrofit): InviteApi =
    retrofit.create(InviteApi::class.java)
```

---

## Écrans waitlist (hommes)

```kotlin
/**
 * Écran affiché aux hommes qui sont en waitlist.
 * 
 * Contenu :
 * - Position dans la file : "#89 sur 327"
 * - Barre de progression animée
 * - Message : "On ouvre Flaam par petits groupes pour garantir
 *   une bonne expérience. Tu seras notifié quand c'est ton tour."
 * - CTA "Passe devant" → partage de lien d'invitation
 *   "Invite 3 amis et passe dans le top 10%"
 * - Champ "Tu as un code ?" → saisie code d'invitation → skip waitlist
 * 
 * L'écran se refresh toutes les 30 minutes (poll léger).
 * Si le status passe à "invited" → animation + redirect vers le feed.
 */

// Navigation : si user.waitlist_status == "waiting" → WaitlistScreen
// Le check est fait dans le SplashRouter (section 34 mobile spec)
```

---

## Écran invitations (femmes + ambassadrices)

```kotlin
/**
 * Accessible via Settings > Inviter des amis
 * 
 * Contenu :
 * - Compteur : "2/3 invitations utilisées"
 * - Liste des codes avec status :
 *   FLAAM-A3K9 → Utilisé par Kofi (badge vert ✓)
 *   FLAAM-M2P1 → Utilisé par Ama (badge vert ✓)
 *   FLAAM-X7R2 → Disponible (bouton "Partager")
 * - Bouton "Partager" → Android ShareSheet avec message :
 *   "Rejoins Flaam avec mon code : FLAAM-X7R2
 *    Télécharge l'app : https://flaam.app/invite/X7R2"
 * - Rewards section :
 *   "1 invitation utilisée → +2 likes offerts ✓"
 *   "3 invitations utilisées → 1 semaine premium 🔒"
 * 
 * Pour les ambassadrices : même écran mais 50 codes
 * et un badge "Ambassadrice" en haut.
 */
```

---

## Onboarding — champ code d'invitation

```kotlin
/**
 * Ajout au flow d'onboarding (après sélection de la ville) :
 * 
 * Écran "Tu as un code d'invitation ?"
 * - Champ texte : "FLAAM-________"
 * - Bouton "Valider" → POST /invites/validate
 *   Si valide : "Code de Ama ✓ — Tu passes devant la file d'attente !"
 *   Si invalide : "Code non reconnu" (rouge)
 * - Lien "Je n'ai pas de code" → continue normalement
 *   (les femmes passent quand même, les hommes vont en waitlist)
 * 
 * Le code est stocké localement et envoyé avec POST /invites/redeem
 * après la fin de l'onboarding.
 */
```

---

# MISE À JOUR 8 — Ghost user detection + onboarding contextuel event

---

## Détection du ghost user au premier OTP

```kotlin
/**
 * Quand l'utilisateur ouvre l'app et fait son OTP, le serveur peut
 * retourner un champ "prefilled" dans la réponse si le numéro
 * correspond à un ghost user (pré-inscrit via page web event).
 * 
 * L'app utilise ces données pour :
 * 1. Skip l'écran prénom (déjà saisi sur le web)
 * 2. Pré-cocher le spot de l'event dans l'écran spots
 * 3. Afficher le message contextuel dans l'onboarding
 * 4. Montrer le badge event dans le feed
 * 
 * Pour les portes 1 (classique) et 2 (invitation), "prefilled" est null.
 * L'onboarding fonctionne exactement comme avant.
 */

data class OtpVerifyResponse(
    val accessToken: String,
    val refreshToken: String,
    val user: UserInfo,
)

data class UserInfo(
    val id: String,
    val onboardingStatus: String,
    val sourceChannel: String?,  // "organic" | "invite" | "event"
    val prefilled: PrefilledData?,
)

data class PrefilledData(
    val firstName: String?,
    val sourceEvent: SourceEvent?,
)

data class SourceEvent(
    val id: String,
    val title: String,
    val spotId: String?,
    val spotName: String?,
    val date: String,
    val attendeesCount: Int,
)
```

---

## OnboardingViewModel — adaptation pour les 3 portes

```kotlin
/**
 * Le OnboardingViewModel gère l'état de l'onboarding.
 * Avec la MàJ 8, il détecte le source_channel et adapte :
 * 
 * Porte 1 (classique) : flow normal, rien de pré-rempli
 * Porte 2 (invitation) : flow normal, code validé en amont
 * Porte 3 (event) : prénom pré-rempli, spot pré-coché, message contextuel
 */

// Dans OnboardingViewModel.init :
fun initFromOtpResponse(response: OtpVerifyResponse) {
    val prefilled = response.user.prefilled

    if (prefilled != null) {
        // Porte 3 : event user
        _state.update { it.copy(
            firstName = prefilled.firstName ?: "",
            skipFirstNameScreen = prefilled.firstName != null,
            preSelectedSpotId = prefilled.sourceEvent?.spotId,
            preSelectedSpotName = prefilled.sourceEvent?.spotName,
            contextualMessage = prefilled.sourceEvent?.let { event ->
                "Tu étais au ${event.title}. " +
                "Complete ton profil pour voir les ${event.attendeesCount} " +
                "personnes qui y étaient !"
            },
            sourceChannel = response.user.sourceChannel,
        )}
    }
    // Sinon : portes 1 et 2, pas de pré-remplissage
}
```

---

## Badge "Était au [event]" dans le feed

```kotlin
/**
 * Dans le feed, les profils des personnes qui étaient au même event
 * affichent un petit badge sous le prénom :
 * 
 *   ┌─────────────────┐
 *   │  [Photo]        │
 *   │  Ama, 24        │
 *   │  🔥 Café 21 ven.│  ← badge event
 *   │  82% zones      │
 *   └─────────────────┘
 * 
 * Le badge est retourné par l'API dans le feed response :
 * "event_badge": { "event_name": "Café 21", "short": "Café 21 ven." }
 * 
 * Visible pendant 7 jours après l'event. Puis disparaît.
 */
```

---

# MISE À JOUR 8 — Porte 3 : conversion ghost user + onboarding event

---

## Détection du ghost user au login

```kotlin
/**
 * Quand POST /auth/otp/verify retourne is_ghost_conversion = true,
 * l'app entre en mode "event onboarding" :
 * 
 * 1. Skip l'écran prénom (ghost_data.first_name pré-rempli)
 * 2. Skip l'écran code d'invitation (arrivé par event, pas par invite)
 * 3. L'écran ville est pré-sélectionné (ville de l'event)
 * 4. L'écran spots pré-coche le spot de l'event
 * 5. L'écran tags pré-coche les suggested_tags de l'event
 * 6. Un bandeau motivant s'affiche en haut de chaque écran :
 *    "7 personnes du Café 21 ont déjà complété leur profil"
 * 7. À la fin : POST /invites/redeem n'est PAS appelé (pas d'invite)
 *    mais POST /events/{event_id}/convert EST appelé (marque la conversion)
 */

data class GhostData(
    val firstName: String,
    val onboardingSource: String,  // "event"
    val eventName: String,
    val eventSpotId: String,
    val suggestedTags: List<String>,
    val attendeesCompleted: Int,
)

// Dans AuthViewModel :
fun handleOtpResponse(response: OtpVerifyResponse) {
    if (response.isGhostConversion && response.ghostData != null) {
        // Stocker les données ghost pour l'onboarding
        _ghostData.value = response.ghostData
        // Naviguer vers l'onboarding en mode event
        _navigateTo.value = Route.Onboarding.EventMode
    } else {
        // Onboarding classique
        _navigateTo.value = Route.Onboarding.Standard
    }
}
```

---

## Écrans onboarding event (différences avec le classique)

```kotlin
/**
 * L'onboarding event réutilise les mêmes écrans que le classique,
 * avec ces modifications :
 * 
 * ÉCRAN PRÉNOM → SKIP (pré-rempli depuis ghost_data)
 * 
 * ÉCRAN CODE INVITATION → SKIP (porte 3, pas porte 2)
 * 
 * ÉCRAN VILLE → Pré-sélectionnée, mais modifiable
 * 
 * ÉCRAN SPOTS → Le spot de l'event est pré-coché avec un badge :
 *   ☑ Café 21 ← "Tu y étais vendredi !"
 *   L'utilisateur peut ajouter d'autres spots normalement.
 * 
 * ÉCRAN TAGS → Les suggested_tags sont pré-cochés :
 *   ☑ afterwork  ☑ sortir  ☑ networking
 *   L'utilisateur peut décocher et ajouter d'autres tags.
 *   Message : "On a pré-sélectionné des tags basés sur l'event."
 * 
 * ÉCRAN MOTIVATION (NOUVEAU, après les photos) :
 *   "7 personnes du Café 21 ont déjà complété leur profil.
 *    Ton profil sera visible pour eux dès demain."
 *   Pas de bouton skip — juste un "Continuer" pour finir.
 * 
 * TOUS LES ÉCRANS → Bandeau en haut :
 *   Petit chip teal : "Afterwork Café 21 — 18 avr"
 *   Rappelle le contexte pendant tout l'onboarding.
 */
```

---

## Badge event dans le feed

```kotlin
/**
 * Dans le FeedCard (composant profil dans le feed), si le candidat
 * partage un event récent (< 14 jours) :
 * 
 * Afficher un badge sous le prénom :
 * "Était au Café 21 il y a 3 jours"
 * 
 * Style : chip avec icône calendar, fond teal-50, texte teal-800.
 * Visible uniquement si les deux utilisateurs étaient checked_in.
 * 
 * Le badge est un lien cliquable qui ouvre le détail de l'event
 * (même s'il est passé — mode "recap").
 */

@Composable
fun EventBadge(eventName: String, daysAgo: Int) {
    Surface(
        shape = RoundedCornerShape(6.dp),
        color = FlaamColors.Teal50,
    ) {
        Row(
            modifier = Modifier.padding(horizontal = 8.dp, vertical = 4.dp),
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.spacedBy(4.dp),
        ) {
            // Icon calendar (Lucide)
            Icon(/* calendar icon */, tint = FlaamColors.Teal800, size = 12.dp)
            Text(
                text = if (daysAgo == 0) "Était au $eventName aujourd'hui"
                       else "Était au $eventName il y a ${daysAgo}j",
                style = FlaamTypography.caption,
                color = FlaamColors.Teal800,
            )
        }
    }
}
```

---

## Ice-breaker contextuel

```kotlin
/**
 * Quand deux participants du même event matchent :
 * Le champ ice_breaker du Match est rempli avec un ice-breaker
 * contextuel au lieu du ice-breaker générique.
 * 
 * Côté mobile, aucun changement d'UI — le ice-breaker est affiché
 * de la même manière. C'est le serveur qui choisit le bon texte.
 * 
 * Exemple :
 * "Vous étiez tous les deux au Café 21. Qu'est-ce que tu en as pensé ?"
 */
```
