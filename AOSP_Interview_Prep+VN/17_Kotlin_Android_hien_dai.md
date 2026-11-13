# Chương 17 — Kotlin & Android hiện đại (cập nhật tới 11/2026)

> **Nguồn**: hai file ghi chú của bạn ở `/home/david/Desktop/temp/` (`Kotlin-Interview-Autosaved.docx`, `KhangDM-ingenico.docx`), sắp xếp lại theo chủ đề, bổ sung chiều sâu và kiến thức mới.
> Ký hiệu **⚠️ Sửa ghi chú**: chỗ ghi chú gốc sai hoặc đã lỗi thời. Đây đúng là các câu interviewer hay "bẫy".
> Phần "mới" (Kotlin 2.x, Android 15/16) viết theo những gì đã phát hành và ổn định. Các chi tiết của phiên bản rất mới thì nên mở trang *What's new* / *Behavior changes* chính thức xem lại trước buổi phỏng vấn.

---

## 17.1 Kotlin chạy trên Android thế nào

### Q1. Kotlin chạy trên Android ra sao?
```
.kt ──kotlinc (K2)──► .class (JVM bytecode) ──D8/R8──► classes.dex ──► APK/AAB
                                                            │
                         trên máy: ART ── dex2oat (AOT, theo profile) + JIT + interpreter
```
- **⚠️ Sửa ghi chú**: ghi chú viết *"thực thi bởi JVM"*. Android **không có JVM**. Bytecode được D8 chuyển sang **DEX**, chạy trên **ART**. ART biên dịch AOT theo profile (Baseline/Cloud profile), cộng thêm JIT. Trong ROM thì app hệ thống được dexpreopt sẵn (xem ch.03, ch.11 Q45 về `LOCAL_DEX_PREOPT := false`).
- Ghi chú cũng viết *"Java biên dịch thẳng ra mã máy, JS thông dịch"*. Không đúng: Java biên dịch ra **bytecode**, sau đó JIT/AOT mới ra mã máy. Kotlin có nhiều target: JVM, Android, JS, **Native** (LLVM, iOS) và **Wasm**.
- D8 còn **desugar** tính năng Java 8+ (lambda, `java.time` qua core library desugaring) cho API thấp.

### Q2. Vì sao dùng Kotlin? (trả lời senior, không chỉ "ngắn gọn")
Null-safety ở mức type system, coroutines (structured concurrency), data/sealed class giúp mô hình hoá state, extension, DSL (Gradle KTS, Compose), interop 100% với Java, Multiplatform. Google "Kotlin-first" từ 2019; Jetpack Compose chỉ viết được bằng Kotlin.

---

## 17.2 Cú pháp & ngữ nghĩa (đi sâu từ ghi chú)

### Q3. `val`, `var`, `const val`?
- `var` đổi được, `val` chỉ gán một lần (**read-only reference, không phải immutable object**: `val list = mutableListOf()` vẫn `add` được).
- `const val`: hằng **compile-time**, chỉ kiểu primitive/String, nằm ở top-level / `object` / `companion`; được **inline** vào chỗ dùng. Đổi giá trị trong một library mà không build lại module dùng nó thì module đó vẫn giữ giá trị cũ.
- `val` có thể có custom getter, nên mỗi lần đọc có thể ra giá trị khác: `val now get() = System.currentTimeMillis()`.

### Q4. Null safety: `?.`, `?:`, `!!`, `as?`, platform type
```kotlin
val len = name?.length ?: -1          // Elvis
val s = obj as? String                // safe cast → null thay vì ClassCastException
val n = name!!.length                 // ném NPE nếu null
requireNotNull(x) { "x null" }; checkNotNull(y)
```
- **⚠️ Sửa ghi chú**: `!!` ném `java.lang.NullPointerException` (từ Kotlin 1.4), **không còn** `KotlinNullPointerException`.
- **Platform type** `String!`: giá trị lấy từ Java không có annotation `@Nullable/@NonNull` thì compiler không kiểm tra null. Đây là nguồn NPE phổ biến nhất trong code Kotlin gọi Android SDK/AOSP Java. Framework AOSP có `@NonNull`/`@Nullable` (androidx.annotation hoặc `android.annotation`), Kotlin đọc được.
- Kotlin không có toán tử 3 ngôi; `if` là **biểu thức**: `val x = if (a > b) a else b`.

### Q5. `==` vs `===`?
- `==` gọi `equals()` (an toàn với null); `===` so sánh **tham chiếu**.
- **⚠️ Sửa ghi chú**: ví dụ `Integer(10)` đã deprecated. Ví dụ đúng là boxing:
```kotlin
val a: Int? = 127;  val b: Int? = 127;  println(a === b)   // true  (cache Integer -128..127)
val c: Int? = 128;  val d: Int? = 128;  println(c === d)   // false (2 object khác nhau)
```
  Với `Int` không nullable, `===` bị compiler cảnh báo deprecated (vì so sánh danh tính của primitive là vô nghĩa).

### Q6. Kotlin có primitive không?
Ở source thì không (chỉ có `Int`, `Double`...), nhưng compiler sinh `int`/`double` khi có thể. Bị **box** thành `Integer` khi: nullable (`Int?`), generic (`List<Int>`), hoặc lưu trong `Any`. Muốn tránh boxing thì dùng `IntArray` thay `Array<Int>`, và `value class`.

### Q7. `data class`: compiler sinh gì, có bẫy gì?
- Sinh `equals/hashCode/toString/copy/componentN` **chỉ từ property trong primary constructor**. Property khai trong thân class không được tính vào `equals`.
- `copy()` là **shallow copy**: list bên trong vẫn dùng chung.
- Không được `abstract/open/sealed/inner`, nhưng **được** kế thừa sealed class/interface (hay dùng cho UI state).
- Mới: `data object` (Kotlin 1.9) cho singleton có `toString` đẹp, hay dùng trong sealed hierarchy:
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
fun tune(id: ChannelId)       // type-safe, runtime thường chỉ là int
```
Không tốn allocation (trừ khi bị box: nullable, generic). Tên hàm có tham số value class bị **mangle** khi nhìn từ Java.

### Q9. `sealed` vs `enum`?
- Enum: mỗi hằng là **một instance duy nhất**, cùng một kiểu dữ liệu. Sealed: mỗi subclass có **state riêng**, có thể tạo nhiều instance.
- `when` trên sealed/enum là **exhaustive** (không cần `else`), nên thêm subclass mới sẽ làm compile lỗi ở các chỗ chưa xử lý. Đó chính là lợi ích.
- **⚠️ Sửa ghi chú**: subclass của sealed **không còn bắt buộc cùng file**. Từ Kotlin 1.5 chỉ cần **cùng package và cùng module**. Có thêm `sealed interface`.
- Enum: dùng `entries` (1.9) thay `values()` (`values()` tạo mảng mới mỗi lần gọi).

### Q10. `object`, `companion object`, object expression: khởi tạo khi nào?
| | Khởi tạo | Ghi chú |
|--|---------|---------|
| `object Foo` (declaration) | lazy, lần truy cập đầu | thread-safe nhờ class initializer của JVM (`static {}`) |
| `companion object` | khi class chứa nó được load | giống Java static initializer |
| `object : Listener {}` (expression) | ngay tại chỗ | tạo **instance mới** mỗi lần chạy qua; giữ tham chiếu tới biến bên ngoài (có thể gây leak như anonymous class) |

Thay cho `static` của Java: top-level function (khuyên dùng), `object`, `companion`. Muốn Java gọi như static thì thêm `@JvmStatic`.

### Q11. `@JvmStatic`, `@JvmField`, `@JvmOverloads`, `@JvmName`, `@Throws`?
- `@JvmStatic`: sinh method static thật (trong companion/object).
- `@JvmField`: lộ field public, không sinh getter/setter.
- `@JvmOverloads`: sinh các overload cho tham số default (bắt buộc khi viết **custom View**: `constructor(context, attrs = null, defStyleAttr = 0)`).
- `@JvmName`: đổi tên khi xung đột chữ ký JVM (do type erasure: `fun f(l: List<String>)` và `fun f(l: List<Int>)`).
- `@Throws(IOException::class)`: Kotlin không có checked exception, nên Java gọi sang cần annotation này mới `catch` được.

### Q12. `lateinit` vs `lazy`?
| | `lateinit var` | `val by lazy {}` |
|--|---------------|-----------------|
| Kiểu | non-null, **không** primitive | bất kỳ |
| Khi nào gán | bên ngoài (DI, `onCreate`, `@Before` test) | lần đọc đầu tiên |
| Thread-safety | không | mặc định `SYNCHRONIZED`; có `PUBLICATION`, `NONE` (dùng trên main thread cho nhanh) |
| Đọc khi chưa gán | `UninitializedPropertyAccessException` | (không xảy ra) |
| Kiểm tra | `::name.isInitialized` | `lazyValue.isInitialized()` |
Pitfall Android: `by lazy { findViewById(...) }` trong **Fragment** giữ View cũ sau khi `onDestroyView`, gây leak và trỏ vào View chết. Dùng ViewBinding và set `null` ở `onDestroyView`.

### Q13. Visibility modifier?
- `public` (mặc định), `internal` (cùng **module** Gradle/Soong), `protected`, `private` (trong class, hoặc trong **file** nếu là top-level).
- **⚠️ Sửa ghi chú**: `protected` của Kotlin **chỉ** thấy trong class và subclass, **không** thấy trong cùng file/package (khác Java, nơi `protected` thấy cả package).
- `internal` khi nhìn từ Java là `public` với tên bị mangle.

### Q14. Constructor & thứ tự khởi tạo?
Primary constructor → các property initializer và `init {}` chạy **theo thứ tự xuất hiện trong code** → thân secondary constructor. Secondary **phải** uỷ quyền về primary (`: this(...)`).
- **⚠️ Sửa ghi chú**: *"mặc định tham số constructor là val"* sai. Không ghi `val/var` thì đó chỉ là **tham số**, không phải property.
- Bẫy: gọi hàm `open` trong `init` của class cha. Lúc đó subclass chưa khởi tạo property, nên đọc được giá trị `null` dù type là non-null.

### Q15. Delegation (`by`)?
- **Class delegation** (composition thay inheritance, đúng như ghi chú "Replace inheritance with delegation"):
```kotlin
class LoggingList<T>(private val inner: MutableList<T> = mutableListOf()) : MutableList<T> by inner {
    override fun add(element: T): Boolean { Log.d("L", "add $element"); return inner.add(element) }
}
```
- **Property delegation**: `by lazy`, `Delegates.observable/vetoable`, `by map`, `by viewModels()`, `by remember { mutableStateOf() }`, `by preferences`... Tự viết bằng `operator fun getValue/setValue` hoặc `ReadWriteProperty`.

### Q16. Scope functions?
| Hàm | Đối tượng trong block | Trả về | Dùng khi |
|-----|----------------------|--------|----------|
| `let` | `it` | kết quả lambda | xử lý giá trị nullable: `x?.let { }` |
| `run` | `this` | kết quả lambda | tính toán từ object |
| `with(obj)` | `this` | kết quả lambda | gọi nhiều hàm trên một object (không phải extension) |
| `apply` | `this` | **chính object** | cấu hình object (builder) |
| `also` | `it` | **chính object** | side effect (log, validate) |
Bẫy: `x?.let { ... } ?: run { ... }` không phải if/else thật. Nếu block `let` trả về `null` thì nhánh `run` **cũng chạy**.

### Q17. Extension function được resolve thế nào?
**Tĩnh**, theo kiểu khai báo (compile-time), không theo kiểu runtime (không polymorphic). Compile thành static method nhận receiver làm tham số đầu. Nếu trùng tên với member function thì **member luôn thắng**. Extension không truy cập được `private`.

### Q18. Lambda, higher-order function, chi phí?
- Lambda **capture** biến thì mỗi lần chạy tạo object mới; không capture thì là singleton.
- **⚠️ Cập nhật ghi chú**: ghi chú viết *"mỗi lambda sinh một anonymous class"*. Từ **Kotlin 2.0**, lambda mặc định sinh bằng `invokedynamic` (`LambdaMetafactory`), không sinh class riêng. Trên Android, D8 desugar lại.
- Lambda with receiver `T.() -> Unit` là nền tảng của DSL (`apply`, `buildString`, Compose, Gradle KTS).

### Q19. `inline`, `noinline`, `crossinline`, `reified`?
- `inline`: chép thân hàm **và lambda** vào nơi gọi. Lợi ích chính là **bỏ object lambda** và cho phép **non-local return**, không phải "tiết kiệm stack".
- `noinline`: lambda đó không inline (khi cần lưu lambda lại hoặc truyền tiếp).
- `crossinline`: lambda được inline nhưng **cấm non-local return** (vì nó chạy trong context khác, ví dụ `Runnable`).
- `reified` (chỉ dùng với `inline`): giữ kiểu T lúc runtime: `inline fun <reified T> Gson.fromJson(s: String) = fromJson(s, T::class.java)`.
- Nhược điểm: phình bytecode. Hàm `public inline` không được truy cập thành viên `private`; muốn dùng thì phải đánh dấu `@PublishedApi internal`.

### Q20. Generics: `in`/`out`, star projection, type erasure
- `out T` (covariant, producer): `List<out E>`, nên `List<Int>` là subtype của `List<Number>`.
- `in T` (contravariant, consumer): `Comparable<in T>`.
- Invariant: `MutableList<T>`, `Array<T>`.
- Use-site variance: `fun copy(from: Array<out Any>, to: Array<Any>)` (giống `? extends` của Java).
- `*`: `List<*>` đọc được như `Any?`, không ghi được.
- Kiểu generic bị **erase** lúc runtime nên không viết được `is List<String>`. Giải pháp là `reified`.
- Mẹo nhớ của ghi chú vẫn đúng: *hàm (lambda) contravariant ở tham số, covariant ở kiểu trả về*: `(Number) -> Int` là subtype của `(Int) -> Number`.

### Q21. Collections vs Sequences, `List` vs `Array`, `map` vs `flatMap`
- Collection: **eager**, mỗi bước tạo một list trung gian. Sequence: **lazy**, xử lý từng phần tử qua cả chuỗi, có short-circuit (`first`, `take`). Sequence có lợi khi dữ liệu lớn **và** nhiều bước, hoặc có `take/first`. Với list nhỏ thì collection thường nhanh hơn.
- **⚠️ Sửa ghi chú**: `List` là interface **read-only**, không phải immutable, và không liên quan tới "kích thước cố định". `Array` có kích thước cố định, mutable, invariant, map sang mảng Java. `IntArray` là `int[]`, không bị box.
- `map`: 1 phần tử → 1 phần tử. `flatMap`: 1 phần tử → nhiều phần tử, rồi làm phẳng thành một list.
- `HashMap` vs `List` (ghi chú ingenico): truy cập `list[i]` là O(1), `list.contains(x)` là O(n), `map[key]` trung bình O(1) (tệ nhất O(log n) khi bucket thành tree trong Java 8+). Trên Android có `ArrayMap`/`SparseArray` tiết kiệm bộ nhớ cho map nhỏ (< ~1000 phần tử) nhưng lookup là O(log n).

### Q22. `when`, label, `Pair/Triple`
- `when` là biểu thức, nhận được `in range`, `is Type` (smart cast), không có subject. **Mới (stable từ Kotlin 2.2)**: **guard condition**:
```kotlin
when (state) {
    is UiState.Success if state.channels.isEmpty() -> showEmpty()
    is UiState.Success -> show(state.channels)
    else -> {}
}
```
- **⚠️ Sửa ghi chú**: label **không phải** "mọi biểu thức". Label là định danh `name@` gắn vào một biểu thức, dùng cho `break@`, `continue@`, `return@`. Ví dụ hay gặp nhất: `list.forEach { if (it < 0) return@forEach }`, chỉ thoát khỏi lambda.
- `Pair/Triple`: dùng tạm. Nếu API public thì nên tạo `data class` có tên field rõ nghĩa.

---

## 17.3 Coroutines & Flow (sửa và đào sâu ghi chú)

### Q23. Coroutine là gì, khác thread chỗ nào?
- Coroutine là một **tính toán có thể tạm dừng**. Compiler biến hàm `suspend` thành **state machine** + `Continuation` (CPS, xem ch.11 Q24). Khi suspend, thread được **trả lại** cho việc khác.
- Rẻ: hàng chục nghìn coroutine chỉ là object trên heap, trong khi mỗi thread tốn stack khoảng ~1MB và cần kernel scheduling.
- **⚠️ Sửa ghi chú**: *"start coroutine sẽ tạo thread mới"* sai. Coroutine chạy trên thread của **dispatcher**. `launch(Dispatchers.Main)` hoàn toàn không tạo thread.
- Concurrency (xen kẽ, có thể chỉ trên 1 core) khác parallelism (chạy cùng lúc trên nhiều core). Ghi chú đã đúng ý này.

### Q24. `launch` vs `async` vs `runBlocking` vs `coroutineScope`?
- `launch` trả `Job` (fire-and-forget); `async` trả `Deferred<T>`, lấy kết quả bằng `await()`.
- `runBlocking`: **block** thread hiện tại. Chỉ dùng trong `main()`/test, **không bao giờ** dùng trên main thread Android.
- `coroutineScope {}`: suspend cho tới khi mọi con xong; một con lỗi thì huỷ cả nhóm. `supervisorScope {}`: con lỗi không huỷ anh em.
- Chạy song song đúng cách:
```kotlin
suspend fun loadHome() = coroutineScope {
    val channels = async { repo.channels() }
    val epg      = async { repo.epg() }
    Home(channels.await(), epg.await())
}
```
Ví dụ trong ghi chú (`one + two.await()`) **compile lỗi**: `one` là `Deferred`, phải viết `one.await() + two.await()`.

### Q25. Structured concurrency, scope của Android
- Mỗi coroutine có cha; cha chỉ kết thúc khi mọi con kết thúc; huỷ cha thì huỷ toàn bộ con.
- `viewModelScope` (`SupervisorJob + Dispatchers.Main.immediate`, huỷ ở `onCleared`), `lifecycleScope` (huỷ ở `DESTROYED`), `rememberCoroutineScope()` (Compose).
- **⚠️ Sửa ghi chú**: `GlobalScope` không gắn với "vòng đời application". Nó **không có Job**, nằm ngoài structured concurrency, không ai huỷ được và dễ leak, nên bị đánh dấu `@DelicateCoroutinesApi`. Việc phải sống lâu hơn màn hình thì tạo **application scope** riêng (`CoroutineScope(SupervisorJob() + Dispatchers.Default)`, inject bằng Hilt) hoặc dùng WorkManager.

### Q26. Dispatchers?
| Dispatcher | Dùng cho | Chi tiết |
|------------|---------|----------|
| `Main` / `Main.immediate` | UI | `immediate`: nếu đang ở main thread thì chạy ngay, không post |
| `Default` | CPU (parse JSON, DiffUtil, sort) | số thread = số core (tối thiểu 2) |
| `IO` | blocking I/O | tối đa `max(64, cores)`, **dùng chung thread pool** với Default |
| `Unconfined` | gần như không dùng trong app | chạy trên thread nào đang resume |
- `Dispatchers.IO.limitedParallelism(1)`: tạo dispatcher tuần tự mà không cần thêm thread. Đây là cách thay cho `newSingleThreadContext`/`newFixedThreadPoolContext`: hai hàm này tạo thread thật, phải `close()`, và đã bị đánh dấu delicate/obsolete. **⚠️ Cập nhật ghi chú.**
- **Main-safety** (ghi chú đã đúng): repository tự `withContext(Dispatchers.IO)`, ViewModel cứ `viewModelScope.launch {}` trên Main. **Không** viết `viewModelScope.launch(Dispatchers.IO)` rồi cập nhật UI trong đó.

### Q27. Huỷ coroutine
- Huỷ là **cooperative**: các hàm suspend của kotlinx (`delay`, `withContext`, `yield`...) tự kiểm tra và ném `CancellationException`.
- **⚠️ Sửa ghi chú**: coroutine đang chạy vòng lặp tính toán **vẫn huỷ được**, nếu trong vòng lặp có kiểm tra `isActive` / `ensureActive()` / `yield()`.
- Bẫy lớn: `try { ... } catch (e: Exception) {}` hoặc `runCatching {}` **nuốt luôn `CancellationException`**, khiến coroutine tiếp tục chạy sau khi đã bị huỷ. Phải rethrow `CancellationException`.
- Dọn dẹp cần gọi suspend sau khi đã bị huỷ: `withContext(NonCancellable) { ... }` trong `finally`.
- Code blocking (socket, `Thread.sleep`) không phản ứng với cancel. Dùng `runInterruptible {}` hoặc API có timeout.

### Q28. Xử lý exception
- Exception trong coroutine con **lan lên cha** ngay lập tức (cả với `async`). Cha huỷ các con còn lại rồi ném tiếp lên trên.
- `CoroutineExceptionHandler` chỉ có tác dụng ở coroutine **gốc** (`launch` trực tiếp trên scope, hoặc con trực tiếp của `SupervisorJob`). Gắn vào coroutine con thì vô dụng.
- `async` ở gốc: exception được giữ lại cho tới `await()`.
- Trong `viewModelScope` (SupervisorJob): một `launch` bị lỗi không làm hỏng các `launch` khác, nhưng nếu không có handler thì exception vẫn tới uncaught handler và **làm crash app**.

### Q29. Flow cơ bản (đã có trong ghi chú) và các điểm sai
- Flow **lạnh**: code trong `flow {}` chỉ chạy khi có `collect`, và chạy lại cho **mỗi** collector.
- **⚠️ Sửa ghi chú**: *"Không thể cancel một flow"* sai. Flow huỷ theo coroutine đang collect nó (huỷ scope là collect dừng); `flow {}` builder còn tự kiểm tra cancel mỗi lần `emit`. Có thể dừng sớm bằng `take(n)`, `first()`, `takeWhile`.
- **Context preservation** (ghi chú đã đúng ý): **không** được `withContext` quanh `emit` trong `flow {}` vì sẽ ném `IllegalStateException: Flow invariant is violated`. Muốn đổi thread phía upstream thì dùng **`flowOn(Dispatchers.IO)`**. `flowOn` chỉ ảnh hưởng các operator **phía trên** nó.
- Operator hay hỏi: `map/filter/transform`, `flatMapLatest` (ví dụ search: chỉ giữ query mới nhất), `combine` vs `zip`, `debounce`, `distinctUntilChanged`, `buffer`/`conflate`/`collectLatest` (xử lý backpressure), `catch` (chỉ bắt lỗi upstream), `retryWhen`, `onEach`/`launchIn`.
- Biến callback API thành Flow:
```kotlin
fun DisplayManager.displayChanges(): Flow<Int> = callbackFlow {
    val l = object : DisplayManager.DisplayListener {
        override fun onDisplayChanged(id: Int) { trySend(id) }
        override fun onDisplayAdded(id: Int) {}
        override fun onDisplayRemoved(id: Int) {}
    }
    registerDisplayListener(l, null)
    awaitClose { unregisterDisplayListener(l) }   // bắt buộc, nếu không sẽ leak listener
}
```

### Q30. `StateFlow`, `SharedFlow`, `Channel`: chọn cái nào?
| | StateFlow | SharedFlow | Channel |
|--|-----------|------------|---------|
| Nóng/lạnh | nóng | nóng | nóng |
| Giá trị hiện tại | luôn có `.value` | tuỳ `replay` | không |
| Nhiều collector | broadcast | broadcast | **mỗi phần tử chỉ một receiver nhận** |
| Trùng giá trị | bỏ qua (conflate + `equals`) | phát hết | phát hết |
| Dùng cho | UI state | event cho nhiều người nghe | event một lần (navigate, toast), pipeline giữa các coroutine |
- **⚠️ Sửa ghi chú**: Channel dùng để giao tiếp **giữa các coroutine trong cùng process**, **không** phải giữa các process. IPC dùng Binder/AIDL/Messenger/ContentProvider.
- Pattern chuẩn trong ViewModel:
```kotlin
val uiState: StateFlow<UiState> = repo.channelsFlow()
    .map<List<Channel>, UiState> { UiState.Success(it) }
    .catch { emit(UiState.Error(it.message.orEmpty())) }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UiState.Loading)
```
  `WhileSubscribed(5000)`: dừng upstream sau 5s không còn collector. Đủ để vượt qua lúc xoay màn hình / recreate mà không phải query lại.
- Collect an toàn trong UI: `repeatOnLifecycle(STARTED)` (View) hoặc `collectAsStateWithLifecycle()` (Compose).

### Q31. Test coroutine?
`runTest` (bỏ qua `delay` bằng virtual time), `StandardTestDispatcher`/`UnconfinedTestDispatcher`, `Dispatchers.setMain(testDispatcher)`, inject dispatcher qua constructor (không hard-code `Dispatchers.IO`), thư viện **Turbine** để test Flow.

---

## 17.4 Android component (sửa ghi chú "ingenico")

### Q32. Process priority (ghi chú đã có 4 mức)
Bổ sung: hệ thống tính bằng `oom_score_adj` (xem bảng ch.12 Q9). Service process đã chạy lâu (> 30 phút) bị hạ xuống `SERVICE_B_ADJ` (ứng với `MAX_SERVICE_INACTIVITY = 30 phút` trong `ActivityManagerConstants.java:181`). A14 thêm **freezer**: process cached bị đóng băng, không được chạy CPU.

### Q33. IPC: AIDL, Messenger, Broadcast, ContentProvider
- **⚠️ Bổ sung ghi chú**:
  - **Messenger** cũng chạy trên Binder (`IMessenger.aidl`), nhưng mọi message đi vào **một Handler**, nên được xử lý tuần tự trên một thread. Dễ viết, không phải lo thread-safety.
  - **AIDL**: các lời gọi tới từ **binder thread pool** (tối đa 15 thread + main), nên phải tự đồng bộ. Mặc định là đồng bộ; khai `oneway` thì không chờ (chi tiết ở ch.04).
  - **ContentProvider**: chia sẻ dữ liệu dạng bảng/file, có permission theo URI. **Broadcast**: một chiều, nhiều người nhận.
  - Dữ liệu lớn: `ParcelFileDescriptor`/`SharedMemory`, vì giới hạn buffer binder ~1MB.

### Q34. Handler / Looper / MessageQueue
- **⚠️ Sửa ghi chú**: *"Thread được tạo ra bởi Handler, Looper, Runnable"* là sai quan hệ. Thread là thread của OS. Một thread **có thể có** một `Looper` (vòng lặp đọc `MessageQueue`). `Handler` gắn với một Looper, dùng để **gửi** `Message`/`Runnable` vào queue đó, và các message được xử lý trên thread của Looper. Chi tiết native (epoll/eventfd) ở ch.11 Q20 và ch.14 Q14.
- `HandlerThread` = Thread + Looper sẵn sàng. Từ API 30, `Handler()` không truyền Looper bị deprecated (bắt buộc chỉ rõ Looper).

### Q35. AsyncTask, IntentService, JobIntentService
- **⚠️ Cập nhật ghi chú**: `AsyncTask` deprecated từ API 30. `IntentService` deprecated từ API 30, và **`JobIntentService` cũng deprecated**. Thay thế:
  - việc nền đảm bảo hoàn thành → **WorkManager**;
  - việc gắn với UI → **coroutines**;
  - việc user thấy được → **FGS** (đúng type) hoặc **User-Initiated Data Transfer job** (A14).
- Lý do AsyncTask leak (ghi chú đã đúng): inner class giữ tham chiếu Activity, và AsyncTask không gắn với lifecycle.

### Q36. Service: các ý sai trong ghi chú
- **⚠️ Sửa ghi chú**:
  - *"FGS không bị kill khi user kill app"*: user **force stop** (hoặc vuốt app trên một số OEM) thì FGS vẫn bị dừng. FGS chỉ được hệ thống ưu tiên giữ (adj `PERCEPTIBLE_APP_ADJ = 200`).
  - *"Bắt đầu FGS khi app ở foreground thì system tạo background service"*: sai. `startForegroundService()` tạo service bình thường, app **phải** gọi `startForeground()` trong 30s (A14).
  - Bound service bị destroy khi **mọi client unbind và service không ở trạng thái started**.
- Luật FGS mới:
  - **A14**: bắt buộc `foregroundServiceType` + permission theo type (ch.11 Q10).
  - **A15**: `dataSync` và `mediaProcessing` (type mới) tối đa **6 giờ / 24 giờ**; hết thời gian hệ thống gọi `Service.onTimeout(int, int)`, app phải `stopSelf()` trong vài giây, nếu không bị coi là lỗi (ANR/crash). Receiver `BOOT_COMPLETED` **không được** start FGS loại `dataSync`, `camera`, `mediaPlayback`, `phoneCall`, `mediaProjection`, `microphone`.
  - Với **app hệ thống STB** (persistent/privileged, ví dụ `DavidExtension`) phần lớn hạn chế được miễn, nhưng cần biết để review app bên thứ ba.

### Q37. Parcelable vs Serializable
- Serializable: dùng reflection, tạo nhiều object tạm, chậm. Parcelable: code ghi/đọc tường minh, nhanh. Trong Kotlin dùng **`@Parcelize`** (plugin `kotlin-parcelize`), không phải viết tay.
- Bẫy: Bundle/Intent đều đi qua binder (giới hạn ~1MB dùng chung), gây `TransactionTooLargeException`. Từ A13 dùng `getParcelableExtra(key, Clazz::class.java)` (bản cũ không truyền class đã deprecated, và an toàn hơn trước tấn công kiểu "Parcel mismatch").

### Q38. Stack vs Heap trên Android
- **⚠️ Sửa ghi chú**: các tuỳ chọn `-Xss/-Xms/-Xmx` là của JVM desktop. Trên ART:
  - heap limit theo `dalvik.vm.heapgrowthlimit` (app thường) và `dalvik.vm.heapsize` (app có `android:largeHeap="true"`);
  - xem bằng `ActivityManager.getMemoryClass()/getLargeMemoryClass()`;
  - stack của main thread do kernel cấp (~8MB); thread Java khác mặc định ~1MB.
- ART dùng **Concurrent Copying / Generational CC GC**, gần như không còn pause dài. Bitmap pixel nằm ở **native heap** từ A8.
- Đệ quy sâu gây `StackOverflowError`; hết heap gây `OutOfMemoryError`. Ghi chú đã đúng phần này.

### Q39. Pass-by-value hay reference?
**⚠️ Làm rõ ghi chú**: Java/Kotlin **luôn truyền theo giá trị**. Với object, "giá trị" được truyền là **bản sao của tham chiếu**. Vì vậy sửa property của object thì bên ngoài thấy, còn gán `param = Other()` thì bên ngoài không đổi.

### Q40. Đồng bộ đa luồng (bổ sung `synchronized`)
`synchronized`/`@Synchronized`, `ReentrantLock`, `@Volatile` (chỉ đảm bảo visibility, không atomic), `AtomicInteger`, `ConcurrentHashMap`. Trong coroutines: `Mutex` (không block thread), `limitedParallelism(1)`, actor/Channel. **Không** dùng `synchronized` bao quanh lời gọi hàm `suspend`.
- Deadlock (ghi chú định nghĩa theo database): ở đây là 2 thread giữ 2 lock theo thứ tự ngược nhau. Phòng tránh bằng thứ tự lock cố định và không gọi binder/callback lạ khi đang giữ lock (đúng như bài học `LedManagerService` ở ch.06).

---

## 17.5 Kiến trúc & Design pattern (sửa ghi chú "ingenico")

### Q41. MVC / MVP / MVVM / MVI
| | Ai giữ logic | View ↔ logic | Ghi chú |
|--|-------------|--------------|---------|
| MVC (Android cũ) | Activity vừa là View vừa là Controller | chặt | khó test |
| MVP | Presenter | qua interface, **hai chiều** | Presenter giữ tham chiếu View (phải detach để tránh leak), không tự sống qua config change |
| MVVM | ViewModel | View **observe** state; ViewModel không biết View | ViewModel sống qua config change |
| MVI | reducer/ViewModel | **một luồng một chiều**: Intent → (reducer) → **một State bất biến** → render | dễ debug và test, state rõ ràng; đổi lại phải tạo object state mới mỗi lần |
- **⚠️ Sửa ghi chú**: *"MVVM là 1-n, nhiều view dùng chung ViewModel"* không phải đặc trưng của MVVM. Thường là 1 màn hình ↔ 1 ViewModel; dùng chung chỉ là trường hợp đặc biệt (`activityViewModels()`, navigation-graph-scoped ViewModel).
- Kiến trúc Google khuyến nghị hiện nay: **UI layer** (UI state + UDF, ViewModel) → **Domain layer** (UseCase, tuỳ chọn) → **Data layer** (Repository, **single source of truth**, offline-first: Room là nguồn chính, network chỉ để đồng bộ). Thêm DI bằng Hilt và modularization (`:feature:*`, `:core:*`).

### Q42. Design pattern: các chỗ nhầm trong ghi chú
- **⚠️ Command ≠ Observer**: Command **đóng gói một yêu cầu thành object** (có thể xếp hàng, undo, log), ví dụ `Runnable` post vào Handler, `WorkRequest`, `ClientTransaction` trong ATMS (ch.12 Q5). Observer mới là publisher/subscriber (`LiveData`, `Flow`, `RemoteCallbackList`).
- **Strategy vs State**: Strategy do **bên ngoài** chọn thuật toán (ví dụ `Comparator`, `Interpolator`). State do **object tự đổi** hành vi theo trạng thái nội bộ (ví dụ `PlayerState`). Ghi chú đã đúng ý này.
- **Singleton**: hàm `getInstance` trong ghi chú **không thread-safe** (2 thread cùng thấy `null` thì tạo 2 instance). Sửa:
```kotlin
companion object {
    @Volatile private var instance: AQIRepository? = null
    fun getInstance(remote: AQIDataSource) =
        instance ?: synchronized(this) { instance ?: AQIRepository(remote).also { instance = it } }
}
```
  Hoặc dùng `object`, hoặc tốt nhất là `@Singleton` trong Hilt (dễ test, dễ thay bằng fake).
- **Builder** trong Kotlin: thường không cần vì đã có named argument + default value + `apply`. Vẫn gặp ở API Java (`AlertDialog.Builder`, `NotificationCompat.Builder`).
- **Facade**: ví dụ Retrofit là facade + **dynamic proxy** (`Proxy.newProxyInstance` sinh implementation cho interface). `XxxManager` trong framework là facade che binder (ch.06).
- **Adapter**: `RecyclerView.Adapter`, `ItemBridgeAdapter` của Leanback.
- **Decorator**: `ContextWrapper` (ch.13 Q10), `InputStream` lồng nhau.
- Trong AOSP còn có: **Proxy/Stub** (Binder), **Template method** (lifecycle `Activity`), **Chain of responsibility** (`InputStage` của `ViewRootImpl`, `dispatchKeyEvent`), **Object pool** (`Message.obtain()`).

### Q43. Interface vs abstract class
**⚠️ Cập nhật ghi chú**: *"function trong interface không có thân hàm"* chỉ đúng với Java ≤ 7. Interface của Kotlin (và Java 8+ `default`) **được có thân hàm** và property abstract hoặc có getter, nhưng **không giữ state** (không có backing field) và **không có constructor**. Abstract class có state và constructor nhưng chỉ kế thừa được một. Interop với Java: trước đây hàm có thân trong interface Kotlin được compile thành lớp `DefaultImpls` (Java không thấy như `default` method), phải bật `-Xjvm-default=all`. Kotlin 2.2 đổi mặc định sang sinh Java `default` method (tuỳ chọn mới `-jvm-default`).

---

## 17.6 Mới từ 2023 tới 11/2026 (hay bị hỏi "bạn có cập nhật không?")

### Q44. Kotlin 1.9 → 2.x có gì?
| Phiên bản | Điểm chính |
|-----------|-----------|
| 1.9 (2023) | `data object`, `enum.entries`, toán tử `..<`; Kotlin Multiplatform **Stable** (11/2023) |
| **2.0** (05/2024) | **compiler K2** stable (build nhanh hơn, smart cast thông minh hơn: smart cast qua biến local, qua `||`, trong lambda), **Compose compiler chuyển vào repo Kotlin** (plugin `org.jetbrains.kotlin.plugin.compose`), lambda sinh bằng `invokedynamic` |
| 2.0.20 | Compose **strong skipping** bật mặc định |
| 2.1 (11/2024) | preview: guard condition trong `when`, non-local `break/continue` trong lambda inline, multi-dollar interpolation `$$"""{"id": "$$id"}"""` |
| 2.2 (06/2025) | ba tính năng trên **stable**; **context parameters** (preview, thay context receivers) |
- Hệ sinh thái: **KSP2** thay kapt (kapt đang ở chế độ bảo trì; Room/Hilt/Moshi đều có KSP); `kotlinx.serialization` thay Gson (Gson bỏ qua default value và null-safety của Kotlin); **Compose Multiplatform cho iOS stable** (2025).

### Q45. Compose: cập nhật
- Strong skipping mặc định nên lambda và tham số unstable vẫn skip được nếu `===`. Chủ đề "stability" (ch.11 Q37) ít đau đầu hơn, nhưng vẫn phải biết.
- `Modifier.Node` thay `composed {}` (hiệu năng tốt hơn); `collectAsStateWithLifecycle`, `LifecycleStartEffect/ResumeEffect`; `LazyLayout` + prefetch.
- Navigation: **type-safe routes** (Navigation 2.8, route là `@Serializable` class); **Navigation 3** (ra mắt 2025): back stack là một list state do app tự quản lý.
- **Compose for TV** (`androidx.tv:tv-material`) là hướng Google khuyến nghị cho app TV mới, thay Leanback. Leanback vẫn chạy nhưng không phát triển tính năng mới. Với STB như project này: launcher/app OEM viết mới nên cân nhắc Compose for TV, nhưng phải đo **cold start và RAM** trên SoC yếu, và dùng Baseline Profile.

### Q46. Android 15 (API 35) thay đổi gì cho app?
- **Edge-to-edge bắt buộc** khi target 35 (`enableEdgeToEdge()`, xử lý `WindowInsets`).
- FGS: timeout 6h cho `dataSync`/`mediaProcessing` + `onTimeout()`; hạn chế start FGS từ `BOOT_COMPLETED` (Q36).
- BAL: phía **creator** `PendingIntent` mặc định không cấp quyền BAL nữa (ch.13 Q17).
- **16 KB page size**: thiết bị mới có thể dùng page 16KB, nên `.so` phải được align 16KB (NDK r27+/AGP 8.5.1+). Google Play yêu cầu từ 11/2025 với app target 35+ có native code.
- Private space, predictive back animation (opt-in), `ApplicationStartInfo` (biết vì sao process được start: cold/warm, do broadcast hay activity).

### Q47. Android 16 (API 36, phát hành 06/2025) thay đổi gì?
- Trên màn hình lớn (sw ≥ 600dp), app target 36 bị **bỏ qua** các khai báo `screenOrientation`, `resizeableActivity=false`, giới hạn aspect ratio → bắt buộc làm adaptive layout. (Có opt-out tạm thời cho target 36.)
- **Edge-to-edge không opt-out được nữa** (bỏ `windowOptOutEdgeToEdgeEnforcement`).
- **Predictive back** mặc định với target 36: `onBackPressed()` không còn được gọi và `KEYCODE_BACK` không còn được dispatch. Phải dùng `OnBackPressedCallback`/`OnBackInvokedCallback`. Với **TV** đây là điểm cần chú ý: remote có phím BACK, app nào còn bắt `KEYCODE_BACK` trong `onKeyDown` sẽ bị lỗi điều hướng khi nâng target.
- Notification dạng **Live Updates / progress-centric**; JobScheduler điều chỉnh quota (job start khi app có FGS cũng bị tính quota); thêm các bảo vệ chống intent redirection.
- Lịch phát hành mới: Google ra **bản SDK chính vào Q2** và một **bản SDK phụ vào Q4** mỗi năm (Android 16 QPR2 mang minor SDK). Câu hỏi "API level nào?" giờ có thể có dạng `36.1`.
- **Android 17 (2026)**: tài liệu này **không liệt kê** chi tiết vì chưa kiểm chứng được. Trước khi phỏng vấn nên đọc trang *Behavior changes: apps targeting Android 17*.

### Q48. Công cụ/thực hành build & hiệu năng hiện nay
- Gradle **version catalog** (`libs.versions.toml`), convention plugins, Kotlin DSL; R8 full mode; **Baseline Profile + Startup Profile** (Macrobenchmark) giảm cold start 20–40%. Rất đáng làm cho launcher STB, vì `DavidLauncher` hiện còn tắt dexpreopt.
- Bảo mật: Credential Manager (passkey), Photo Picker thay quyền đọc media, Play Integrity API.
- AI on-device: Gemini Nano qua AICore / ML Kit GenAI API (trên một số thiết bị). Chỉ cần biết khái niệm.

---

## 17.7 Rapid-fire Kotlin (≤ 30 giây/câu)

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | `Unit` vs `Nothing` vs `Any`? | `Unit` có đúng 1 giá trị (như `void`); `Nothing` không có giá trị nào (hàm luôn ném lỗi/không return, là subtype của mọi kiểu); `Any` là gốc của mọi kiểu non-null |
| 2 | `typealias` có tạo kiểu mới không? | Không, chỉ là tên khác. Muốn kiểu mới type-safe thì dùng `value class` |
| 3 | `object` trong Kotlin có thread-safe không? | Có, phần khởi tạo dựa vào class init của JVM/ART |
| 4 | `lazy` mặc định thread mode? | `SYNCHRONIZED` |
| 5 | `infix` cần điều kiện gì? | member/extension, đúng 1 tham số, không vararg/default |
| 6 | `operator` ví dụ? | `plus`, `get/set` (`[]`), `invoke`, `contains` (`in`), `compareTo`, `getValue/setValue` (delegate), `componentN` |
| 7 | `suspend` gọi được từ đâu? | Từ hàm `suspend` khác hoặc từ coroutine builder |
| 8 | `withContext` vs `async{}.await()`? | Cùng kết quả, nhưng `withContext` gọn hơn và không tạo `Deferred` |
| 9 | `flowOn` ảnh hưởng phần nào? | Operator **phía trên** nó (upstream) |
| 10 | `SharedFlow(replay=0)` emit khi không có ai collect? | Giá trị bị mất (với `tryEmit`), hoặc `emit` suspend nếu có buffer + collector chậm |
| 11 | `Dispatchers.Main` khi unit test JVM? | Không có, phải `Dispatchers.setMain(testDispatcher)` |
| 12 | `@Parcelize` cần gì? | plugin `kotlin-parcelize`, class implement `Parcelable`, property trong primary constructor |
| 13 | `sealed interface` có lợi gì hơn `sealed class`? | Một class có thể implement nhiều sealed interface; enum cũng implement được |
| 14 | `const` dùng được trong class thường không? | Không, chỉ ở top-level, `object`, `companion` |
| 15 | `!!` ném exception gì (Kotlin 1.4+)? | `java.lang.NullPointerException` |
