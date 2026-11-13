# Chương 11 — Tầng App (Android 14, Android TV/STB)

> Nguồn: thảo luận Claude ↔ Gemini, **đã kiểm chứng với source A14 trong project**. Các chỗ Gemini viết sai đã được sửa và đánh dấu **⚠️ Đã sửa**. Bản gốc của Gemini nằm ở `gemini_raw/01_gemini_app_layer.md` (dùng để đọc thêm, nhớ đối chiếu với chương 16).

---

## 11.1 Activity lifecycle, Configuration change, Process death

### Q1. Configuration change khác process death thế nào? State được khôi phục ra sao?

| | Configuration change | Process death (LMK kill khi ở background) |
|--|---------------------|-------------------------------------------|
| `ActivityRecord` trong system_server | còn | còn (giữ `Bundle` icicle) |
| Process | còn | bị kill |
| Object Java (ViewModel, field) | **ViewModel còn** (qua `NonConfigurationInstances`) | mất hết |
| Bundle `onSaveInstanceState` | có | có (đã gửi lên ATMS qua `activityStopped()`) |
| `SavedStateHandle` | có | có |

- Config change: `ActivityThread.handleRelaunchActivity` → `performDestroyActivity` giữ `ActivityClientRecord.lastNonConfigurationInstances` (**chỉ nằm trong RAM của process app**, không gửi lên system_server) → Activity mới `attach()` nhận lại → `ComponentActivity.getViewModelStore()` lấy lại store cũ.
- Process death: Bundle đã gửi lên system_server lúc `onStop` (`ActivityClientController.activityStopped`) → khi user quay lại, ATMS start process mới và truyền Bundle vào `onCreate(savedInstanceState)`.

```kotlin
class ChannelViewModel(private val state: SavedStateHandle) : ViewModel() {
    val channelId: StateFlow<String> = state.getStateFlow("ch", "1")   // sống qua cả process death
    fun tune(id: String) { state["ch"] = id }
}
```
**Pitfall**: nhét list EPG/Bitmap vào Bundle → `TransactionTooLargeException` (buffer binder ~1MB **dùng chung cho cả process**).

### Q2. STB đổi độ phân giải HDMI (1080p ↔ 4K, 50 ↔ 60Hz) thì Activity bị gì?

- DisplayManager nhận mode mới → `Configuration` đổi (`screenWidthDp`, `screenHeightDp`, `smallestScreenWidthDp`, `densityDpi`, đôi khi `colorMode`) → `ActivityRecord.ensureActivityConfiguration()` → nếu Activity không khai báo xử lý các thay đổi đó thì **relaunch**.
- Player toàn màn hình nên khai báo:
```xml
<activity android:name=".PlayerActivity"
    android:configChanges="screenSize|smallestScreenSize|screenLayout|density|orientation|keyboard|keyboardHidden|navigation|uiMode" />
```
- **Pitfall**: chỉ khai `screenSize` mà quên `smallestScreenSize`/`screenLayout` → vẫn bị recreate → MediaCodec bị release/tạo lại → màn hình đen 1–3s.
- Trong `onDestroy()` phân biệt `isFinishing` (giải phóng thật) và `isChangingConfigurations` (chỉ tách Surface, giữ player).

### Q3. SavedStateRegistry hoạt động thế nào?

- `ComponentActivity` là `SavedStateRegistryOwner`; mỗi component đăng ký `SavedStateProvider` với 1 key. Lúc `onSaveInstanceState`, registry gọi `saveState()` của từng provider, gộp vào Bundle. Lúc restore, component lấy lại bằng `consumeRestoredStateForKey(key)`.
- Đăng ký trùng key → `IllegalArgumentException`.

---

## 11.2 Launch mode, Task, Intent flags

### Q4. `standard`, `singleTop`, `singleTask`, `singleInstance`, `singleInstancePerTask`?

| launchMode | Hành vi |
|-----------|---------|
| `standard` | luôn tạo instance mới trong task của caller |
| `singleTop` | nếu đang ở đỉnh task → `onNewIntent()`, không tạo mới |
| `singleTask` | 1 instance trong task theo `taskAffinity`; nếu đã có → đưa task lên, **pop mọi activity phía trên**, `onNewIntent()` |
| `singleInstance` | như singleTask nhưng task **chỉ chứa nó**; activity nó mở sẽ sang task khác |
| `singleInstancePerTask` (A12+) | phải là root của task; có thể có nhiều instance nếu dùng `FLAG_ACTIVITY_MULTIPLE_TASK`/`NEW_DOCUMENT` |

Xử lý trong `ActivityStarter` (`frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java`).

### Q5. `NEW_TASK | CLEAR_TOP` vs `CLEAR_TASK`?

- `CLEAR_TOP`: finish mọi activity phía trên target; nếu target là `standard` và không có `FLAG_ACTIVITY_SINGLE_TOP` → **chính target cũng bị finish và tạo lại**. Thêm `SINGLE_TOP` → giữ instance, gọi `onNewIntent`.
- `CLEAR_TASK` (phải đi cùng `NEW_TASK`): xoá sạch task rồi tạo activity mới làm root.
```kotlin
startActivity(Intent(this, HomeActivity::class.java).addFlags(
    Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP or Intent.FLAG_ACTIVITY_SINGLE_TOP))
```

### Q6. Task hijacking (StrandHogg) và `taskAffinity`?

- App độc hại khai `taskAffinity` trùng package nạn nhân (+ `allowTaskReparenting`) → activity giả chen vào task của nạn nhân.
- Phòng: `taskAffinity=""` cho màn hình nhạy cảm, `exported=false`, kiểm tra caller. Google đã vá dần từ A11–A12 (hạn chế reparent/launch vào task app khác).
- **⚠️ Đã sửa**: Gemini khẳng định "A14 cấm hẳn gộp task giữa các UID" — không có căn cứ rõ ràng trong source, đừng nói chắc như vậy khi phỏng vấn.

### Q7. Mở Detail Screen từ Voice Search/EPG, bấm Back phải về màn hình list của app?

Dùng `TaskStackBuilder` + `android:parentActivityName` để dựng back stack nhân tạo:
```kotlin
val pi = TaskStackBuilder.create(ctx)
    .addNextIntentWithParentStack(Intent(ctx, DetailActivity::class.java).putExtra("id", id))
    .getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
```
A12+ bắt buộc chỉ rõ `FLAG_IMMUTABLE`/`FLAG_MUTABLE` cho PendingIntent.

---

## 11.3 Service & Foreground Service (A14)

### Q8. Service chạy trên thread nào? Tại sao vẫn ANR dù đã dùng worker thread?

- `onCreate/onStartCommand/onBind` chạy trên **main thread**. AMS đặt timer khi gửi lệnh (`ActiveServices.bumpServiceExecutingLocked` → `scheduleServiceTimeoutLocked`), client xong thì gọi `serviceDoneExecuting` để huỷ.
- Timeout: **20s** (process foreground) / **200s** (background) — `ActivityManagerConstants.java:261-266` (`DEFAULT_SERVICE_TIMEOUT = 20s × Build.HW_TIMEOUT_MULTIPLIER`, background ×10; multiplier > 1 trên emulator/thiết bị chậm).
- Nếu main thread bận (lock, I/O) trước khi return `onStartCommand` → ANR dù việc nặng nằm ở thread khác.

### Q9. Started + Bound: khi nào Service bị destroy?

`ServiceRecord`: `startRequested` + `bindings` (`IntentBindRecord` → `AppBindRecord` → `ConnectionRecord`). Chỉ destroy khi **đã stop** (`stopSelf/stopService`) **và** không còn binding nào có `BIND_AUTO_CREATE`.

### Q10. Foreground Service trên A14 phải đáp ứng gì?

1. `targetSdk 34`: **bắt buộc** khai báo `android:foregroundServiceType` trong manifest.
2. Xin permission theo type: `FOREGROUND_SERVICE_MEDIA_PLAYBACK`, `..._DATA_SYNC`, `..._CONNECTED_DEVICE`, `..._SPECIAL_USE`, `..._SYSTEM_EXEMPTED`...
3. Type truyền vào `startForeground(id, notif, type)` phải ⊆ type trong manifest.
4. Sau `startForegroundService()` phải gọi `startForeground()` trong **30s trên A14** (`DEFAULT_SERVICE_START_FOREGROUND_TIMEOUT_MS = 30 * 1000`).

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK"/>
<service android:name=".PlaybackService" android:foregroundServiceType="mediaPlayback" android:exported="false"/>
```
- Manifest không khai type mà start FGS → `MissingForegroundServiceTypeException`; type không hợp lệ/thiếu permission → `InvalidForegroundServiceTypeException`/`SecurityException` (xem `ActiveServices.java`, quanh dòng 2560).
- **⚠️ Đã sửa (2 lỗi của Gemini)**:
  - `startForeground(id, notif)` **không** truyền type vẫn hợp lệ trên A14 (dùng `FOREGROUND_SERVICE_TYPE_MANIFEST` = toàn bộ type trong manifest); chỉ lỗi khi manifest không khai type.
  - Giới hạn **6 giờ cho `dataSync`** là của **Android 15**, không có trong A14.

### Q11. Start service từ background (ví dụ nhận gói multicast cảnh báo khẩn cấp)?

- A8+: `startService()` từ background → `IllegalStateException` (app thường); A12+: `startForegroundService()` từ background → `ForegroundServiceStartNotAllowedException`, trừ các trường hợp được miễn (có `SYSTEM_ALERT_WINDOW`, FCM high-priority, `LOCKED_BOOT_COMPLETED`/`BOOT_COMPLETED`, companion device, app privileged/persistent...).
- Trên STB, app OEM thường là **privileged/persistent** (như `DavidExtension`), nên được miễn phần lớn hạn chế. App thường: dùng JobScheduler/WorkManager expedited, hoặc notification.
- **⚠️ Lưu ý**: `fullScreenIntent` trên A14 cần permission `USE_FULL_SCREEN_INTENT` và chỉ được cấp mặc định cho app gọi điện/báo thức. Gemini không nhắc điều này.

---

## 11.4 BroadcastReceiver

### Q12. A14: `RECEIVER_EXPORTED` / `RECEIVER_NOT_EXPORTED`?

- targetSdk 34: `registerReceiver()` cho broadcast **không phải broadcast hệ thống** phải truyền 1 trong 2 cờ, nếu không → `SecurityException`.
```kotlin
ContextCompat.registerReceiver(ctx, receiver, IntentFilter("com.david.ACTION_X"), ContextCompat.RECEIVER_NOT_EXPORTED)
```

### Q13. Ordered vs normal broadcast, `abortBroadcast()`?

- Normal: phát song song; ordered: tuần tự theo `priority`, receiver có thể sửa result hoặc `abortBroadcast()`.
- Gọi `abortBroadcast()` trong broadcast thường: framework **log lỗi** "BroadcastReceiver trying to return result during a non-ordered broadcast" (tạo RuntimeException để in stack), **không crash app**. **⚠️ Đã sửa**: Gemini nói là "ném exception".
- A14 dùng `BroadcastQueueModernImpl` (mặc định bật: `BroadcastConstants.DEFAULT_MODERN_QUEUE_ENABLED = true`).

### Q14. Sticky broadcast?

- Deprecated từ API 21: không có bảo vệ (ai cũng gửi/sửa được), dữ liệu treo trong system_server. Vẫn dùng nội bộ cho `ACTION_BATTERY_CHANGED`, ... (`registerReceiver(null, filter)` lấy giá trị cuối).

### Q15. Implicit broadcast ban (A8)?

- Receiver khai báo trong manifest **không nhận** implicit broadcast (trừ danh sách miễn trừ) → phải đăng ký động hoặc gửi explicit (`setPackage`/`setComponent`).
- **⚠️ Đã sửa**: danh sách miễn trừ **không** nằm ở `BroadcastConstants.java`. Đúng là: các action được gắn `FLAG_RECEIVER_INCLUDE_BACKGROUND` trong code platform, cộng với danh sách `allow-implicit-broadcast` trong sysconfig XML (`SystemConfig.getAllowImplicitBroadcasts()` → `ActivityManagerService.mBackgroundLaunchBroadcasts`, `ActivityManagerService.java:2602`).
```xml
<!-- /system/etc/sysconfig/xxx.xml : cho phép static receiver nhận broadcast OEM -->
<config><allow-implicit-broadcast action="com.david.action.FW_UPDATE"/></config>
```

---

## 11.5 ContentProvider

### Q16. Tại sao `ContentProvider.onCreate()` chạy trước `Application.onCreate()`?

`ActivityThread.handleBindApplication()`: `makeApplicationInner()` (gọi `attachBaseContext`) → `installContentProviders()` (gọi `onCreate` của mọi provider) → `publishContentProviders` lên AMS → `mInstrumentation.callApplicationOnCreate()`. Thư viện (App Startup, WorkManager, Firebase) lợi dụng điều này để tự khởi tạo → làm **chậm cold start** trên STB.

### Q17. CursorWindow & giới hạn kích thước?

- Kết quả `query()` đi qua **shared memory** (ashmem) trong `CursorWindow`, chỉ truyền fd qua binder.
- **⚠️ Đã sửa**: Gemini nói 4MB. Trên A14 trong project là **2MB**: `core/res/res/values/config.xml: config_cursorWindowSize = 2048` (KB). Row quá lớn → `SQLiteBlobTooBigException`/`CursorWindowAllocationException`.
- Giải pháp: phân trang (`QUERY_ARG_LIMIT/OFFSET`), không lưu blob/ảnh trong DB, trả file qua `openFile()`/`openPipeHelper()`.

### Q18. FileProvider, `FLAG_GRANT_READ_URI_PERMISSION`?

- `file://` ra ngoài app → `FileUriExposedException` (A7+). Dùng `content://` từ FileProvider; quyền tạm cấp qua `UriGrantsManagerService`.
- **Pitfall**: `<root-path path="."/>` làm lộ toàn bộ storage.

### Q19. Truyền file 100MB giữa 2 app?

Không truyền `byte[]` qua binder. Dùng `ParcelFileDescriptor` (`openFile()`, `openPipeHelper()`, `ParcelFileDescriptor.createPipe()`), hoặc `SharedMemory`.

---

## 11.6 Handler / Looper / MessageQueue

### Q20. Cấu trúc MessageQueue & cách ngủ/thức?

- Danh sách liên kết đơn sắp theo `Message.when` (`uptimeMillis`).
- `next()` → `nativePollOnce(ptr, timeout)` → `Looper::pollOnce` → `epoll_wait`; thức dậy khi `enqueueMessage` gọi `nativeWake` → ghi vào `eventfd` (`system/core/libutils/Looper.cpp`).
- Main thread không được `quit()` → `IllegalStateException("Main thread not allowed to quit.")`.

### Q21. Sync barrier là gì? `ViewRootImpl` dùng thế nào?

- Barrier = `Message` có `target == null`. Khi gặp barrier, `next()` **bỏ qua message đồng bộ**, chỉ lấy message `isAsynchronous()`.
- `ViewRootImpl.scheduleTraversals()` gọi `postSyncBarrier()` (`ViewRootImpl.java:2439`) rồi đăng ký callback Choreographer (async) → frame được ưu tiên; `unscheduleTraversals/doTraversal` gọi `removeSyncBarrier()`.
- `postSyncBarrier()` không có trong SDK public (`@TestApi` + `@UnsupportedAppUsage`, `MessageQueue.java:470-472`). Nếu barrier bị "rò" (không remove) thì main thread không xử lý input/lifecycle → app treo.

### Q22. IdleHandler?

`queueIdle()` được gọi khi queue rỗng hoặc message kế tiếp chưa đến giờ; trả `false` để chỉ chạy 1 lần. Dùng để lazy-init sau khi frame đầu đã vẽ. Không làm việc nặng trong đó.

### Q23. Handler leak?

Chuỗi GC root: `Looper (static sMainLooper)` → `MessageQueue` → `Message.target (Handler)` → `Activity` (inner class). Giải pháp: static Handler + `WeakReference` **và** `removeCallbacksAndMessages(null)` trong `onDestroy`.

---

## 11.7 Kotlin Coroutines & Flow

### Q24. `suspend` biên dịch thành gì?
CPS: thêm tham số `Continuation`, trả về `Any?` (giá trị hoặc `COROUTINE_SUSPENDED`), thân hàm thành **state machine** `label` + biến được "spill" vào object continuation. Không dùng `Thread.sleep` trong coroutine (block thread pool).

### Q25. `Job` vs `SupervisorJob`, tại sao `scope.launch` với SupervisorJob vẫn crash?
- Job: con lỗi → huỷ cha và các anh em. SupervisorJob: lỗi không lan sang anh em.
- Nhưng exception chưa được xử lý trong `launch` vẫn đi tới `CoroutineExceptionHandler`/uncaught handler của thread → **crash** nếu không có handler. `async` thì giữ exception đến khi `await()`.
- `withContext(SupervisorJob())` không có tác dụng như mong muốn (vỡ structured concurrency).

### Q26. Dispatchers?
- `Default` = số core (tối thiểu 2), `IO` = `max(64, cores)` và **dùng chung pool** `CoroutineScheduler` với Default nên chuyển qua lại thường không tốn context switch.
- `Main.immediate`: nếu đang ở main thread thì chạy ngay, không post.

### Q27. StateFlow vs SharedFlow?
- `StateFlow` ≈ `SharedFlow(replay=1, onBufferOverflow=DROP_OLDEST)` + `distinctUntilChanged` + luôn có giá trị đầu.
- Event một lần (navigate/toast) dùng `SharedFlow(replay=0)` hoặc `Channel`, không dùng StateFlow (bị phát lại sau recreate).

---

## 11.8 Jetpack

### Q28. ViewModel sống sót config change bằng cơ chế nào?
`Activity.retainNonConfigurationInstances()` → `ComponentActivity.onRetainNonConfigurationInstance()` trả `NonConfigurationInstances{viewModelStore}` → `ActivityClientRecord.lastNonConfigurationInstances` → Activity mới `getLastNonConfigurationInstance()`. `viewModelStore.clear()` chỉ khi `isFinishing` (không phải config change).
- **⚠️ Đã sửa**: object này **không** được gửi lên system_server (Gemini viết "gửi lên system server hoặc giữ trong memory").

### Q29. LiveData `postValue` mất giá trị? Vì sao cần `repeatOnLifecycle`?
- `postValue` gộp vào `mPendingData`, chỉ post 1 Runnable → nhiều lần post trước khi main thread chạy thì chỉ giữ giá trị cuối.
- `lifecycleScope.launch { flow.collect{} }` vẫn collect khi STOPPED → dùng `repeatOnLifecycle(STARTED)` (huỷ & khởi động lại). `launchWhenStarted` chỉ suspend, không huỷ upstream (đã deprecated).

### Q30. Room Flow tự cập nhật thế nào?
`InvalidationTracker` tạo TEMP TRIGGER ghi vào bảng `room_table_modification_log`; khi bảng thay đổi → observer chạy lại query → emit. JOIN nhiều bảng lớn sẽ query lại thường xuyên.

### Q31. WorkManager đảm bảo chạy thế nào?
Lưu mọi work vào DB nội bộ (`androidx.work.workdb`); API 23+ dùng `SystemJobScheduler` (JobScheduler); khi reboot thì `RescheduleReceiver` lên lịch lại. Không dùng cho tác vụ cần chính xác theo giây.

---

## 11.9 View rendering

### Q32. Từ `invalidate()/requestLayout()` đến pixel trên TV?
1. `requestLayout` đặt `PFLAG_FORCE_LAYOUT`, lan lên `ViewRootImpl`.
2. `scheduleTraversals()`: sync barrier + `Choreographer.postCallback(CALLBACK_TRAVERSAL)` + xin VSYNC qua `DisplayEventReceiver`.
3. VSYNC-app → `Choreographer.doFrame()`: input → animation → traversal (`performMeasure/Layout/Draw`).
4. `draw` ghi **DisplayList** (RenderNode); `ThreadedRenderer` sync sang **RenderThread** → GPU (Skia/GL/Vulkan).
5. Buffer gửi qua `BLASTBufferQueue` → SurfaceFlinger → HWC → HDMI.

### Q33. UI thread rảnh mà vẫn jank?
GPU bound (GPU Mali yếu ở 1080p/4K), upload texture lớn trên RenderThread, hết buffer (SurfaceFlinger/HWC giữ), SurfaceFlinger bị trễ. Dùng Perfetto: xem slice `DrawFrame` của RenderThread, `dequeueBuffer`, `queueBuffer`.

### Q34. Measure nhiều lần ("double taxation")?
`RelativeLayout` luôn measure 2 lần; `LinearLayout` có `layout_weight` cũng measure 2 lần → lồng nhau thì số lần tăng theo cấp số nhân. Dùng `ConstraintLayout` phẳng.

### Q35. Overdraw khi phát video?
Bỏ `windowBackground` (`window.setBackgroundDrawable(null)`), không set background chồng nhau; dùng `SurfaceView` (video đi thẳng vào HWC overlay plane) thay `TextureView` (phải đi qua GPU, tốn thêm buffer).

---

## 11.10 Jetpack Compose

### Q36. Compose compiler biến `@Composable` thành gì?
Thêm tham số `$composer`, `$changed` (bitmask); chèn `startRestartGroup/endRestartGroup`; nếu mọi tham số không đổi và ổn định (stable) thì `skipToGroupEnd()`. State lưu trong **SlotTable** (gap buffer).

### Q37. Stability?
`@Immutable` (mọi thuộc tính `val` + kiểu bất biến), `@Stable` (thay đổi có báo cho Compose, như `MutableState`). `List<T>` là interface → unstable → composable nhận `List` không skip được. Dùng `kotlinx.collections.immutable` hoặc bọc `@Immutable` (hoặc bật strong skipping ở compiler mới).

### Q38. Side effects?
`LaunchedEffect(key)` (coroutine gắn với composition, restart khi key đổi), `rememberCoroutineScope` (launch từ callback như onClick/onKey), `SideEffect` (chạy sau mỗi lần recompose thành công), `DisposableEffect` (cần `onDispose` để gỡ listener).

### Q39. Animation focus scale không gây recomposition?
Đọc state trong phase **draw**: `Modifier.graphicsLayer { scaleX = s.value }`; `Modifier.offset { IntOffset(...) }` dạng lambda (đọc ở layout phase).

---

## 11.11 ANR & Memory leak ở tầng app

### Q40. Các loại ANR & timeout (A14)?
| Loại | Timeout | Nơi phát hiện |
|------|---------|---------------|
| Input dispatch | 5s | `InputDispatcher` (native) |
| Broadcast fg / bg | 10s / 60s | `BroadcastQueueModernImpl` (`deliveryTimeoutSoft/Hard`) |
| Service fg / bg | 20s / 200s | `ActiveServices` |
| `startForeground` sau `startForegroundService` | 30s (A14) | `ActiveServices` |
| ContentProvider publish | 10s | AMS |

- Khi ANR: AMS gửi **SIGQUIT** → thread "Signal Catcher" của ART dump stack ra `/data/anr/anr_<thời gian>` (bản cũ là `traces.txt`). Đọc `"main"` → `Blocked`/`waiting to lock <0x..> held by thread N` → tìm thread N.
- ANR thường xảy ra khi **CPU gần như rảnh**: main thread đang chờ lock hoặc binder.

### Q41. LeakCanary hoạt động thế nào?
Theo dõi object bị destroy bằng `KeyedWeakReference` + `ReferenceQueue`; sau 5s, gọi GC, nếu chưa vào queue → retained; ≥5 retained (app visible) → `Debug.dumpHprofData()` → thư viện Shark tìm đường ngắn nhất từ GC root. Không bật trên bản release (dump heap làm đứng process).

### Q42. Leak đặc thù TV?
Drawable/viền focus lưu static; player singleton giữ `SurfaceView`/Activity; listener Leanback `ViewHolder` capture Presenter/Fragment; `DisplayManager.registerDisplayListener` bằng Activity Context mà quên unregister (HDMI hotplug liên tục).

---

## 11.12 Android TV app

### Q43. D-pad: `FocusFinder` tìm view tiếp theo thế nào?
- Phím không ai xử lý → `ViewRootImpl` gọi `focusSearch(direction)` → `FocusFinder.findNextFocus`.
- Ứng viên phải nằm đúng hướng; chọn view có khoảng cách trọng số nhỏ nhất:
  **`13 × major² + minor²`** (`FocusFinder.java:537`). **⚠️ Đã sửa**: Gemini ghi công thức tuyến tính `13×major + minor`.
- Can thiệp: `nextFocusRight/Down...`, `focusSearch()` override, `addFocusables()`, `descendantFocusability`, `setOnKeyListener`.

### Q44. Leanback: Presenter / ArrayObjectAdapter / ItemBridgeAdapter?
`Presenter` = tạo/bind ViewHolder cho 1 kiểu item; `ArrayObjectAdapter` + `PresenterSelector` giữ data; `ItemBridgeAdapter` nối sang `RecyclerView.Adapter`; `HorizontalGridView`/`VerticalGridView` giữ item focus ở vị trí cố định (window alignment). Dùng `DiffCallback`/`setItems` thay vì `notifyDataSetChanged` để không mất focus.

### Q45. Viết Launcher (HOME) cho Android TV?
```xml
<activity android:name=".HomeActivity" android:launchMode="singleTask"
    android:clearTaskOnLaunch="true" android:stateNotNeeded="true" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.HOME"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
    <intent-filter>   <!-- để hiện trong danh sách app của launcher TV -->
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LEANBACK_LAUNCHER"/>
    </intent-filter>
</activity>
```
- **⚠️ Đã sửa**: `LEANBACK_LAUNCHER` **không** phải điều kiện để làm HOME. Nó chỉ dùng để app xuất hiện trong danh sách app của launcher TV. HOME cần `HOME` + `DEFAULT`.
- **Ví dụ project**: `DavidLauncher` là prebuilt APK privileged, ký platform (`vendor/david/apps/DavidLauncher/Android.mk`: `LOCAL_PRIVILEGED_MODULE := true`, `LOCAL_CERTIFICATE := platform`), có allowlist `privapp-permissions-launcher.xml` + `default-launcher-permissions.xml` (auto-grant location/BT). Câu hỏi hay: *vì sao `LOCAL_DEX_PREOPT := false` làm boot đầu chậm hơn?* → APK không được AOT lúc build, phải dexopt lúc boot/khi chạy.
- Channels & Programs: `TvContractCompat.PreviewPrograms` / `WatchNextPrograms` ghi vào TvProvider để hiện hàng gợi ý trên Home.

### Q46. TIF overlay view của TvInputService hiển thị thế nào?
`Session.onCreateOverlayView()` → `TvInputService` thêm view qua `WindowManager.addView` với window type **`TYPE_APPLICATION_MEDIA_OVERLAY`** (`TvInputService.java:1930-1947`), dùng window token từ `TvView`. Video vẽ vào Surface của `SurfaceView` trong `TvView`. **⚠️ Đã sửa**: Gemini viết "attach vào một SurfaceControl riêng".
