# Android Development — Deep Dive Roadmap

We'll go from fundamentals → UI toolkit → app architecture → data & networking → concurrency → platform integration → performance → production.

*Covers native Android development with Kotlin and Jetpack Compose. Cross-references the React Native notes throughout — since React Native ultimately renders to the same native Android views this file covers directly, and the Java Backend notes, since Kotlin/JVM tooling and several architectural patterns (DI, Repository, MVVM) carry over directly from that file.*

---

## 1. Android Fundamentals

**Definition:** Android is a Linux-kernel-based mobile operating system running applications on the **ART (Android Runtime)**, which compiles app code (Kotlin/Java, compiled to JVM bytecode, then to Dex bytecode) ahead-of-time into native machine code — the same JVM-based compilation lineage as the Java Backend notes' Java applications, but targeting ART's mobile-optimized execution model rather than a server-side JVM.

**Kotlin as the primary language — Definition:** Kotlin is Google's officially recommended language for Android development — fully interoperable with Java (Java Backend notes) since both compile to JVM-compatible bytecode, but adds null-safety enforced at the type-system level (`String` vs `String?`, eliminating most `NullPointerException`s at compile time), more concise syntax (data classes, extension functions), and first-class coroutines support (section 10) — the same relationship TypeScript has to JavaScript (JS/TS notes): a stricter, safer superset-like layer over an existing, more permissive ecosystem.

**Project structure: Gradle, modules, manifest — Definition:** an Android project is built with **Gradle** (the same JVM-ecosystem build tool covered in the Java Backend notes, using Kotlin DSL build scripts, `build.gradle.kts`), organized into **modules** (the `app` module by default, with additional feature/library modules as an app grows, section 16), with `AndroidManifest.xml` (section 2) declaring the app's components, required permissions, and metadata.

**APK/AAB, build variants — Definition:** a Gradle build produces either an **APK** (a directly-installable package) or an **AAB** (Android App Bundle, the required Play Store submission format, letting Google Play generate device-optimized APKs at install time) — the same distinction already covered from the React Native notes' build-process section, here as the actual underlying mechanism that section was abstracting over; **build variants** (combining a build type — debug/release — with a product flavor, section 17) let a single codebase produce multiple distinct build outputs (e.g. a free vs paid app variant) from shared source.

---

## 2. App Components & the Manifest

**The four component types — Definition:** every Android app is composed of some combination of four fundamental component types, each with a distinct lifecycle managed by the OS rather than by application code directly:
- **Activities** — a single, focused screen with a UI (the closest analogue to a React Native "screen," section 4 of the React Native notes' navigation section).
- **Services** — components that run in the background without a UI, for long-running operations (section 11).
- **Broadcast Receivers** — components that respond to system-wide or app-wide broadcast events (e.g. "battery low," "connectivity changed") without needing to be actively running when the broadcast is sent.
- **Content Providers** — a standardized interface for sharing structured data between apps (e.g. the system contacts database), the Android platform's built-in mechanism for controlled cross-app data access.

**`AndroidManifest.xml` — Definition:** the mandatory declaration file listing every component the app defines, the permissions it requires (section 12), and app-level metadata (package name, minimum/target SDK version) — the OS reads this manifest to know what an app is capable of and is allowed to do *before* running any of its code, the same "declare your capabilities up front" model already discussed for iOS's `Info.plist` in the React Native notes' section 7.

```xml
<manifest>
  <uses-permission android:name="android.permission.INTERNET" />
  <application android:name=".MyApp">
    <activity android:name=".MainActivity" android:exported="true">
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
    </activity>
  </application>
</manifest>
```

**The Activity/App lifecycle in depth — Definition:** an Activity moves through a well-defined sequence of lifecycle callbacks the OS invokes as the user navigates to/away from/back to it — `onCreate()` → `onStart()` → `onResume()` (now visible and interactive) → `onPause()` (losing focus, e.g. a dialog appearing on top) → `onStop()` (no longer visible) → `onDestroy()` — correctly handling this lifecycle (saving/restoring state across a configuration change like screen rotation, releasing resources in `onPause`/`onStop`) is one of the most fundamental and historically error-prone aspects of Android development, which modern architecture (ViewModel, section 6) is specifically designed to insulate app logic from.

**Intents: explicit vs implicit, intent filters — Definition:** an `Intent` is Android's message-passing object used to request an action from another component — an **explicit** Intent names the exact target component (e.g. launching a specific Activity within the same app); an **implicit** Intent describes an action to perform (`ACTION_VIEW` a URL) without naming a specific component, letting the OS find any installed app whose manifest declares a matching **intent filter** — the underlying mechanism behind Android's inter-app integration (e.g. "share to" sheets, opening a link in whichever browser is installed) and the deep-linking mechanism the React Native notes' navigation section builds on at a higher level.

---

## 3. UI with Jetpack Compose

**Declarative UI fundamentals, `@Composable` functions — Definition:** Jetpack Compose is Android's modern, declarative UI toolkit — a `@Composable` function describes **what** the UI should look like given the current state, and Compose's runtime handles actually updating the displayed UI when that state changes — the same declarative philosophy already covered for React (React notes' section 1) and SwiftUI-equivalent frameworks, replacing the older imperative View/XML system (below) where a developer manually mutated View objects (`textView.setText(...)`) in response to state changes.

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!")
}
```

**State management: `remember`, `mutableStateOf`, state hoisting — Definition:** `remember { mutableStateOf(initial) }` creates a piece of Compose-observed state that survives recomposition (below) but not a configuration change on its own (needing `rememberSaveable` for that) — directly analogous to React's `useState` (React notes' section 2); **state hoisting** — lifting state up to a common ancestor and passing it down along with a callback to modify it, rather than a component owning its own state internally — is Compose's idiomatic pattern for building reusable, testable composables, the exact same "lift state up" principle already covered for React's controlled-component pattern.

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("Count: $count") }
}
```

**Recomposition — Definition:** when observed state changes, Compose automatically re-invokes (**recomposes**) only the `@Composable` functions that actually read that specific piece of state — not the entire UI tree — the same targeted-update efficiency principle behind React's reconciliation (React notes' section 6), here implemented via Compose's own state-observation and "smart recomposition" system rather than a virtual DOM diff.

**Compose vs the legacy View/XML system (brief) — Definition:** Android UI was traditionally built by defining layouts in XML and manipulating `View` objects imperatively from Kotlin/Java code (`findViewById`, `setOnClickListener`) — Compose is now Google's recommended default for all new UI, offering less boilerplate and a unified language (pure Kotlin, no separate XML layer) — the same "declarative UI toolkit supersedes an older imperative one" transition React Native itself represents relative to fully manual native UIKit/View code.

---

## 4. Layouts & Material Design

**Compose layout primitives — Definition:** `Column` arranges children vertically, `Row` arranges children horizontally (Compose's direct equivalents of a Flexbox `column`/`row` direction, React Native notes' section 3), `Box` stacks children on top of one another (for overlays, exactly like `position: absolute` layering), and `LazyColumn`/`LazyRow` render large or unbounded lists **efficiently**, only composing items near the visible viewport — Compose's direct equivalent of React Native's `FlatList` virtualization (React Native notes' section 2/14).

```kotlin
@Composable
fun UserList(users: List<User>) {
    LazyColumn {
        items(users) { user -> Text(user.name) }
    }
}
```

**Modifiers system — Definition:** every composable accepts a `Modifier` parameter — a chainable sequence of styling/behavior/layout instructions (`Modifier.padding(16.dp).fillMaxWidth().clickable { ... }`) applied in the exact order they're chained — Compose's equivalent of combining CSS properties and event handlers, but expressed as a single composable, ordered chain rather than a separate style object.

**Material Design 3 components and theming — Definition:** Compose ships pre-built, Material-Design-3-compliant components (`Button`, `TextField`, `Scaffold`) and a `MaterialTheme` wrapper providing consistent color schemes, typography, and shape theming across an app — Android's equivalent of adopting a design system (HTML/CSS notes' section 10's design-system discussion, or a component library like Material UI in the React ecosystem), here as the platform's own first-party, deeply integrated default.

---

## 5. Navigation

**Jetpack Navigation Compose — Definition:** the recommended navigation library for Compose apps — manages a back stack of composable "destinations" the same way React Navigation manages a stack of screens (React Native notes' section 4), letting a single-Activity app (the modern recommended architecture — one Activity hosting all screens as composables, rather than one Activity per screen as in the older View system) handle all in-app navigation.

```kotlin
@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(navController, startDestination = "home") {
        composable("home") { HomeScreen(onNavigate = { navController.navigate("profile/42") }) }
        composable("profile/{userId}") { backStackEntry ->
            val userId = backStackEntry.arguments?.getString("userId")
            ProfileScreen(userId)
        }
    }
}
```

**NavHost, NavController, passing arguments — Definition:** `NavHost` declares the navigation graph (the set of possible destinations and how to reach them); `NavController` is the object used to actually trigger navigation (`navController.navigate(...)`) and read the current back stack state; route arguments are embedded directly in the route string pattern (`"profile/{userId}"`), extracted from the resulting `NavBackStackEntry` — conceptually identical to React Navigation's typed-params model (React Native notes' section 4), just expressed through Kotlin route strings rather than a TypeScript param-list type.

**Deep linking, nested navigation graphs — Definition:** Navigation Compose supports the same OS-level deep-linking (an external URL/intent opening the app directly to a specific destination) already covered generally in section 2's Intents and the React Native notes' section 4; nested navigation graphs group related destinations (e.g. an entire onboarding flow) into their own sub-graph, composable as a single navigable unit within the larger app graph — Android's equivalent of React Navigation's nested-navigator pattern.

---

## 6. App Architecture (MVVM & Architecture Components)

**Recommended app architecture: UI/domain/data layers — Definition:** Google's officially recommended architecture separates an app into a **UI layer** (composables + ViewModel, displaying state and forwarding user actions), an optional **domain layer** (use-case classes encapsulating business logic reused across multiple ViewModels), and a **data layer** (repositories, section 8's persistence and section 9's networking, the single source of truth for app data) — the same layered-architecture principle already covered generally in the System Design notes and Java Backend notes' Spring layering (Controller/Service/Repository), adapted to Android's specific component model.

**ViewModel — lifecycle-aware state holder — Definition:** a `ViewModel` (from Android's Architecture Components) holds UI-related state and **survives configuration changes** (like screen rotation) that would otherwise destroy and recreate the hosting Activity/Composable — solving the exact "state lost on rotation" problem section 2's Activity lifecycle discussion flags, by scoping state to a lifecycle-aware holder the OS specifically preserves across such changes rather than to the Activity/Composable itself.

```kotlin
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Success(repository.getUser(id))
        }
    }
}
```

**LiveData vs StateFlow — Definition:** `LiveData` was the original lifecycle-aware observable data holder; `StateFlow` (built on Kotlin coroutines' `Flow`, section 10) is the modern, now-preferred alternative — offering a richer set of operators (the same reactive-stream operators `Flow` provides generally) and better integration with coroutine-based code, at the cost of needing a small amount of manual lifecycle-awareness (`repeatOnLifecycle`) that `LiveData` handled automatically — most new code defaults to `StateFlow`, with `LiveData` remaining common in older/legacy codebases.

**Unidirectional data flow (UDF) — Definition:** state flows **down** from the ViewModel to the composable UI (as an observed `StateFlow`), while events flow **up** from the UI to the ViewModel (as function calls, e.g. `viewModel.loadUser(id)`) — never the reverse — the exact same unidirectional data flow principle already covered for Redux/Flux-style state management in the React notes' section 5, here as the recommended default architecture for essentially every Android screen, not an optional pattern reserved for complex state.

---

## 7. Dependency Injection with Hilt

**Why DI matters in Android — Definition:** the same Dependency Injection pattern already covered architecturally in the Design Patterns notes and concretely in the Java Backend notes' Spring `@Autowired`/NestJS notes' provider system — Android's added wrinkle is that many of its core components (Activities, Fragments) are instantiated directly by the OS, not by application code, making constructor injection (the simplest DI form) impossible for them without a dedicated framework to bridge that gap.

**Hilt setup — Definition:** Hilt (built on Google's Dagger, a compile-time DI code generator) is the officially recommended DI library for Android — `@HiltAndroidApp` on the `Application` class bootstraps Hilt's DI container application-wide; `@AndroidEntryPoint` on an Activity/Fragment/Service opts that OS-instantiated component into having its own dependencies injected (bridging the "OS instantiates this, not me" gap above); `@Inject` on a class's constructor marks it as available for Hilt to provide, exactly like Spring's `@Autowired`/NestJS's constructor injection.

```kotlin
@HiltAndroidApp
class MyApp : Application()

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    private val viewModel: UserViewModel by viewModels() // Hilt injects its dependencies automatically
}

class UserRepository @Inject constructor(private val api: UserApi, private val dao: UserDao)
```

**Modules, scopes, qualifiers — Definition:** a Hilt `@Module` with `@Provides`/`@Binds` methods tells Hilt how to construct types it can't build automatically (an interface needing a specific implementation, exactly like NestJS's `useClass`/`useFactory` custom providers); **scopes** (`@Singleton`, `@ActivityScoped`) control an instance's lifetime, mirroring NestJS/Spring's provider-scope concept (NestJS notes' section 4); **qualifiers** disambiguate when multiple bindings exist for the same type (e.g. two different `OkHttpClient` configurations).

---

## 8. Data Persistence

**Room (SQLite ORM) — Definition:** Room is Android's official ORM layer over SQLite (the same embedded relational database concept covered in the SQL notes) — `@Entity` classes map to tables, `@Dao` interfaces declare type-safe query methods (validated at compile time against the actual SQL), and a `@Database` class ties them together — Android's equivalent of the Java Backend notes' TypeORM/Prisma-style ORM pattern, specifically for on-device relational storage.

```kotlin
@Entity data class User(@PrimaryKey val id: String, val name: String)

@Dao
interface UserDao {
    @Query("SELECT * FROM User WHERE id = :id")
    suspend fun getUser(id: String): User?
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)
}
```

**DataStore (replacing SharedPreferences) — Definition:** `DataStore` is the modern, coroutine/Flow-based key-value (Preferences DataStore) or typed-object (Proto DataStore) persistence API, replacing the older `SharedPreferences` — offering asynchronous, non-blocking reads by default (avoiding `SharedPreferences`' historical main-thread-blocking-read pitfall) and safer, transactional writes — Android's rough equivalent of the React Native notes' AsyncStorage/MMKV distinction (section 5), here as the platform's own recommended, built-in solution.

**File storage, scoped storage — Definition:** modern Android (API 29+) enforces **scoped storage** — apps can freely read/write their own app-specific directory without special permission, but accessing files outside it (shared media, another app's files) requires the system-mediated Storage Access Framework or a specifically-declared, narrower permission — a deliberate privacy-focused restriction (section 12) preventing apps from freely browsing a device's entire shared storage the way older Android versions permitted.

---

## 9. Networking

**Retrofit for REST APIs — Definition:** Retrofit is the de facto standard HTTP client library for Android — declares an API as a Kotlin interface with annotated methods (`@GET`, `@POST`), and generates the actual networking implementation at compile time — a more structured, type-safe alternative to raw `fetch`-style calls (React Native notes' section 6), closer in spirit to a generated API client than a manually-written HTTP call.

```kotlin
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User
}

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
val api = retrofit.create(UserApi::class.java)
```

**OkHttp interceptors — Definition:** Retrofit is built on **OkHttp**, whose interceptor mechanism lets code inspect/modify every outgoing request and incoming response — used for attaching an auth header to every request (the same cross-cutting-concern pattern as an Express/NestJS middleware or interceptor, Node.js/NestJS notes), logging, or automatic retry logic — a single, centralized place to implement concerns that would otherwise need to be duplicated across every individual API call.

**Coroutines integration for async network calls — Definition:** Retrofit interfaces declared with `suspend` functions (as above) integrate directly with Kotlin coroutines (section 10) — a network call reads as straightforward sequential code (`val user = api.getUser(id)`) without callback nesting, the same async/await-style readability benefit already covered for JavaScript's `async`/`await` in the JS/TS notes.

**Handling connectivity changes (recap)** — Android's `ConnectivityManager` provides the same network-state-monitoring capability as the React Native notes' `NetInfo` (section 6), used to detect connectivity loss and adapt app behavior (queueing writes, showing offline UI) accordingly.

---

## 10. Kotlin Coroutines & Flow

**Coroutines fundamentals: suspend functions, structured concurrency — Definition:** a `suspend` function can pause its execution at an await-point (e.g. waiting on network I/O) **without blocking the underlying thread**, letting that thread do other work in the meantime and resuming the suspended function later — Kotlin's answer to async programming, conceptually parallel to JavaScript's `async`/`await` (JS/TS notes' section 7) but implemented via actual language-level continuation-passing rather than an event loop, and additionally enforcing **structured concurrency** — every coroutine must be launched within a `CoroutineScope`, which ties its lifetime to a well-defined parent, so a scope being cancelled automatically cancels every coroutine launched within it, preventing orphaned, leaked background work.

**CoroutineScope, Dispatchers, Jobs — Definition:** a `CoroutineScope` (e.g. `viewModelScope`, automatically cancelled when a ViewModel is cleared, section 6) defines a coroutine's lifetime boundary; a `Dispatcher` (`Dispatchers.Main`, `.IO`, `.Default`) determines which thread pool a coroutine actually runs on (`.IO` for network/disk work, `.Default` for CPU-intensive work, `.Main` for UI updates) — Kotlin's structured, type-safe alternative to manually managing raw threads; a `Job` represents a handle to a running coroutine, cancellable directly (`job.cancel()`), automatically propagating cancellation to any child coroutines it launched.

**Flow, StateFlow, SharedFlow — Definition:** `Flow` is Kotlin's cold, asynchronous stream type — conceptually similar to an RxJS Observable (a stream of values over time, backpressure-aware) but built natively into the coroutines system; `StateFlow` (section 6) is a **hot**, state-holding specialization always having a current value, Android's Kotlin-native reactive state-holder; `SharedFlow` is a more general hot stream for broadcasting one-off events (not persistent state) to multiple collectors — together forming Kotlin's reactive-streams toolkit, filling the same role RxJS fills in a JavaScript codebase.

```kotlin
fun observeUsers(): Flow<List<User>> = userDao.getAllUsersFlow() // Room can return a Flow directly
    .map { users -> users.filter { it.isActive } }
```

**Comparison with RxJava (brief) — Definition:** RxJava was the previous-generation standard reactive-streams library for Android/JVM — Coroutines + Flow is now Google's recommended default, offering tighter language integration (structured concurrency, `suspend` functions) and generally simpler, more readable code for the same problems RxJava solved with a steeper operator-chaining learning curve — RxJava knowledge remains relevant mainly for maintaining existing, older Android codebases, much like the Pages-Router-vs-App-Router relationship already covered in the Next.js notes.

---

## 11. Background Work

**WorkManager for deferrable, guaranteed background tasks — Definition:** `WorkManager` schedules background work guaranteed to eventually execute — even across app restarts or device reboots — while respecting battery-optimization system constraints (deferring work when the device is in Doze mode, requiring network connectivity, etc.) — the right tool for deferrable tasks like periodic data sync or uploading a file once a network becomes available, Android's equivalent conceptually to the job-queue pattern already covered in the NestJS notes' section 17, but specifically OS-integrated and battery-aware.

```kotlin
val syncWork = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
    .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
    .build()
WorkManager.getInstance(context).enqueue(syncWork)
```

**Foreground Services for user-visible ongoing tasks — Definition:** a foreground Service runs with an associated, persistent user-visible notification (e.g. music playback, an active navigation session, an ongoing file download) — required by the OS specifically *because* it consumes resources the user should be aware of and able to stop, unlike WorkManager's deferrable, invisible-to-the-user background work.

**Choosing the right background execution API — Definition:** WorkManager for deferrable, guaranteed-eventually work not requiring immediate execution or user visibility; a foreground Service for immediate, ongoing, user-visible work; a plain coroutine scoped to a ViewModel (section 10) for work that should simply stop when its screen is no longer active — picking the wrong one is a common source of Android's most notorious background-execution bugs (work silently killed by the OS, or unnecessary battery drain from an inappropriately persistent Service).

---

## 12. Permissions & Security

**Runtime permissions model — Definition:** since Android 6.0, dangerous permissions (camera, location, contacts) must be requested **at runtime**, not just declared in the manifest (section 2) — the user sees an explicit system prompt and can grant/deny each permission individually, and denial must be handled gracefully rather than assumed away — the same runtime-permission model already covered from the app-consuming side in the React Native notes' section 7/10, here as the underlying native mechanism.

**Scoped storage, privacy-focused APIs (recap)** — see section 8; Android's ongoing platform trend restricts broad, unscoped access to shared data (storage, location precision, background location) by default, requiring increasingly specific, narrowly-scoped permissions and APIs — a consistent design philosophy across recent Android versions worth recognizing as a pattern, not a one-off restriction.

**Secure data storage (EncryptedSharedPreferences, Keystore) — Definition:** `EncryptedSharedPreferences` wraps standard key-value storage with automatic encryption backed by the **Android Keystore system** (hardware-backed key storage on supporting devices) — Android's native equivalent of the React Native notes' SecureStore (section 12 there), used for the same purpose: storing sensitive values (auth tokens, API keys) encrypted at rest rather than in plaintext.

**ProGuard/R8 code obfuscation — Definition:** R8 (the modern successor to ProGuard) shrinks, optimizes, and obfuscates release-build code — removing unused code (reducing APK size, section 15) and renaming classes/methods to short, meaningless identifiers, making reverse-engineering a shipped app meaningfully harder — a release-build-only step, since obfuscated stack traces require an uploaded mapping file to de-obfuscate crash reports back into readable form.

---

## 13. Testing Android Applications

**Unit testing with JUnit + MockK — Definition:** standard JUnit (the JVM-ecosystem test framework already covered in the Java Backend notes) runs plain Kotlin unit tests on the local JVM (no device/emulator needed) for pure logic (ViewModels, repositories, use cases); MockK is a Kotlin-idiomatic mocking library (the same dependency-mocking principle covered across every other backend's testing notes in this workspace) for isolating the class under test from its real dependencies.

**Instrumented tests with Espresso — Definition:** Espresso runs tests **on an actual device or emulator**, driving real UI interactions against the legacy View system (`onView(withId(...)).perform(click())`) — Android's equivalent of the React Native notes' Detox/Maestro E2E testing, validating genuine end-to-end behavior rather than isolated unit logic.

**Compose UI testing — Definition:** Compose has its own dedicated testing APIs (`composeTestRule`, `onNodeWithText(...).performClick()`) — conceptually parallel to React (Native) Testing Library's query-by-what-the-user-sees philosophy (React notes' section 14), adapted specifically to Compose's semantics tree rather than the legacy View hierarchy Espresso targets.

**Testing ViewModels and coroutines — Definition:** testing coroutine-based ViewModel logic (section 6/10) requires `kotlinx-coroutines-test`'s `TestDispatcher`/`runTest` to run suspend functions deterministically and synchronously within a test, rather than actually waiting on real asynchronous delays — the same "control time/async execution deterministically in tests" principle already covered generally in the JS/TS and React testing sections, here specific to Kotlin's structured-concurrency model.

---

## 14. Push Notifications & Background Messaging

**Firebase Cloud Messaging (FCM) — Definition:** FCM is Android's (and, cross-platform, the React Native notes' underlying mechanism's) standard push-notification delivery service — an app registers for a device token, sends it to a backend server, and that backend triggers notifications through FCM's servers to reach the device even when the app isn't running — the same remote-notification architecture already covered from the cross-platform framework's perspective in the React Native notes' section 11, here as the native Android implementation directly.

**Notification channels (Android 8+) — Definition:** every notification must belong to a **channel** (e.g. "Messages," "Promotions"), which the *user* can individually configure (mute, change priority/sound) from system settings — a deliberate, user-controlled granularity Android introduced specifically to prevent a single blanket "allow/deny all notifications" toggle from being an app's only choice, requiring apps to declare and register their channels upfront rather than sending arbitrary, uncategorized notifications.

**Handling notification taps and deep links (recap)** — see section 5's deep linking and section 2's Intents; a notification's tap action is itself backed by a `PendingIntent`, carrying the same Intent-based navigation mechanism covered throughout this file, letting a tap route the user directly into a specific in-app destination.

---

## 15. Performance Optimization

**Avoiding unnecessary recomposition in Compose — Definition:** Compose recomposes (section 3) more efficiently than naive re-rendering, but can still recompose more than necessary if state is read too broadly (e.g. an entire list re-composing when only one item's data changed) or if unstable types are passed as parameters (preventing Compose's skip-if-unchanged optimization) — the same "minimize what re-renders" discipline already covered for React (React notes' section 11) and React Native (section 14 there), here specific to Compose's stability/skippability rules.

**Memory leaks (LeakCanary, common causes) — Definition:** LeakCanary is a widely-used library that automatically detects and reports memory leaks during development/testing — common Android-specific leak causes include holding a long-lived reference to an Activity/Context (e.g. from a singleton or a not-properly-cancelled background callback) past that Activity's actual lifetime, preventing it from being garbage collected — a category of bug largely specific to Android's Activity-lifecycle model, without a direct equivalent in a typical web app's simpler page-lifecycle.

**App startup time optimization (Baseline Profiles) — Definition:** a **Baseline Profile** is a list of critical code paths (the classes/methods exercised during app startup and key user flows) that the OS uses to ahead-of-time-compile that specific code at install time, rather than relying purely on ART's just-in-time compilation during the app's first runs — directly analogous to the React Native notes' Hermes engine discussion (section 14 there) of ahead-of-time bytecode compilation for faster startup, here as Android's own platform-level mechanism for the same underlying goal.

**ANRs and the main thread — Definition:** an **ANR (Application Not Responding)** dialog is shown by the OS when the main (UI) thread is blocked for too long (historically ~5 seconds) — the Android-specific, OS-enforced consequence of doing slow, blocking work (a synchronous network call, a heavy computation) directly on the main thread rather than offloading it via coroutines/`Dispatchers.IO` (section 10) — a stricter, more user-visible penalty for blocking the main thread than the web notes' generally-discussed "don't block the JS event loop" guidance, since Android will actively kill an unresponsive app if the user chooses to force-close it from the ANR dialog.

---

## 16. Multi-Module Architecture

**Why modularize a large app — Definition:** splitting a large single-module (`app`) codebase into multiple Gradle modules improves build times (Gradle can build/cache unchanged modules incrementally, parallelized), enforces clearer architectural boundaries (a module's dependencies are explicit in its `build.gradle.kts`, preventing accidental tangled coupling), and enables better team scalability (different teams owning different feature modules with a narrower, well-defined API surface between them) — the same motivations behind the Node.js notes' feature-based folder structure and the NestJS notes' module system, here enforced at the build-system level rather than just a folder convention.

**Feature modules vs core/shared modules — Definition:** a **feature module** contains a self-contained slice of app functionality (e.g. a `:feature:profile` module); **core/shared modules** (`:core:network`, `:core:database`, `:core:designsystem`) contain cross-cutting infrastructure/UI-primitives that multiple feature modules depend on — feature modules should generally not depend on one another directly, only on shared core modules, keeping the dependency graph a clean, largely one-directional structure rather than an unmanageable web.

**Build time benefits, dependency graph management** — Gradle's build-graph-aware incremental compilation means a change in one leaf feature module doesn't require rebuilding unrelated modules — a genuinely significant practical benefit for large codebases with long build times, and a primary, concrete motivation for modularizing beyond purely architectural cleanliness.

---

## 17. Build Variants, Flavors & CI/CD

**Build types vs product flavors — Definition:** a **build type** (`debug`/`release`) controls how code is compiled (debuggable, minification/R8 obfuscation on or off, section 12); a **product flavor** (e.g. `free`/`paid`, or `dev`/`staging`/`prod` pointing at different backend environments) represents a variant of the *product itself* — Gradle combines every build type with every flavor to produce the full matrix of possible build variants (e.g. `paidRelease`, `devDebug`) — Android's build-configuration equivalent of the multi-environment `.env` file pattern already covered across this workspace's Node.js/Deployment notes, expressed as a build-system-level concept instead.

**Signing configs — Definition:** each variant can be configured with its own signing config (section 18's key material), commonly keeping debug builds unsigned/auto-signed with a shared debug key for convenience while requiring release builds to use the real, securely-stored production signing key — mismanaging this (e.g. accidentally committing a release signing key to source control) is a serious, hard-to-recover-from security mistake, given a lost/leaked signing key's implications (section 18).

**Gradle Play Publisher, CI/CD pipelines (recap)** — the Gradle Play Publisher plugin automates uploading a built AAB directly to the Play Console from a CI pipeline — the same CI/CD automation principles already covered generally in the Deployment notes, here specific to Android's build-and-publish toolchain, letting a merged PR trigger an automatic build, test run, and (for release branches) Play Store track upload without manual, error-prone local steps.

---

## 18. Play Store Deployment

**App signing (upload key vs app signing key) — Definition:** modern Play App Signing separates a developer's own **upload key** (used to sign the AAB submitted to Google Play) from the actual **app signing key** (which Google securely manages and uses to sign the final APK served to users) — this separation means a compromised or lost upload key can be safely revoked and replaced (via a request to Google) without the catastrophic, permanent consequence a lost signing key represented under the older, pre-Play-App-Signing model (still true today for the underlying app signing key itself, which Google now custodies rather than the developer directly).

**Play Console: tracks, staged rollouts — Definition:** the Play Console offers multiple release **tracks** — **internal** (a small, fast-iteration tester group), **closed**/**open testing** (a larger beta audience), and **production** — letting a build be validated progressively before reaching all users; a **staged rollout** within the production track releases a new version to only a percentage of users initially (e.g. 10%), automatically monitorable for crash-rate regressions before manually increasing the rollout percentage — directly mitigating the risk of a bad release reaching 100% of users immediately, the mobile-store equivalent of a canary deployment already covered conceptually in the Deployment/System Design notes.

**App Bundles, Play Feature Delivery — Definition:** building on the AAB format (section 1), Play Feature Delivery lets specific app modules (section 16) be marked as **on-demand** or **conditional** — downloaded only when a user actually needs that feature, rather than bundled into every install — directly reducing initial install size for large apps with rarely-used features, a capability the AAB format's device-specific-APK-generation was specifically designed to enable.

---

## 19. Android Interview Prep

**Common interview questions** — explain the Activity lifecycle and why a ViewModel is needed to survive configuration changes (sections 2 & 6); what's the difference between `LiveData` and `StateFlow`, and why has the ecosystem moved toward the latter (section 6/10); walk through structured concurrency and why an orphaned coroutine can't normally happen (section 10); when would you choose WorkManager versus a foreground Service (section 11); explain recomposition and what makes a composable "skippable" (sections 3 & 15); why does Play App Signing's upload-key/app-signing-key split matter (section 18).

**Android vs React Native vs Flutter — when native wins** — native Android development is the right choice when an app needs deep, immediate access to newly-released platform APIs (native apps get first access, before cross-platform frameworks catch up), maximum possible performance for graphics/compute-intensive work, or when the team is already deeply invested in Kotlin/JVM expertise (Java Backend notes) with no meaningful cross-platform code-sharing need (an Android-only app, for instance) — the same tradeoff table already presented from the opposite direction in the React Native notes' section 18.

**Where Design Patterns show up natively in modern Android architecture — Definition:** direct mappings back to the Design Patterns notes, mirroring the same exercise already done for NestJS:
- **Dependency Injection** — Hilt/Dagger (section 7) *is* the DI pattern, exactly as in the NestJS/Java Backend notes.
- **Repository pattern** — the data layer's Repository classes (section 6) abstract Room/Retrofit data sources behind a single interface, the same pattern already covered architecturally in the Design Patterns notes and concretely in the Java Backend/NestJS notes' data-layer sections.
- **Observer pattern** — `StateFlow`/`Flow`/`LiveData` (section 6/10) are direct, modern implementations of Observer, the same reactive-stream lineage as RxJS/RxJava.
- **MVVM (Model-View-ViewModel)** — Android's entire recommended architecture (section 6) is a named instance of the MVVM pattern already covered in the Design Patterns notes' architectural-patterns section, here as the platform's own first-party, explicitly-endorsed default rather than an optional choice.
- **Builder pattern** — Retrofit's `Retrofit.Builder()` (section 9) and many other Android/Kotlin APIs use the classic Builder pattern for constructing complex configuration objects step by step.
