# Chapter 11 — App Layer (Android 14, Android TV/STB)

> Source: Claude ↔ Gemini discussion, **verified against the A14 source in the project**. Places where Gemini was wrong have been corrected and marked **⚠️ Corrected**. Gemini's original is in `gemini_raw/01_gemini_app_layer.md` (for further reading; remember to cross-check with chapter 16).

---

## 11.1 Activity lifecycle, configuration change, process death

### Q1. How does a configuration change differ from process death? How is state restored?

| | Configuration change | Process death (LMK kill while in background) |
|--|---------------------|-------------------------------------------|
| `ActivityRecord` in system_server | kept | kept (holds the icicle `Bundle`) |
| Process | kept | killed |
| Java objects (ViewModel, fields) | **ViewModel survives** (via `NonConfigurationInstances`) | all lost |
| `onSaveInstanceState` Bundle | yes | yes (already sent to ATMS via `activityStopped()`) |
| `SavedStateHandle` | yes | yes |

- Config change: `ActivityThread.handleRelaunchActivity` → `performDestroyActivity` keeps `ActivityClientRecord.lastNonConfigurationInstances` (**only in the app process's RAM**, not sent to system_server) → the new Activity receives it in `attach()` → `ComponentActivity.getViewModelStore()` gets the old store back.
- Process death: the Bundle was sent to system_server at `onStop` (`ActivityClientController.activityStopped`) → when the user returns, ATMS starts a new process and passes the Bundle into `onCreate(savedInstanceState)`.

```kotlin
class ChannelViewModel(private val state: SavedStateHandle) : ViewModel() {
    val channelId: StateFlow<String> = state.getStateFlow("ch", "1")   // survives process death too
    fun tune(id: String) { state["ch"] = id }
}
```
**Pitfall**: stuffing an EPG list/Bitmap into the Bundle → `TransactionTooLargeException` (the ~1MB binder buffer is **shared by the whole process**).

### Q2. The STB switches HDMI resolution (1080p ↔ 4K, 50 ↔ 60Hz) — what happens to the Activity?

- DisplayManager receives the new mode → `Configuration` changes (`screenWidthDp`, `screenHeightDp`, `smallestScreenWidthDp`, `densityDpi`, sometimes `colorMode`) → `ActivityRecord.ensureActivityConfiguration()` → if the Activity doesn't declare that it handles those changes, it is **relaunched**.
- A fullscreen player should declare:
```xml
<activity android:name=".PlayerActivity"
    android:configChanges="screenSize|smallestScreenSize|screenLayout|density|orientation|keyboard|keyboardHidden|navigation|uiMode" />
```
- **Pitfall**: declaring only `screenSize` and forgetting `smallestScreenSize`/`screenLayout` → still recreated → MediaCodec is released/recreated → 1–3s black screen.
- In `onDestroy()` distinguish `isFinishing` (really release) from `isChangingConfigurations` (only detach the Surface, keep the player).

### Q3. How does SavedStateRegistry work?

- `ComponentActivity` is a `SavedStateRegistryOwner`; each component registers a `SavedStateProvider` under a key. During `onSaveInstanceState`, the registry calls each provider's `saveState()` and merges them into the Bundle. On restore, a component gets its state back via `consumeRestoredStateForKey(key)`.
- Registering a duplicate key → `IllegalArgumentException`.

---

## 11.2 Launch modes, Tasks, Intent flags

### Q4. `standard`, `singleTop`, `singleTask`, `singleInstance`, `singleInstancePerTask`?

| launchMode | Behavior |
|-----------|---------|
| `standard` | always creates a new instance in the caller's task |
| `singleTop` | if already at the top of the task → `onNewIntent()`, no new instance |
| `singleTask` | 1 instance in the task chosen by `taskAffinity`; if it exists → bring the task forward, **pop every activity above it**, `onNewIntent()` |
| `singleInstance` | like singleTask but the task **contains only it**; activities it starts go to another task |
| `singleInstancePerTask` (A12+) | must be the root of the task; multiple instances are possible with `FLAG_ACTIVITY_MULTIPLE_TASK`/`NEW_DOCUMENT` |

Handled in `ActivityStarter` (`frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java`).

### Q5. `NEW_TASK | CLEAR_TOP` vs `CLEAR_TASK`?

- `CLEAR_TOP`: finishes every activity above the target; if the target is `standard` and there's no `FLAG_ACTIVITY_SINGLE_TOP` → **the target itself is finished and recreated**. Adding `SINGLE_TOP` → keeps the instance and calls `onNewIntent`.
- `CLEAR_TASK` (must be combined with `NEW_TASK`): wipes the task, then creates the activity as the new root.
```kotlin
startActivity(Intent(this, HomeActivity::class.java).addFlags(
    Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP or Intent.FLAG_ACTIVITY_SINGLE_TOP))
```

### Q6. Task hijacking (StrandHogg) and `taskAffinity`?

- A malicious app declares a `taskAffinity` equal to the victim's package (+ `allowTaskReparenting`) → a fake activity slips into the victim's task.
- Defense: `taskAffinity=""` for sensitive screens, `exported=false`, verify the caller. Google patched this progressively in A11–A12 (restricting reparenting/launching into other apps' tasks).
- **⚠️ Corrected**: Gemini claimed "A14 completely forbids merging tasks across UIDs" — there is no clear basis for that in the source; don't state it that confidently in an interview.

### Q7. Opening a Detail Screen from Voice Search/EPG, pressing Back must return to the app's list screen?

Use `TaskStackBuilder` + `android:parentActivityName` to build a synthetic back stack:
```kotlin
val pi = TaskStackBuilder.create(ctx)
    .addNextIntentWithParentStack(Intent(ctx, DetailActivity::class.java).putExtra("id", id))
    .getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
```
A12+ requires PendingIntents to specify `FLAG_IMMUTABLE`/`FLAG_MUTABLE` explicitly.

---

## 11.3 Service & Foreground Service (A14)

### Q8. Which thread does a Service run on? Why can it still ANR when work is on a worker thread?

- `onCreate/onStartCommand/onBind` run on the **main thread**. AMS arms a timer when it sends a command (`ActiveServices.bumpServiceExecutingLocked` → `scheduleServiceTimeoutLocked`); the client calls `serviceDoneExecuting` when done to cancel it.
- Timeout: **20s** (foreground process) / **200s** (background) — `ActivityManagerConstants.java:261-266` (`DEFAULT_SERVICE_TIMEOUT = 20s × Build.HW_TIMEOUT_MULTIPLIER`, background ×10; the multiplier is > 1 on emulators/slow devices).
- If the main thread is busy (locks, I/O) before `onStartCommand` returns → ANR even though the heavy work is on another thread.

### Q9. Started + Bound: when is a Service destroyed?

`ServiceRecord`: `startRequested` + `bindings` (`IntentBindRecord` → `AppBindRecord` → `ConnectionRecord`). It is destroyed only when it **has been stopped** (`stopSelf/stopService`) **and** no binding with `BIND_AUTO_CREATE` remains.

### Q10. What must a Foreground Service satisfy on A14?

1. `targetSdk 34`: declaring `android:foregroundServiceType` in the manifest is **mandatory**.
2. Request the per-type permission: `FOREGROUND_SERVICE_MEDIA_PLAYBACK`, `..._DATA_SYNC`, `..._CONNECTED_DEVICE`, `..._SPECIAL_USE`, `..._SYSTEM_EXEMPTED`...
3. The type passed to `startForeground(id, notif, type)` must be ⊆ the types in the manifest.
4. After `startForegroundService()`, `startForeground()` must be called within **30s on A14** (`DEFAULT_SERVICE_START_FOREGROUND_TIMEOUT_MS = 30 * 1000`).

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK"/>
<service android:name=".PlaybackService" android:foregroundServiceType="mediaPlayback" android:exported="false"/>
```
- Manifest has no type and an FGS is started → `MissingForegroundServiceTypeException`; invalid type/missing permission → `InvalidForegroundServiceTypeException`/`SecurityException` (see `ActiveServices.java`, around line 2560).
- **⚠️ Corrected (2 Gemini errors)**:
  - `startForeground(id, notif)` **without** a type is still valid on A14 (uses `FOREGROUND_SERVICE_TYPE_MANIFEST` = all types in the manifest); it only fails when the manifest declares no type.
  - The **6-hour limit for `dataSync`** belongs to **Android 15**, it does not exist in A14.

### Q11. Starting a service from the background (e.g. receiving an emergency-alert multicast packet)?

- A8+: `startService()` from the background → `IllegalStateException` (normal apps); A12+: `startForegroundService()` from the background → `ForegroundServiceStartNotAllowedException`, except for exemptions (holding `SYSTEM_ALERT_WINDOW`, FCM high-priority, `LOCKED_BOOT_COMPLETED`/`BOOT_COMPLETED`, companion devices, privileged/persistent apps...).
- On an STB, OEM apps are usually **privileged/persistent** (like `DavidExtension`), so most restrictions are waived. Normal apps: use JobScheduler/WorkManager expedited work, or a notification.
- **⚠️ Note**: on A14 `fullScreenIntent` requires the `USE_FULL_SCREEN_INTENT` permission, which is granted by default only to calling/alarm apps. Gemini didn't mention this.

---

## 11.4 BroadcastReceiver

### Q12. A14: `RECEIVER_EXPORTED` / `RECEIVER_NOT_EXPORTED`?

- targetSdk 34: `registerReceiver()` for broadcasts **that aren't system broadcasts** must pass one of the 2 flags, otherwise → `SecurityException`.
```kotlin
ContextCompat.registerReceiver(ctx, receiver, IntentFilter("com.david.ACTION_X"), ContextCompat.RECEIVER_NOT_EXPORTED)
```

### Q13. Ordered vs normal broadcasts, `abortBroadcast()`?

- Normal: delivered in parallel; ordered: sequential by `priority`, a receiver can modify the result or `abortBroadcast()`.
- Calling `abortBroadcast()` in a normal broadcast: the framework **logs an error** "BroadcastReceiver trying to return result during a non-ordered broadcast" (creating a RuntimeException just to print the stack), it **does not crash the app**. **⚠️ Corrected**: Gemini said it "throws an exception".
- A14 uses `BroadcastQueueModernImpl` (on by default: `BroadcastConstants.DEFAULT_MODERN_QUEUE_ENABLED = true`).

### Q14. Sticky broadcasts?

- Deprecated since API 21: no protection (anyone can send/modify them), data lingers in system_server. Still used internally for `ACTION_BATTERY_CHANGED`, ... (`registerReceiver(null, filter)` returns the last value).

### Q15. Implicit broadcast ban (A8)?

- Manifest-declared receivers **do not receive** implicit broadcasts (except the exemption list) → register dynamically or send explicitly (`setPackage`/`setComponent`).
- **⚠️ Corrected**: the exemption list is **not** in `BroadcastConstants.java`. The truth: actions flagged with `FLAG_RECEIVER_INCLUDE_BACKGROUND` in platform code, plus the `allow-implicit-broadcast` list in sysconfig XML (`SystemConfig.getAllowImplicitBroadcasts()` → `ActivityManagerService.mBackgroundLaunchBroadcasts`, `ActivityManagerService.java:2602`).
```xml
<!-- /system/etc/sysconfig/xxx.xml : allow static receivers to receive an OEM broadcast -->
<config><allow-implicit-broadcast action="com.david.action.FW_UPDATE"/></config>
```

---

## 11.5 ContentProvider

### Q16. Why does `ContentProvider.onCreate()` run before `Application.onCreate()`?

`ActivityThread.handleBindApplication()`: `makeApplicationInner()` (calls `attachBaseContext`) → `installContentProviders()` (calls `onCreate` of every provider) → `publishContentProviders` to AMS → `mInstrumentation.callApplicationOnCreate()`. Libraries (App Startup, WorkManager, Firebase) exploit this to self-initialize → **slower cold start** on STBs.

### Q17. CursorWindow & size limits?

- `query()` results travel through **shared memory** (ashmem) in a `CursorWindow`; only the fd goes over binder.
- **⚠️ Corrected**: Gemini said 4MB. On A14 in the project it is **2MB**: `core/res/res/values/config.xml: config_cursorWindowSize = 2048` (KB). A row that's too big → `SQLiteBlobTooBigException`/`CursorWindowAllocationException`.
- Solutions: pagination (`QUERY_ARG_LIMIT/OFFSET`), don't store blobs/images in the DB, return files via `openFile()`/`openPipeHelper()`.

### Q18. FileProvider, `FLAG_GRANT_READ_URI_PERMISSION`?

- A `file://` URI leaving the app → `FileUriExposedException` (A7+). Use `content://` from a FileProvider; temporary grants go through `UriGrantsManagerService`.
- **Pitfall**: `<root-path path="."/>` exposes the whole storage.

### Q19. Passing a 100MB file between 2 apps?

Don't pass a `byte[]` over binder. Use a `ParcelFileDescriptor` (`openFile()`, `openPipeHelper()`, `ParcelFileDescriptor.createPipe()`), or `SharedMemory`.

---

## 11.6 Handler / Looper / MessageQueue

### Q20. MessageQueue structure & how it sleeps/wakes?

- A singly linked list sorted by `Message.when` (`uptimeMillis`).
- `next()` → `nativePollOnce(ptr, timeout)` → `Looper::pollOnce` → `epoll_wait`; woken when `enqueueMessage` calls `nativeWake` → writes to an `eventfd` (`system/core/libutils/Looper.cpp`).
- The main thread can't `quit()` → `IllegalStateException("Main thread not allowed to quit.")`.

### Q21. What is a sync barrier? How does `ViewRootImpl` use it?

- A barrier = a `Message` with `target == null`. When `next()` meets a barrier it **skips synchronous messages** and only takes `isAsynchronous()` ones.
- `ViewRootImpl.scheduleTraversals()` calls `postSyncBarrier()` (`ViewRootImpl.java:2439`) then registers a Choreographer callback (async) → the frame gets priority; `unscheduleTraversals/doTraversal` call `removeSyncBarrier()`.
- `postSyncBarrier()` is not in the public SDK (`@TestApi` + `@UnsupportedAppUsage`, `MessageQueue.java:470-472`). If a barrier "leaks" (never removed), the main thread stops processing input/lifecycle → the app hangs.

### Q22. IdleHandler?

`queueIdle()` is called when the queue is empty or the next message isn't due yet; return `false` to run only once. Use it for lazy init after the first frame is drawn. Don't do heavy work there.

### Q23. Handler leak?

GC root chain: `Looper (static sMainLooper)` → `MessageQueue` → `Message.target (Handler)` → `Activity` (inner class). Fix: a static Handler + `WeakReference` **and** `removeCallbacksAndMessages(null)` in `onDestroy`.

---

## 11.7 Kotlin Coroutines & Flow

### Q24. What does `suspend` compile into?
CPS: adds a `Continuation` parameter, returns `Any?` (the value or `COROUTINE_SUSPENDED`), and the body becomes a **state machine** with a `label`, with variables "spilled" into the continuation object. Don't use `Thread.sleep` in a coroutine (blocks the thread pool).

### Q25. `Job` vs `SupervisorJob` — why does `scope.launch` with a SupervisorJob still crash?
- Job: a failing child cancels the parent and its siblings. SupervisorJob: failures don't spread to siblings.
- But an unhandled exception in `launch` still goes to the `CoroutineExceptionHandler`/the thread's uncaught handler → **crash** if there's no handler. `async` keeps the exception until `await()`.
- `withContext(SupervisorJob())` doesn't do what you expect (breaks structured concurrency).

### Q26. Dispatchers?
- `Default` = number of cores (minimum 2), `IO` = `max(64, cores)` and it **shares the `CoroutineScheduler` pool** with Default, so switching between them usually costs no context switch.
- `Main.immediate`: if already on the main thread, runs immediately without posting.

### Q27. StateFlow vs SharedFlow?
- `StateFlow` ≈ `SharedFlow(replay=1, onBufferOverflow=DROP_OLDEST)` + `distinctUntilChanged` + always has an initial value.
- One-shot events (navigate/toast) use `SharedFlow(replay=0)` or a `Channel`, not StateFlow (it would be replayed after recreation).

---

## 11.8 Jetpack

### Q28. Which mechanism lets a ViewModel survive configuration changes?
`Activity.retainNonConfigurationInstances()` → `ComponentActivity.onRetainNonConfigurationInstance()` returns `NonConfigurationInstances{viewModelStore}` → `ActivityClientRecord.lastNonConfigurationInstances` → the new Activity's `getLastNonConfigurationInstance()`. `viewModelStore.clear()` happens only when `isFinishing` (not on config change).
- **⚠️ Corrected**: this object is **not** sent to system_server (Gemini wrote "sent to the system server or kept in memory").

### Q29. LiveData `postValue` loses values? Why is `repeatOnLifecycle` needed?
- `postValue` merges into `mPendingData` and posts a single Runnable → multiple posts before the main thread runs keep only the last value.
- `lifecycleScope.launch { flow.collect{} }` keeps collecting while STOPPED → use `repeatOnLifecycle(STARTED)` (cancels & restarts). `launchWhenStarted` only suspends, it doesn't cancel the upstream (deprecated).

### Q30. How does a Room Flow update itself?
`InvalidationTracker` creates TEMP TRIGGERs writing into the `room_table_modification_log` table; when a table changes → the observer re-runs the query → emits. JOINs across many large tables get re-queried frequently.

### Q31. How does WorkManager guarantee execution?
It stores all work in an internal DB (`androidx.work.workdb`); on API 23+ it uses `SystemJobScheduler` (JobScheduler); after a reboot, `RescheduleReceiver` reschedules. Not for tasks that need to-the-second precision.

---

## 11.9 View rendering

### Q32. From `invalidate()/requestLayout()` to pixels on the TV?
1. `requestLayout` sets `PFLAG_FORCE_LAYOUT`, propagates up to `ViewRootImpl`.
2. `scheduleTraversals()`: sync barrier + `Choreographer.postCallback(CALLBACK_TRAVERSAL)` + requests VSYNC via `DisplayEventReceiver`.
3. VSYNC-app → `Choreographer.doFrame()`: input → animation → traversal (`performMeasure/Layout/Draw`).
4. `draw` records a **DisplayList** (RenderNode); `ThreadedRenderer` syncs it to the **RenderThread** → GPU (Skia/GL/Vulkan).
5. The buffer goes through `BLASTBufferQueue` → SurfaceFlinger → HWC → HDMI.

### Q33. The UI thread is idle but there's still jank?
GPU bound (weak Mali GPU at 1080p/4K), large texture uploads on the RenderThread, running out of buffers (held by SurfaceFlinger/HWC), SurfaceFlinger running late. In Perfetto look at the RenderThread's `DrawFrame` slices, `dequeueBuffer`, `queueBuffer`.

### Q34. Measuring multiple times ("double taxation")?
`RelativeLayout` always measures twice; a `LinearLayout` with `layout_weight` also measures twice → when nested, the number of passes grows exponentially. Use a flat `ConstraintLayout`.

### Q35. Overdraw during video playback?
Remove the `windowBackground` (`window.setBackgroundDrawable(null)`), don't stack backgrounds; use `SurfaceView` (video goes straight to an HWC overlay plane) instead of `TextureView` (goes through the GPU, costs extra buffers).

---

## 11.10 Jetpack Compose

### Q36. What does the Compose compiler turn `@Composable` into?
Adds `$composer`, `$changed` (bitmask) parameters; inserts `startRestartGroup/endRestartGroup`; if all parameters are unchanged and stable, it calls `skipToGroupEnd()`. State lives in the **SlotTable** (gap buffer).

### Q37. Stability?
`@Immutable` (all properties `val` + immutable types), `@Stable` (changes are notified to Compose, like `MutableState`). `List<T>` is an interface → unstable → a composable taking a `List` can't skip. Use `kotlinx.collections.immutable` or wrap it with `@Immutable` (or enable strong skipping in newer compilers).

### Q38. Side effects?
`LaunchedEffect(key)` (a coroutine tied to the composition, restarted when the key changes), `rememberCoroutineScope` (launch from callbacks like onClick/onKey), `SideEffect` (runs after every successful recomposition), `DisposableEffect` (needs `onDispose` to remove listeners).

### Q39. A focus scale animation without recomposition?
Read state in the **draw** phase: `Modifier.graphicsLayer { scaleX = s.value }`; lambda-based `Modifier.offset { IntOffset(...) }` (read in the layout phase).

---

## 11.11 ANR & memory leaks at the app layer

### Q40. ANR types & timeouts (A14)?
| Type | Timeout | Detected by |
|------|---------|---------------|
| Input dispatch | 5s | `InputDispatcher` (native) |
| Broadcast fg / bg | 10s / 60s | `BroadcastQueueModernImpl` (`deliveryTimeoutSoft/Hard`) |
| Service fg / bg | 20s / 200s | `ActiveServices` |
| `startForeground` after `startForegroundService` | 30s (A14) | `ActiveServices` |
| ContentProvider publish | 10s | AMS |

- On ANR: AMS sends **SIGQUIT** → ART's "Signal Catcher" thread dumps stacks to `/data/anr/anr_<time>` (older versions: `traces.txt`). Read `"main"` → `Blocked`/`waiting to lock <0x..> held by thread N` → find thread N.
- ANRs often happen while the **CPU is almost idle**: the main thread is waiting for a lock or a binder call.

### Q41. How does LeakCanary work?
Tracks destroyed objects with `KeyedWeakReference` + `ReferenceQueue`; after 5s it triggers GC; if the reference hasn't been enqueued → retained; ≥5 retained (app visible) → `Debug.dumpHprofData()` → the Shark library finds the shortest path from a GC root. Don't enable it in release builds (heap dumps freeze the process).

### Q42. TV-specific leaks?
Focus highlight drawables stored statically; a player singleton holding a `SurfaceView`/Activity; Leanback `ViewHolder` listeners capturing a Presenter/Fragment; `DisplayManager.registerDisplayListener` with an Activity Context and forgetting to unregister (frequent HDMI hotplug).

---

## 11.12 Android TV apps

### Q43. D-pad: how does `FocusFinder` find the next view?
- A key nobody handles → `ViewRootImpl` calls `focusSearch(direction)` → `FocusFinder.findNextFocus`.
- Candidates must lie in the right direction; it picks the view with the smallest weighted distance:
  **`13 × major² + minor²`** (`FocusFinder.java:537`). **⚠️ Corrected**: Gemini wrote the linear formula `13×major + minor`.
- Ways to intervene: `nextFocusRight/Down...`, overriding `focusSearch()`, `addFocusables()`, `descendantFocusability`, `setOnKeyListener`.

### Q44. Leanback: Presenter / ArrayObjectAdapter / ItemBridgeAdapter?
`Presenter` = creates/binds the ViewHolder for one item type; `ArrayObjectAdapter` + `PresenterSelector` hold the data; `ItemBridgeAdapter` bridges to a `RecyclerView.Adapter`; `HorizontalGridView`/`VerticalGridView` keep the focused item at a fixed position (window alignment). Use `DiffCallback`/`setItems` instead of `notifyDataSetChanged` so focus isn't lost.

### Q45. Writing a Launcher (HOME) for Android TV?
```xml
<activity android:name=".HomeActivity" android:launchMode="singleTask"
    android:clearTaskOnLaunch="true" android:stateNotNeeded="true" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.HOME"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
    <intent-filter>   <!-- to appear in the TV launcher's app list -->
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LEANBACK_LAUNCHER"/>
    </intent-filter>
</activity>
```
- **⚠️ Corrected**: `LEANBACK_LAUNCHER` is **not** a requirement for being HOME. It only makes the app appear in the TV launcher's app list. HOME needs `HOME` + `DEFAULT`.
- **Project example**: `DavidLauncher` is a privileged prebuilt APK, platform-signed (`vendor/david/apps/DavidLauncher/Android.mk`: `LOCAL_PRIVILEGED_MODULE := true`, `LOCAL_CERTIFICATE := platform`), with the allowlist `privapp-permissions-launcher.xml` + `default-launcher-permissions.xml` (auto-grants location/BT). Good question: *why does `LOCAL_DEX_PREOPT := false` make the first boot slower?* → the APK isn't AOT-compiled at build time, so it must be dexopted at boot/at run time.
- Channels & Programs: `TvContractCompat.PreviewPrograms` / `WatchNextPrograms` written into TvProvider to show recommendation rows on Home.

### Q46. How is a TvInputService's TIF overlay view displayed?
`Session.onCreateOverlayView()` → `TvInputService` adds the view via `WindowManager.addView` with window type **`TYPE_APPLICATION_MEDIA_OVERLAY`** (`TvInputService.java:1930-1947`), using the window token from `TvView`. Video is drawn into the Surface of the `SurfaceView` inside `TvView`. **⚠️ Corrected**: Gemini wrote "attached to a separate SurfaceControl".
