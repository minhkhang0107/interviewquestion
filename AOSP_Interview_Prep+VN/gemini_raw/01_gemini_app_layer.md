Dưới đây là bộ câu hỏi và câu trả lời phỏng vấn chuyên sâu dành cho vị trí **Senior Android Engineer (Android 14, Android TV / Set-top Box Platform)**, tiếp cận từ góc nhìn của Platform/Framework Engineer.

---

# CHỦ ĐỀ 1: ACTIVITY LIFECYCLE, CONFIGURATION CHANGE, PROCESS DEATH & SAVEDSTATE

### Q1. Phân biệt bản chất giữa Configuration Change và Low Memory Process Death. Cơ chế AOSP xử lý khôi phục trạng thái trong hai trường hợp này khác nhau như thế nào?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * **Configuration Change:** Khi cấu hình phần cứng thay đổi (VD: STB đổi độ phân giải HDMI qua `DisplayManagerService`), `ActivityTaskManagerService` (ATMS) gửi yêu cầu relaunch tới client qua Binder. Trong `ActivityThread.java`, phương thức `handleRelaunchActivityLocally()` được gọi. `ActivityRecord` trên system server vẫn tồn tại nguyên vẹn. Hệ thống giữ lại đối tượng `ActivityClientRecord.lastNonConfigurationInstances` để chuyển giao trực tiếp ViewModelStore và các retained instance sang `Activity` instance mới mà không cần serialize qua Binder.
  * **Process Death:** Xảy ra khi STB rơi vào tình trạng thiếu RAM (`LowMemoryKiller` - LMKD kill process ở background). Toàn bộ process của App bị hủy sạch, heap memory bị xóa. Tuy nhiên, `ActivityRecord` trong ATMS vẫn lưu `Bundle` trạng thái mà App đã gửi lên system server trước đó thông qua `onSaveInstanceState()` (gọi Binder transaction `activityStopped()` -> `ActivityRecord.saveState()`). Khi user quay lại, ATMS fork một process mới qua Zygote, tạo `ActivityThread`, và truyền `savedInstanceState` (Bundle) vào `Activity.onCreate()`. ViewModel trong trường hợp này bị hủy hoàn toàn trừ khi sử dụng `SavedStateHandle`.
* **Code minh họa (SavedStateHandle):**
```kotlin
class ChannelDetailViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    // Sống sót qua cả Config Change lẫn Process Death
    val channelId: StateFlow<String> = savedStateHandle.getStateFlow("KEY_CHANNEL_ID", "default_ch_01")

    fun updateChannel(id: String) {
        savedStateHandle["KEY_CHANNEL_ID"] = id
    }
}
```
* **Pitfall:** Lưu Bitmap, danh sách EPG hàng nghìn items vào `onSaveInstanceState()`. Bundle này được gửi qua IPC Binder tới `system_server`. Binder buffer của toàn process có giới hạn chỉ **1MB** (chia sẻ cho mọi transaction đang chạy). Vượt quá ngưỡng này sẽ ném `TransactionTooLargeException` gây crash ngay lập tức.

---

### Q2. Trên Android TV STB, khi người dùng cắm/rút cáp HDMI (Hotplug) hoặc TV tự động switch Display Mode (4K 60Hz sang 1080p 50Hz/HDR), Lifecycle của Activity bị ảnh hưởng thế nào và cách xử lý triệt để?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Khi HDMI display thay đổi, `DisplayManagerService` phát hiện EDID mới từ phần cứng và dispatch event tới `WindowManagerService` (WMS) và ATMS. Cấu hình `Configuration` hệ thống bị thay đổi các trường: `densityDpi`, `screenWidthDp`, `screenHeightDp`, `smallestScreenWidthDp`, và đôi khi là `colorMode` (khi bật HDR).
  * ATMS duyệt qua `Task` hiện tại, gọi `ActivityRecord.ensureActivityConfiguration()` và xác định xem Activity có tự handle các config này hay không (`ActivityRecord.canHandleConfigChanges()`). Nếu không, Activity sẽ bị recreate (`onDestroy()` -> `onCreate()`).
* **Code minh họa:**
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
    // Tự xử lý: tính toán lại UI scale hoặc SurfaceView aspect ratio
    playerPresenter.relayoutSurfaces(newConfig.screenWidthDp, newConfig.screenHeightDp)
}
```
* **Pitfall:** Quên khai báo `smallestScreenSize` hoặc `screenLayout`. Trên một số dòng SoC Android TV (Amlogic, Realtek), switch độ phân giải HDMI làm thay đổi `smallestScreenWidthDp`. Nếu thiếu flag này, dù đã khai báo `screenSize`, Activity vẫn bị destroy/recreate, dẫn đến hiện tượng rách hình hoặc ExoPlayer bị gián đoạn giải mã phần cứng (re-instantiate MediaCodec).

---

### Q3. Phân biệt `isFinishing()` và `isChangingConfigurations()` trong `onDestroy()`. Tại sao điều này sống còn đối với quản lý tài nguyên phần cứng (Codec, Tuner, ExoPlayer) trên STB?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Trong `Activity.java`, biến `mFinished` được set thành `true` khi gọi `finish()`, hoặc user nhấn nút Back. Hàm `isFinishing()` trả về giá trị này.
  * Khi xảy ra Configuration Change, `ActivityThread.java` gọi `handleRelaunchActivity()`, biến `mChangingConfigurations` trong Activity được gán là `true`. Hàm `isChangingConfigurations()` trả về trạng thái này.
* **Code minh họa:**
```kotlin
override fun onDestroy() {
    super.onDestroy()
    if (isFinishing) {
        // Activity thực sự bị kill hoàn toàn: Giải phóng toàn bộ phần cứng
        exoPlayer.release()
        hardwareTunerSession.close()
    } else if (isChangingConfigurations) {
        // Chỉ là recreate do HDMI mode switch: giữ lại ExoPlayer session qua NonConfigurationInstances
        // hoặc detach SurfaceView để tránh giải phóng Codec nặng nề
        exoPlayer.clearVideoSurface()
    }
}
```
* **Pitfall:** Gọi `exoPlayer.release()` vô điều kiện trong `onDestroy()`. Khi HDMI hotplug xảy ra, player giải phóng MediaCodec decoder, sau đó `onCreate()` khởi tạo lại từ đầu, gây hiện tượng đen màn hình (black screen) từ 1-3 giây và gián đoạn âm thanh HDMI Passthrough.

---

### Q4. Cơ chế hoạt động của `SavedStateRegistry` trong `androidx.activity.ComponentActivity`. Làm thế nào để một Custom View hoặc Component độc lập tự khôi phục state mà không cần phụ thuộc vào callback của Activity?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * `ComponentActivity` implement `SavedStateRegistryOwner`. Class `SavedStateRegistryController` quản lý một thể hiện của `SavedStateRegistry`.
  * `SavedStateRegistry` lưu trữ một map các `SavedStateProvider`. Khi Activity kích hoạt `onSaveInstanceState(outState)`, nó gọi `SavedStateRegistry.performSave(outState)`. Hệ thống lặp qua tất cả các provider đã đăng ký, thu thập `Bundle` của từng component và đóng gói vào một Bundle tổng (`androidx.lifecycle.BundlableSavedStateRegistry.key`).
  * Khi recreate, `performRestore(savedInstanceState)` giải nén Bundle con cho từng component dựa theo `key`.
* **Code minh họa:**
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
* **Pitfall:** Đăng ký trùng `key` trong `SavedStateRegistry` sẽ ném ra `IllegalArgumentException("SavedStateProvider with the given key is already registered")`. Ngoài ra, nếu gọi `consumeRestoredStateForKey()` sau khi lifecycle state của registry đã hoàn tất khởi tạo thì dữ liệu phục hồi sẽ bị mất (`null`).

---

# CHỦ ĐỀ 2: LAUNCHMODE, TASKAFFINITY, INTENT FLAGS & BACK STACK

### Q5. Phân tích sự khác biệt giữa `singleTask`, `singleInstance`, và `singleInstancePerTask` (Android 12+). Khi nào thì Back Stack bị phân nhánh hoặc gộp Task?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Hệ thống quản lý thông qua `Task.java` và `ActivityStarter.java` trong `services/core/.../wm/`.
  * `singleTask`: Nếu chưa có task nào mang `taskAffinity` tương ứng, một Task mới được tạo và Activity làm root. Nếu Task đã tồn tại, hệ thống đưa Task đó lên foreground và gọi `onNewIntent()`. Tất cả các Activity nằm đè lên trên `singleTask` Activity đó mặc định sẽ bị pop khỏi stack (finish) để đưa `singleTask` Activity lên top.
  * `singleInstance`: Tương tự `singleTask`, nhưng hệ thống đảm bảo Task chứa nó **chỉ có duy nhất một Activity này**. Mọi Activity khác được launch từ nó sẽ bị đẩy sang một Task khác (sử dụng taskAffinity mặc định của app đó).
  * `singleInstancePerTask` (A12+): Cho phép tạo nhiều instance cùng loại nhưng mỗi instance bắt buộc phải nằm ở Root của một Task riêng biệt (thường kết hợp với multi-window hoặc multi-display trên TV).
* **Code minh họa:**
```xml
<!-- Deep link Player mở từ Launcher hoặc Voice Command -->
<activity 
    android:name=".PlayerActivity"
    android:launchMode="singleTask"
    android:taskAffinity="com.stb.tv.playback" />
```
* **Pitfall:** Dùng `singleInstance` cho màn hình thanh toán hoặc cài đặt phụ. Khi từ màn hình đó mở tiếp một màn hình con, màn hình con bị văng sang task khác. Nhấn phím Back trên remote TV sẽ gây cảm giác khó hiểu cho người dùng vì không quay về màn hình trước đó mà nhảy thẳng về màn hình của task khác hoặc Home.

---

### Q6. Khi kết hợp `FLAG_ACTIVITY_NEW_TASK` và `FLAG_ACTIVITY_CLEAR_TOP`, cơ chế xử lý trong `ActivityStarter.java` diễn ra thế nào? Nó khác gì với `FLAG_ACTIVITY_CLEAR_TASK`?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Trong `ActivityStarter.java` (AOSP `frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java`), khi gặp `FLAG_ACTIVITY_NEW_TASK`: Hệ thống tìm kiếm Task có affinity trùng khớp.
  * Nếu kết hợp thêm `FLAG_ACTIVITY_CLEAR_TOP`:
    * Hệ thống scan trong Task target xem Activity đó đã tồn tại chưa (`Task.findActivityInHistory()`).
    * Nếu tìm thấy: Hệ thống hủy (finish) toàn bộ Activity nằm phía trên nó. Mặc định, nếu Activity đích không có `launchMode="singleTop"` hoặc không có cờ `FLAG_ACTIVITY_SINGLE_TOP`, chính bản thân Activity đích cũng bị finish và recreate lại!
    * Nếu có `FLAG_ACTIVITY_SINGLE_TOP` đi kèm: Activity đích không bị recreate mà nhận intent mới qua `onNewIntent()`.
  * `FLAG_ACTIVITY_CLEAR_TASK`: Bắt buộc phải đi cùng `FLAG_ACTIVITY_NEW_TASK`. Hệ thống xóa trắng toàn bộ các Activity đang tồn tại trong Task đó (kể cả Activity đích) và tạo mới một instance làm root activity.
* **Code minh họa:**
```kotlin
// Quay về Home Activity và giữ nguyên instance, clear toàn bộ màn hình con
val intent = Intent(context, TvHomeActivity::class.java).apply {
    flags = Intent.FLAG_ACTIVITY_NEW_TASK or 
            Intent.FLAG_ACTIVITY_CLEAR_TOP or 
            Intent.FLAG_ACTIVITY_SINGLE_TOP
}
context.startActivity(intent)
```
* **Pitfall:** Sử dụng `FLAG_ACTIVITY_CLEAR_TOP` mà không có `FLAG_ACTIVITY_SINGLE_TOP` (hoặc không khai báo `singleTop`/`singleTask` trong manifest). Hậu quả: Activity gốc bị destroy và tạo lại từ đầu, làm gián đoạn các luồng dữ liệu đang hiển thị và trigger lại toàn bộ animation mở màn hình.

---

### Q7. Lỗ hổng bảo mật StrandHogg (Task Hijacking) liên quan thế nào đến `taskAffinity`? Cơ chế phòng ngừa trên Android 14 là gì?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Mặc định, tất cả các Activity trong cùng một package có cùng một `taskAffinity`. Khi một app độc hại khai báo một Activity có `taskAffinity` trùng với app ngân hàng/truyền hình của nạn nhân và set `allowTaskReparenting="true"` hoặc launch với `FLAG_ACTIVITY_NEW_TASK`.
  * Khi nạn nhân mở app chính chủ, Activity của app độc hại sẽ được đưa lên đỉnh của Task nạn nhân, tạo giao diện giả để lừa người dùng nhập thông tin đăng nhập/mã OTP.
  * Từ Android 14, `ActivityRecord.java` và `Task.java` siết chặt `taskAffinity`. Flag `android:allowTaskReparenting` bị hạn chế tối đa và framework mặc định chặn việc gộp task giữa các UID khác nhau trừ khi được cấp quyền hệ thống hoặc cùng signing key.
* **Code minh họa (Phòng ngừa):**
```xml
<!-- Khóa không cho reparent và reset affinity cho các màn hình nhạy cảm -->
<activity
    android:name=".auth.TvLoginActivity"
    android:taskAffinity=""
    android:allowTaskReparenting="false"
    android:launchMode="singleTask"
    android:exported="false" />
```
* **Pitfall:** Để `exported="true"` kết hợp với `taskAffinity` mặc định mà không kiểm tra caller UID qua `getCallingPackage()` hoặc `Binder.getCallingUid()`.

---

### Q8. Khi người dùng bấm vào Search Result từ Android TV Global Voice Search (Google Assistant) hoặc Live Channels EPG để vào Detail Screen, làm sao tạo "Synthetic Back Stack" chuẩn để khi bấm Back không bị thoát ra Home?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Khi mở từ bên ngoài, Detail Screen sẽ được đẩy vào một Task mới hoặc Task của Google Assistant. Nếu bấm Back, người dùng sẽ bị văng ra Assistant thay vì quay về màn hình danh sách kênh (Channel List) của app.
  * Lớp `TaskStackBuilder.java` (AOSP `frameworks/base/core/java/android/app/TaskStackBuilder.java`) cung cấp khả năng dựng lại một chuỗi back stack ảo dựa trên khai báo `android:parentActivityName` trong `AndroidManifest.xml`.
  * `TaskStackBuilder.startActivities()` hoặc `getPendingIntent()` sẽ tạo ra một chuỗi các `Intent`. Khi thực thi, `ActivityStarter` sẽ khởi chạy parent activity trước rồi mới đẩy child activity lên top.
* **Code minh họa:**
```kotlin
fun createPendingIntentForVoiceSearch(context: Context, channelId: String): PendingIntent {
    val detailIntent = Intent(context, ChannelDetailActivity::class.java).apply {
        putExtra("EXTRA_CHANNEL_ID", channelId)
    }
    
    return TaskStackBuilder.create(context).run {
        // Thêm tất cả parent activities vào stack dựa theo AndroidManifest.xml
        addNextIntentWithParentStack(detailIntent)
        getPendingIntent(
            0,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )!!
    }
}
```
* **Pitfall:** Thiếu cờ `PendingIntent.FLAG_IMMUTABLE` trên Android 12+ dẫn đến crash `IllegalArgumentException`. Thứ hai, nếu quên khai báo `android:parentActivityName` trong Manifest, `addNextIntentWithParentStack` chỉ đẩy duy nhất một Intent vào stack, Back stack vẫn bị rỗng.

---

# CHỦ ĐỀ 3: SERVICE (STARTED, BOUND, FOREGROUND & BACKGROUND LIMITS A8-A14)

### Q9. Cơ chế trigger ANR của Service (`SERVICE_TIMEOUT`) diễn ra ở đâu trong system_server? Tại sao code chạy trên worker thread của Service vẫn có thể bị ANR?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Quản lý bởi `ActiveServices.java` (trong `com.android.server.am`).
  * Khi một Service được start/bound, `ActiveServices.bumpServiceExecutingLocked()` post một delayed message tới `Handler` của `ActivityManagerService` (`SERVICE_TIMEOUT_MSG`):
    * Foreground Service timeout: 20 giây.
    * Background Service timeout: 200 giây.
  * Khi client xử lý xong lifecycle (`onCreate()`, `onStartCommand()`, hoặc `onBind()`), nó gửi IPC Binder `serviceDoneExecuting()` về system server để remove message này khỏi queue.
  * Nếu main thread của App bị nghẽn (do lock contention, SQLite write trên main thread...), `ActivityThread.handleServiceArgs()` không được gọi đúng hạn, message timeout trong AMS kích hoạt và gọi `AppErrors.appNotResponding()`.
* **Code minh họa:**
```kotlin
class BackgroundSyncService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // DÙ chạy background thread:
        Thread {
            doHeavyTuning()
        }.start()
        
        // NHƯNG nếu trước khi return, bạn gọi một blocking call trên Main Thread:
        runBlocking { 
            database.clearAllTables() // -> Block main thread -> ANR ServiceTimeout!
        }
        return START_NOT_STICKY
    }
    override fun onBind(intent: Intent?): IBinder? = null
}
```
* **Pitfall:** Lầm tưởng Service tự động chạy trên worker thread. Service mặc định chạy trên **Main (UI) Thread** của App Process. Bất kỳ blocking operation nào trong `onCreate()` hoặc `onStartCommand()` đều kích hoạt timer ANR 20s.

---

### Q10. Android 14 siết chặt `foregroundServiceType`. Nếu một app STB chạy ngầm làm nhiệm vụ stream P2P TV hoặc đồng bộ EPG thì phải đáp ứng những yêu cầu kỹ thuật nào? Điều gì xảy ra nếu vi phạm?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Trên Android 14 (API 34), `ActiveServices.startForegroundServiceDelegateLocked()` kiểm tra nghiêm ngặt `foregroundServiceType`.
  * Bắt buộc: (1) Khai báo permission đặc thù (VD: `FOREGROUND_SERVICE_DATA_SYNC`, `FOREGROUND_SERVICE_MEDIA_PLAYBACK`); (2) Khai báo `android:foregroundServiceType` trong Manifest; (3) Truyền đúng type đó vào hàm `startForeground(id, notif, type)`.
  * Nếu thiếu type hoặc type không khớp với Manifest, hệ thống ném ra `MissingForegroundServiceTypeException` hoặc `SecurityException` ngay lập tức.
  * Riêng type `dataSync`, Android 14 áp dụng runtime timeout (tối đa 6 tiếng). Quá thời gian này, AMS gọi callback timeout hoặc stop service.
* **Code minh họa:**
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
// Android 14 bắt buộc truyền type vào hàm startForeground
ServiceCompat.startForeground(
    this,
    NOTIFICATION_ID,
    notification,
    ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK
)
```
* **Pitfall:** Gọi `startForeground()` mà không truyền type hoặc truyền `FOREGROUND_SERVICE_TYPE_NONE` trên Android 14. App sẽ crash với `IllegalArgumentException: default fgs type is not allowed`.

---

### Q11. Khi một Service vừa được `startService()` vừa được `bindService(..., BIND_AUTO_CREATE)`, lifecycle của nó kết thúc khi nào? Cơ chế quản lý tham chiếu trong `ServiceRecord.java` ra sao?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Trong `ActiveServices.java`, mỗi Service được đại diện bằng một `ServiceRecord.java`.
  * `ServiceRecord` duy trì hai trạng thái độc lập:
    * `startRequested = true`: Khi có ít nhất một lệnh `startService()`. Service muốn sống cho đến khi gọi `stopSelf()` hoặc `stopService()`.
    * `bindings`: Một map chứa danh sách các `AppBindRecord` và `ConnectionRecord`. Khi có client bind qua `bindService()`, reference counter tăng lên.
  * **Điều kiện hủy:** Service chỉ thực sự bị gọi `onDestroy()` khi và chỉ khi:
    1. Đã được yêu cầu stop: `stopSelf()` hoặc `stopService()` được gọi (`startRequested = false`).
    2. **VÀ** toàn bộ client đã unbind (`bindings.isEmpty() == true`).
* **Code minh họa:**
```kotlin
// Client 1 (PlaybackManager)
context.startService(serviceIntent) // Service sống độc lập

// Client 2 (UiController)
context.bindService(serviceIntent, connection, Context.BIND_AUTO_CREATE) // Lấy Binder điều khiển

// Khi UiController thoát:
context.unbindService(connection) // Service VẪN CHẠY vì startRequested vẫn là true

// Khi bài hát kết thúc hoặc người dùng bấm Exit trên Remote:
playbackService.stopSelf() // Lúc này Service mới chính thức onDestroy()
```
* **Pitfall:** Nghĩ rằng gọi `stopSelf()` là service sẽ tắt ngay. Nếu UI Component vẫn đang giữ `ServiceConnection` (quên gọi `unbindService()`), Service vẫn tiếp tục sống ngầm trong memory, gây leak memory toàn bộ Service và Context đi kèm.

---

### Q12. Trên STB, khi nhận được UDP broadcast / Multicast packet cảnh báo khẩn cấp từ Headend ở background, làm thế nào để khởi động một Service hiển thị overlay mà không bị `BackgroundServiceStartNotAllowedException`?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Từ Android 8.0 và đặc biệt là Android 12+, `ActiveServices.java` chặn hoàn toàn việc gọi `startService()` hoặc `startForegroundService()` từ background, trừ các trường hợp nằm trong whitelist exemption (nhận High Priority Push Notification qua FCM, system broadcast đặc thù, hoặc app có quyền `SYSTEM_ALERT_WINDOW`).
  * Khi vi phạm, hàm `ActiveServices.startServiceLocked()` sẽ kiểm tra `isAllowedResourceUsage()` và ném ra `BackgroundServiceStartNotAllowedException`.
  * **Giải pháp trên STB:**
    1. Đăng ký nhận packet thông qua một persistent worker hoặc `JobScheduler` với priority cao.
    2. Với app hệ thống (System STB app ký platform key): cấp quyền `START_ACTIVITIES_FROM_BACKGROUND` hoặc cấu hình privilege whitelist trong `/etc/permissions/privapp-permissions-xxx.xml`.
    3. Sử dụng `WorkManager` với `OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST` hoặc gửi Notification có `fullScreenIntent`.
* **Code minh họa:**
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
        .setFullScreenIntent(pendingIntent, true) // Mở trực tiếp Activity từ background
        .build()

    NotificationManagerCompat.from(context).notify(999, notification)
}
```
* **Pitfall:** Trực tiếp gọi `context.startService()` từ trong một Coroutine IO hoặc raw Java Thread lắng nghe socket UDP khi App đang không có Activity nào ở foreground. App sẽ crash ngay tức thì trên production.

---

# CHỦ ĐỀ 4: BROADCASTRECEIVER & ANDROID 14 CHANGES

### Q13. Phân tích sự thay đổi bắt buộc của cờ `RECEIVER_EXPORTED` / `RECEIVER_NOT_EXPORTED` trên Android 14. Cơ chế bảo vệ nội bộ trong `ContextImpl.java` hoạt động ra sao?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Trước Android 14, dynamic receiver mặc định có thể nhận broadcast từ bất kỳ app nào khác trong hệ điều hành nếu không khai báo permission bảo vệ.
  * Trên Android 14 (`ContextImpl.java` -> `registerReceiverInternal()`), hệ thống kiểm tra nếu target SDK >= 34, cờ `RECEIVER_EXPORTED` hoặc `RECEIVER_NOT_EXPORTED` bắt buộc phải được truyền vào, trừ khi broadcast đó là Protected Broadcast của hệ thống (như `ACTION_BOOT_COMPLETED`, `ACTION_BATTERY_CHANGED`).
  * Nếu không set cờ, hệ thống ném `SecurityException: com.package: Targeting U+ (version 34 and above) disallows registering receiver without specifying RECEIVER_EXPORTED or RECEIVER_NOT_EXPORTED`.
* **Code minh họa:**
```kotlin
val filter = IntentFilter("com.stb.action.CUSTOM_HDMI_EVENT")
val receiver = HdmiEventReceiver()

// Dynamic register đúng chuẩn Android 14:
ContextCompat.registerReceiver(
    context,
    receiver,
    filter,
    ContextCompat.RECEIVER_NOT_EXPORTED // Chỉ nội bộ app hoặc cùng UID mới được gửi tới
)
```
* **Pitfall:** Đánh dấu `RECEIVER_EXPORTED` cho các receiver nhạy cảm (như trigger upgrade firmware, xóa cache TV). Mọi ứng dụng bên thứ 3 cài trên STB đều có thể inject Intent độc hại để kích hoạt logic này mà không gặp rào cản nào.

---

### Q14. Cơ chế dispatching của `OrderedBroadcast` vs standard Broadcast trong `BroadcastQueueModernImpl` (Android 14). Tại sao gọi `abortBroadcast()` có thể chặn đứng luồng dữ liệu của các app khác?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Standard Broadcast: Dispatch đồng loạt tới các dynamic receivers đã đăng ký.
  * `OrderedBroadcast`: Quản lý trong `BroadcastRecord.java`. Danh sách receivers được sắp xếp theo thứ tự `priority` giảm dần (`IntentFilter.getPriority()`).
  * `BroadcastQueueModernImpl` dispatch tới từng receiver một theo kiểu tuần tự (serial). Chỉ khi receiver trước đó gọi `onReceive()` xong (hoặc gọi `finishReceiver()` qua Binder IPC), hệ thống mới dispatch tiếp cho receiver tiếp theo.
  * Biến `mAbortBroadcast` nằm trong `BroadcastRecord`. Khi một receiver gọi `abortBroadcast()`, system server lập tức đánh dấu cờ này thành `true` và xóa sạch danh sách receivers còn lại đang chờ trong queue.
* **Code minh họa:**
```kotlin
class HighPriorityKeyInterceptor : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == "com.stb.action.GLOBAL_KEY") {
            val keyCode = intent.getIntExtra("KEYCODE", 0)
            if (keyCode == KeyEvent.KEYCODE_PROG_RED) {
                // Tiêu thụ sự kiện, không cho các launcher/app khác nhận được
                abortBroadcast()
            }
        }
    }
}
```
* **Pitfall:** Gọi `abortBroadcast()` trong một Standard Broadcast (`context.sendBroadcast()`). Phương thức này sẽ ném ra `RuntimeException: BroadcastReceiver trying to return result during a non-ordered broadcast`.

---

### Q15. Tại sao Sticky Broadcast bị cấm hoàn toàn (`SecurityException` hoặc deprecated)? Vấn đề kiến trúc nào trong `ActivityManagerService` dẫn đến quyết định này?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Khi gọi `sendStickyBroadcast()`, intent đó không mất đi sau khi dispatch mà được lưu trữ vĩnh viễn trong một map tĩnh `mStickyBroadcasts` trong `ActivityManagerService.java` trên `system_server`.
  * Bất kỳ app nào đăng ký sau (với `registerReceiver(null, filter)`) đều lập tức nhận được Intent cuối cùng này.
  * **Vấn đề cốt lõi:**
    1. **Bảo mật:** Không có cơ chế permission nào bảo vệ `mStickyBroadcasts`. Bất kỳ app nào cũng có thể đọc, sửa đổi hoặc giả mạo broadcast này.
    2. **Memory Leak:** Các sticky broadcast tích tụ trong RAM của `system_server`, không bao giờ được GC trừ khi gọi `removeStickyBroadcast()`.
    3. **Architecture:** Khái niệm State nên được quản lý qua Service hoặc ContentProvider thay vì dùng cơ chế messaging fire-and-forget của Broadcast.
* **Pitfall:** Cố gắng sử dụng sticky broadcast để chia sẻ trạng thái kết nối mạng hoặc trạng thái cắm cáp quang/HDMI trên STB. Luôn thay thế bằng StateFlow trong local process hoặc ContentProvider/Binder interface đối với multi-process.

---

### Q16. Implicit Broadcast Ban từ Android 8.0: Cơ chế nào trong hệ thống chặn Static Receiver trong Manifest và giải pháp trên Android TV STB là gì?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Khi một implicit broadcast được gửi, `PackageManagerService` (PMS) resolve tất cả các receiver trong Manifest.
  * Tuy nhiên, `BroadcastQueue` kiểm tra nếu Intent không có Explicit Component (`Intent.setComponent()` hoặc `Intent.setPackage()`), nó sẽ tự động lọc bỏ các static receiver đăng ký cho hành động đó nhằm ngăn chặn hàng trăm app bị đánh thức cùng lúc gây nghẽn CPU và giật lag hệ thống.
  * Danh sách ngoại lệ (Whitelisted Broadcasts) được định nghĩa tĩnh trong AOSP tại `frameworks/base/services/core/java/com/android/server/am/BroadcastConstants.java` (ví dụ: `ACTION_BOOT_COMPLETED`, `ACTION_LOCALE_CHANGED`).
* **Code minh họa (Giải pháp gửi explicit broadcast):**
```kotlin
val intent = Intent("com.stb.action.CHANNEL_TUNED").apply {
    // Chỉ định rõ package đích hoặc component class
    setPackage("com.stb.tv.epg")
}
context.sendBroadcast(intent)
```
* **Pitfall:** Định nghĩa Static Receiver trong `AndroidManifest.xml` để đón các Intent như `android.net.conn.CONNECTIVITY_CHANGE`. Receiver này sẽ không bao giờ được trigger trên các thiết bị Android hiện đại.

---

# CHỦ ĐỀ 5: CONTENTPROVIDER & FILEPROVIDER

### Q17. Lifecycle của ContentProvider: Tại sao `onCreate()` của ContentProvider chạy trước cả `Application.onCreate()`? Tác động tiêu cực của nó đến Cold Startup time trên STB?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Trong `ActivityThread.java`, khi process được fork và khởi tạo, hàm `handleBindApplication()` được thực thi.
  * Thứ tự thực thi cụ thể:
    1. Tạo `Application` context: `LoadedApk.makeApplication()`.
    2. Gọi `Application.attachBaseContext()`.
    3. Cài đặt các ContentProvider: `installContentProviders(app, data.providers)`. Duyệt qua tất cả các provider trong Manifest và gọi `ContentProvider.onCreate()`.
    4. Cuối cùng mới gọi: `mInstrumentation.callApplicationOnCreate(app)`.
  * **Hậu quả:** Tất cả các ContentProvider (kể cả từ các thư viện bên thứ 3 như Firebase, App Startup, WorkManager, LeakCanary) đều được inject và chạy `onCreate()` trên **Main Thread** trước khi bất kỳ dòng code nào trong `Application.onCreate()` được chạy. Nếu một provider thực hiện khởi tạo SDK, đọc file SharedPreferences, mở SQLite database, Cold Startup time của TV Launcher sẽ tăng vọt, gây hiện tượng màn hình đen khi bật STB.
* **Code minh họa (Disable auto-initialization):**
```xml
<!-- Gỡ bỏ auto-init của thư viện để tự init lười (Lazy) trong worker thread -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    tools:node="remove" />
```
* **Pitfall:** Thực hiện network call hoặc khởi tạo Room database blocking ngay trong `ContentProvider.onCreate()`. App process sẽ bị treo ngay từ phase khởi tạo, dẫn đến ANR trước khi hiện bất kỳ frame UI nào.

---

### Q18. Cơ chế chia sẻ bộ nhớ của `CursorWindow` trong ContentProvider query qua Binder IPC. Giới hạn 2MB/4MB và nguyên nhân ném `CursorWindowAllocationException`?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Khi gọi `ContentProvider.query()`, dữ liệu không được copy trực tiếp qua Binder parcel thông thường mà sử dụng bộ nhớ chia sẻ ẩn (Anonymous Shared Memory - `ashmem` qua class `CursorWindow.cpp` trong `frameworks/base/libs/androidfw/`).
  * System server hoặc process cung cấp dữ liệu cấp phát một `CursorWindow` (mặc định giới hạn kích thước là **2MB** trên các bản Android cũ và **4MB** trên Android hiện đại). Nó ghi dữ liệu SQLite vào vùng nhớ ashmem này và chỉ truyền file descriptor (FD) qua Binder tới client process.
  * Phía client map FD này vào memory space để đọc dữ liệu mà không tốn công copy qua lại.
  * Nếu một row dữ liệu (chứa ảnh thumbnail kênh, blob metadata) lớn hơn dung lượng trống còn lại của Window, hoặc tổng số lượng rows quá lớn mà không được paging, hệ thống ném `CursorWindowAllocationException`.
* **Code minh họa (Đọc phân trang an toàn):**
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
* **Pitfall:** Query bảng EPG gồm 10,000 dòng chứa kèm ảnh Base64 hoặc text description quá dài trong một lần query duy nhất `query(uri, null, null, null)`. Tràn CursorWindow khiến app crash không thể cứu vãn.

---

### Q19. Cơ chế bảo mật và ánh xạ path của `FileProvider`. Tại sao chia sẻ file qua `file://` bị ném `FileUriExposedException` và `FLAG_GRANT_READ_URI_PERMISSION` hoạt động thế nào trong `UriGrantsManagerService`?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Chia sẻ URI `file://` làm lộ đường dẫn private thực tế trên ổ cứng (`/data/user/0/...`) của app. App nhận không có quyền đọc thư mục sandbox này, dẫn đến crash hoặc lộ lỗ hổng bảo mật. Từ Android 7.0, `StrictMode` chặn hoàn toàn và ném `FileUriExposedException`.
  * `FileProvider` biến đổi file thực tế thành một URI dạng `content://authority/alias_path/file_name` dựa trên file cấu hình XML mapping.
  * Khi gọi Intent với `FLAG_GRANT_READ_URI_PERMISSION`, `UriGrantsManagerService.java` (UGMS) trong system_server ghi nhận một `UriPermission`. Nó cấp quyền tạm thời truy cập file cụ thể đó cho target UID. Khi target process thực hiện IPC ngược lại để đọc file qua `openFileDescriptor()`, UGMS kiểm tra bảng quyền này và cho phép truy cập.
* **Code minh họa:**
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
context.startActivity(Intent.createChooser(intent, "Gửi log"))
```
* **Pitfall:** Cấu hình `<root-path name="root" path="." />` trong `file_paths.xml`. Điều này tạo ra lỗ hổng Directory Traversal nghiêm trọng, cho phép các app khác đọc toàn bộ private storage của app.

---

### Q20. Kỹ thuật truyền payload nhị phân dung lượng cực lớn (Firmware OTA chunk 100MB hoặc DB Dump) giữa 2 app STB qua ContentProvider mà không bị crash Binder và không tốn memory?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Sử dụng cơ chế Pipe streaming thông qua `ContentProvider.openPipeHelper()`.
  * Phương thức này tạo ra một cặp `ParcelFileDescriptor` kết nối qua Linux pipe (`pipe()` system call). Phía ContentProvider đóng vai trò writer, ghi data vào pipe output stream trên một background worker thread.
  * Phía Client nhận `ParcelFileDescriptor` đóng vai trò reader, đọc trực tiếp dữ liệu từ file descriptor qua input stream mà không cần buffer toàn bộ dữ liệu vào Java Heap và không thông qua CursorWindow.
* **Code minh họa:**
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
* **Pitfall:** Đọc toàn bộ `byte[]` của file dung lượng lớn vào RAM trước khi gửi. Trên các thiết bị STB cấu hình thấp (RAM 1GB/2GB), thao tác này sẽ kích hoạt OOM killer tiêu diệt process ngay lập tức.

---

# CHỦ ĐỀ 6: HANDLER, LOOPER, MESSAGEQUEUE, IDLEHANDLER & SYNC BARRIER

### Q21. Cấu trúc dữ liệu của `MessageQueue.java` là gì? Tại sao không dùng FIFO thông thường? Cơ chế non-blocking / sleep qua `nativePollOnce` sử dụng Linux API nào?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * `MessageQueue.java` sử dụng cấu trúc dữ liệu **Singly-Linked List** (danh sách liên kết đơn), được sắp xếp tăng dần theo mốc thời gian thực thi: `Message.when` (dựa trên `SystemClock.uptimeMillis()`).
  * Không dùng FIFO thông thường vì các message có thể được gửi kèm delay: `sendMessageDelayed()`. Message có `when` nhỏ nhất (cần chạy sớm nhất) luôn nằm ở đầu danh sách (`mMessages`).
  * **Cơ chế sleep/wake up:**
    * Khi gọi `MessageQueue.next()`, nếu danh sách rỗng hoặc message đầu tiên chưa đến thời điểm thực thi (`when > now`), thread sẽ sleep mà không tốn CPU thông qua hàm native `nativePollOnce(ptr, nextPollTimeoutMillis)`.
    * Dưới tầng C++ (`android_os_MessageQueue.cpp` và `Looper.cpp`), hệ thống sử dụng **Linux `epoll`** (`epoll_wait()`) và **`eventfd`**.
    * Khi có message mới được đưa vào queue (`enqueueMessage()`) có thời gian chạy sớm hơn message đầu hiện tại, nó gọi `nativeWake(ptr)`. Native code sẽ ghi một byte vào `eventfd`, đánh thức `epoll_wait()`, đưa UI thread trở lại trạng thái runnable.
* **Code minh họa (Bản chất xử lý loop):**
```java
// Logic cốt lõi trong Looper.loop()
for (;;) {
    Message msg = queue.next(); // Có thể sleep bên trong nativePollOnce, giải phóng CPU core
    if (msg == null) return;
    msg.target.dispatchMessage(msg);
    msg.recycleUnchecked();
}
```
* **Pitfall:** Gọi `Looper.myLooper().quit()` trên Main Thread. Hệ thống sẽ ném `IllegalStateException: Main thread not allowed to quit`.

---

### Q22. Synchronous Barrier là gì? Phương thức ẩn `postSyncBarrier()` được `ViewRootImpl` sử dụng như thế nào để ưu tiên xử lý VSYNC/Rendering?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Trong `MessageQueue.java`, một **Sync Barrier** thực chất là một `Message` đặc biệt có thuộc tính `target == null`.
  * Phương thức `MessageQueue.postSyncBarrier()` (hidden API) chèn barrier này vào queue theo thứ tự `when`.
  * Khi hàm `MessageQueue.next()` duyệt qua queue:
    * Nếu gặp một message có `target == null`, nó biết đây là Barrier.
    * Nó sẽ **bỏ qua toàn bộ các message thông thường (Synchronous Messages)** phía sau barrier.
    * Nó scan tiếp danh sách liên kết để tìm và chỉ thực thi các **Asynchronous Message** (`msg.isAsynchronous() == true`).
  * **Áp dụng trong `ViewRootImpl.java`:**
    * Khi cần vẽ frame (`scheduleTraversals()`), `ViewRootImpl` lập tức đặt một `postSyncBarrier()`.
    * Sau đó nó post một Asynchronous Message chứa Choreographer callback (`mTraversalRunnable`).
    * Nhờ cơ chế này, lệnh vẽ UI được đảm bảo nhảy cóc lên đầu queue, bỏ qua mọi network callbacks hay normal handler tasks đang chờ, đảm bảo frame được render kịp VSYNC. Sau khi vẽ xong, nó gọi `removeSyncBarrier()`.
* **Code minh họa:**
```kotlin
// Tạo Asynchronous Message để chạy vượt hàng đợi khi có Sync Barrier
val handler = Handler(Looper.getMainLooper())
val asyncMsg = Message.obtain(handler) {
    // Critical rendering logic
}.apply {
    isAsynchronous = true // Đánh dấu là Async message
}
handler.sendMessage(asyncMsg)
```
* **Pitfall:** Lạm dụng hoặc dùng Reflection gọi `postSyncBarrier()` mà quên gọi `removeSyncBarrier()`. Queue sẽ bị kẹt vĩnh viễn: mọi message thông thường (bao gồm Touch input, click D-pad, lifecycle event) sẽ không bao giờ được dispatch, gây ANR ngay lập tức.

---

### Q23. `IdleHandler` là gì? Phương thức `queueIdle()` được gọi khi nào và cách ứng dụng nó để lazy load các component nặng trên TV Launcher?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * `IdleHandler` là một interface static lồng trong `MessageQueue`.
  * Trong `MessageQueue.next()`, sau khi duyệt qua queue và nhận thấy: (1) Queue hoàn toàn rỗng; hoặc (2) Message đầu tiên có `when > now` (thread chuẩn bị sleep trong `nativePollOnce`), hệ thống sẽ kiểm tra danh sách `mIdleHandlers`.
  * Nếu danh sách không rỗng, UI thread sẽ tranh thủ gọi `queueIdle()` của từng `IdleHandler` trước khi đi ngủ.
  * Giá trị trả về của `queueIdle()`:
    * `true`: Giữ lại handler này, lần sau thread idle sẽ tiếp tục gọi.
    * `false`: Xóa handler này ra khỏi danh sách sau khi chạy xong.
* **Code minh họa:**
```kotlin
class TvHomeActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_tv_home)

        // Tận dụng thời gian UI thread rảnh rỗi để preload EPG Cache
        Looper.myQueue().addIdleHandler {
            preloadNextChannelEpgData()
            false // Chỉ chạy 1 lần duy nhất lúc khởi động xong
        }
    }
}
```
* **Pitfall:** Viết code xử lý quá nặng hoặc synchronous I/O trong `queueIdle()`. Dù thread đang rảnh, nhưng nếu bạn block bên trong `queueIdle()` quá lâu (VD: > 16ms), khi có VSYNC hoặc phím D-pad tới, UI thread không thể xử lý kịp, dẫn đến rớt frame (jank) hoặc lag navigation.

---

### Q24. Phân tích chuỗi tham chiếu gây Memory Leak kinh điển của non-static `Handler` trong Activity. Tại sao `WeakReference` đơn thuần đôi khi vẫn chưa đủ?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Khi khai báo một anonymous inner class `Handler` hoặc non-static inner class trong Activity:
    1. Handler ngầm định giữ một strong reference tới Activity outer class (`this$0`).
    2. Khi gửi `sendMessageDelayed(msg, 10_000)`: Message được đưa vào `MessageQueue`.
    3. Đối tượng `Message` này có trường `target` tham chiếu ngược lại chính `Handler`.
    4. `MessageQueue` lại được giữ bởi `Looper.getMainLooper()`, vốn là một biến static tồn tại vĩnh viễn trong suốt vòng đời của process.
    5. **Chuỗi tham chiếu (GC Root):** `Looper (Static)` -> `MessageQueue` -> `Message` -> `Handler` -> `Activity`.
    6. Nếu Activity bị finish trước 10s, Garbage Collector không thể thu hồi Activity -> Leak!
  * **Tại sao `WeakReference` vẫn chưa đủ?**
    * Dùng `WeakReference` giúp Activity được GC thu hồi.
    * NHƯNG bản thân đối tượng `Message` và `Runnable` vẫn tồn tại lơ lửng trong `MessageQueue` cho đến khi hết timeout. Nếu Runnable đó capture các biến local nặng hoặc callback, nó vẫn leak các đối tượng đó và gây crash `NullPointerException` khi cố gắng access Activity đã chết nếu không kiểm tra cẩn thận.
* **Code minh họa (Xử lý an toàn triệt để):**
```kotlin
class SafeTvActivity : AppCompatActivity() {
    private class SafeHandler(activity: SafeTvActivity) : Handler(Looper.getMainLooper()) {
        private val activityRef = WeakReference(activity)
        override fun handleMessage(msg: Message) {
            val activity = activityRef.get() ?: return
            // Xử lý an toàn...
        }
    }

    private val safeHandler = SafeHandler(this)

    override fun onDestroy() {
        super.onDestroy()
        // BẮT BUỘC: Xóa sạch toàn bộ message/runnable còn sót lại khỏi Queue
        safeHandler.removeCallbacksAndMessages(null)
    }
}
```
* **Pitfall:** Quên gọi `handler.removeCallbacksAndMessages(null)` trong `onDestroy()`. Dù dùng `WeakReference`, message vẫn chiếm bộ nhớ queue và có thể trigger logic không mong muốn khi màn hình đã đóng.

---

# CHỦ ĐỀ 7: KOTLIN COROUTINES & FLOW (UNDER THE HOOD)

### Q25. Trình biên dịch biến đổi một hàm `suspend` thành Bytecode Java như thế nào? Phân tích cơ chế CPS (Continuation Passing Style), biến State Machine và nhãn `label`.

**Trả lời:**
* **Cơ chế bên trong:**
  * Kotlin Compiler sử dụng kỹ thuật **CPS (Continuation Passing Style)**:
    1. Bổ sung thêm một tham số ngầm định vào cuối hàm: `$completion: Continuation<T>`.
    2. Kiểu trả về thực tế đổi thành `Any?` (trả về kết quả thực tế hoặc hằng số đặc biệt `COROUTINE_SUSPENDED`).
  * Bên trong hàm, Compiler sinh ra một anonymous class kế thừa `ContinuationImpl`. Class này chứa:
    * Biến `label: Int`: Lưu trạng thái hiện tại của State Machine.
    * Các trường lưu biến local (spilled variables) để giữ nguyên giá trị qua các lần suspend.
    * Hàm `invokeSuspend()` chứa một câu lệnh `switch(label)`:
      * `case 0`: Chạy từ đầu tới điểm suspend thứ nhất. Trước khi suspend, gán `label = 1`, truyền `this` vào hàm suspend tiếp theo. Nếu hàm đó trả về `COROUTINE_SUSPENDED`, toàn bộ call-stack thoát ra, giải phóng thread.
      * `case 1`: Khi coroutine được resume, `invokeSuspend()` được gọi lại. Nhờ `label = 1`, switch nhảy thẳng tới code sau điểm suspend thứ nhất, đọc lại các biến local đã lưu và chạy tiếp.
* **Code minh họa (Cách Compiler dịch hàm suspend):**
```kotlin
// Hàm gốc
suspend fun fetchChannelAndEpg(channelId: String): EpgData {
    val channel = api.getChannel(channelId) // suspend point 1
    val epg = api.getEpg(channel.id)       // suspend point 2
    return epg
}

// Giả mã Bytecode sau khi được biên dịch:
fun fetchChannelAndEpg(channelId: String, $completion: Continuation<Any?>): Any? {
    val $continuation = object : ContinuationImpl($completion) {
        var result: Any? = null
        var label: Int = 0
        var L$0: Any? = null // Lưu channel
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
            // Nếu chạy đồng bộ, rơi xuống case 1
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
* **Pitfall:** Gọi `Thread.sleep()` bên trong hàm `suspend`. Thao tác này block toàn bộ thread bên dưới (thread pool worker) thay vì kích hoạt cơ chế state machine suspension, vô hiệu hóa hoàn toàn lợi thế non-blocking của Coroutine.

---

### Q26. Structured Concurrency: Cơ chế lan truyền lỗi (Exception Propagation) giữa `Job` và `SupervisorJob`. Tại sao viết `CoroutineScope(Dispatchers.IO + SupervisorJob()).launch { ... }` có thể tạo ra unhandled exception làm sập app?

**Trả lời:**
* **Cơ chế bên trong:**
  * **Job thông thường:** Lan truyền lỗi hai chiều. Khi một child job bị fail (throw exception khác `CancellationException`), nó lập tức: (1) Cancel chính nó; (2) Lan truyền exception lên Parent Job; (3) Parent Job cancel toàn bộ các child jobs còn lại; (4) Parent tiếp tục bắn exception lên trên.
  * **SupervisorJob:** Lan truyền lỗi một chiều (xuống dưới). Nếu một child job bị fail, `SupervisorJob` không tự hủy và không cancel các child khác.
  * **Vấn đề cốt lõi:** `SupervisorJob` chỉ có tác dụng khi nó là **Parent trực tiếp** của các coroutine con.
  * Nếu dùng `launch { ... }`, builder này mặc định cài đặt cơ chế rethrow uncaught exception tới Thread Default Exception Handler nếu không có `CoroutineExceptionHandler`.
* **Code minh họa:**
```kotlin
val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())

// NGUY HIỂM: Nếu child1 crash, SupervisorJob ngăn không cho scope chết,
// NHƯNG bản thân exception vẫn không được bắt và ném thẳng ra Thread UncaughtExceptionHandler -> CRASH APP!
scope.launch {
    throw RuntimeException("Network Error!")
}

// CÁCH VIẾT ĐÚNG: Bổ sung CoroutineExceptionHandler
val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
    Log.e("STB", "Caught safely: ${throwable.message}")
}
scope.launch(exceptionHandler) {
    throw RuntimeException("Network Error!") // An toàn, không sập app
}
```
* **Pitfall:** Truyền `SupervisorJob()` vào `withContext(SupervisorJob()) { ... }`. Trong `withContext`, tham số `Job` bị ghi đè và ép buộc theo cơ chế structured concurrency thông thường, khiến `SupervisorJob` hoàn toàn mất tác dụng.

---

### Q27. Deep dive `Dispatchers.Main` vs `Dispatchers.Default` vs `Dispatchers.IO`: Thread Pool của Coroutines hoạt động thế nào và tại sao `Dispatchers.IO` chia sẻ threads với `Default`?

**Trả lời:**
* **Cơ chế bên trong:**
  * Dưới tầng framework của `kotlinx.coroutines`, cả `Default` và `IO` đều trỏ chung về một instance của **`CoroutineScheduler`** (Work-Stealing Thread Pool).
  * `Dispatchers.Default`: Tối ưu cho CPU-bound tasks. Số lượng threads tối đa bằng số CPU Cores của STB (thường là 4 cores trên các chip Amlogic S905X). Mỗi worker thread có một local deque (hàng đợi cục bộ). Khi hết việc, worker sẽ "steal" (trộm) task từ deque của worker khác.
  * `Dispatchers.IO`: Tối ưu cho blocking I/O (Disk, Socket). Giới hạn tối đa là `max(64, CPU_cores)`.
  * **Cơ chế chia sẻ:** Khi chuyển đổi từ `Default` sang `IO` qua `withContext(Dispatchers.IO)`, coroutine không nhất thiết phải nhảy sang một OS thread hoàn toàn mới. `CoroutineScheduler` chỉ đơn giản chuyển đổi trạng thái của worker thread hiện tại từ chế độ CPU core limit sang chế độ non-core elastic thread pool. Điều này giảm thiểu tối đa chi phí Context Switch của hệ điều hành.
  * `Dispatchers.Main.immediate`: Khác với `Dispatchers.Main` thông thường (luôn gọi `handler.post()`), `.immediate` sẽ kiểm tra trước: nếu thread hiện tại đã là UI Thread, nó **thực thi code đồng bộ ngay lập tức** mà không cần qua MessageQueue, loại bỏ độ trễ 1 frame rendering.
* **Pitfall:** Sử dụng `Dispatchers.Default` cho các tác vụ blocking I/O (như ghi file cache, đọc DB). Toàn bộ 4 CPU threads sẽ bị block, khiến mọi coroutine tính toán khác trong app bị "chết đói" (starvation).

---

### Q28. So sánh bản chất giữa `SharedFlow` và `StateFlow`. Cơ chế Backpressure và buffer overflow (`BufferOverflow.SUSPEND` vs `DROP_OLDEST`) hoạt động thế nào?

**Trả lời:**
* **Cơ chế bên trong:**
  * `StateFlow` thực chất là một biến thể chuyên biệt của `SharedFlow` (`SharedFlowImpl`):
    * Cố định cấu hình: `replay = 1`, `extraBufferCapacity = 0`, `onBufferOverflow = BufferOverflow.DROP_OLDEST`.
    * Luôn có giá trị khởi tạo (`initialValue`).
    * Có cơ chế **Equality Conflation**: Nếu emit một giá trị `new == old` (qua `Any.equals()`), StateFlow sẽ bỏ qua, không dispatch tới collector.
  * `SharedFlow`:
    * Không bắt buộc có giá trị ban đầu.
    * Có thể cấu hình `replay >= 0` và `extraBufferCapacity >= 0`.
    * Hỗ trợ 3 chiến lược Backpressure:
      * `BufferOverflow.SUSPEND`: Hàm `emit()` sẽ bị treo (suspend) khi buffer bị đầy cho tới khi collector chậm nhất giải phóng buffer.
      * `BufferOverflow.DROP_OLDEST`: Xóa giá trị cũ nhất trong buffer để nhường chỗ cho item mới (không bao giờ suspend `emit()`).
      * `BufferOverflow.DROP_LATEST`: Bỏ qua luôn giá trị mới nhất vừa emit.
* **Code minh họa:**
```kotlin
// Dùng cho One-time Event (Navigation, Show Toast trên TV)
private val _navigationEvent = MutableSharedFlow<TvNavDest>(
    replay = 0,
    extraBufferCapacity = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
val navigationEvent = _navigationEvent.asSharedFlow()
```
* **Pitfall:** Dùng `StateFlow` để phát sự kiện One-time Event (như hiển thị Toast hoặc Dialog). Nếu màn hình xoay hoặc recreate, StateFlow sẽ re-emit lại giá trị cuối cùng (`replay = 1`), khiến Toast/Dialog bị hiển thị lặp lại ngoài ý muốn.

---

# CHỦ ĐỀ 8: JETPACK (VIEWMODEL, LIVEDATA/STATEFLOW, ROOM, WORKMANAGER, HILT)

### Q29. Bằng cơ chế chính xác nào trong AOSP mà `ViewModel` sống sót qua được Configuration Change? Lần theo chuỗi thực thi từ `Activity.java` đến `ComponentActivity.java`.

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Trong `Activity.java`: Khi chuẩn bị recreate do config change, hệ thống gọi phương thức ẩn `retainNonConfigurationInstances()`.
  * `ComponentActivity.java` (AndroidX) override phương thức này và đóng gói đối tượng `NonConfigurationInstances` chứa trường `viewModelStore`.
  * `ActivityThread.java` gửi đối tượng này lên system server hoặc giữ trong memory client qua `ActivityClientRecord.lastNonConfigurationInstances`.
  * Sau khi `Activity` instance mới được tạo qua reflection:
    1. `ActivityThread.attach()` gán lại `lastNonConfigurationInstances`.
    2. Trong `ComponentActivity.onCreate()`, khi gọi `ViewModelProvider(this).get(...)`, hệ thống kiểm tra `ViewModelStoreOwner`.
    3. `ComponentActivity.getViewModelStore()` gọi `getLastNonConfigurationInstance()`. Nếu tìm thấy instance cũ, nó lấy lại đúng đối tượng `ViewModelStore` trước đó thay vì khởi tạo mới.
    4. Khi Activity bị kill thực sự (`isFinishing == true`), `ComponentActivity.onDestroy()` mới gọi `viewModelStore.clear()`, lúc này các hàm `onCleared()` của ViewModel mới được thực thi.
* **Code minh họa cơ chế AOSP:**
```java
// Logic cốt lõi trong ComponentActivity.java
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
* **Pitfall:** Truyền `Activity Context` vào ViewModel (dưới dạng biến thành viên). Vì ViewModel sống xuyên qua vòng đời của Activity instance, biến Context cũ này sẽ giữ toàn bộ view hierarchy trong bộ nhớ, gây ra critical memory leak.

---

### Q30. So sánh `LiveData` vs `StateFlow`. Tại sao `LiveData.postValue()` có thể làm rớt dữ liệu (Drop Data) và tại sao bắt buộc phải dùng `repeatOnLifecycle` khi collect Flow trên UI?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * **Cơ chế `LiveData.postValue()`:**
    * `LiveData` duy trì một biến `mPendingData` và một Runnable gửi tới Main Thread qua `ArchTaskExecutor`.
    * Sử dụng synchronized block: Nếu background thread gọi `postValue(1)` rồi lập tức gọi `postValue(2)` trước khi Main Thread kịp xử lý Runnable, biến `mPendingData` sẽ bị ghi đè thành `2`.
    * Khi Main Thread thức dậy, nó chỉ lấy giá trị cuối cùng (`2`). Giá trị `1` bị mất hoàn toàn!
  * **Tại sao cần `repeatOnLifecycle` với StateFlow?**
    * Flow không có cơ chế tự nhận biết Android Lifecycle như LiveData.
    * Nếu dùng `lifecycleScope.launch { flow.collect { ... } }`, flow sẽ tiếp tục collect và giữ kết nối ngay cả khi Activity ở trạng thái `STOPPED` (app về background hoặc chuyển sang app TV khác).
    * Điều này gây lãng phí tài nguyên CPU/Network và crash app nếu UI cập nhật khi view đã detach.
    * `repeatOnLifecycle(Lifecycle.State.STARTED)` sử dụng coroutine nội bộ: tự động hủy (cancel) coroutine thu thập khi lifecycle rơi xuống dưới `STARTED`, và tự động restart coroutine mới khi lifecycle đạt `STARTED` trở lên.
* **Code minh họa:**
```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.channelState.collect { channels ->
            channelAdapter.submitList(channels)
        }
    }
}
```
* **Pitfall:** Thu thập StateFlow bằng `launchWhenStarted`. API này chỉ *suspend* quá trình emit chứ không *cancel* producer, khiến luồng upstream flow (như GPS, websocket hoặc EPG polling) vẫn chạy ngầm tốn tài nguyên ở background.

---

### Q31. Cơ chế `InvalidationTracker` của Room hoạt động thế nào dưới tầng SQLite? Tại sao một hàm DAO trả về Flow lại có thể tự động trigger query mới khi bảng bị thay đổi?

**Trả lời:**
* **Cơ chế bên trong:**
  * Room sử dụng class `InvalidationTracker`.
  * Khi khởi tạo database, Room tự động tạo các **SQLite Triggers** (`TEMP TRIGGER`) trên các bảng được quan sát (`room_table_modification_tracker`).
  * Khi có bất kỳ câu lệnh `INSERT`, `UPDATE`, hoặc `DELETE` xảy ra trên bảng đích, SQLite trigger sẽ tự động chèn một cờ đánh dấu vào bảng tạm `room_table_modification_tracker`.
  * Khi bạn query trả về `Flow<List<T>>`: Room tạo ra một `Flow` phát ra giá trị đầu tiên, đồng thời đăng ký một `Observer` với `InvalidationTracker`.
  * Khi database thay đổi, `InvalidationTracker` thực hiện query kiểm tra bảng tạm trên một background thread. Nếu phát hiện cờ thay đổi, nó thông báo cho Observer. Observer này sẽ tự động thực thi lại câu lệnh query SQL ban đầu và emit kết quả mới nhất vào luồng Flow.
* **Pitfall:** Thực hiện query Room dạng `Flow` với các câu lệnh `JOIN` phức tạp trên nhiều bảng lớn. Mỗi khi một bảng thành phần có bất kỳ thay đổi nhỏ nào (dù không ảnh hưởng đến kết quả join), toàn bộ câu query join nặng nề sẽ bị re-run từ đầu, gây giật lag STB.

---

### Q32. Kiến trúc của `WorkManager`: Làm thế nào nó trừu tượng hóa giữa `JobScheduler`, `AlarmManager` và `BroadcastReceiver`? Cơ chế đảm bảo tính bền vững (guaranteed execution) nằm ở đâu?

**Trả lời:**
* **Cơ chế bên trong:**
  * WorkManager sử dụng cơ chế fallback linh hoạt tùy theo phiên bản Android:
    * Android 6.0+ (API 23+): Sử dụng `SystemJobScheduler` (giao tiếp trực tiếp với `JobSchedulerService` của AOSP).
    * Android < 6.0 hoặc các thiết bị không có Google Play Services: Fallback về sự kết hợp giữa `SystemAlarmScheduler` (`AlarmManager`) và các `BroadcastReceiver`.
  * **Cơ chế đảm bảo bền vững:**
    * Trái tim của WorkManager là một SQLite database nội bộ có tên `androidx.work.workdb`.
    * Bất cứ khi nào gọi `WorkManager.enqueue()`, một bản ghi chứa toàn bộ metadata, constraints (chỉ chạy khi có mạng, khi cắm sạc) và trạng thái (`ENQUEUED`, `RUNNING`, `SUCCEEDED`) lập tức được ghi bền vững (persisted) vào `workdb`.
    * Kể cả khi thiết bị STB bị rút nguồn đột ngột hoặc crash hệ thống, sau khi khởi động lại, `RescheduleReceiver` nhận broadcast `BOOT_COMPLETED`, đọc lại `workdb` và nạp lại toàn bộ các job dở dang vào `JobScheduler`.
* **Pitfall:** Sử dụng `WorkManager` cho các tác vụ cần chạy chính xác từng giây (in-app timing). WorkManager áp dụng cơ chế batching job của `JobScheduler` để tiết kiệm điện, độ trễ có thể dao động từ vài phút đến cả tiếng tùy thuộc vào Doze Mode.

---

# CHỦ ĐỀ 9: VIEW RENDERING (MEASURE/LAYOUT/DRAW, CHOREOGRAPHER, VSYNC, RENDERTHREAD & JANK)

### Q33. Vẽ lại chu trình render từ khi gọi `requestLayout()` / `invalidate()` đến khi pixel xuất hiện trên TV: Phân tích vai trò của `ViewRootImpl`, `Choreographer`, `SurfaceFlinger` và VSYNC.

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  1. **Trigger:** `View.requestLayout()` đánh dấu cờ `PFLAG_FORCE_LAYOUT` và gọi ngược dần lên cha cho tới `ViewRootImpl.java`.
  2. **Lên lịch (Schedule):** `ViewRootImpl.scheduleTraversals()` chèn một Sync Barrier vào `MessageQueue` và đăng ký nhận tín hiệu VSYNC tiếp theo với `Choreographer.java` qua `DisplayEventReceiver`.
  3. **Đồng bộ VSYNC:** Phần cứng phát tín hiệu VSYNC (VD: chu kỳ 16.6ms cho TV 60Hz, 8.3ms cho TV 120Hz). `Choreographer` nhận tín hiệu, dispatch callback `CALLBACK_TRAVERSAL` tới `ViewRootImpl`.
  4. **Traversals:** `ViewRootImpl.performTraversals()` thực thi tuần tự 3 phase trên UI Thread:
     * `performMeasure()` -> `View.measure()` -> `onMeasure()`.
     * `performLayout()` -> `View.layout()` -> `onLayout()`.
     * `performDraw()` -> `View.draw()` -> `onDraw()` (sinh ra các Drawing Commands ghi vào `DisplayList`).
  5. **RenderThread:** UI Thread chuyển giao (sync) `DisplayList` cho `RenderThread` (Native C++). RenderThread tối ưu hóa commands, giao tiếp với GPU driver qua OpenGL ES hoặc Vulkan để render frame vào buffer.
  6. **SurfaceFlinger:** RenderThread swap buffer, đưa buffer vào hàng đợi `BufferQueue`. Process `SurfaceFlinger` compositing các layer (Status bar, App Window, Video Surface) và đẩy dữ liệu ra Hardware Framebuffer hiển thị lên màn hình TV qua cổng HDMI.
* **Pitfall:** Gọi `requestLayout()` lặp đi lặp lại bên trong `onLayout()` hoặc `onDraw()`. Thao tác này kích hoạt vòng lặp layout vô tận, làm đơ hoàn toàn giao diện người dùng.

---

### Q34. Khác biệt cốt lõi giữa UI Thread và RenderThread (`DrawFrameTask`). Tại sao UI Thread hoàn toàn rảnh rỗi mà App vẫn có thể bị rớt frame (Jank)?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * **UI Thread:** Chịu trách nhiệm thực thi mã Java/Kotlin, bắt sự kiện click/remote, cập nhật View hierarchy, tính toán kích thước, và xây dựng danh sách lệnh vẽ (`DisplayListCanvas`).
  * **RenderThread (tách riêng từ Android 5.0 - `ThreadedRenderer.cpp`):** Nhận `DisplayList`, thực hiện compile shaders, upload texture (Bitmap) lên VRAM của GPU, và gọi các lệnh driver phần cứng.
  * **Nguyên nhân UI Thread rảnh nhưng vẫn Jank:**
    1. **GPU Bound:** Shader quá phức tạp hoặc GPU của chip STB quá yếu không thể hoàn thành việc rasterize các pixel trong vòng 16.6ms.
    2. **Texture Upload Janks:** UI Thread giải mã một ảnh Bitmap khổng lồ (VD: poster phim 4K). Mặc dù giải mã ở worker thread, nhưng khi RenderThread thực hiện bind texture lần đầu (`glTexImage2D`), nó phải upload hàng chục MB qua bus bộ nhớ lên GPU. Thao tác này block RenderThread, khiến frame tiếp theo bị trễ dù UI thread không hề có tác vụ nào chạy.
    3. **Buffer Stuffing / Triple Buffering Exhaustion:** Cả 3 buffers trong `BufferQueue` đều đang bị khóa bởi GPU hoặc `SurfaceFlinger`, RenderThread phải sleep chờ buffer trống.
* **Pitfall:** Load Bitmap kích thước nguyên bản từ internet và hiển thị vào ImageView mà không resize về đúng kích thước hiển thị (`targetSize`) của View trên TV.

---

### Q35. Vấn đề "Double Taxation" (Unbounded Measure Pass) là gì? Tại sao lồng ghép các `LinearLayout` có thuộc tính `layout_weight="1"` hoặc `RelativeLayout` đa tầng lại gây chết hiệu năng trên CPU yếu của STB?

**Trả lời:**
* **Cơ chế bên trong:**
  * Thông thường, một View chỉ nên được đo kích thước (measure) một lần duy nhất trong mỗi chu kỳ render.
  * **LinearLayout với `layout_weight`:** Để chia tỉ lệ, LinearLayout bắt buộc phải thực hiện **2 passes đo đạc**:
    * Pass 1: Đo kích thước tự nhiên của các con không có weight.
    * Pass 2: Tính toán phần không gian còn thừa và gọi `measure()` lần thứ hai trên tất cả các view có `weight` để áp đặt kích thước mới.
  * **RelativeLayout:** Mặc định luôn luôn thực hiện **tối thiểu 2 passes measure** trên toàn bộ các view con để resolve các quan hệ ràng buộc đa chiều (trái, phải, trên, dưới).
  * **Hậu quả cấp số nhân:** Nếu bạn lồng một `LinearLayout (weight)` bên trong một `RelativeLayout`, số lần measure trên các View ở tầng sâu nhất sẽ tăng theo cấp số nhân: $2 \times 2 \times 2... = 2^N$ lần measure!
  * Trên các SoC TV giá rẻ (quad-core ARM Cortex-A35/A53 tốc độ chỉ 1.2GHz), việc kích hoạt hàng nghìn phép tính toán measure lặp đi lặp lại sẽ làm UI thread nghẽn cứng, gây rớt frame nghiêm trọng khi scroll danh sách phim.
* **Pitfall:** Dùng lồng ghép nhiều tầng `LinearLayout` có weight thay vì dùng một tầng phẳng `ConstraintLayout` duy nhất để dàn layout.

---

### Q36. Overdraw là gì? Làm thế nào để loại bỏ Overdraw triệt để trong các ứng dụng Video/Media Player trên Android TV?

**Trả lời:**
* **Cơ chế bên trong:**
  * Overdraw xảy ra khi GPU phải vẽ đè một pixel lên cùng một vị trí trên màn hình nhiều lần trong một khung hình duy nhất (vẽ màu nền của Window -> vẽ đè màu nền của Layout cha -> vẽ đè màu nền của View con).
* **Chiến lược tối ưu hóa triệt để:**
  1. **Gỡ bỏ Window Background mặc định:** Mọi Activity mặc định đều có background màu xám hoặc đen định nghĩa trong Theme (`android:windowBackground`).
  2. **Trong Custom View:** Sử dụng `canvas.clipRect()` để loại bỏ các vùng bị che khuất bởi các thành phần khác.
  3. **Khử overdraw khi Play Video:** Khi đang phát video, `SurfaceView` sẽ đục một lỗ thủng (punch hole) trên window để phần cứng overlay video trực tiếp. Nếu player container có set background color, GPU sẽ vẽ màu đó đè lên video surface, gây lãng phí băng thông bộ nhớ khủng khiếp.
* **Code minh họa:**
```kotlin
class TvPlayerActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        // Gỡ bỏ background của Window để tránh GPU vẽ 1 lớp nền vô ích bên dưới Video
        window.setBackgroundDrawable(null)
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_tv_player)
    }
}
```
* **Pitfall:** Đặt thuộc tính `android:background="#000000"` trên mọi Layout từ `root` đến các `CardView` con. Bật tính năng "Debug GPU Overdraw" trên TV sẽ thấy màn hình hiển thị toàn màu đỏ (Overdraw > 4x).

---

# CHỦ ĐỀ 10: JETPACK COMPOSE (UNDER THE HOOD, STATE, SIDE-EFFECTS & STABILITY)

### Q37. Cơ chế "Slot Table" và `GapBuffer` trong Compose Runtime hoạt động ra sao? Trình biên dịch Compose biến đổi hàm `@Composable` thông qua IR Transformer như thế nào?

**Trả lời:**
* **Cơ chế bên trong:**
  * Compose Runtime không sử dụng cấu trúc cây phân cấp truyền thống dạng Object pointer để lưu UI state mà sử dụng một mảng phẳng gọi là **Slot Table** (dựa trên cấu trúc dữ liệu `Gap Buffer`).
  * `GapBuffer` là một mảng liên tục có một "khoảng trống" (gap). Khi duyệt cây Composable, con trỏ đọc/ghi tuần tự các giá trị (state, memoized values, groups). Khi cần chèn hoặc xóa node do recomposition, gap được di chuyển tới vị trí đó và mở rộng/thu hẹp, cho phép chèn/xóa với độ phức tạp $O(1)$.
  * **IR Transformer (Compiler Plugin):**
    * Mọi hàm đánh dấu `@Composable` bị compiler rewrite:
    * Thêm tham số `$composer: Composer` và `$changed: Int` (bitmask).
    * Compiler chèn các lệnh `composer.startRestartGroup(key)`, `composer.startReplaceableGroup(key)` ở đầu hàm và `composer.endRestartGroup()` ở cuối hàm.
    * Bitmask `$changed` lưu trữ trạng thái của các tham số: nếu tất cả tham số đều không đổi (so sánh qua `equals`), composer sẽ nhảy cóc qua toàn bộ thân hàm (Skip recomposition).
* **Code minh họa (Giả mã sau khi Compose Compiler biên dịch):**
```kotlin
// Hàm gốc
@Composable fun ChannelTitle(name: String) { Text(name) }

// Sau khi Compiler biến đổi:
fun ChannelTitle(name: String, $composer: Composer, $changed: Int) {
    $composer.startRestartGroup(12345)
    // Kiểm tra bitmask: Nếu không đổi thì skip
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
* **Pitfall:** Truyền các lambda không ổn định hoặc tạo trực tiếp instance object mới bên trong thân Composable mà không bọc trong `remember { ... }`. Hành vi này phá vỡ cơ chế kiểm tra `$changed`, khiến hàm luôn bị recompose liên tục.

---

### Q38. Hệ thống Stability trong Compose: Phân biệt `@Stable`, `@Immutable` và Unstable. Tại sao Kotlin `List<T>` mặc định bị xem là Unstable và cách khắc phục?

**Trả lời:**
* **Cơ chế bên trong:**
  * Compose Compiler phân loại các class thành:
    * **Immutable:** Giá trị không bao giờ thay đổi sau khi tạo (VD: primitive types, `String`). Tất cả các public properties phải là `val` và có kiểu dữ liệu Immutable.
    * **Stable:** Có thể thay đổi (mutable), nhưng khi thay đổi nó sẽ thông báo cho Compose Runtime (VD: `MutableState`).
    * **Unstable:** Chứa các thuộc tính `var`, hoặc các interface/class mà Compiler không thể đảm bảo tính bất biến tại compile-time.
  * **Vấn đề của `List<T>`:**
    * Trong Kotlin, `List` chỉ là một interface. Kiểu thực tế lúc runtime hoàn toàn có thể là một `ArrayList` (có thể bị mutate thêm bớt phần tử).
    * Do đó, Compose Compiler coi mọi implementation của `List<T>` là **Unstable**!
    * Hậu quả: Bất kỳ Composable nào nhận tham số dạng `List<T>` sẽ **không bao giờ được skip recomposition**, ngay cả khi dữ liệu danh sách không hề thay đổi!
* **Code minh họa (Khắc phục triệt để):**
```kotlin
// Cách 1: Dùng thư viện kotlinx.collections.immutable
@Composable
fun TvChannelRow(channels: PersistentList<ChannelItem>) { ... }

// Cách 2: Bọc trong một Data Class đánh dấu @Immutable
@Immutable
data class ChannelListWrapper(val list: List<ChannelItem>)

@Composable
fun TvChannelRow(wrapper: ChannelListWrapper) { ... }
```
* **Pitfall:** Tin rằng sử dụng `data class` với toàn bộ thuộc tính `val` là tự động Stable. Nếu trong đó có một trường `val tags: List<String>`, toàn bộ data class đó sẽ bị compiler đánh tụt hạng thành Unstable.

---

### Q39. Phân biệt bản chất và Use Case thực tế giữa các Side-Effect APIs: `LaunchedEffect`, `rememberCoroutineScope`, `SideEffect` và `DisposableEffect` trên giao diện Android TV.

**Trả lời:**
* **Cơ chế bên trong:**
  * `LaunchedEffect(keys)`: Khởi chạy một coroutine bên trong Composition context. Coroutine tự động bị hủy (cancelled) và khởi chạy lại khi `keys` thay đổi, hoặc bị hủy khi composable rời khỏi Composition (Leave the composition). Dùng để load data khi chuyển kênh TV.
  * `rememberCoroutineScope()`: Trả về một `CoroutineScope` gắn liền với lifecycle của điểm Composition hiện tại, nhưng được dùng để launch coroutine từ **bên ngoài** các composable callbacks (như `onClick`, sự kiện nhấn phím D-pad remote).
  * `SideEffect`: Chạy sau **mỗi lần Recomposition thành công**. Không phải là coroutine. Dùng để đồng bộ hóa Compose state với các object không do Compose quản lý (VD: gửi log analytics hoặc cập nhật biến C++ native).
  * `DisposableEffect(keys)`: Tương tự `LaunchedEffect` nhưng dành cho các tác vụ non-coroutine cần dọn dẹp (cleanup). Bắt buộc phải có khối `onDispose { ... }`. Cực kỳ quan trọng để add/remove FocusListener hoặc release MediaPlayer trên TV.
* **Code minh họa:**
```kotlin
@Composable
fun DpadFocusTracker(viewId: String, onFocusLost: () -> Unit) {
    DisposableEffect(viewId) {
        val listener = View.OnFocusChangeListener { _, hasFocus ->
            if (!hasFocus) onFocusLost()
        }
        // Attach listener...
        onDispose {
            // Cleanup khi rời khỏi màn hình hoặc viewId đổi
            // Tránh leak listener
        }
    }
}
```
* **Pitfall:** Thực hiện side-effect (như ghi biến, điều hướng navigation, call API) trực tiếp bên trong thân hàm `@Composable` mà không bọc trong các Side-Effect APIs. Vì recomposition có thể bị cancel hoặc chạy nhiều lần liên tiếp, các logic này sẽ bị kích hoạt sai lệch nhiều lần.

---

### Q40. Custom Layout trong Compose: 3 Phase (Composition -> Measurement/Layout -> Drawing). Làm thế nào để bỏ qua Phase 1 & 2 khi thực hiện animation trên TV?

**Trả lời:**
* **Cơ chế bên trong:**
  * Compose có 3 phases chạy tuần tự:
    1. **Composition:** Chạy các hàm composable để tạo cây UI (tốn CPU nhất).
    2. **Layout:** Thực hiện đo đạc (measure) và định vị vị trí (place).
    3. **Drawing:** Vẽ canvas vào RenderNode (nhẹ nhất).
  * Nếu một State thay đổi được đọc trực tiếp ở Phase 1, toàn bộ 3 phases đều phải chạy lại.
  * Nếu đọc State ở Phase 2 (Layout), Compose bỏ qua Phase 1, chỉ chạy lại Layout và Drawing.
  * Nếu đọc State ở Phase 3 (Drawing), Compose **bỏ qua hoàn toàn cả Composition lẫn Layout phase**, chỉ vẽ lại canvas.
* **Code minh họa (Tối ưu animation Focus Scale trên TV):**
```kotlin
// KÉM HIỆU NĂNG: Đọc state ở Phase 1 -> Recompose liên tục 60fps
val scale by animateFloatAsState(if (isFocused) 1.2f else 1.0f)
Box(modifier = Modifier.size(100.dp * scale)) // Trigger Recomposition!

// TỐI ƯU TUYỆT ĐỐI: Dùng Modifier.graphicsLayer đọc state ở Phase 3
val scale = remember { Animatable(1.0f) }
Box(modifier = Modifier
    .size(100.dp)
    .graphicsLayer { // Lambda này được gọi ở Draw phase!
        scaleX = scale.value
        scaleY = scale.value
    }
)
```
* **Pitfall:** Dùng `Modifier.offset(x, y)` với giá trị tính toán trực tiếp thay vì dùng `Modifier.offset { IntOffset(x.value, y.value) }`. Dạng lambda cho phép defer việc đọc state sang tận Layout phase, tránh recomposition.

---

# CHỦ ĐỀ 11: MEMORY LEAK, LEAKCANARY & ANR Ở TẦNG APP

### Q41. Phân loại 4 nguyên nhân gây ANR ở tầng App. Cơ chế kernel gửi tín hiệu `SIGQUIT` tới `Signal Catcher` thread và cách phân tích file `/data/anr/traces.txt` để tìm Deadlock?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * **4 loại ANR:**
    1. *Input Dispatching Timeout:* 5 giây không phản hồi sự kiện phím D-pad hoặc Touch (`InputDispatcher.cpp`).
    2. *Broadcast Timeout:* `BroadcastReceiver.onReceive()` chạy quá 10 giây (foreground) hoặc 60 giây (background).
    3. *Service Timeout:* Service lifecycle chạy quá 20 giây (foreground) hoặc 200 giây (background).
    4. *ContentProvider Timeout:* `ContentProvider` không publish xong trong 10 giây.
  * **Cơ chế Dump Stacktrace:**
    * Khi phát hiện timeout, AMS gửi tín hiệu **`SIGQUIT` (Signal 3)** tới process bị lỗi thông qua Linux kernel call `kill(pid, SIGQUIT)`.
    * Trong ART runtime, thread `Signal Catcher` (`art/runtime/signal_catcher.cc`) bắt tín hiệu này, suspend toàn bộ các threads khác trong process và dump toàn bộ call-stack của từng thread ra file `/data/anr/traces.txt`.
  * **Cách đọc traces.txt tìm Deadlock:**
    * Tìm `tid=1 sysTid=... "main"`: Xem trạng thái là `BLOCKED`, `WAITING`, hay `MONITOR`.
    * Nếu trạng thái là `BLOCKED (on a monitor)`: Xem dòng `waiting to lock <0x01234567> held by thread X`.
    * Nhảy tới `thread X` trong file: Nếu thấy `thread X` cũng đang `waiting to lock <0x09876543> held by thread 1 (main)` -> Xác nhận **Mutual Deadlock**!
* **Pitfall:** Nghĩ rằng chỉ có CPU 100% mới gây ANR. Trên thực tế, phần lớn ANR nghiêm trọng trên Android TV xảy ra khi CPU gần như 0% do Main Thread bị `BLOCKED` chờ lock từ một worker thread đang thực hiện SQLite query hoặc Disk I/O.

---

### Q42. Cơ chế bên trong của `LeakCanary`: Nó sử dụng `WeakReference`, `ReferenceQueue` và `AppWatcher` như thế nào để phát hiện leak? Quá trình Dump Hprof và Shark phân tích heap diễn ra ra sao?

**Trả lời:**
* **Cơ chế bên trong:**
  1. **Giám sát:** `AppWatcher` hook vào `ActivityLifecycleCallbacks.onActivityDestroyed()`.
  2. **Theo dõi:** Khi `onDestroy()` được gọi, nó bọc đối tượng Activity vào một `KeyedWeakReference` (kế thừa từ `WeakReference`) và gắn kèm một `ReferenceQueue`.
  3. **Chờ đợi:** Nó post một background task kiểm tra sau 5 giây (thông qua `IdleHandler` hoặc Handler delay).
  4. **Kiểm tra:** Hệ thống ép GC chạy (`Runtime.getRuntime().gc()`). Nếu Activity đã được dọn sạch, `WeakReference` của nó sẽ được đưa vào `ReferenceQueue`. Nếu sau khi poll `ReferenceQueue` mà **không tìm thấy** reference đó, LeakCanary xác định đối tượng này đã bị "Retained" (giữ trái phép).
  5. **Dump & Analyze:** Khi số lượng retained objects vượt quá ngưỡng (mặc định là 5), nó gọi `Debug.dumpHprofData()` để ghi heap ra file `.hprof`. Thư viện **Shark** (parser Hprof viết bằng Kotlin) sẽ đọc file này, dựng lại đồ thị đối tượng và tìm đường đi ngắn nhất từ **GC Root** tới đối tượng bị leak (**Shortest Retained Path**).
* **Pitfall:** Bật LeakCanary trên bản build Production hoặc Performance Benchmark. Quá trình `Debug.dumpHprofData()` sẽ "freeze" (đóng băng) toàn bộ process của app từ 2 đến 10 giây trên phần cứng yếu của STB, trực tiếp gây ra ANR giả mạo.

---

### Q43. Phân tích 3 lỗi Memory Leak "kinh điển" đặc thù nhất trên hệ điều hành Android TV / Set-top Box.

**Trả lời:**
* **3 Lỗi đặc thù:**
  1. **Leak Static TV Focus Highlight:** Để tùy biến viền sáng (Focus Border) khi remote D-pad di chuyển tới, lập trình viên thường dùng một Custom Drawable và vô tình lưu nó vào một Singleton hoặc static variable. Vì Drawable giữ reference tới `Resources`, và `Resources` lại giữ reference tới `Context` (Activity), điều này làm leak toàn bộ Activity.
  2. **Leak ExoPlayer qua `SurfaceView` / `TextureView`:** `SurfaceView` chứa một `SurfaceControl` kết nối trực tiếp với `SurfaceFlinger`. Nếu truyền Activity Context vào Player singleton và không gọi `player.clearVideoSurfaceView()` khi Activity destroy, `SurfaceView` và `SurfaceSession` sẽ không bao giờ được GC, làm cạn kiệt Native Graphic Memory của STB.
  3. **Leanback `Presenter.ViewHolder` captures Presenter/Adapter Context:** Trong thư viện Leanback, nếu tạo listener (như focus change, click) dạng anonymous inner class bên trong `ViewHolder` mà tham chiếu ngược ra biến của `Presenter` hoặc `ArrayObjectAdapter`, toàn bộ danh sách rows items sẽ bị neo lại trong bộ nhớ dù fragment đã bị pop khỏi back stack.
* **Code minh họa (Sửa lỗi ExoPlayer leak):**
```kotlin
override fun onStop() {
    super.onStop()
    // Tách rời SurfaceView khỏi Player trước khi Activity bị destroy
    exoPlayer.clearVideoSurface()
}
```
* **Pitfall:** Sử dụng `TextureView` thay vì `SurfaceView` để phát video trên Android TV. `TextureView` tiêu tốn thêm từ 1-3 buffers RAM đồ họa nội bộ (tốn thêm hàng chục MB RAM) và tăng chi phí compositing của GPU lên 30%, dễ gây Out Of Memory (OOM) trên các STB RAM 1GB.

---

# CHỦ ĐỀ 12: ANDROID TV (LEANBACK, FOCUS NAVIGATION, TIF & LAUNCHER/HOME)

### Q44. Focus Navigation Engine trên Android TV: Khi người dùng bấm phím D-pad, `FocusFinder.java` tìm kiếm View tiếp theo như thế nào? Cơ chế thuật toán Directional Search và cách can thiệp?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Khi bấm phím D-pad, `ViewRootImpl.java` nhận `KeyEvent` từ Linux kernel driver qua `InputEventReceiver` và dispatch vào View hierarchy qua `dispatchKeyEvent()`.
  * Nếu không có view nào tiêu thụ phím, hệ thống gọi `focusSearch(direction)` (với `direction` là `FOCUS_UP`, `FOCUS_DOWN`, `FOCUS_LEFT`, `FOCUS_RIGHT`).
  * `ViewGroup.java` ủy quyền cho singleton **`FocusFinder.java`** (AOSP `frameworks/base/core/java/android/view/FocusFinder.java`).
  * **Thuật toán Directional Search:**
    1. `FocusFinder` lấy tọa độ hình học (Rect) của View đang giữ focus trên màn hình.
    2. Quét qua toàn bộ các focusable views khả dụng trong cùng window.
    3. Tính toán trọng số khoảng cách hình học bằng công thức:
       $$Distance = 13 \times MajorAxisDistance + MinorAxisDistance$$
       *(Khoảng cách theo trục chính của hướng di chuyển được nhân hệ số 13 để ưu tiên các view nằm thẳng hàng trước khi xét các view chéo góc).*
    4. View nào có `Distance` nhỏ nhất sẽ được chọn để trao focus (`view.requestFocus()`).
* **Code minh họa (Override luồng Focus bằng code và XML):**
```xml
<!-- Ép buộc phím D-pad Right nhảy chính xác tới Button mong muốn -->
<Button
    android:id="@+id/btn_play"
    android:nextFocusRight="@+id/btn_settings"
    android:focusable="true" />
```
```kotlin
// Can thiệp thủ công bằng code khi thuật toán mặc định nhảy sai
btnPlay.setOnKeyListener { v, keyCode, event ->
    if (event.action == KeyEvent.ACTION_DOWN && keyCode == KeyEvent.KEYCODE_DPAD_DOWN) {
        customTargetView.requestFocus()
        true // Đã tiêu thụ sự kiện, chặn FocusFinder can thiệp
    } else {
        false
    }
}
```
* **Pitfall:** Một View có `visibility = View.GONE` hoặc `isFocusable = false` nằm giữa đường đi của D-pad. `FocusFinder` sẽ bị kẹt hoặc nhảy loạn xạ sang một component hoàn toàn khác ở góc màn hình. Luôn set `android:focusable="true"` rõ ràng cho các custom view.

---

### Q45. Phân tích kiến trúc của Leanback Library: Cơ chế phối hợp giữa `BrowseSupportFragment`, `ArrayObjectAdapter`, `Presenter`, và `ItemBridgeAdapter` để tái sử dụng View khi điều hướng bằng D-pad.

**Trả lời:**
* **Cơ chế bên trong:**
  * Thư viện Leanback được thiết kế chuyên biệt cho điều hướng 10-foot experience:
    * **`Presenter`:** Tương đương với `Adapter` thu nhỏ cho một kiểu item cụ thể. Chịu trách nhiệm tạo `ViewHolder` (`onCreateViewHolder`) và bind dữ liệu (`onBindViewHolder`). Không chứa logic danh sách.
    * **`ArrayObjectAdapter`:** Lưu trữ tập hợp dữ liệu và ánh xạ từng kiểu dữ liệu (Model) với một `Presenter` tương ứng thông qua `PresenterSelector`.
    * **`ItemBridgeAdapter`:** Là cầu nối trung gian chuyển đổi từ `ArrayObjectAdapter` của Leanback sang chuẩn `RecyclerView.Adapter` truyền thống.
    * **`BrowseSupportFragment`:** Quản lý hai phần: `HeadersSupportFragment` (cột danh mục bên trái) và `RowsSupportFragment` (danh sách các hàng ngang `ListRow` bên phải).
  * **Tối ưu hóa D-pad:** `RowsSupportFragment` sử dụng `HorizontalGridView` (kế thừa từ `RecyclerView`). Khác với phone, nó có cơ chế căn chỉnh tiêu cự (Focus Alignment): giữ cho item đang được focus luôn nằm cố định ở một tỉ lệ phần trăm nhất định trên màn hình (VD: luôn cách lề trái 15%), danh sách sẽ cuộn mượt mà bên dưới item đó.
* **Code minh họa:**
```kotlin
val cardPresenter = CardPresenter()
val listRowAdapter = ArrayObjectAdapter(cardPresenter).apply {
    add(Movie("Inception"))
    add(Movie("Interstellar"))
}
val headerItem = HeaderItem(0, "Phim Khoa Học Viễn Tưởng")
val rootAdapter = ArrayObjectAdapter(ListRowPresenter()).apply {
    add(ListRow(headerItem, listRowAdapter))
}
browseSupportFragment.adapter = rootAdapter
```
* **Pitfall:** Gọi `notifyDataSetChanged()` trên `ArrayObjectAdapter` khi đang cuộn dở bằng remote D-pad. Thao tác này xóa sạch focus state hiện tại, khiến con trỏ focus bị văng ngược về phần tử đầu tiên hoặc nhảy mất lên thanh menu header.

---

### Q46. Kiến trúc TV Input Framework (TIF): Phân tích luồng hoạt động giữa `TvView`, `TvInputManagerService` và `TvInputService.Session`. Làm sao overlay UI hiển thị độc lập với hardware video surface?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * TIF là kiến trúc chuẩn của AOSP để hiển thị truyền hình (DVB-T2/C/S2, IPTV, HDMI In):
    1. **`TvView`:** Nằm trong ứng dụng Live TV (Client). Nó nhúng một `SurfaceView` để hiển thị luồng video và giao tiếp với hệ thống qua Binder IPC.
    2. **`TvInputManagerService` (TIMS):** System server service quản lý toàn bộ các hardware tuners và các app input truyền hình.
    3. **`TvInputService`:** Service chạy ngầm do nhà cung cấp thiết bị hoặc app IPTV triển khai.
  * **Luồng Tune Kênh:**
    * `TvView.tune(uri)` -> Gửi IPC tới TIMS.
    * TIMS yêu cầu `TvInputService` tạo một `TvInputService.Session`.
    * `Session.onSetSurface(surface)`: System truyền thẳng đối tượng `Surface` (từ `SurfaceView` của `TvView`) xuống cho `Session`.
    * Session truyền Surface này vào phần cứng giải mã (Hardware Decoder hoặc MediaCodec).
  * **Cơ chế Overlay UI độc lập:**
    * `Session.onCreateOverlayView()` cho phép trả về một `View` (chứa EPG, phụ đề, audio track selector).
    * `TvInputService` sẽ attach View này vào một `SurfaceControl` riêng biệt nằm đè lên trên Video Surface. Nhờ đó, việc vẽ lại UI overlay (60fps) không can thiệp hay làm gián đoạn luồng render trực tiếp từ phần cứng của video stream bên dưới.
* **Pitfall:** Thực hiện việc render phụ đề hoặc icon player trực tiếp lên cùng `Canvas` với Video Surface. Bất kỳ sự chậm trễ nào trong quá trình vẽ UI cũng sẽ làm giật luồng video phần cứng.

---

### Q47. Để một ứng dụng trở thành Custom Android TV Launcher (Home App), cần cấu hình Intent Filter chính xác như thế nào trong Manifest? Cơ chế Channels & Programs API (`TvContractCompat`) hoạt động ra sao?

**Trả lời:**
* **Cơ chế bên trong & AOSP Source:**
  * Để được hệ thống nhận diện là một TV Launcher chuẩn và thay thế màn hình chính mặc định khi bấm phím Home trên remote:
```xml
<activity
    android:name=".TvLauncherActivity"
    android:launchMode="singleTask"
    android:clearTaskOnLaunch="true"
    android:stateNotNeeded="true"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <!-- Đánh dấu là màn hình Home -->
        <category android:name="android.intent.category.HOME" />
        <category android:name="android.intent.category.DEFAULT" />
        <!-- BẮT BUỘC để hiển thị trên Android TV -->
        <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
    </intent-filter>
</activity>
```
  * **Channels & Programs API (`TvContractCompat`):**
    * Cho phép ứng dụng publish các kênh nội dung và các video gợi ý (Programs) trực tiếp ra màn hình Android TV Home Screen.
    * Giao tiếp thông qua `ContentProvider` của hệ thống (`android.media.tv.provider`).
    * App sử dụng `PreviewChannelHelper` để:
      1. Tạo một `PreviewChannel` (VD: "Kênh Phim Thịnh Hành").
      2. Chèn các `PreviewProgram` vào channel đó (chứa title, video trailer URL, thumbnail, deep link Intent).
      3. Ghi dữ liệu vào database hệ thống qua `TvContractCompat`. Android TV Home app sẽ query dữ liệu này và hiển thị thành các hàng ngang (rows) ngoài màn hình chính.
* **Code minh họa (Publish Preview Program):**
```kotlin
val program = PreviewProgram.Builder()
    .setChannelId(channelId)
    .setTitle("Phim Mới 2024")
    .setDescription("Xem ngay trên App STB")
    .setPosterArtUri(Uri.parse("https://stb.com/poster.jpg"))
    .setIntentUri(Uri.parse("stb://player?videoId=999"))
    .setType(TvContractCompat.PreviewPrograms.TYPE_MOVIE)
    .build()

val programUri = context.contentResolver.insert(
    TvContractCompat.PreviewPrograms.CONTENT_URI,
    program.toContentValues()
)
```
* **Pitfall:** Quên khai báo `android.intent.category.DEFAULT` cùng với `CATEGORY_HOME`. Khi bấm nút Home trên remote, hệ thống sẽ không hiển thị Resolver Activity để người dùng chọn Launcher mới mà nhảy thẳng về Launcher mặc định của nhà sản xuất.
