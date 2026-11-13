# Chapter 17 — Kotlin & Modern Android (updated to 11/2026)

> **Source**: your two note files in `/home/david/Desktop/temp/` (`Kotlin-Interview-Autosaved.docx`, `KhangDM-ingenico.docx`), reorganized by topic, with more depth and new material.
> Marker **⚠️ Notes fix**: places where the original notes were wrong or outdated. These are exactly the "trap" questions interviewers like.
> The "new" parts (Kotlin 2.x, Android 15/16) cover what has been released and is stable. For very recent versions, re-check the official *What's new* / *Behavior changes* pages before the interview.

---

## 17.1 How Kotlin runs on Android

### Q1. How does Kotlin run on Android?
```
.kt ──kotlinc (K2)──► .class (JVM bytecode) ──D8/R8──► classes.dex ──► APK/AAB
                                                            │
                         on device: ART ── dex2oat (AOT, profile-guided) + JIT + interpreter
```
- **⚠️ Notes fix**: the notes say *"executed by the JVM"*. Android **has no JVM**. The bytecode is converted to **DEX** by D8 and runs on **ART**. ART compiles AOT based on profiles (Baseline/Cloud profiles), plus JIT. In a ROM, system apps are dexpreopted ahead of time (see ch.03, and ch.11 Q45 on `LOCAL_DEX_PREOPT := false`).
- The notes also say *"Java compiles straight to machine code, JS is interpreted"*. Not right: Java compiles to **bytecode**, and only JIT/AOT turns it into machine code. Kotlin has many targets: JVM, Android, JS, **Native** (LLVM, iOS) and **Wasm**.
- D8 also **desugars** Java 8+ features (lambdas, `java.time` via core library desugaring) for low API levels.

### Q2. Why Kotlin? (a senior answer, not just "it's concise")
Null-safety in the type system, coroutines (structured concurrency), data/sealed classes for modeling state, extensions, DSLs (Gradle KTS, Compose), 100% Java interop, Multiplatform. Google has been "Kotlin-first" since 2019; Jetpack Compose can only be written in Kotlin.

---

## 17.2 Syntax & semantics (going deeper than the notes)

### Q3. `val`, `var`, `const val`?
- `var` is mutable, `val` is assigned once (**a read-only reference, not an immutable object**: `val list = mutableListOf()` can still `add`).
- `const val`: a **compile-time** constant, primitive/String only, at top level / in an `object` / `companion`; it is **inlined** at use sites. Change its value in a library without rebuilding the module using it, and that module keeps the old value.
- A `val` can have a custom getter, so each read may return a different value: `val now get() = System.currentTimeMillis()`.

### Q4. Null safety: `?.`, `?:`, `!!`, `as?`, platform types
```kotlin
val len = name?.length ?: -1          // Elvis
val s = obj as? String                // safe cast → null instead of ClassCastException
val n = name!!.length                 // throws NPE if null
requireNotNull(x) { "x null" }; checkNotNull(y)
```
- **⚠️ Notes fix**: `!!` throws `java.lang.NullPointerException` (since Kotlin 1.4), **no longer** `KotlinNullPointerException`.
- **Platform type** `String!`: a value coming from Java without `@Nullable/@NonNull` annotations is not null-checked by the compiler. This is the biggest source of NPEs in Kotlin code calling the Android SDK/AOSP Java. AOSP framework code has `@NonNull`/`@Nullable` (androidx.annotation or `android.annotation`), which Kotlin understands.
- Kotlin has no ternary operator; `if` is an **expression**: `val x = if (a > b) a else b`.

### Q5. `==` vs `===`?
- `==` calls `equals()` (null-safe); `===` compares **references**.
- **⚠️ Notes fix**: the `Integer(10)` example is deprecated. The proper example is boxing:
```kotlin
val a: Int? = 127;  val b: Int? = 127;  println(a === b)   // true  (Integer cache -128..127)
val c: Int? = 128;  val d: Int? = 128;  println(c === d)   // false (2 different objects)
```
  For non-nullable `Int`, `===` gives a deprecation warning (identity comparison of primitives is meaningless).

### Q6. Does Kotlin have primitives?
Not in source (only `Int`, `Double`...), but the compiler emits `int`/`double` whenever possible. Values are **boxed** to `Integer` when: nullable (`Int?`), generic (`List<Int>`), or stored as `Any`. To avoid boxing, use `IntArray` instead of `Array<Int>`, and `value class`.

### Q7. `data class`: what does the compiler generate, what are the traps?
- Generates `equals/hashCode/toString/copy/componentN` **only from properties in the primary constructor**. Properties declared in the class body are not part of `equals`.
- `copy()` is a **shallow copy**: inner lists are still shared.
- Cannot be `abstract/open/sealed/inner`, but **can** extend a sealed class/interface (common for UI state).
- New: `data object` (Kotlin 1.9) for singletons with a nice `toString`, common in sealed hierarchies:
```kotlin
sealed interface UiState {
    data object Loading : UiState
    data class Success(val channels: List<Channel>) : UiState
    data class Error(val msg: String) : UiState
}
```

### Q8. `value class` (inline class)?
```kotlin
@JvmInline value class ChannelId(val raw: Int)
fun tune(id: ChannelId)       // type-safe, usually just an int at runtime
```
No allocation (unless boxed: nullable, generic). Functions taking a value class have **mangled** names when seen from Java.

### Q9. `sealed` vs `enum`?
- Enum: each constant is **a single instance**, all of the same type. Sealed: each subclass has **its own state** and can have many instances.
- `when` on a sealed type/enum is **exhaustive** (no `else` needed), so adding a new subclass causes compile errors wherever it isn't handled. That's exactly the benefit.
- **⚠️ Notes fix**: sealed subclasses **no longer have to be in the same file**. Since Kotlin 1.5 they only need to be in the **same package and same module**. There's also `sealed interface`.
- Enum: use `entries` (1.9) instead of `values()` (`values()` allocates a new array on every call).

### Q10. `object`, `companion object`, object expressions: when are they initialized?
| | Initialized | Notes |
|--|---------|---------|
| `object Foo` (declaration) | lazily, on first access | thread-safe thanks to the JVM class initializer (`static {}`) |
| `companion object` | when the enclosing class is loaded | like a Java static initializer |
| `object : Listener {}` (expression) | right there | creates a **new instance** every time it runs; captures outer variables (can leak like an anonymous class) |

Replacements for Java's `static`: top-level functions (recommended), `object`, `companion`. To let Java call them as static, add `@JvmStatic`.

### Q11. `@JvmStatic`, `@JvmField`, `@JvmOverloads`, `@JvmName`, `@Throws`?
- `@JvmStatic`: generates a real static method (in a companion/object).
- `@JvmField`: exposes a public field, no getter/setter generated.
- `@JvmOverloads`: generates overloads for default parameters (required when writing a **custom View**: `constructor(context, attrs = null, defStyleAttr = 0)`).
- `@JvmName`: renames on JVM signature clashes (due to type erasure: `fun f(l: List<String>)` and `fun f(l: List<Int>)`).
- `@Throws(IOException::class)`: Kotlin has no checked exceptions, so Java callers need this annotation to be able to `catch`.

### Q12. `lateinit` vs `lazy`?
| | `lateinit var` | `val by lazy {}` |
|--|---------------|-----------------|
| Type | non-null, **not** primitive | any |
| When assigned | from outside (DI, `onCreate`, `@Before` in tests) | on first read |
| Thread-safety | none | `SYNCHRONIZED` by default; also `PUBLICATION`, `NONE` (use on the main thread for speed) |
| Read before assignment | `UninitializedPropertyAccessException` | (can't happen) |
| Check | `::name.isInitialized` | `lazyValue.isInitialized()` |
Android pitfall: `by lazy { findViewById(...) }` in a **Fragment** keeps the old View after `onDestroyView`, causing a leak and pointing at a dead View. Use ViewBinding and set it to `null` in `onDestroyView`.

### Q13. Visibility modifiers?
- `public` (default), `internal` (same Gradle/Soong **module**), `protected`, `private` (inside the class, or inside the **file** for top-level declarations).
- **⚠️ Notes fix**: Kotlin's `protected` is visible **only** in the class and its subclasses, **not** in the same file/package (unlike Java, where `protected` is visible in the whole package).
- `internal` seen from Java is `public` with a mangled name.

### Q14. Constructors & initialization order?
Primary constructor → property initializers and `init {}` blocks run **in the order they appear in the code** → secondary constructor bodies. A secondary constructor **must** delegate to the primary (`: this(...)`).
- **⚠️ Notes fix**: *"constructor parameters are val by default"* is wrong. Without `val/var` it's just a **parameter**, not a property.
- Trap: calling an `open` function from a parent class's `init`. At that point the subclass hasn't initialized its properties, so you can read `null` even though the type is non-null.

### Q15. Delegation (`by`)?
- **Class delegation** (composition instead of inheritance, exactly as the notes' "Replace inheritance with delegation"):
```kotlin
class LoggingList<T>(private val inner: MutableList<T> = mutableListOf()) : MutableList<T> by inner {
    override fun add(element: T): Boolean { Log.d("L", "add $element"); return inner.add(element) }
}
```
- **Property delegation**: `by lazy`, `Delegates.observable/vetoable`, `by map`, `by viewModels()`, `by remember { mutableStateOf() }`, `by preferences`... Write your own with `operator fun getValue/setValue` or `ReadWriteProperty`.

### Q16. Scope functions?
| Function | Object inside the block | Returns | Use when |
|-----|----------------------|--------|----------|
| `let` | `it` | the lambda result | handling a nullable value: `x?.let { }` |
| `run` | `this` | the lambda result | computing from an object |
| `with(obj)` | `this` | the lambda result | calling many functions on one object (not an extension) |
| `apply` | `this` | **the object itself** | configuring an object (builder) |
| `also` | `it` | **the object itself** | side effects (log, validate) |
Trap: `x?.let { ... } ?: run { ... }` is not a real if/else. If the `let` block returns `null`, the `run` branch **also runs**.

### Q17. How are extension functions resolved?
**Statically**, by the declared type (compile time), not the runtime type (not polymorphic). They compile into static methods taking the receiver as the first parameter. If a member function has the same name, **the member always wins**. Extensions can't access `private` members.

### Q18. Lambdas, higher-order functions, cost?
- A lambda that **captures** variables allocates a new object each time it runs; a non-capturing one is a singleton.
- **⚠️ Notes update**: the notes say *"every lambda generates an anonymous class"*. Since **Kotlin 2.0**, lambdas are generated with `invokedynamic` (`LambdaMetafactory`) by default, not as separate classes. On Android, D8 desugars them again.
- A lambda with receiver `T.() -> Unit` is the foundation of DSLs (`apply`, `buildString`, Compose, Gradle KTS).

### Q19. `inline`, `noinline`, `crossinline`, `reified`?
- `inline`: copies the function body **and its lambdas** to the call site. The main benefit is **eliminating the lambda object** and enabling **non-local returns**, not "saving stack".
- `noinline`: that lambda is not inlined (when you need to store the lambda or pass it on).
- `crossinline`: the lambda is inlined but **non-local returns are forbidden** (because it runs in another context, e.g. a `Runnable`).
- `reified` (only with `inline`): keeps the type T at runtime: `inline fun <reified T> Gson.fromJson(s: String) = fromJson(s, T::class.java)`.
- Downside: bytecode bloat. A `public inline` function can't access `private` members; to use them they must be marked `@PublishedApi internal`.

### Q20. Generics: `in`/`out`, star projection, type erasure
- `out T` (covariant, producer): `List<out E>`, so `List<Int>` is a subtype of `List<Number>`.
- `in T` (contravariant, consumer): `Comparable<in T>`.
- Invariant: `MutableList<T>`, `Array<T>`.
- Use-site variance: `fun copy(from: Array<out Any>, to: Array<Any>)` (like Java's `? extends`).
- `*`: `List<*>` can be read as `Any?`, can't be written.
- Generic types are **erased** at runtime, so you can't write `is List<String>`. The fix is `reified`.
- The notes' mnemonic is still right: *functions (lambdas) are contravariant in parameters, covariant in return type*: `(Number) -> Int` is a subtype of `(Int) -> Number`.

### Q21. Collections vs Sequences, `List` vs `Array`, `map` vs `flatMap`
- Collection: **eager**, each step creates an intermediate list. Sequence: **lazy**, each element flows through the whole chain, with short-circuiting (`first`, `take`). Sequences pay off when the data is large **and** there are many steps, or with `take/first`. For small lists, collections are usually faster.
- **⚠️ Notes fix**: `List` is a **read-only** interface, not immutable, and has nothing to do with "fixed size". `Array` has a fixed size, is mutable, invariant, and maps to a Java array. `IntArray` is `int[]`, not boxed.
- `map`: 1 element → 1 element. `flatMap`: 1 element → many elements, then flattened into one list.
- `HashMap` vs `List` (ingenico notes): `list[i]` is O(1), `list.contains(x)` is O(n), `map[key]` is O(1) on average (worst case O(log n) when a bucket becomes a tree in Java 8+). Android has `ArrayMap`/`SparseArray`, memory-efficient for small maps (< ~1000 entries), but lookups are O(log n).

### Q22. `when`, labels, `Pair/Triple`
- `when` is an expression, accepts `in range`, `is Type` (smart cast), or no subject. **New (stable since Kotlin 2.2)**: **guard conditions**:
```kotlin
when (state) {
    is UiState.Success if state.channels.isEmpty() -> showEmpty()
    is UiState.Success -> show(state.channels)
    else -> {}
}
```
- **⚠️ Notes fix**: a label is **not** "any expression". A label is an identifier `name@` attached to an expression, used with `break@`, `continue@`, `return@`. The most common example: `list.forEach { if (it < 0) return@forEach }`, which exits only the lambda.
- `Pair/Triple`: fine for temporary use. For a public API, create a `data class` with meaningful field names.

---

## 17.3 Coroutines & Flow (fixing and deepening the notes)

### Q23. What is a coroutine, and how does it differ from a thread?
- A coroutine is a **suspendable computation**. The compiler turns a `suspend` function into a **state machine** + `Continuation` (CPS, see ch.11 Q24). On suspension, the thread is **given back** for other work.
- Cheap: tens of thousands of coroutines are just objects on the heap, while each thread costs ~1MB of stack and needs kernel scheduling.
- **⚠️ Notes fix**: *"starting a coroutine creates a new thread"* is wrong. Coroutines run on the **dispatcher's** threads. `launch(Dispatchers.Main)` creates no thread at all.
- Concurrency (interleaving, possibly on 1 core) vs parallelism (truly simultaneous on multiple cores). The notes got this right.

### Q24. `launch` vs `async` vs `runBlocking` vs `coroutineScope`?
- `launch` returns a `Job` (fire-and-forget); `async` returns a `Deferred<T>`, whose result you get with `await()`.
- `runBlocking`: **blocks** the current thread. Only for `main()`/tests, **never** on the Android main thread.
- `coroutineScope {}`: suspends until all children finish; one failing child cancels the whole group. `supervisorScope {}`: a failing child doesn't cancel its siblings.
- Running things in parallel properly:
```kotlin
suspend fun loadHome() = coroutineScope {
    val channels = async { repo.channels() }
    val epg      = async { repo.epg() }
    Home(channels.await(), epg.await())
}
```
The notes' example (`one + two.await()`) **doesn't compile**: `one` is a `Deferred`; it must be `one.await() + two.await()`.

### Q25. Structured concurrency, Android scopes
- Every coroutine has a parent; the parent finishes only when all children finish; cancelling the parent cancels all children.
- `viewModelScope` (`SupervisorJob + Dispatchers.Main.immediate`, cancelled in `onCleared`), `lifecycleScope` (cancelled at `DESTROYED`), `rememberCoroutineScope()` (Compose).
- **⚠️ Notes fix**: `GlobalScope` is not tied to "the application lifecycle". It **has no Job**, sits outside structured concurrency, can't be cancelled by anyone and leaks easily, which is why it's marked `@DelicateCoroutinesApi`. For work that must outlive a screen, create your own **application scope** (`CoroutineScope(SupervisorJob() + Dispatchers.Default)`, injected with Hilt) or use WorkManager.

### Q26. Dispatchers?
| Dispatcher | Used for | Details |
|------------|---------|----------|
| `Main` / `Main.immediate` | UI | `immediate`: if already on the main thread, runs immediately without posting |
| `Default` | CPU (JSON parsing, DiffUtil, sorting) | threads = number of cores (minimum 2) |
| `IO` | blocking I/O | up to `max(64, cores)`, **shares the thread pool** with Default |
| `Unconfined` | almost never used in apps | runs on whatever thread resumes it |
- `Dispatchers.IO.limitedParallelism(1)`: a sequential dispatcher without extra threads. This replaces `newSingleThreadContext`/`newFixedThreadPoolContext`: those create real threads, must be `close()`d, and are marked delicate/obsolete. **⚠️ Notes update.**
- **Main-safety** (the notes got this right): the repository does its own `withContext(Dispatchers.IO)`, the ViewModel just does `viewModelScope.launch {}` on Main. **Don't** write `viewModelScope.launch(Dispatchers.IO)` and then update the UI inside it.

### Q27. Cancelling coroutines
- Cancellation is **cooperative**: kotlinx suspend functions (`delay`, `withContext`, `yield`...) check for it and throw `CancellationException`.
- **⚠️ Notes fix**: a coroutine running a computation loop **can still be cancelled**, as long as the loop checks `isActive` / `ensureActive()` / `yield()`.
- Big trap: `try { ... } catch (e: Exception) {}` or `runCatching {}` **also swallow `CancellationException`**, so the coroutine keeps running after being cancelled. Always rethrow `CancellationException`.
- Cleanup that needs to suspend after cancellation: `withContext(NonCancellable) { ... }` in `finally`.
- Blocking code (sockets, `Thread.sleep`) doesn't react to cancellation. Use `runInterruptible {}` or APIs with timeouts.

### Q28. Exception handling
- An exception in a child coroutine **propagates to the parent** immediately (even with `async`). The parent cancels the remaining children and rethrows upward.
- A `CoroutineExceptionHandler` only works on a **root** coroutine (a `launch` directly on a scope, or a direct child of a `SupervisorJob`). Attaching it to a child coroutine does nothing.
- `async` at the root: the exception is held until `await()`.
- In `viewModelScope` (SupervisorJob): one failing `launch` doesn't break other `launch`es, but without a handler the exception still reaches the uncaught handler and **crashes the app**.

### Q29. Flow basics (already in the notes) and the wrong parts
- Flows are **cold**: the code in `flow {}` only runs on `collect`, and runs again for **each** collector.
- **⚠️ Notes fix**: *"a flow can't be cancelled"* is wrong. A flow is cancelled with the coroutine collecting it (cancel the scope and collection stops); the `flow {}` builder also checks for cancellation on every `emit`. Stop early with `take(n)`, `first()`, `takeWhile`.
- **Context preservation** (the notes got the idea right): you **must not** wrap `emit` in `withContext` inside `flow {}`; it throws `IllegalStateException: Flow invariant is violated`. To change the upstream thread, use **`flowOn(Dispatchers.IO)`**. `flowOn` affects only the operators **above** it.
- Frequently asked operators: `map/filter/transform`, `flatMapLatest` (e.g. search: keep only the latest query), `combine` vs `zip`, `debounce`, `distinctUntilChanged`, `buffer`/`conflate`/`collectLatest` (backpressure handling), `catch` (only catches upstream errors), `retryWhen`, `onEach`/`launchIn`.
- Turning a callback API into a Flow:
```kotlin
fun DisplayManager.displayChanges(): Flow<Int> = callbackFlow {
    val l = object : DisplayManager.DisplayListener {
        override fun onDisplayChanged(id: Int) { trySend(id) }
        override fun onDisplayAdded(id: Int) {}
        override fun onDisplayRemoved(id: Int) {}
    }
    registerDisplayListener(l, null)
    awaitClose { unregisterDisplayListener(l) }   // mandatory, otherwise the listener leaks
}
```

### Q30. `StateFlow`, `SharedFlow`, `Channel`: which one?
| | StateFlow | SharedFlow | Channel |
|--|-----------|------------|---------|
| Hot/cold | hot | hot | hot |
| Current value | always has `.value` | depends on `replay` | no |
| Multiple collectors | broadcast | broadcast | **each element goes to only one receiver** |
| Duplicate values | skipped (conflation + `equals`) | all emitted | all emitted |
| Used for | UI state | events for many listeners | one-shot events (navigate, toast), pipelines between coroutines |
- **⚠️ Notes fix**: Channels are for communication **between coroutines in the same process**, **not** between processes. IPC uses Binder/AIDL/Messenger/ContentProvider.
- Standard ViewModel pattern:
```kotlin
val uiState: StateFlow<UiState> = repo.channelsFlow()
    .map<List<Channel>, UiState> { UiState.Success(it) }
    .catch { emit(UiState.Error(it.message.orEmpty())) }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UiState.Loading)
```
  `WhileSubscribed(5000)`: stops the upstream after 5s without collectors. That's enough to survive rotation/recreation without re-querying.
- Collecting safely in the UI: `repeatOnLifecycle(STARTED)` (Views) or `collectAsStateWithLifecycle()` (Compose).

### Q31. Testing coroutines?
`runTest` (skips `delay` with virtual time), `StandardTestDispatcher`/`UnconfinedTestDispatcher`, `Dispatchers.setMain(testDispatcher)`, inject dispatchers through the constructor (don't hard-code `Dispatchers.IO`), and the **Turbine** library for testing Flows.

---

## 17.4 Android components (fixing the "ingenico" notes)

### Q32. Process priority (the notes have 4 levels)
Addition: the system computes it as `oom_score_adj` (see the table in ch.12 Q9). A service process that has been running for a long time (> 30 minutes) is demoted to `SERVICE_B_ADJ` (matching `MAX_SERVICE_INACTIVITY = 30 minutes` in `ActivityManagerConstants.java:181`). A14 adds the **freezer**: cached processes are frozen and get no CPU.

### Q33. IPC: AIDL, Messenger, Broadcast, ContentProvider
- **⚠️ Notes addition**:
  - **Messenger** also runs on Binder (`IMessenger.aidl`), but every message goes into **one Handler**, so they're processed sequentially on one thread. Easy to write, no thread-safety worries.
  - **AIDL**: calls arrive on the **binder thread pool** (up to 15 threads + main), so you must synchronize yourself. Synchronous by default; declare `oneway` to not wait (details in ch.04).
  - **ContentProvider**: shares tabular/file data with per-URI permissions. **Broadcast**: one-way, many receivers.
  - Large data: `ParcelFileDescriptor`/`SharedMemory`, because of the ~1MB binder buffer limit.

### Q34. Handler / Looper / MessageQueue
- **⚠️ Notes fix**: *"a Thread is made of a Handler, a Looper and a Runnable"* gets the relationship wrong. A thread is an OS thread. A thread **may have** one `Looper` (a loop reading the `MessageQueue`). A `Handler` is attached to a Looper and is used to **send** `Message`s/`Runnable`s into that queue; the messages are processed on the Looper's thread. Native details (epoll/eventfd) in ch.11 Q20 and ch.14 Q14.
- `HandlerThread` = a Thread with a ready Looper. Since API 30, `Handler()` without a Looper is deprecated (you must specify the Looper).

### Q35. AsyncTask, IntentService, JobIntentService
- **⚠️ Notes update**: `AsyncTask` is deprecated since API 30. `IntentService` is deprecated since API 30, and **`JobIntentService` is deprecated too**. Replacements:
  - background work that must complete → **WorkManager**;
  - work tied to the UI → **coroutines**;
  - user-visible work → an **FGS** (with the right type) or a **User-Initiated Data Transfer job** (A14).
- Why AsyncTask leaks (the notes got this right): the inner class holds a reference to the Activity, and AsyncTask isn't tied to the lifecycle.

### Q36. Service: the wrong points in the notes
- **⚠️ Notes fix**:
  - *"An FGS isn't killed when the user kills the app"*: when the user **force stops** the app (or swipes it away on some OEMs), the FGS is stopped too. An FGS is merely kept with priority by the system (adj `PERCEPTIBLE_APP_ADJ = 200`).
  - *"Starting an FGS while the app is in the foreground makes the system create a background service"*: wrong. `startForegroundService()` creates a normal service, and the app **must** call `startForeground()` within 30s (A14).
  - A bound service is destroyed when **all clients unbind and the service isn't in the started state**.
- New FGS rules:
  - **A14**: `foregroundServiceType` + a per-type permission are mandatory (ch.11 Q10).
  - **A15**: `dataSync` and `mediaProcessing` (a new type) are limited to **6 hours per 24 hours**; when time runs out the system calls `Service.onTimeout(int, int)` and the app must `stopSelf()` within a few seconds, otherwise it's treated as an error (ANR/crash). `BOOT_COMPLETED` receivers **may not** start FGSs of type `dataSync`, `camera`, `mediaPlayback`, `phoneCall`, `mediaProjection`, `microphone`.
  - For **STB system apps** (persistent/privileged, e.g. `DavidExtension`) most restrictions are waived, but you need to know them to review third-party apps.

### Q37. Parcelable vs Serializable
- Serializable: uses reflection, creates many temporary objects, slow. Parcelable: explicit write/read code, fast. In Kotlin use **`@Parcelize`** (the `kotlin-parcelize` plugin) instead of writing it by hand.
- Trap: Bundles/Intents all go through binder (the shared ~1MB limit), causing `TransactionTooLargeException`. Since A13 use `getParcelableExtra(key, Clazz::class.java)` (the old version without a class is deprecated, and the new one is safer against "Parcel mismatch" attacks).

### Q38. Stack vs Heap on Android
- **⚠️ Notes fix**: the `-Xss/-Xms/-Xmx` options belong to the desktop JVM. On ART:
  - the heap limit comes from `dalvik.vm.heapgrowthlimit` (normal apps) and `dalvik.vm.heapsize` (apps with `android:largeHeap="true"`);
  - inspect them with `ActivityManager.getMemoryClass()/getLargeMemoryClass()`;
  - the main thread's stack is provided by the kernel (~8MB); other Java threads default to ~1MB.
- ART uses **Concurrent Copying / Generational CC GC**, with almost no long pauses. Bitmap pixels live on the **native heap** since A8.
- Deep recursion causes `StackOverflowError`; running out of heap causes `OutOfMemoryError`. The notes got this part right.

### Q39. Pass-by-value or by reference?
**⚠️ Notes clarification**: Java/Kotlin are **always pass-by-value**. For objects, the "value" passed is **a copy of the reference**. So modifying an object's property is visible outside, but assigning `param = Other()` doesn't change anything outside.

### Q40. Multithreaded synchronization (beyond `synchronized`)
`synchronized`/`@Synchronized`, `ReentrantLock`, `@Volatile` (only guarantees visibility, not atomicity), `AtomicInteger`, `ConcurrentHashMap`. With coroutines: `Mutex` (doesn't block the thread), `limitedParallelism(1)`, actors/Channels. **Don't** wrap `synchronized` around calls to `suspend` functions.
- Deadlock (the notes define it in database terms): here it's 2 threads holding 2 locks in opposite order. Prevent it with a fixed lock order and by not calling binder/foreign callbacks while holding a lock (exactly the `LedManagerService` lesson in ch.06).

---

## 17.5 Architecture & Design patterns (fixing the "ingenico" notes)

### Q41. MVC / MVP / MVVM / MVI
| | Who holds the logic | View ↔ logic | Notes |
|--|-------------|--------------|---------|
| MVC (old Android) | the Activity is both View and Controller | tight | hard to test |
| MVP | Presenter | through an interface, **two-way** | the Presenter holds a View reference (must detach to avoid leaks), doesn't survive config changes by itself |
| MVVM | ViewModel | the View **observes** state; the ViewModel doesn't know the View | the ViewModel survives config changes |
| MVI | reducer/ViewModel | **one unidirectional flow**: Intent → (reducer) → **one immutable State** → render | easy to debug and test, explicit state; the cost is creating a new state object every time |
- **⚠️ Notes fix**: *"MVVM is 1-n, many views share one ViewModel"* is not a defining trait of MVVM. Usually it's 1 screen ↔ 1 ViewModel; sharing is a special case (`activityViewModels()`, navigation-graph-scoped ViewModels).
- Google's current recommended architecture: **UI layer** (UI state + UDF, ViewModel) → **Domain layer** (UseCases, optional) → **Data layer** (Repositories, **single source of truth**, offline-first: Room is the primary source, the network only syncs). Plus DI with Hilt and modularization (`:feature:*`, `:core:*`).

### Q42. Design patterns: the mix-ups in the notes
- **⚠️ Command ≠ Observer**: Command **encapsulates a request as an object** (can be queued, undone, logged), e.g. a `Runnable` posted to a Handler, a `WorkRequest`, ATMS's `ClientTransaction` (ch.12 Q5). Observer is publisher/subscriber (`LiveData`, `Flow`, `RemoteCallbackList`).
- **Strategy vs State**: in Strategy the **outside** chooses the algorithm (e.g. `Comparator`, `Interpolator`). In State the **object itself changes** its behavior based on internal state (e.g. `PlayerState`). The notes got this right.
- **Singleton**: the `getInstance` function in the notes is **not thread-safe** (2 threads can both see `null` and create 2 instances). Fix:
```kotlin
companion object {
    @Volatile private var instance: AQIRepository? = null
    fun getInstance(remote: AQIDataSource) =
        instance ?: synchronized(this) { instance ?: AQIRepository(remote).also { instance = it } }
}
```
  Or use `object`, or best of all `@Singleton` in Hilt (easy to test, easy to replace with a fake).
- **Builder** in Kotlin: usually unnecessary thanks to named arguments + default values + `apply`. Still seen in Java APIs (`AlertDialog.Builder`, `NotificationCompat.Builder`).
- **Facade**: Retrofit, the notes' example, is a facade + a **dynamic proxy** (`Proxy.newProxyInstance` generates the implementation of the interface). `XxxManager` classes in the framework are facades over binder (ch.06).
- **Adapter**: `RecyclerView.Adapter`, Leanback's `ItemBridgeAdapter`.
- **Decorator**: `ContextWrapper` (ch.13 Q10), nested `InputStream`s.
- AOSP also has: **Proxy/Stub** (Binder), **Template method** (the `Activity` lifecycle), **Chain of responsibility** (`ViewRootImpl`'s `InputStage`, `dispatchKeyEvent`), **Object pool** (`Message.obtain()`).

### Q43. Interface vs abstract class
**⚠️ Notes update**: *"functions in an interface have no body"* is only true for Java ≤ 7. Kotlin interfaces (and Java 8+ `default`) **may have bodies** and abstract properties or properties with getters, but **can't hold state** (no backing fields) and **have no constructor**. An abstract class has state and constructors but you can extend only one. Java interop: function bodies in Kotlin interfaces used to compile into a `DefaultImpls` class (Java didn't see them as `default` methods), requiring `-Xjvm-default=all`. Kotlin 2.2 changed the default to emit Java `default` methods (new option `-jvm-default`).

---

## 17.6 New from 2023 to 11/2026 (often asked: "are you up to date?")

### Q44. What's new from Kotlin 1.9 → 2.x?
| Version | Highlights |
|-----------|-----------|
| 1.9 (2023) | `data object`, `enum.entries`, the `..<` operator; Kotlin Multiplatform **Stable** (11/2023) |
| **2.0** (05/2024) | **K2 compiler** stable (faster builds, smarter smart casts: through local variables, through `||`, inside lambdas), **the Compose compiler moved into the Kotlin repo** (plugin `org.jetbrains.kotlin.plugin.compose`), lambdas generated with `invokedynamic` |
| 2.0.20 | Compose **strong skipping** on by default |
| 2.1 (11/2024) | previews: guard conditions in `when`, non-local `break/continue` in inline lambdas, multi-dollar interpolation `$$"""{"id": "$$id"}"""` |
| 2.2 (06/2025) | those three features **stable**; **context parameters** (preview, replacing context receivers) |
- Ecosystem: **KSP2** replaces kapt (kapt is in maintenance mode; Room/Hilt/Moshi all support KSP); `kotlinx.serialization` replaces Gson (Gson ignores Kotlin's default values and null-safety); **Compose Multiplatform for iOS stable** (2025).

### Q45. Compose: what's new
- Strong skipping is on by default, so lambdas and unstable parameters can still skip if they are `===`. "Stability" (ch.11 Q37) is less painful now, but you still need to know it.
- `Modifier.Node` replaces `composed {}` (better performance); `collectAsStateWithLifecycle`, `LifecycleStartEffect/ResumeEffect`; `LazyLayout` + prefetch.
- Navigation: **type-safe routes** (Navigation 2.8, routes are `@Serializable` classes); **Navigation 3** (launched in 2025): the back stack is a list state managed by the app itself.
- **Compose for TV** (`androidx.tv:tv-material`) is Google's recommended direction for new TV apps, replacing Leanback. Leanback still works but gets no new features. For an STB like this project: a new OEM launcher/app should consider Compose for TV, but you must measure **cold start and RAM** on a weak SoC, and use a Baseline Profile.

### Q46. What did Android 15 (API 35) change for apps?
- **Edge-to-edge is enforced** when targeting 35 (`enableEdgeToEdge()`, handle `WindowInsets`).
- FGS: 6h timeout for `dataSync`/`mediaProcessing` + `onTimeout()`; restrictions on starting FGSs from `BOOT_COMPLETED` (Q36).
- BAL: the PendingIntent **creator** no longer grants BAL privileges by default (ch.13 Q17).
- **16 KB page size**: new devices may use 16KB pages, so `.so` files must be 16KB-aligned (NDK r27+/AGP 8.5.1+). Google Play requires it from 11/2025 for apps targeting 35+ with native code.
- Private space, predictive back animations (opt-in), `ApplicationStartInfo` (tells why a process was started: cold/warm, by a broadcast or an activity).

### Q47. What did Android 16 (API 36, released 06/2025) change?
- On large screens (sw ≥ 600dp), apps targeting 36 have their `screenOrientation`, `resizeableActivity=false` and aspect-ratio restrictions **ignored** → adaptive layouts become mandatory. (There's a temporary opt-out for target 36.)
- **Edge-to-edge can no longer be opted out of** (`windowOptOutEdgeToEdgeEnforcement` removed).
- **Predictive back** is on by default for target 36: `onBackPressed()` is no longer called and `KEYCODE_BACK` is no longer dispatched. You must use `OnBackPressedCallback`/`OnBackInvokedCallback`. On **TV** this is a key point: the remote has a BACK key, and any app still catching `KEYCODE_BACK` in `onKeyDown` will have broken navigation when it raises its target.
- **Live Updates / progress-centric** notifications; JobScheduler quota adjustments (jobs started while the app has an FGS also count against quota); extra protections against intent redirection.
- New release cadence: Google ships a **major SDK release in Q2** and a **minor SDK release in Q4** each year (Android 16 QPR2 brought a minor SDK). The question "which API level?" can now have answers like `36.1`.
- **Android 17 (2026)**: this document **does not list** details because they couldn't be verified. Read the *Behavior changes: apps targeting Android 17* page before the interview.

### Q48. Current build & performance tools/practices
- Gradle **version catalog** (`libs.versions.toml`), convention plugins, Kotlin DSL; R8 full mode; **Baseline Profile + Startup Profile** (Macrobenchmark) cut cold start by 20–40%. Very worth doing for an STB launcher, since `DavidLauncher` currently has dexpreopt disabled.
- Security: Credential Manager (passkeys), the Photo Picker instead of media read permissions, the Play Integrity API.
- On-device AI: Gemini Nano via AICore / ML Kit GenAI APIs (on some devices). Just know the concept.

---

## 17.7 Kotlin rapid-fire (≤ 30 seconds/question)

| # | Question | Answer |
|---|---------|---------|
| 1 | `Unit` vs `Nothing` vs `Any`? | `Unit` has exactly 1 value (like `void`); `Nothing` has no values (a function that always throws/never returns; a subtype of every type); `Any` is the root of all non-null types |
| 2 | Does `typealias` create a new type? | No, just another name. For a new type-safe type use a `value class` |
| 3 | Is a Kotlin `object` thread-safe? | Yes, its initialization relies on JVM/ART class init |
| 4 | Default thread mode of `lazy`? | `SYNCHRONIZED` |
| 5 | Requirements for `infix`? | member/extension, exactly 1 parameter, no vararg/default |
| 6 | `operator` examples? | `plus`, `get/set` (`[]`), `invoke`, `contains` (`in`), `compareTo`, `getValue/setValue` (delegates), `componentN` |
| 7 | Where can `suspend` be called from? | From another `suspend` function or from a coroutine builder |
| 8 | `withContext` vs `async{}.await()`? | Same result, but `withContext` is cleaner and doesn't create a `Deferred` |
| 9 | Which part does `flowOn` affect? | The operators **above** it (upstream) |
| 10 | `SharedFlow(replay=0)` emitting with no collectors? | The value is lost (with `tryEmit`), or `emit` suspends if there's a buffer + a slow collector |
| 11 | `Dispatchers.Main` in a JVM unit test? | Not available; use `Dispatchers.setMain(testDispatcher)` |
| 12 | What does `@Parcelize` need? | the `kotlin-parcelize` plugin, the class implements `Parcelable`, properties in the primary constructor |
| 13 | Advantage of `sealed interface` over `sealed class`? | A class can implement several sealed interfaces; enums can implement them too |
| 14 | Can `const` be used in a regular class? | No, only at top level, in an `object`, or a `companion` |
| 15 | Which exception does `!!` throw (Kotlin 1.4+)? | `java.lang.NullPointerException` |
