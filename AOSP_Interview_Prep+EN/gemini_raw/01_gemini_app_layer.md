Below is an in-depth set of interview questions and answers for the position of **Senior Android Engineer (Android 14, Android TV / Set-top Box Platform)**, approached from the perspective of a Platform/Framework Engineer.

---

# TOPIC 1: ACTIVITY LIFECYCLE, CONFIGURATION CHANGE, PROCESS DEATH & SAVEDSTATE

### Q1. Distinguish the fundamental differences between Configuration Change and Low Memory Process Death. How does the AOSP mechanism handle state restoration differently in these two cases?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * **Configuration Change:** When a hardware configuration changes (e.g., STB changes HDMI resolution via `DisplayManagerService`), `ActivityTaskManagerService` (ATMS) sends a relaunch request to the client via Binder. In `ActivityThread.java`, the `handleRelaunchActivityLocally()` method is called. The `ActivityRecord` on the system server remains intact. The system retains the `ActivityClientRecord.lastNonConfigurationInstances` object to transfer the ViewModelStore and retained instances directly to the new `Activity` instance without requiring serialization over Binder.
  * **Process Death:** Occurs when the STB encounters low RAM conditions (`LowMemoryKiller` - LMKD kills background processes). The entire app process is destroyed, and heap memory is cleared. However, the `ActivityRecord` in ATMS still retains the state `Bundle` that the app previously sent to the system server via `onSaveInstanceState()` (calling the Binder transaction `activityStopped()` -> `ActivityRecord.saveState()`). When the user returns, ATMS forks a new process via Zygote, creates `ActivityThread`, and passes `savedInstanceState` (Bundle) into `Activity.onCreate()`. In this scenario, ViewModel is completely destroyed unless `SavedStateHandle` is used.
* **Illustrative Code (SavedStateHandle):**
```kotlin
class ChannelDetailViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    // Survives both Config Change and Process Death
    val channelId: StateFlow<String> = savedStateHandle.getStateFlow("KEY_CHANNEL_ID", "default_ch_01")

    fun updateChannel(id: String) {
        savedStateHandle["KEY_CHANNEL_ID"] = id
    }
}
```
* **Pitfall:** Storing Bitmaps or EPG lists with thousands of items in `onSaveInstanceState()`. This Bundle is sent via IPC Binder to `system_server`. The Binder buffer for the entire process is limited to only **1MB** (shared across all running transactions). Exceeding this threshold throws a `TransactionTooLargeException`, causing an immediate crash.

---

### Q2. On Android TV STBs, when the user plugs/unplugs an HDMI cable (Hotplug) or the TV automatically switches Display Mode (4K 60Hz to 1080p 50Hz/HDR), how is the Activity Lifecycle affected and what is the comprehensive way to handle it?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * When the HDMI display changes, `DisplayManagerService` detects the new EDID from the hardware and dispatches events to `WindowManagerService` (WMS) and ATMS. System `Configuration` fields are changed: `densityDpi`, `screenWidthDp`, `screenHeightDp`, `smallestScreenWidthDp`, and sometimes `colorMode` (when HDR is enabled).
  * ATMS traverses the current `Task`, invokes `ActivityRecord.ensureActivityConfiguration()`, and determines whether the Activity handles these configs itself (`ActivityRecord.canHandleConfigChanges()`). If not, the Activity will be recreated (`onDestroy()` -> `onCreate()`).
* **Illustrative Code:**
```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".player.TvPlayerActivity"
    android:configChanges="density|keyboard|keyboardHidden|navigation|orientation|screenLayout|screenSize|smallestScreenSize|uiMode"
    android:exported="true" />
```
```kotlin
// TvPlayerActivity.kt
override fun onConfigurationChanged(newConfig: Configuration) {
    super.onConfigurationChanged(newConfig)
    // Handle manually: recalculate UI scale or SurfaceView aspect ratio
    playerPresenter.relayoutSurfaces(newConfig.screenWidthDp, newConfig.screenHeightDp)
}
```
* **Pitfall:** Forgetting to declare `smallestScreenSize` or `screenLayout`. On some Android TV SoC chipsets (Amlogic, Realtek), switching HDMI resolution changes `smallestScreenWidthDp`. If this flag is missing, even if `screenSize` is declared, the Activity will still be destroyed/recreated, leading to screen tearing or ExoPlayer interrupting hardware decoding (re-instantiating MediaCodec).

---

### Q3. Distinguish between `isFinishing()` and `isChangingConfigurations()` in `onDestroy()`. Why is this vital for hardware resource management (Codec, Tuner, ExoPlayer) on an STB?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * In `Activity.java`, the `mFinished` variable is set to `true` when `finish()` is called or the user presses the Back button. The `isFinishing()` function returns this value.
  * When a Configuration Change occurs, `ActivityThread.java` calls `handleRelaunchActivity()`, and the `mChangingConfigurations` variable in Activity is set to `true`. The `isChangingConfigurations()` function returns this state.
* **Illustrative Code:**
```kotlin
override fun onDestroy() {
    super.onDestroy()
    if (isFinishing) {
        // Activity is actually killed completely: Release all hardware
        exoPlayer.release()
        hardwareTunerSession.close()
    } else if (isChangingConfigurations) {
        // Just a recreate due to HDMI mode switch: retain ExoPlayer session via NonConfigurationInstances
        // or detach SurfaceView to avoid heavy Codec release
        exoPlayer.clearVideoSurface()
    }
}
```
* **Pitfall:** Unconditionally calling `exoPlayer.release()` in `onDestroy()`. When an HDMI hotplug occurs, the player releases the MediaCodec decoder, and then `onCreate()` reinitializes from scratch, causing a black screen for 1-3 seconds and interrupting HDMI Passthrough audio.

---

### Q4. How does `SavedStateRegistry` operate in `androidx.activity.ComponentActivity`? How can a Custom View or independent Component restore its state by itself without depending on Activity callbacks?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * `ComponentActivity` implements `SavedStateRegistryOwner`. The `SavedStateRegistryController` class manages an instance of `SavedStateRegistry`.
  * `SavedStateRegistry` stores a map of `SavedStateProvider`s. When the Activity triggers `onSaveInstanceState(outState)`, it calls `SavedStateRegistry.performSave(outState)`. The system iterates through all registered providers, collects each component's `Bundle`, and packages them into an aggregate Bundle (`androidx.lifecycle.BundlableSavedStateRegistry.key`).
  * Upon recreation, `performRestore(savedInstanceState)` unbundles the child Bundle for each component based on the `key`.
* **Illustrative Code:**
```kotlin
class CustomTvChannelView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : FrameLayout(context, attrs), SavedStateRegistry.SavedStateProvider {

    private var currentChannelId: String = ""

    fun registerToHost(owner: SavedStateRegistryOwner) {
        owner.savedStateRegistry.registerSavedStateProvider("KEY_TV_VIEW", this)
        val restoredBundle = owner.savedStateRegistry.consumeRestoredStateForKey("KEY_TV_VIEW")
        currentChannelId = restoredBundle?.getString("EXTRA_CH") ?: "CH_01"
    }

    override fun saveState(): Bundle = Bundle().apply {
        putString("EXTRA_CH", currentChannelId)
    }
}
```
* **Pitfall:** Registering duplicate keys in `SavedStateRegistry` throws `IllegalArgumentException("SavedStateProvider with the given key is already registered")`. Additionally, calling `consumeRestoredStateForKey()` after the registry's lifecycle state has completed initialization results in lost restored data (`null`).

---

# TOPIC 2: LAUNCHMODE, TASKAFFINITY, INTENT FLAGS & BACK STACK

### Q5. Analyze the differences between `singleTask`, `singleInstance`, and `singleInstancePerTask` (Android 12+). When does the Back Stack branch or merge Tasks?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * The system manages this via `Task.java` and `ActivityStarter.java` in `services/core/.../wm/`.
  * `singleTask`: If no task with the corresponding `taskAffinity` exists yet, a new Task is created with the Activity as root. If the Task already exists, the system brings that Task to the foreground and invokes `onNewIntent()`. All Activities stacked on top of that `singleTask` Activity are popped from the stack (finished) by default to bring the `singleTask` Activity to the top.
  * `singleInstance`: Similar to `singleTask`, but the system guarantees that the Task containing it **holds only this single Activity**. Any other Activity launched from it is pushed into a different Task (using that app's default taskAffinity).
  * `singleInstancePerTask` (A12+): Allows creating multiple instances of the same type, but each instance must reside at the Root of a separate Task (often combined with multi-window or multi-display on TV).
* **Illustrative Code:**
```xml
<!-- Deep link Player opened from Launcher or Voice Command -->
<activity 
    android:name=".PlayerActivity"
    android:launchMode="singleTask"
    android:taskAffinity="com.stb.tv.playback" />
```
* **Pitfall:** Using `singleInstance` for payment or secondary settings screens. When opening a child screen from that screen, the child screen gets pushed into another task. Pressing the Back button on the TV remote causes confusion for users because it does not return to the previous screen, but instead jumps directly to another task's screen or Home.

---

### Q6. When combining `FLAG_ACTIVITY_NEW_TASK` and `FLAG_ACTIVITY_CLEAR_TOP`, how does the processing mechanism in `ActivityStarter.java` work? How does it differ from `FLAG_ACTIVITY_CLEAR_TASK`?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * In `ActivityStarter.java` (AOSP `frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java`), when encountering `FLAG_ACTIVITY_NEW_TASK`: The system searches for a Task with a matching affinity.
  * If additionally combined with `FLAG_ACTIVITY_CLEAR_TOP`:
    * The system scans the target Task to see if that Activity already exists (`Task.findActivityInHistory()`).
    * If found: The system destroys (finishes) all Activities on top of it. By default, if the target Activity does not have `launchMode="singleTop"` or lacks the `FLAG_ACTIVITY_SINGLE_TOP` flag, the target Activity itself is also finished and recreated!
    * If `FLAG_ACTIVITY_SINGLE_TOP` is accompanied: The target Activity is not recreated and instead receives the new intent via `onNewIntent()`.
  * `FLAG_ACTIVITY_CLEAR_TASK`: Must be used in conjunction with `FLAG_ACTIVITY_NEW_TASK`. The system clears all existing Activities in that Task (including the target Activity) and creates a new instance as the root activity.
* **Illustrative Code:**
```kotlin
// Return to Home Activity retaining the instance, clear all child screens
val intent = Intent(context, TvHomeActivity::class.java).apply {
    flags = Intent.FLAG_ACTIVITY_NEW_TASK or 
            Intent.FLAG_ACTIVITY_CLEAR_TOP or 
            Intent.FLAG_ACTIVITY_SINGLE_TOP
}
context.startActivity(intent)
```
* **Pitfall:** Using `FLAG_ACTIVITY_CLEAR_TOP` without `FLAG_ACTIVITY_SINGLE_TOP` (or without declaring `singleTop`/`singleTask` in the manifest). Consequence: The original Activity is destroyed and recreated from scratch, disrupting active data streams being displayed and re-triggering the entire screen transition animation.

---

### Q7. How does the StrandHogg (Task Hijacking) vulnerability relate to `taskAffinity`? What is the mitigation mechanism on Android 14?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * By default, all Activities in the same package share the same `taskAffinity`. A malicious app declares an Activity with a `taskAffinity` matching the victim's banking/television app and sets `allowTaskReparenting="true"` or launches with `FLAG_ACTIVITY_NEW_TASK`.
  * When the victim opens the legitimate app, the malicious app's Activity is brought to the top of the victim's Task, presenting a fake interface to trick users into entering login credentials/OTP codes.
  * Starting with Android 14, `ActivityRecord.java` and `Task.java` tighten `taskAffinity`. The `android:allowTaskReparenting` flag is strictly restricted, and the framework by default blocks task merging across different UIDs unless granted system permissions or sharing the same signing key.
* **Illustrative Code (Prevention):**
```xml
<!-- Lock against reparenting and reset affinity for sensitive screens -->
<activity
    android:name=".auth.TvLoginActivity"
    android:taskAffinity=""
    android:allowTaskReparenting="false"
    android:launchMode="singleTask"
    android:exported="false" />
```
* **Pitfall:** Leaving `exported="true"` combined with default `taskAffinity` without verifying the caller UID via `getCallingPackage()` or `Binder.getCallingUid()`.

---

### Q8. When a user clicks on a Search Result from Android TV Global Voice Search (Google Assistant) or Live Channels EPG to enter the Detail Screen, how do you construct a proper "Synthetic Back Stack" so that pressing Back does not exit directly to Home?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * When opened externally, the Detail Screen is pushed into a new Task or Google Assistant's Task. If Back is pressed, the user is thrown back to Assistant instead of returning to the app's Channel List screen.
  * The `TaskStackBuilder.java` class (AOSP `frameworks/base/core/java/android/app/TaskStackBuilder.java`) provides the ability to synthesize a virtual back stack based on the `android:parentActivityName` declaration in `AndroidManifest.xml`.
  * `TaskStackBuilder.startActivities()` or `getPendingIntent()` generates a sequence of `Intent`s. Upon execution, `ActivityStarter` launches the parent activity first before pushing the child activity to the top.
* **Illustrative Code:**
```kotlin
fun createPendingIntentForVoiceSearch(context: Context, channelId: String): PendingIntent {
    val detailIntent = Intent(context, ChannelDetailActivity::class.java).apply {
        putExtra("EXTRA_CHANNEL_ID", channelId)
    }
    
    return TaskStackBuilder.create(context).run {
        // Add all parent activities to stack based on AndroidManifest.xml
        addNextIntentWithParentStack(detailIntent)
        getPendingIntent(
            0,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )!!
    }
}
```
* **Pitfall:** Missing the `PendingIntent.FLAG_IMMUTABLE` flag on Android 12+ results in an `IllegalArgumentException` crash. Second, forgetting to declare `android:parentActivityName` in the Manifest causes `addNextIntentWithParentStack` to push only a single Intent into the stack, leaving the Back stack empty.

---

# TOPIC 3: SERVICE (STARTED, BOUND, FOREGROUND & BACKGROUND LIMITS A8-A14)

### Q9. Where in system_server is the Service ANR trigger (`SERVICE_TIMEOUT`) handled? Why can code running on a Service's worker thread still trigger an ANR?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * Managed by `ActiveServices.java` (in `com.android.server.am`).
  * When a Service is started/bound, `ActiveServices.bumpServiceExecutingLocked()` posts a delayed message to `ActivityManagerService`'s `Handler` (`SERVICE_TIMEOUT_MSG`):
    * Foreground Service timeout: 20 seconds.
    * Background Service timeout: 200 seconds.
  * When the client finishes processing the lifecycle (`onCreate()`, `onStartCommand()`, or `onBind()`), it sends the `serviceDoneExecuting()` Binder IPC back to the system server to remove this message from the queue.
  * If the app's main thread is blocked (due to lock contention, SQLite write on the main thread...), `ActivityThread.handleServiceArgs()` is not executed in time, the timeout message in AMS triggers and invokes `AppErrors.appNotResponding()`.
* **Illustrative Code:**
```kotlin
class BackgroundSyncService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // EVEN IF running on background thread:
        Thread {
            doHeavyTuning()
        }.start()
        
        // BUT if before returning, you invoke a blocking call on the Main Thread:
        runBlocking { 
            database.clearAllTables() // -> Block main thread -> ANR ServiceTimeout!
        }
        return START_NOT_STICKY
    }
    override fun onBind(intent: Intent?): IBinder? = null
}
```
* **Pitfall:** Mistakenly believing that Service automatically runs on a worker thread. A Service runs on the **Main (UI) Thread** of the App Process by default. Any blocking operation in `onCreate()` or `onStartCommand()` triggers the 20s ANR timer.

---

### Q10. Android 14 tightens `foregroundServiceType`. If an STB app runs in the background to stream P2P TV or sync EPG, what technical requirements must it fulfill? What happens if violated?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * On Android 14 (API 34), `ActiveServices.startForegroundServiceDelegateLocked()` strictly verifies `foregroundServiceType`.
  * Mandatory requirements: (1) Declare specific permissions (e.g., `FOREGROUND_SERVICE_DATA_SYNC`, `FOREGROUND_SERVICE_MEDIA_PLAYBACK`); (2) Declare `android:foregroundServiceType` in the Manifest; (3) Pass that exact type into the `startForeground(id, notif, type)` function.
  * If the type is missing or does not match the Manifest, the system throws `MissingForegroundServiceTypeException` or `SecurityException` immediately.
  * Specifically for the `dataSync` type, Android 14 enforces a runtime timeout (maximum 6 hours). Exceeding this limit causes AMS to invoke a timeout callback or stop the service.
* **Illustrative Code:**
```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<service
    android:name=".playback.TvPlaybackService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="false" />
```
```kotlin
// TvPlaybackService.kt
val notification = createPlaybackNotification()
// Android 14 requires passing type to startForeground
ServiceCompat.startForeground(
    this,
    NOTIFICATION_ID,
    notification,
    ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK
)
```
* **Pitfall:** Calling `startForeground()` without passing a type or passing `FOREGROUND_SERVICE_TYPE_NONE` on Android 14. The app crashes with `IllegalArgumentException: default fgs type is not allowed`.

---

### Q11. When a Service is both started with `startService()` and bound with `bindService(..., BIND_AUTO_CREATE)`, when does its lifecycle end? How does reference management in `ServiceRecord.java` work?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * In `ActiveServices.java`, each Service is represented by a `ServiceRecord.java`.
  * `ServiceRecord` maintains two independent states:
    * `startRequested = true`: When there is at least one `startService()` call. The Service intends to live until `stopSelf()` or `stopService()` is called.
    * `bindings`: A map containing the list of `AppBindRecord`s and `ConnectionRecord`s. When a client binds via `bindService()`, the reference counter increments.
  * **Destruction condition:** The Service actually has `onDestroy()` invoked if and only if:
    1. A stop was requested: `stopSelf()` or `stopService()` was called (`startRequested = false`).
    2. **AND** all clients have unbound (`bindings.isEmpty() == true`).
* **Illustrative Code:**
```kotlin
// Client 1 (PlaybackManager)
context.startService(serviceIntent) // Service lives independently

// Client 2 (UiController)
context.bindService(serviceIntent, connection, Context.BIND_AUTO_CREATE) // Obtain controlling Binder

// When UiController exits:
context.unbindService(connection) // Service STILL RUNS because startRequested is still true

// When the song ends or user presses Exit on the Remote:
playbackService.stopSelf() // At this point the Service officially calls onDestroy()
```
* **Pitfall:** Assuming that calling `stopSelf()` shuts down the service immediately. If a UI Component still holds a `ServiceConnection` (forgetting to call `unbindService()`), the Service continues to live in memory, leaking the entire Service and associated Context.

---

### Q12. On an STB, upon receiving an emergency alert UDP broadcast / Multicast packet from the Headend in the background, how do you start a Service to display an overlay without triggering `BackgroundServiceStartNotAllowedException`?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * Since Android 8.0 and especially Android 12+, `ActiveServices.java` completely blocks calling `startService()` or `startForegroundService()` from the background, except in cases within whitelist exemptions (receiving High Priority Push Notifications via FCM, specific system broadcasts, or apps having the `SYSTEM_ALERT_WINDOW` permission).
  * When violated, `ActiveServices.startServiceLocked()` checks `isAllowedResourceUsage()` and throws `BackgroundServiceStartNotAllowedException`.
  * **Solutions on STB:**
    1. Register to receive packets through a persistent worker or `JobScheduler` with high priority.
    2. For system apps (System STB apps signed with platform key): grant `START_ACTIVITIES_FROM_BACKGROUND` permission or configure privilege whitelist in `/etc/permissions/privapp-permissions-xxx.xml`.
    3. Use `WorkManager` with `OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST` or send a Notification with a `fullScreenIntent`.
* **Illustrative Code:**
```kotlin
fun showEmergencyAlert(context: Context, alertMsg: String) {
    val fullScreenIntent = Intent(context, EmergencyAlertActivity::class.java).apply {
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
        putExtra("ALERT_DATA", alertMsg)
    }
    val pendingIntent = PendingIntent.getActivity(
        context, 0, fullScreenIntent, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )

    val notification = NotificationCompat.Builder(context, EMERGENCY_CHANNEL_ID)
        .setSmallIcon(R.drawable.ic_warning)
        .setContentTitle("CẢNH BÁO KHẨN CẤP")
        .setContentText(alertMsg)
        .setPriority(NotificationCompat.PRIORITY_MAX)
        .setCategory(NotificationCompat.CATEGORY_ALARM)
        .setFullScreenIntent(pendingIntent, true) // Open Activity directly from background
        .build()

    NotificationManagerCompat.from(context).notify(999, notification)
}
```
* **Pitfall:** Directly calling `context.startService()` from within an IO Coroutine or raw Java Thread listening on a UDP socket when the app has no Activity in the foreground. The app will crash immediately in production.

---

# TOPIC 4: BROADCASTRECEIVER & ANDROID 14 CHANGES

### Q13. Analyze the mandatory change of the `RECEIVER_EXPORTED` / `RECEIVER_NOT_EXPORTED` flags in Android 14. How does the internal protection mechanism in `ContextImpl.java` work?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * Prior to Android 14, dynamic receivers could by default receive broadcasts from any other app in the operating system unless protected by declaring a permission.
  * In Android 14 (`ContextImpl.java` -> `registerReceiverInternal()`), the system checks if the target SDK >= 34; the `RECEIVER_EXPORTED` or `RECEIVER_NOT_EXPORTED` flag is mandatory to pass in, unless that broadcast is a system Protected Broadcast (such as `ACTION_BOOT_COMPLETED`, `ACTION_BATTERY_CHANGED`).
  * If the flag is not set, the system throws `SecurityException: com.package: Targeting U+ (version 34 and above) disallows registering receiver without specifying RECEIVER_EXPORTED or RECEIVER_NOT_EXPORTED`.
* **Illustrative Code:**
```kotlin
val filter = IntentFilter("com.stb.action.CUSTOM_HDMI_EVENT")
val receiver = HdmiEventReceiver()

// Standard Android 14 dynamic registration:
ContextCompat.registerReceiver(
    context,
    receiver,
    filter,
    ContextCompat.RECEIVER_NOT_EXPORTED // Can only be sent by the app internally or by the same UID
)
```
* **Pitfall:** Marking `RECEIVER_EXPORTED` for sensitive receivers (such as triggering firmware upgrades, clearing TV cache). Any third-party application installed on the STB can inject malicious Intents to trigger this logic without encountering any barriers.

---

### Q14. Dispatching mechanism of `OrderedBroadcast` vs standard Broadcast in `BroadcastQueueModernImpl` (Android 14). Why can calling `abortBroadcast()` halt the data stream of other apps?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * Standard Broadcast: Dispatches simultaneously to registered dynamic receivers.
  * `OrderedBroadcast`: Managed in `BroadcastRecord.java`. The list of receivers is sorted in descending order of `priority` (`IntentFilter.getPriority()`).
  * `BroadcastQueueModernImpl` dispatches to each receiver one by one in a serial manner. Only when the preceding receiver finishes calling `onReceive()` (or calls `finishReceiver()` via Binder IPC) does the system dispatch to the next receiver.
  * The `mAbortBroadcast` variable is located in `BroadcastRecord`. When a receiver calls `abortBroadcast()`, the system server immediately marks this flag as `true` and clears the remaining receivers waiting in the queue.
* **Illustrative Code:**
```kotlin
class HighPriorityKeyInterceptor : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == "com.stb.action.GLOBAL_KEY") {
            val keyCode = intent.getIntExtra("KEYCODE", 0)
            if (keyCode == KeyEvent.KEYCODE_PROG_RED) {
                // Consume the event, preventing other launchers/apps from receiving it
                abortBroadcast()
            }
        }
    }
}
```
* **Pitfall:** Calling `abortBroadcast()` in a Standard Broadcast (`context.sendBroadcast()`). This method will throw a `RuntimeException: BroadcastReceiver trying to return result during a non-ordered broadcast`.

---

### Q15. Why are Sticky Broadcasts completely banned (`SecurityException` or deprecated)? What architectural issue in `ActivityManagerService` led to this decision?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * When calling `sendStickyBroadcast()`, the intent is not discarded after dispatching but is stored permanently in a static map `mStickyBroadcasts` in `ActivityManagerService.java` on `system_server`.
  * Any app that registers later (with `registerReceiver(null, filter)`) immediately receives this last Intent.
  * **Core issues:**
    1. **Security:** There is no permission mechanism protecting `mStickyBroadcasts`. Any app can read, modify, or spoof this broadcast.
    2. **Memory Leak:** Sticky broadcasts accumulate in the RAM of `system_server`, never being garbage collected unless `removeStickyBroadcast()` is called.
    3. **Architecture:** The concept of State should be managed via a Service or ContentProvider instead of using the fire-and-forget messaging mechanism of Broadcast.
* **Pitfall:** Attempting to use sticky broadcasts to share network connection status or fiber optic/HDMI plug state on STBs. Always replace with StateFlow in a local process or ContentProvider/Binder interface for multi-process.

---

### Q16. Implicit Broadcast Ban from Android 8.0: What mechanism in the system blocks Static Receivers in the Manifest, and what is the solution on Android TV STBs?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * When an implicit broadcast is sent, `PackageManagerService` (PMS) resolves all receivers in the Manifest.
  * However, `BroadcastQueue` checks if the Intent lacks an Explicit Component (`Intent.setComponent()` or `Intent.setPackage()`), and it will automatically filter out static receivers registered for that action to prevent hundreds of apps from being woken up simultaneously, causing CPU bottlenecks and system lag.
  * The exception list (Whitelisted Broadcasts) is statically defined in AOSP at `frameworks/base/services/core/java/com/android/server/am/BroadcastConstants.java` (e.g., `ACTION_BOOT_COMPLETED`, `ACTION_LOCALE_CHANGED`).
* **Illustrative Code (Solution for sending an explicit broadcast):**
```kotlin
val intent = Intent("com.stb.action.CHANNEL_TUNED").apply {
    // Explicitly specify the target package or component class
    setPackage("com.stb.tv.epg")
}
context.sendBroadcast(intent)
```
* **Pitfall:** Defining a Static Receiver in `AndroidManifest.xml` to receive Intents such as `android.net.conn.CONNECTIVITY_CHANGE`. This receiver will never be triggered on modern Android devices.

---

# TOPIC 5: CONTENTPROVIDER & FILEPROVIDER

### Q17. Lifecycle of ContentProvider: Why does ContentProvider's `onCreate()` run even before `Application.onCreate()`? What is its negative impact on Cold Startup time on STBs?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * In `ActivityThread.java`, when a process is forked and initialized, the `handleBindApplication()` function is executed.
  * Specific execution order:
    1. Create `Application` context: `LoadedApk.makeApplication()`.
    2. Call `Application.attachBaseContext()`.
    3. Install ContentProviders: `installContentProviders(app, data.providers)`. Iterate through all providers in the Manifest and call `ContentProvider.onCreate()`.
    4. Finally, call: `mInstrumentation.callApplicationOnCreate(app)`.
  * **Consequence:** All ContentProviders (including those from third-party libraries such as Firebase, App Startup, WorkManager, LeakCanary) are injected and run `onCreate()` on the **Main Thread** before any line of code in `Application.onCreate()` is executed. If a provider performs SDK initialization, reads SharedPreferences files, or opens an SQLite database, the Cold Startup time of the TV Launcher will skyrocket, causing a black screen issue when turning on the STB.
* **Illustrative Code (Disable auto-initialization):**
```xml
<!-- Remove library auto-init to lazily initialize in a worker thread -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    tools:node="remove" />
```
* **Pitfall:** Performing network calls or blocking Room database initialization directly in `ContentProvider.onCreate()`. The app process will hang right from the initialization phase, resulting in an ANR before displaying any UI frame.

---

### Q18. Memory sharing mechanism of `CursorWindow` in ContentProvider query via Binder IPC. 2MB/4MB limit and reasons for throwing `CursorWindowAllocationException`?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * When calling `ContentProvider.query()`, data is not copied directly via standard Binder parcels, but instead uses anonymous shared memory (`ashmem` via the `CursorWindow.cpp` class in `frameworks/base/libs/androidfw/`).
  * The system server or data-providing process allocates a `CursorWindow` (by default size-limited to **2MB** on older Android versions and **4MB** on modern Android). It writes the SQLite data into this ashmem memory region and transmits only the file descriptor (FD) via Binder to the client process.
  * The client side maps this FD into memory space to read data without the overhead of copying back and forth.
  * If a single row of data (containing channel thumbnail images, blob metadata) is larger than the remaining free capacity of the Window, or if the total number of rows is too large without paging, the system throws `CursorWindowAllocationException`.
* **Illustrative Code (Safe paged reading):**
```kotlin
fun queryEpgPaged(context: Context, offset: Int, limit: Int): List<EpgItem> {
    val bundle = Bundle().apply {
        putInt(ContentResolver.QUERY_ARG_OFFSET, offset)
        putInt(ContentResolver.QUERY_ARG_LIMIT, limit)
        putString(ContentResolver.QUERY_ARG_SORT_COLUMNS, "time_start DESC")
    }
    return context.contentResolver.query(
        EpgContract.CONTENT_URI, null, bundle, null
    )?.use { cursor ->
        // Parse cursor
        emptyList()
    } ?: emptyList()
}
```
* **Pitfall:** Querying an EPG table of 10,000 rows containing Base64 images or overly long text descriptions in a single query `query(uri, null, null, null)`. CursorWindow overflow causes the app to crash irrecoverably.

---

### Q19. Security mechanism and path mapping of `FileProvider`. Why does sharing files via `file://` throw `FileUriExposedException` and how does `FLAG_GRANT_READ_URI_PERMISSION` work in `UriGrantsManagerService`?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * Sharing `file://` URIs exposes the actual private path on disk (`/data/user/0/...`) of the app. The receiving app lacks permission to read this sandbox directory, leading to crashes or exposed security vulnerabilities. Starting with Android 7.0, `StrictMode` blocks this completely and throws `FileUriExposedException`.
  * `FileProvider` transforms the actual file into a URI of the form `content://authority/alias_path/file_name` based on an XML mapping configuration file.
  * When an Intent is invoked with `FLAG_GRANT_READ_URI_PERMISSION`, `UriGrantsManagerService.java` (UGMS) in system_server records a `UriPermission`. It grants temporary permission to access that specific file to the target UID. When the target process performs reverse IPC to read the file via `openFileDescriptor()`, UGMS checks this permission table and allows access.
* **Illustrative Code:**
```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <cache-path name="logs" path="crash_reports/" />
</paths>
```
```kotlin
val file = File(context.cacheDir, "crash_reports/dump.txt")
val contentUri = FileProvider.getUriForFile(context, "com.stb.tv.fileprovider", file)

val intent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_STREAM, contentUri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
context.startActivity(Intent.createChooser(intent, "Send log"))
```
* **Pitfall:** Configuring `<root-path name="root" path="." />` in `file_paths.xml`. This creates a severe Directory Traversal vulnerability, allowing other apps to read the app's entire private storage.

---

### Q20. Technique for transferring extremely large binary payloads (100MB Firmware OTA chunk or DB Dump) between 2 STB apps via ContentProvider without Binder crashes and without consuming memory?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * Using the Pipe streaming mechanism via `ContentProvider.openPipeHelper()`.
  * This method creates a pair of `ParcelFileDescriptor`s connected via a Linux pipe (`pipe()` system call). The ContentProvider acts as the writer, writing data to the pipe output stream on a background worker thread.
  * The Client receives the `ParcelFileDescriptor` acting as the reader, directly reading data from the file descriptor via the input stream without needing to buffer the entire data into the Java Heap and bypassing CursorWindow.
* **Illustrative Code:**
```kotlin
class FirmwareContentProvider : ContentProvider() {
    override fun openFile(uri: Uri, mode: String): ParcelFileDescriptor? {
        val file = File(context!!.filesDir, "ota_chunk.bin")
        return openPipeHelper(uri, "application/octet-stream", null, file) { output, _, _, _, f ->
            FileInputStream(f).use { input ->
                input.copyTo(FileOutputStream(output.fileDescriptor))
            }
        }
    }
    // implement other methods...
    override fun onCreate(): Boolean = true
    override fun query(uri: Uri, p: Array<String>?, s: String?, a: Array<String>?, o: String?): Cursor? = null
    override fun getType(uri: Uri): String? = null
    override fun insert(uri: Uri, v: ContentValues?): Uri? = null
    override fun delete(uri: Uri, s: String?, a: Array<String>?): Int = 0
    override fun update(uri: Uri, v: ContentValues?, s: String?, a: Array<String>?): Int = 0
}
```
* **Pitfall:** Reading the entire `byte[]` of a large file into RAM before sending. On low-spec STB devices (1GB/2GB RAM), this operation will trigger the OOM killer to terminate the process immediately.

---

# TOPIC 6: HANDLER, LOOPER, MESSAGEQUEUE, IDLEHANDLER & SYNC BARRIER

### Q21. What is the data structure of `MessageQueue.java`? Why not use standard FIFO? Which Linux API does the non-blocking / sleep mechanism via `nativePollOnce` use?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * `MessageQueue.java` uses a **Singly-Linked List** data structure, sorted in ascending order by execution timestamp: `Message.when` (based on `SystemClock.uptimeMillis()`).
  * Standard FIFO is not used because messages can be sent with a delay: `sendMessageDelayed()`. The message with the smallest `when` (needs to run earliest) is always at the head of the list (`mMessages`).
  * **Sleep/wake up mechanism:**
    * When calling `MessageQueue.next()`, if the list is empty or the first message has not yet reached its execution time (`when > now`), the thread will sleep without consuming CPU via the native function `nativePollOnce(ptr, nextPollTimeoutMillis)`.
    * At the C++ layer (`android_os_MessageQueue.cpp` and `Looper.cpp`), the system uses **Linux `epoll`** (`epoll_wait()`) and **`eventfd`**.
    * When a new message is enqueued (`enqueueMessage()`) with an earlier execution time than the current head message, it calls `nativeWake(ptr)`. Native code writes a byte to `eventfd`, waking up `epoll_wait()` and returning the UI thread to a runnable state.
* **Illustrative Code (Loop handling essence):**
```java
// Core logic in Looper.loop()
for (;;) {
    Message msg = queue.next(); // Can sleep inside nativePollOnce, freeing CPU core
    if (msg == null) return;
    msg.target.dispatchMessage(msg);
    msg.recycleUnchecked();
}
```
* **Pitfall:** Calling `Looper.myLooper().quit()` on the Main Thread. The system will throw `IllegalStateException: Main thread not allowed to quit`.

---

### Q22. What is a Synchronous Barrier? How is the hidden `postSyncBarrier()` method used by `ViewRootImpl` to prioritize VSYNC/Rendering processing?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * In `MessageQueue.java`, a **Sync Barrier** is essentially a special `Message` with the attribute `target == null`.
  * The `MessageQueue.postSyncBarrier()` method (hidden API) inserts this barrier into the queue ordered by `when`.
  * When the `MessageQueue.next()` function iterates through the queue:
    * If it encounters a message with `target == null`, it knows this is a Barrier.
    * It will **bypass all normal messages (Synchronous Messages)** behind the barrier.
    * It continues scanning the linked list to find and execute only **Asynchronous Messages** (`msg.isAsynchronous() == true`).
  * **Application in `ViewRootImpl.java`:**
    * When needing to draw a frame (`scheduleTraversals()`), `ViewRootImpl` immediately sets a `postSyncBarrier()`.
    * Then it posts an Asynchronous Message containing the Choreographer callback (`mTraversalRunnable`).
    * Thanks to this mechanism, UI rendering commands are guaranteed to jump ahead in the queue, bypassing any pending network callbacks or normal handler tasks, ensuring frames are rendered in time for VSYNC. After drawing finishes, it calls `removeSyncBarrier()`.
* **Illustrative Code:**
```kotlin
// Create an Asynchronous Message to bypass the queue when a Sync Barrier is present
val handler = Handler(Looper.getMainLooper())
val asyncMsg = Message.obtain(handler) {
    // Critical rendering logic
}.apply {
    isAsynchronous = true // Mark as Async message
}
handler.sendMessage(asyncMsg)
```
* **Pitfall:** Abusing or using Reflection to call `postSyncBarrier()` and forgetting to call `removeSyncBarrier()`. The queue will be permanently stuck: all normal messages (including Touch input, D-pad clicks, lifecycle events) will never be dispatched, causing an immediate ANR.

---

### Q23. What is `IdleHandler`? When is the `queueIdle()` method called, and how can it be applied to lazy load heavy components on a TV Launcher?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * `IdleHandler` is a static nested interface in `MessageQueue`.
  * In `MessageQueue.next()`, after traversing the queue and observing that: (1) The queue is completely empty; or (2) The first message has `when > now` (the thread is about to sleep in `nativePollOnce`), the system checks the `mIdleHandlers` list.
  * If the list is not empty, the UI thread takes the opportunity to call `queueIdle()` of each `IdleHandler` before going to sleep.
  * Return value of `queueIdle()`:
    * `true`: Keep this handler; next time the thread is idle, it will continue to be called.
    * `false`: Remove this handler from the list after execution finishes.
* **Illustrative Code:**
```kotlin
class TvHomeActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_tv_home)

        // Take advantage of UI thread idle time to preload EPG Cache
        Looper.myQueue().addIdleHandler {
            preloadNextChannelEpgData()
            false // Run only once upon completing startup
        }
    }
}
```
* **Pitfall:** Writing overly heavy processing code or synchronous I/O in `queueIdle()`. Even though the thread is idle, if you block inside `queueIdle()` for too long (e.g., > 16ms), when VSYNC or a D-pad key event arrives, the UI thread cannot handle it in time, leading to dropped frames (jank) or navigation lag.

---

### Q24. Analyze the classic reference chain causing Memory Leak with non-static `Handler`s in Activities. Why is `WeakReference` alone sometimes still insufficient?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * When declaring an anonymous inner class `Handler` or non-static inner class in an Activity:
    1. The Handler implicitly holds a strong reference to the Activity outer class (`this$0`).
    2. When sending `sendMessageDelayed(msg, 10_000)`: The Message is enqueued into `MessageQueue`.
    3. This `Message` object has a `target` field referencing back to the `Handler` itself.
    4. `MessageQueue` in turn is held by `Looper.getMainLooper()`, which is a static variable that exists permanently throughout the process lifecycle.
    5. **Reference chain (GC Root):** `Looper (Static)` -> `MessageQueue` -> `Message` -> `Handler` -> `Activity`.
    6. If the Activity finishes before 10s, the Garbage Collector cannot reclaim the Activity -> Leak!
  * **Why is `WeakReference` still not enough?**
    * Using `WeakReference` allows the Activity to be reclaimed by the GC.
    * BUT the `Message` and `Runnable` objects themselves still linger in `MessageQueue` until the timeout expires. If that Runnable captures heavy local variables or callbacks, it still leaks those objects and causes a `NullPointerException` crash when attempting to access the dead Activity without careful checks.
* **Illustrative Code (Thoroughly safe handling):**
```kotlin
class SafeTvActivity : AppCompatActivity() {
    private class SafeHandler(activity: SafeTvActivity) : Handler(Looper.getMainLooper()) {
        private val activityRef = WeakReference(activity)
        override fun handleMessage(msg: Message) {
            val activity = activityRef.get() ?: return
            // Safe handling...
        }
    }

    private val safeHandler = SafeHandler(this)

    override fun onDestroy() {
        super.onDestroy()
        // MANDATORY: Clear all remaining messages/runnables from the Queue
        safeHandler.removeCallbacksAndMessages(null)
    }
}
```
* **Pitfall:** Forgetting to call `handler.removeCallbacksAndMessages(null)` in `onDestroy()`. Even with `WeakReference`, messages still occupy queue memory and can trigger unwanted logic after the screen has closed.

---

# TOPIC 7: KOTLIN COROUTINES & FLOW (UNDER THE HOOD)

### Q25. How does the compiler transform a `suspend` function into Java Bytecode? Analyze the CPS (Continuation Passing Style) mechanism, the State Machine variable, and the `label`.

**Answer:**
* **Internal mechanism:**
  * The Kotlin Compiler uses the **CPS (Continuation Passing Style)** technique:
    1. Adds an implicit parameter to the end of the function: `$completion: Continuation<T>`.
    2. The actual return type changes to `Any?` (returning the actual result or the special constant `COROUTINE_SUSPENDED`).
  * Inside the function, the Compiler generates an anonymous class inheriting from `ContinuationImpl`. This class contains:
    * A `label: Int` variable: Stores the current state of the State Machine.
    * Fields storing local variables (spilled variables) to preserve their values across suspensions.
    * An `invokeSuspend()` function containing a `switch(label)` statement:
      * `case 0`: Runs from the beginning to the first suspend point. Before suspending, sets `label = 1`, passes `this` into the next suspend function. If that function returns `COROUTINE_SUSPENDED`, the entire call-stack unwinds, releasing the thread.
      * `case 1`: When the coroutine is resumed, `invokeSuspend()` is called again. Thanks to `label = 1`, the switch jumps directly to the code following the first suspend point, reads the saved local variables, and continues execution.
* **Illustrative code (How the Compiler translates a suspend function):**
```kotlin
// Original function
suspend fun fetchChannelAndEpg(channelId: String): EpgData {
    val channel = api.getChannel(channelId) // suspend point 1
    val epg = api.getEpg(channel.id)       // suspend point 2
    return epg
}

// Bytecode pseudocode after compilation:
fun fetchChannelAndEpg(channelId: String, $completion: Continuation<Any?>): Any? {
    val $continuation = object : ContinuationImpl($completion) {
        var result: Any? = null
        var label: Int = 0
        var L$0: Any? = null // Store channel
        override fun invokeSuspend(_result: Any?): Any? {
            this.result = _result
            this.label = this.label or Int.MIN_VALUE
            return fetchChannelAndEpg(null, this)
        }
    }
    when ($continuation.label) {
        0 -> {
            $continuation.label = 1
            val res = api.getChannel(channelId, $continuation)
            if (res == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
            // If executed synchronously, falls through to case 1
        }
        1 -> {
            val channel = $continuation.result as Channel
            $continuation.L$0 = channel
            $continuation.label = 2
            val res = api.getEpg(channel.id, $continuation)
            if (res == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        2 -> {
            val epg = $continuation.result as EpgData
            return epg
        }
    }
}
```
* **Pitfall:** Calling `Thread.sleep()` inside a `suspend` function. This operation blocks the underlying thread entirely (thread pool worker) instead of triggering the state machine suspension mechanism, completely nullifying the non-blocking advantage of Coroutines.

---

### Q26. Structured Concurrency: Exception Propagation mechanism between `Job` and `SupervisorJob`. Why can writing `CoroutineScope(Dispatchers.IO + SupervisorJob()).launch { ... }` produce an unhandled exception that crashes the app?

**Answer:**
* **Internal mechanism:**
  * **Regular Job:** Bi-directional exception propagation. When a child job fails (throws an exception other than `CancellationException`), it immediately: (1) Cancels itself; (2) Propagates the exception up to the Parent Job; (3) The Parent Job cancels all remaining child jobs; (4) The Parent continues propagating the exception upward.
  * **SupervisorJob:** Uni-directional exception propagation (downward). If a child job fails, the `SupervisorJob` does not cancel itself and does not cancel other children.
  * **Core issue:** `SupervisorJob` is only effective when it is the **direct Parent** of the child coroutines.
  * If using `launch { ... }`, this builder by default installs a mechanism that rethrows uncaught exceptions to the Thread Default Exception Handler if there is no `CoroutineExceptionHandler`.
* **Illustrative code:**
```kotlin
val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())

// DANGEROUS: If child1 crashes, SupervisorJob prevents the scope from dying,
// BUT the exception itself is still not caught and is thrown directly to Thread UncaughtExceptionHandler -> CRASH APP!
scope.launch {
    throw RuntimeException("Network Error!")
}

// CORRECT APPROACH: Add CoroutineExceptionHandler
val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
    Log.e("STB", "Caught safely: ${throwable.message}")
}
scope.launch(exceptionHandler) {
    throw RuntimeException("Network Error!") // Safe, does not crash the app
}
```
* **Pitfall:** Passing `SupervisorJob()` into `withContext(SupervisorJob()) { ... }`. In `withContext`, the `Job` parameter is overridden and enforced under standard structured concurrency rules, causing `SupervisorJob` to lose its effect completely.

---

### Q27. Deep dive `Dispatchers.Main` vs `Dispatchers.Default` vs `Dispatchers.IO`: How does the Coroutines Thread Pool work and why does `Dispatchers.IO` share threads with `Default`?

**Answer:**
* **Internal mechanism:**
  * At the framework layer of `kotlinx.coroutines`, both `Default` and `IO` point to the same instance of **`CoroutineScheduler`** (Work-Stealing Thread Pool).
  * `Dispatchers.Default`: Optimized for CPU-bound tasks. The maximum number of threads equals the number of CPU cores of the STB (typically 4 cores on Amlogic S905X chips). Each worker thread has a local deque (local queue). When out of work, a worker will "steal" tasks from another worker's deque.
  * `Dispatchers.IO`: Optimized for blocking I/O (Disk, Socket). The maximum limit is `max(64, CPU_cores)`.
  * **Sharing mechanism:** When switching from `Default` to `IO` via `withContext(Dispatchers.IO)`, the coroutine does not necessarily have to jump to a completely new OS thread. `CoroutineScheduler` simply transitions the state of the current worker thread from CPU core limit mode to non-core elastic thread pool mode. This minimizes OS Context Switch overhead.
  * `Dispatchers.Main.immediate`: Unlike standard `Dispatchers.Main` (which always calls `handler.post()`), `.immediate` checks beforehand: if the current thread is already the UI Thread, it **executes code synchronously immediately** without going through the MessageQueue, eliminating a 1-frame rendering latency.
* **Pitfall:** Using `Dispatchers.Default` for blocking I/O tasks (such as writing cache files, reading DB). All 4 CPU threads will be blocked, causing all other computational coroutines in the app to suffer from "starvation".

---

### Q28. Comparison of the nature of `SharedFlow` vs `StateFlow`. How do Backpressure and buffer overflow (`BufferOverflow.SUSPEND` vs `DROP_OLDEST`) mechanisms work?

**Answer:**
* **Internal mechanism:**
  * `StateFlow` is essentially a specialized variant of `SharedFlow` (`SharedFlowImpl`):
    * Fixed configuration: `replay = 1`, `extraBufferCapacity = 0`, `onBufferOverflow = BufferOverflow.DROP_OLDEST`.
    * Always has an initial value (`initialValue`).
    * Has an **Equality Conflation** mechanism: If emitting a value where `new == old` (via `Any.equals()`), StateFlow ignores it and does not dispatch to collectors.
  * `SharedFlow`:
    * Does not require an initial value.
    * Can configure `replay >= 0` and `extraBufferCapacity >= 0`.
    * Supports 3 Backpressure strategies:
      * `BufferOverflow.SUSPEND`: The `emit()` function suspends when the buffer is full until the slowest collector frees up buffer space.
      * `BufferOverflow.DROP_OLDEST`: Drops the oldest value in the buffer to make room for the new item (never suspends `emit()`).
      * `BufferOverflow.DROP_LATEST`: Drops the latest value being emitted directly.
* **Illustrative code:**
```kotlin
// Used for One-time Events (Navigation, Show Toast on TV)
private val _navigationEvent = MutableSharedFlow<TvNavDest>(
    replay = 0,
    extraBufferCapacity = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
val navigationEvent = _navigationEvent.asSharedFlow()
```
* **Pitfall:** Using `StateFlow` to emit One-time Events (such as showing a Toast or Dialog). If the screen rotates or recreates, StateFlow will re-emit the last value (`replay = 1`), causing the Toast/Dialog to be redisplayed unintentionally.

---

# TOPIC 8: JETPACK (VIEWMODEL, LIVEDATA/STATEFLOW, ROOM, WORKMANAGER, HILT)

### Q29. By what exact mechanism in AOSP does `ViewModel` survive Configuration Changes? Trace the execution chain from `Activity.java` to `ComponentActivity.java`.

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * In `Activity.java`: When preparing to recreate due to a config change, the system calls the hidden method `retainNonConfigurationInstances()`.
  * `ComponentActivity.java` (AndroidX) overrides this method and packages a `NonConfigurationInstances` object containing the `viewModelStore` field.
  * `ActivityThread.java` sends this object to the system server or retains it in client memory via `ActivityClientRecord.lastNonConfigurationInstances`.
  * After the new `Activity` instance is created via reflection:
    1. `ActivityThread.attach()` reassigns `lastNonConfigurationInstances`.
    2. In `ComponentActivity.onCreate()`, when calling `ViewModelProvider(this).get(...)`, the system checks `ViewModelStoreOwner`.
    3. `ComponentActivity.getViewModelStore()` calls `getLastNonConfigurationInstance()`. If it finds the old instance, it retrieves the exact previous `ViewModelStore` object instead of initializing a new one.
    4. When the Activity is truly killed (`isFinishing == true`), `ComponentActivity.onDestroy()` finally calls `viewModelStore.clear()`, at which point the ViewModel's `onCleared()` methods are executed.
* **Illustrative code of AOSP mechanism:**
```java
// Core logic in ComponentActivity.java
@Override
public final Object onRetainNonConfigurationInstance() {
    Object custom = onRetainCustomNonConfigurationInstance();
    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
        NonConfigurationInstances nc = (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) viewModelStore = nc.viewModelStore;
    }
    if (viewModelStore == null && custom == null) return null;
    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.viewModelStore = viewModelStore;
    return nci;
}
```
* **Pitfall:** Passing `Activity Context` into ViewModel (as a member variable). Because ViewModel outlives the lifecycle of the Activity instance, this old Context variable will retain the entire view hierarchy in memory, causing a critical memory leak.

---

### Q30. Comparison of `LiveData` vs `StateFlow`. Why can `LiveData.postValue()` drop data, and why is it mandatory to use `repeatOnLifecycle` when collecting Flow on the UI?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * **Mechanism of `LiveData.postValue()`:**
    * `LiveData` maintains an `mPendingData` variable and a Runnable sent to the Main Thread via `ArchTaskExecutor`.
    * Uses a synchronized block: If a background thread calls `postValue(1)` and immediately calls `postValue(2)` before the Main Thread has processed the Runnable, the `mPendingData` variable is overwritten with `2`.
    * When the Main Thread wakes up, it only takes the latest value (`2`). Value `1` is completely lost!
  * **Why is `repeatOnLifecycle` needed with StateFlow?**
    * Flow does not have an automatic Android Lifecycle awareness mechanism like LiveData.
    * If using `lifecycleScope.launch { flow.collect { ... } }`, the flow continues collecting and maintains connections even when the Activity is in the `STOPPED` state (app moves to the background or switches to another TV app).
    * This wastes CPU/Network resources and crashes the app if the UI updates after the view has been detached.
    * `repeatOnLifecycle(Lifecycle.State.STARTED)` uses an internal coroutine: it automatically cancels the collecting coroutine when the lifecycle falls below `STARTED`, and automatically restarts a new coroutine when the lifecycle reaches `STARTED` or higher.
* **Illustrative code:**
```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.channelState.collect { channels ->
            channelAdapter.submitList(channels)
        }
    }
}
```
* **Pitfall:** Collecting StateFlow using `launchWhenStarted`. This API only *suspends* the emission process rather than *canceling* the producer, causing the upstream flow (such as GPS, websocket, or EPG polling) to continue running in the background and wasting resources.

---

### Q31. How does Room's `InvalidationTracker` mechanism work at the SQLite layer? Why can a DAO function returning Flow automatically trigger a new query when the table changes?

**Answer:**
* **Internal mechanism:**
  * Room uses the `InvalidationTracker` class.
  * Upon database initialization, Room automatically creates **SQLite Triggers** (`TEMP TRIGGER`) on the observed tables (`room_table_modification_tracker`).
  * When any `INSERT`, `UPDATE`, or `DELETE` statement occurs on the target table, the SQLite trigger automatically inserts a marker flag into the temporary table `room_table_modification_tracker`.
  * When you query returning `Flow<List<T>>`: Room creates a `Flow` emitting the first value, while also registering an `Observer` with `InvalidationTracker`.
  * When the database changes, `InvalidationTracker` queries the temporary table on a background thread. If it detects a change flag, it notifies the Observer. This Observer automatically re-executes the original SQL query and emits the latest result into the Flow stream.
* **Pitfall:** Executing Room queries returning `Flow` with complex `JOIN` statements across multiple large tables. Whenever a component table has any minor change (even if it does not affect the join result), the entire heavy join query will be re-run from scratch, causing STB stutter/lag.

---

### Q32. Architecture of `WorkManager`: How does it abstract between `JobScheduler`, `AlarmManager`, and `BroadcastReceiver`? Where is the guaranteed execution mechanism located?

**Answer:**
* **Internal mechanism:**
  * WorkManager uses a flexible fallback mechanism depending on the Android version:
    * Android 6.0+ (API 23+): Uses `SystemJobScheduler` (communicates directly with AOSP's `JobSchedulerService`).
    * Android < 6.0 or devices without Google Play Services: Falls back to a combination of `SystemAlarmScheduler` (`AlarmManager`) and `BroadcastReceiver`s.
  * **Guaranteed persistence mechanism:**
    * The heart of WorkManager is an internal SQLite database named `androidx.work.workdb`.
    * Whenever `WorkManager.enqueue()` is called, a record containing all metadata, constraints (run only when network is connected, when charging) and state (`ENQUEUED`, `RUNNING`, `SUCCEEDED`) is immediately persisted into `workdb`.
    * Even if the STB device experiences a sudden power cut or a system crash, after rebooting, `RescheduleReceiver` receives the `BOOT_COMPLETED` broadcast, reads `workdb` again, and re-enqueues all unfinished jobs into `JobScheduler`.
* **Pitfall:** Using `WorkManager` for tasks that require second-accurate execution (in-app timing). WorkManager applies the job batching mechanism of `JobScheduler` to save power, and latency can fluctuate from a few minutes up to an hour depending on Doze Mode.

---

# TOPIC 9: VIEW RENDERING (MEASURE/LAYOUT/DRAW, CHOREOGRAPHER, VSYNC, RENDERTHREAD & JANK)

### Q33. Retrace the rendering pipeline from calling `requestLayout()` / `invalidate()` to pixels appearing on TV: Analyze the roles of `ViewRootImpl`, `Choreographer`, `SurfaceFlinger`, and VSYNC.

**Answer:**
* **Internal Mechanism & AOSP Source:**
  1. **Trigger:** `View.requestLayout()` marks the `PFLAG_FORCE_LAYOUT` flag and calls upward through parent views until reaching `ViewRootImpl.java`.
  2. **Schedule:** `ViewRootImpl.scheduleTraversals()` posts a Sync Barrier into the `MessageQueue` and registers to receive the next VSYNC signal with `Choreographer.java` via `DisplayEventReceiver`.
  3. **VSYNC Synchronization:** Hardware emits a VSYNC signal (e.g., 16.6ms cycle for 60Hz TV, 8.3ms for 120Hz TV). `Choreographer` receives the signal and dispatches the `CALLBACK_TRAVERSAL` callback to `ViewRootImpl`.
  4. **Traversals:** `ViewRootImpl.performTraversals()` sequentially executes 3 phases on the UI Thread:
     * `performMeasure()` -> `View.measure()` -> `onMeasure()`.
     * `performLayout()` -> `View.layout()` -> `onLayout()`.
     * `performDraw()` -> `View.draw()` -> `onDraw()` (generates Drawing Commands written to `DisplayList`).
  5. **RenderThread:** The UI Thread hands off (syncs) the `DisplayList` to `RenderThread` (Native C++). RenderThread optimizes commands, communicates with the GPU driver via OpenGL ES or Vulkan to render the frame into a buffer.
  6. **SurfaceFlinger:** RenderThread swaps buffers, enqueuing the buffer into `BufferQueue`. The `SurfaceFlinger` process composites the layers (Status bar, App Window, Video Surface) and pushes data to the Hardware Framebuffer to display on the TV screen via the HDMI port.
* **Pitfall:** Repeatedly calling `requestLayout()` inside `onLayout()` or `onDraw()`. This triggers an infinite layout loop, completely freezing the user interface.

---

### Q34. Core difference between UI Thread and RenderThread (`DrawFrameTask`). Why can the App still drop frames (Jank) even when the UI Thread is completely idle?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * **UI Thread:** Responsible for executing Java/Kotlin code, handling click/remote events, updating the View hierarchy, calculating dimensions, and building the drawing command list (`DisplayListCanvas`).
  * **RenderThread (split since Android 5.0 - `ThreadedRenderer.cpp`):** Receives the `DisplayList`, compiles shaders, uploads textures (Bitmaps) to GPU VRAM, and invokes hardware driver commands.
  * **Causes of Jank even when the UI Thread is idle:**
    1. **GPU Bound:** Shaders are too complex or the STB chip's GPU is too weak to finish rasterizing pixels within 16.6ms.
    2. **Texture Upload Janks:** The UI Thread decodes a huge Bitmap image (e.g., a 4K movie poster). Even though decoding occurs on a worker thread, when RenderThread binds the texture for the first time (`glTexImage2D`), it must upload tens of megabytes over the memory bus to the GPU. This operation blocks RenderThread, causing the next frame to be delayed even if the UI thread has no running tasks.
    3. **Buffer Stuffing / Triple Buffering Exhaustion:** All 3 buffers in `BufferQueue` are locked by the GPU or `SurfaceFlinger`, forcing RenderThread to sleep waiting for an empty buffer.
* **Pitfall:** Loading a full-resolution Bitmap from the internet and displaying it in an ImageView without resizing it to the exact display dimensions (`targetSize`) of the View on TV.

---

### Q35. What is the "Double Taxation" (Unbounded Measure Pass) issue? Why does nesting `LinearLayout`s with `layout_weight="1"` or multi-level `RelativeLayout`s kill performance on weak STB CPUs?

**Answer:**
* **Internal mechanism:**
  * Typically, a View should only be measured once per render cycle.
  * **LinearLayout with `layout_weight`:** To distribute proportions, LinearLayout is forced to execute **2 measurement passes**:
    * Pass 1: Measures the natural size of children without weights.
    * Pass 2: Calculates remaining space and calls `measure()` a second time on all views with `weight` to enforce the new dimensions.
  * **RelativeLayout:** By default, always performs **at least 2 measure passes** across all child views to resolve multi-dimensional constraints (left, right, top, bottom).
  * **Exponential consequence:** If you nest a `LinearLayout (weight)` inside a `RelativeLayout`, the number of measure passes on views at the deepest levels increases exponentially: $2 \times 2 \times 2... = 2^N$ measure passes!
  * On low-cost TV SoCs (quad-core ARM Cortex-A35/A53 clocked at only 1.2GHz), triggering thousands of repeated measurement calculations congests the UI thread, causing severe frame drops when scrolling through movie lists.
* **Pitfall:** Nesting multiple levels of `LinearLayout` with weights instead of using a single flat `ConstraintLayout` layer to arrange the layout.

---

### Q36. What is Overdraw? How can Overdraw be completely eliminated in Video/Media Player applications on Android TV?

**Answer:**
* **Internal mechanism:**
  * Overdraw occurs when the GPU must redraw a pixel at the same screen position multiple times within a single frame (drawing Window background color -> drawing parent Layout background color on top -> drawing child View background color on top).
* **Comprehensive optimization strategies:**
  1. **Remove default Window Background:** Every Activity by default has a gray or black background defined in the Theme (`android:windowBackground`).
  2. **In Custom Views:** Use `canvas.clipRect()` to exclude regions obscured by other components.
  3. **Eliminate overdraw during Video Playback:** During video playback, `SurfaceView` punches a hole in the window for hardware overlay of the video directly. If the player container has a background color set, the GPU draws that color on top of the video surface, causing massive memory bandwidth waste.
* **Illustrative code:**
```kotlin
class TvPlayerActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        // Remove the Window background to prevent the GPU from drawing a useless background layer underneath the Video
        window.setBackgroundDrawable(null)
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_tv_player)
    }
}
```
* **Pitfall:** Setting the `android:background="#000000"` attribute on every Layout from `root` down to child `CardView`s. Enabling "Debug GPU Overdraw" on TV will show the screen entirely red (Overdraw > 4x).

---

# TOPIC 10: JETPACK COMPOSE (UNDER THE HOOD, STATE, SIDE-EFFECTS & STABILITY)

### Q37. How do the "Slot Table" mechanism and `GapBuffer` work in Compose Runtime? How does the Compose compiler transform `@Composable` functions via the IR Transformer?

**Answer:**
* **Internal mechanism:**
  * Compose Runtime does not use a traditional hierarchical tree structure of Object pointers to store UI state, but instead uses a flat array called the **Slot Table** (based on the `Gap Buffer` data structure).
  * `GapBuffer` is a contiguous array with a "gap". When traversing the Composable tree, the read/write pointer sequentially reads/writes values (state, memoized values, groups). When inserting or deleting a node due to recomposition is needed, the gap is moved to that position and expanded/shrunk, allowing insertion/deletion with $O(1)$ complexity.
  * **IR Transformer (Compiler Plugin):**
    * Every function annotated with `@Composable` is rewritten by the compiler:
    * Adds parameters `$composer: Composer` and `$changed: Int` (bitmask).
    * The compiler inserts instructions `composer.startRestartGroup(key)`, `composer.startReplaceableGroup(key)` at the beginning of the function and `composer.endRestartGroup()` at the end of the function.
    * The `$changed` bitmask stores the status of the parameters: if all parameters remain unchanged (compared via `equals`), the composer skips over the entire function body (Skip recomposition).
* **Code illustration (Pseudocode after Compose Compiler compilation):**
```kotlin
// Original function
@Composable fun ChannelTitle(name: String) { Text(name) }

// After Compiler transformation:
fun ChannelTitle(name: String, $composer: Composer, $changed: Int) {
    $composer.startRestartGroup(12345)
    // Check bitmask: If unchanged, then skip
    if ($changed and 0b0001 == 0 && $composer.skipping) {
        $composer.skipToGroupEnd()
    } else {
        Text(name, $composer, ...)
    }
    $composer.endRestartGroup()?.updateScope { nextComposer ->
        ChannelTitle(name, nextComposer, $changed or 0b0001)
    }
}
```
* **Pitfall:** Passing unstable lambdas or directly creating new object instances inside the Composable body without wrapping them in `remember { ... }`. This behavior breaks the `$changed` checking mechanism, causing the function to always recompose continuously.

---

### Q38. The Stability system in Compose: Distinguish between `@Stable`, `@Immutable`, and Unstable. Why is Kotlin's `List<T>` considered Unstable by default, and how to fix it?

**Answer:**
* **Internal mechanism:**
  * The Compose Compiler classifies classes into:
    * **Immutable:** Values never change after creation (e.g., primitive types, `String`). All public properties must be `val` and have Immutable data types.
    * **Stable:** Can change (mutable), but when it changes, it notifies the Compose Runtime (e.g., `MutableState`).
    * **Unstable:** Contains `var` properties, or interfaces/classes for which the Compiler cannot guarantee immutability at compile-time.
  * **The issue with `List<T>`:**
    * In Kotlin, `List` is only an interface. The actual type at runtime could very well be an `ArrayList` (which can be mutated by adding or removing elements).
    * Therefore, the Compose Compiler treats every implementation of `List<T>` as **Unstable**!
    * Consequence: Any Composable that receives a parameter of type `List<T>` will **never skip recomposition**, even if the list data has not changed at all!
* **Code illustration (Complete fix):**
```kotlin
// Approach 1: Use kotlinx.collections.immutable library
@Composable
fun TvChannelRow(channels: PersistentList<ChannelItem>) { ... }

// Approach 2: Wrap in a Data Class annotated with @Immutable
@Immutable
data class ChannelListWrapper(val list: List<ChannelItem>)

@Composable
fun TvChannelRow(wrapper: ChannelListWrapper) { ... }
```
* **Pitfall:** Believing that using a `data class` with all `val` properties is automatically Stable. If it contains a field `val tags: List<String>`, the entire data class will be downgraded to Unstable by the compiler.

---

### Q39. Distinguish the nature and practical Use Cases between Side-Effect APIs: `LaunchedEffect`, `rememberCoroutineScope`, `SideEffect`, and `DisposableEffect` on the Android TV interface.

**Answer:**
* **Internal mechanism:**
  * `LaunchedEffect(keys)`: Launches a coroutine inside the Composition context. The coroutine is automatically cancelled and relaunched when `keys` change, or cancelled when the composable leaves the Composition (Leave the composition). Used for loading data when changing TV channels.
  * `rememberCoroutineScope()`: Returns a `CoroutineScope` bound to the lifecycle of the current Composition point, but is used to launch coroutines from **outside** composable callbacks (such as `onClick`, remote D-pad key press events).
  * `SideEffect`: Runs after **every successful Recomposition**. It is not a coroutine. Used to synchronize Compose state with objects not managed by Compose (e.g., sending analytics logs or updating native C++ variables).
  * `DisposableEffect(keys)`: Similar to `LaunchedEffect` but intended for non-coroutine tasks that require cleanup. Must have an `onDispose { ... }` block. Extremely important for adding/removing FocusListener or releasing MediaPlayer on TV.
* **Code illustration:**
```kotlin
@Composable
fun DpadFocusTracker(viewId: String, onFocusLost: () -> Unit) {
    DisposableEffect(viewId) {
        val listener = View.OnFocusChangeListener { _, hasFocus ->
            if (!hasFocus) onFocusLost()
        }
        // Attach listener...
        onDispose {
            // Cleanup when leaving the screen or viewId changes
            // Avoid listener leak
        }
    }
}
```
* **Pitfall:** Executing side-effects (such as mutating variables, navigation, calling APIs) directly inside the `@Composable` function body without wrapping them in Side-Effect APIs. Because recomposition can be cancelled or run multiple times consecutively, these logics will be incorrectly triggered multiple times.

---

### Q40. Custom Layout in Compose: 3 Phases (Composition -> Measurement/Layout -> Drawing). How to skip Phase 1 & 2 when performing animations on TV?

**Answer:**
* **Internal mechanism:**
  * Compose has 3 phases that run sequentially:
    1. **Composition:** Runs composable functions to build the UI tree (most CPU-intensive).
    2. **Layout:** Performs measurement (measure) and placement (place).
    3. **Drawing:** Draws canvas onto the RenderNode (lightest).
  * If a changed State is read directly in Phase 1, all 3 phases must re-run.
  * If reading State in Phase 2 (Layout), Compose skips Phase 1, re-running only Layout and Drawing.
  * If reading State in Phase 3 (Drawing), Compose **completely skips both Composition and Layout phases**, redrawing only the canvas.
* **Code illustration (Optimizing Focus Scale animation on TV):**
```kotlin
// POOR PERFORMANCE: Reading state in Phase 1 -> Constant recomposition at 60fps
val scale by animateFloatAsState(if (isFocused) 1.2f else 1.0f)
Box(modifier = Modifier.size(100.dp * scale)) // Trigger Recomposition!

// ABSOLUTE OPTIMIZATION: Use Modifier.graphicsLayer reading state in Phase 3
val scale = remember { Animatable(1.0f) }
Box(modifier = Modifier
    .size(100.dp)
    .graphicsLayer { // This lambda is called in the Draw phase!
        scaleX = scale.value
        scaleY = scale.value
    }
)
```
* **Pitfall:** Using `Modifier.offset(x, y)` with directly calculated values instead of using `Modifier.offset { IntOffset(x.value, y.value) }`. The lambda variant allows deferring state reading until the Layout phase, avoiding recomposition.

---

# TOPIC 11: MEMORY LEAK, LEAKCANARY & ANR AT THE APP LAYER

### Q41. Classification of 4 causes of ANR at the App layer. How does the kernel mechanism send the `SIGQUIT` signal to the `Signal Catcher` thread, and how to analyze the `/data/anr/traces.txt` file to find Deadlocks?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * **4 types of ANR:**
    1. *Input Dispatching Timeout:* 5 seconds without responding to D-pad or Touch key events (`InputDispatcher.cpp`).
    2. *Broadcast Timeout:* `BroadcastReceiver.onReceive()` runs longer than 10 seconds (foreground) or 60 seconds (background).
    3. *Service Timeout:* Service lifecycle runs longer than 20 seconds (foreground) or 200 seconds (background).
    4. *ContentProvider Timeout:* `ContentProvider` does not finish publishing within 10 seconds.
  * **Stacktrace Dump Mechanism:**
    * When a timeout is detected, AMS sends a **`SIGQUIT` (Signal 3)** signal to the faulty process via the Linux kernel call `kill(pid, SIGQUIT)`.
    * In the ART runtime, the `Signal Catcher` thread (`art/runtime/signal_catcher.cc`) catches this signal, suspends all other threads in the process, and dumps the entire call stack of each thread to the file `/data/anr/traces.txt`.
  * **How to read traces.txt to find Deadlocks:**
    * Find `tid=1 sysTid=... "main"`: Check whether the state is `BLOCKED`, `WAITING`, or `MONITOR`.
    * If the state is `BLOCKED (on a monitor)`: Look at the line `waiting to lock <0x01234567> held by thread X`.
    * Jump to `thread X` in the file: If `thread X` is also `waiting to lock <0x09876543> held by thread 1 (main)` -> Confirm **Mutual Deadlock**!
* **Pitfall:** Thinking that only 100% CPU causes ANR. In reality, the majority of severe ANRs on Android TV occur when CPU is near 0% because the Main Thread is `BLOCKED` waiting for a lock held by a worker thread executing an SQLite query or Disk I/O.

---

### Q42. The internal mechanism of `LeakCanary`: How does it use `WeakReference`, `ReferenceQueue`, and `AppWatcher` to detect leaks? How do the Hprof Dump and Shark heap analysis processes take place?

**Answer:**
* **Internal mechanism:**
  1. **Monitoring:** `AppWatcher` hooks into `ActivityLifecycleCallbacks.onActivityDestroyed()`.
  2. **Tracking:** When `onDestroy()` is called, it wraps the Activity object in a `KeyedWeakReference` (inheriting from `WeakReference`) and attaches a `ReferenceQueue`.
  3. **Waiting:** It posts a background task to check after 5 seconds (via `IdleHandler` or Handler delay).
  4. **Checking:** The system forces GC to run (`Runtime.getRuntime().gc()`). If the Activity has been cleared, its `WeakReference` will be enqueued in the `ReferenceQueue`. If after polling the `ReferenceQueue`, that reference is **not found**, LeakCanary determines that this object has been "Retained" (illegally retained).
  5. **Dump & Analyze:** When the number of retained objects exceeds the threshold (default is 5), it calls `Debug.dumpHprofData()` to write the heap to an `.hprof` file. The **Shark** library (an Hprof parser written in Kotlin) reads this file, reconstructs the object graph, and finds the shortest path from a **GC Root** to the leaked object (**Shortest Retained Path**).
* **Pitfall:** Enabling LeakCanary on Production or Performance Benchmark builds. The `Debug.dumpHprofData()` process will "freeze" the app's entire process for 2 to 10 seconds on weak STB hardware, directly causing fake ANRs.

---

### Q43. Analysis of 3 "classic" Memory Leak issues most specific to the Android TV / Set-top Box operating system.

**Answer:**
* **3 Specific issues:**
  1. **Leak Static TV Focus Highlight:** To customize the glowing border (Focus Border) when the remote D-pad navigates to it, developers often use a Custom Drawable and unintentionally store it in a Singleton or static variable. Because the Drawable holds a reference to `Resources`, and `Resources` in turn holds a reference to `Context` (Activity), this leaks the entire Activity.
  2. **ExoPlayer Leak via `SurfaceView` / `TextureView`:** `SurfaceView` contains a `SurfaceControl` directly connected to `SurfaceFlinger`. If an Activity Context is passed into the Player singleton and `player.clearVideoSurfaceView()` is not called when the Activity is destroyed, `SurfaceView` and `SurfaceSession` will never be GC-ed, exhausting the STB's Native Graphic Memory.
  3. **Leanback `Presenter.ViewHolder` captures Presenter/Adapter Context:** In the Leanback library, if a listener (such as focus change, click) is created as an anonymous inner class inside a `ViewHolder` and references variables of `Presenter` or `ArrayObjectAdapter`, the entire list of rows items will be anchored in memory even after the fragment has been popped from the back stack.
* **Code illustration (Fixing the ExoPlayer leak):**
```kotlin
override fun onStop() {
    super.onStop()
    // Detach SurfaceView from Player before Activity is destroyed
    exoPlayer.clearVideoSurface()
}
```
* **Pitfall:** Using `TextureView` instead of `SurfaceView` to play video on Android TV. `TextureView` consumes an additional 1-3 internal graphic RAM buffers (costing tens of extra MBs of RAM) and increases GPU compositing cost by 30%, easily causing Out Of Memory (OOM) on 1GB RAM STBs.

---

# TOPIC 12: ANDROID TV (LEANBACK, FOCUS NAVIGATION, TIF & LAUNCHER/HOME)

### Q44. Focus Navigation Engine on Android TV: When the user presses a D-pad key, how does `FocusFinder.java` search for the next View? Directional Search algorithm mechanism and how to intervene?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * When pressing a D-pad key, `ViewRootImpl.java` receives `KeyEvent` from the Linux kernel driver via `InputEventReceiver` and dispatches it into the View hierarchy via `dispatchKeyEvent()`.
  * If no view consumes the key, the system calls `focusSearch(direction)` (where `direction` is `FOCUS_UP`, `FOCUS_DOWN`, `FOCUS_LEFT`, `FOCUS_RIGHT`).
  * `ViewGroup.java` delegates to the singleton **`FocusFinder.java`** (AOSP `frameworks/base/core/java/android/view/FocusFinder.java`).
  * **Directional Search Algorithm:**
    1. `FocusFinder` retrieves the geometric coordinates (Rect) of the View currently holding focus on the screen.
    2. Scans across all available focusable views within the same window.
    3. Calculates the geometric distance weight using the formula:
       $$Distance = 13 \times MajorAxisDistance + MinorAxisDistance$$
       *(The distance along the major axis of movement is multiplied by a factor of 13 to prioritize views that are directly aligned before considering diagonal views).*
    4. Whichever View has the smallest `Distance` will be selected to receive focus (`view.requestFocus()`).
* **Code illustration (Overriding the Focus flow using code and XML):**
```xml
<!-- Force D-pad Right key to jump exactly to the desired Button -->
<Button
    android:id="@+id/btn_play"
    android:nextFocusRight="@+id/btn_settings"
    android:focusable="true" />
```
```kotlin
// Manually intervene via code when the default algorithm jumps incorrectly
btnPlay.setOnKeyListener { v, keyCode, event ->
    if (event.action == KeyEvent.ACTION_DOWN && keyCode == KeyEvent.KEYCODE_DPAD_DOWN) {
        customTargetView.requestFocus()
        true // Event consumed, prevent FocusFinder from intervening
    } else {
        false
    }
}
```
* **Pitfall:** A View with `visibility = View.GONE` or `isFocusable = false` lies in the path of the D-pad. `FocusFinder` will get stuck or jump erratically to a completely different component in the corner of the screen. Always set `android:focusable="true"` explicitly for custom views.

---

### Q45. Analysis of the Leanback Library architecture: The coordination mechanism between `BrowseSupportFragment`, `ArrayObjectAdapter`, `Presenter`, and `ItemBridgeAdapter` to reuse Views during D-pad navigation.

**Answer:**
* **Internal mechanism:**
  * The Leanback library is specially designed for 10-foot experience navigation:
    * **`Presenter`:** Equivalent to a mini-`Adapter` for a specific item type. Responsible for creating the `ViewHolder` (`onCreateViewHolder`) and binding data (`onBindViewHolder`). Does not contain list logic.
    * **`ArrayObjectAdapter`:** Stores the dataset and maps each data type (Model) to a corresponding `Presenter` via `PresenterSelector`.
    * **`ItemBridgeAdapter`:** Acts as an intermediary bridge converting from Leanback's `ArrayObjectAdapter` to the traditional `RecyclerView.Adapter` standard.
    * **`BrowseSupportFragment`:** Manages two parts: `HeadersSupportFragment` (category column on the left) and `RowsSupportFragment` (list of horizontal `ListRow`s on the right).
  * **D-pad Optimization:** `RowsSupportFragment` uses `HorizontalGridView` (inherits from `RecyclerView`). Unlike phones, it has a Focus Alignment mechanism: keeping the currently focused item fixed at a certain percentage position on the screen (e.g., always 15% from the left margin), while the list scrolls smoothly beneath that item.
* **Code illustration:**
```kotlin
val cardPresenter = CardPresenter()
val listRowAdapter = ArrayObjectAdapter(cardPresenter).apply {
    add(Movie("Inception"))
    add(Movie("Interstellar"))
}
val headerItem = HeaderItem(0, "Science Fiction Movies")
val rootAdapter = ArrayObjectAdapter(ListRowPresenter()).apply {
    add(ListRow(headerItem, listRowAdapter))
}
browseSupportFragment.adapter = rootAdapter
```
* **Pitfall:** Calling `notifyDataSetChanged()` on `ArrayObjectAdapter` while scrolling with the D-pad remote. This operation clears the current focus state, causing the focus cursor to jump back to the first element or disappear onto the header menu bar.

---

### Q46. TV Input Framework (TIF) architecture: Analysis of the workflow between `TvView`, `TvInputManagerService`, and `TvInputService.Session`. How does the UI overlay display independently from the hardware video surface?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * TIF is the standard AOSP architecture for displaying broadcast TV (DVB-T2/C/S2, IPTV, HDMI In):
    1. **`TvView`:** Resides in the Live TV application (Client). It embeds a `SurfaceView` to display the video stream and communicates with the system via Binder IPC.
    2. **`TvInputManagerService` (TIMS):** System server service that manages all hardware tuners and TV input apps.
    3. **`TvInputService`:** Background service implemented by the device manufacturer or IPTV app.
  * **Channel Tuning Flow:**
    * `TvView.tune(uri)` -> Sends IPC to TIMS.
    * TIMS requests `TvInputService` to create a `TvInputService.Session`.
    * `Session.onSetSurface(surface)`: System passes the `Surface` object directly (from `TvView`'s `SurfaceView`) down to the `Session`.
    * Session passes this Surface to the decoding hardware (Hardware Decoder or MediaCodec).
  * **Independent UI Overlay Mechanism:**
    * `Session.onCreateOverlayView()` allows returning a `View` (containing EPG, subtitles, audio track selector).
    * `TvInputService` attaches this View to a separate `SurfaceControl` that sits on top of the Video Surface. Thanks to this, redrawing the UI overlay (at 60fps) does not interfere with or disrupt the direct hardware render pipeline of the underlying video stream.
* **Pitfall:** Rendering subtitles or player icons directly onto the same `Canvas` as the Video Surface. Any delay during UI drawing will cause stuttering in the hardware video stream.

---

### Q47. For an application to become a Custom Android TV Launcher (Home App), how must the Intent Filter be configured accurately in the Manifest? How does the Channels & Programs API (`TvContractCompat`) mechanism work?

**Answer:**
* **Internal Mechanism & AOSP Source:**
  * To be recognized by the system as a standard TV Launcher and replace the default home screen when the Home key is pressed on the remote:
```xml
<activity
    android:name=".TvLauncherActivity"
    android:launchMode="singleTask"
    android:clearTaskOnLaunch="true"
    android:stateNotNeeded="true"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <!-- Mark as Home screen -->
        <category android:name="android.intent.category.HOME" />
        <category android:name="android.intent.category.DEFAULT" />
        <!-- MANDATORY to display on Android TV -->
        <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
    </intent-filter>
</activity>
```
  * **Channels & Programs API (`TvContractCompat`):**
    * Allows applications to publish content channels and recommended videos (Programs) directly to the Android TV Home Screen.
    * Communicates via the system's `ContentProvider` (`android.media.tv.provider`).
    * The app uses `PreviewChannelHelper` to:
      1. Create a `PreviewChannel` (e.g., "Trending Movies Channel").
      2. Insert `PreviewProgram`s into that channel (containing title, video trailer URL, thumbnail, deep link Intent).
      3. Write data to the system database via `TvContractCompat`. The Android TV Home app will query this data and display it as horizontal rows on the home screen.
* **Code illustration (Publish Preview Program):**
```kotlin
val program = PreviewProgram.Builder()
    .setChannelId(channelId)
    .setTitle("New Movies 2024")
    .setDescription("Watch now on the STB App")
    .setPosterArtUri(Uri.parse("https://stb.com/poster.jpg"))
    .setIntentUri(Uri.parse("stb://player?videoId=999"))
    .setType(TvContractCompat.PreviewPrograms.TYPE_MOVIE)
    .build()

val programUri = context.contentResolver.insert(
    TvContractCompat.PreviewPrograms.CONTENT_URI,
    program.toContentValues()
)
```
* **Pitfall:** Forgetting to declare `android.intent.category.DEFAULT` alongside `CATEGORY_HOME`. When pressing the Home button on the remote, the system will not display the Resolver Activity for the user to choose the new Launcher, but will jump directly back to the manufacturer's default Launcher.

