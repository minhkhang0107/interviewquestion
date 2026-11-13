Dưới đây là bộ câu hỏi và câu trả lời phỏng vấn chuyên sâu dành cho vị trí **Senior Android Platform/Framework Engineer** (tập trung vào **Android 14 - AOSP**, thiết bị **Android TV / Set-Top Box - STB**).

---

# TỔNG HỢP CÂU HỎI PHỎNG VẤN SENIOR FRAMEWORK (PHẦN 1)

---

## CHỦ ĐỀ 1: Luồng startActivity chi tiết từ Client đến ClientTransaction

### Q1. Trình bày chi tiết luồng thực thi từ `Activity.startActivity()` ở Client sang System Server. Tại sao AOSP phải đi qua `Instrumentation` và interface AIDL nào chịu trách nhiệm giao tiếp giữa hai process?
**Trả lời:**
* **Cơ chế bên trong:**
  1. Khi gọi `Activity.startActivity(Intent)`, lời gọi chuyển tiếp qua `Activity.startActivityForResult()` -> `Instrumentation.execStartActivity()`.
  2. `Instrumentation` đóng vai trò là choke-point kiểm soát mọi tương tác vòng đời và điều hướng của ứng dụng. Tại đây, `Instrumentation` kiểm tra kết quả trả về từ `ActivityTaskManager.getService().startActivity(...)` (ném ra các exception như `ActivityNotFoundException`, `SecurityException` nếu app không có quyền export).
  3. Quá trình IPC sang System Server được thực hiện qua interface AIDL `IActivityTaskManager` (trước Android 10 nằm trong `IActivityManager`). Proxy phía Client là `ActivityTaskManagerService.IActivityTaskManager.Stub.Proxy`.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/Activity.java`
  - `frameworks/base/core/java/android/app/Instrumentation.java`
  - `frameworks/base/core/java/android/app/ActivityTaskManager.java`
  - `frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java` (`ATMS`)
* **Code minh họa luồng Client:**
```java
// Instrumentation.java
public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, 
        Activity target, Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    // IPC chuyển giao sang ATMS trong System Server
    int result = ActivityTaskManager.getService().startActivity(
            whoThread, who.getBasePackageName(), who.getAttributionTag(), 
            intent, intent.resolveTypeIfNeeded(who.getContentResolver()), 
            token, target != null ? target.mEmbeddedID : null, 
            requestCode, 0, null, options);
    checkStartActivityResult(result, intent);
    return null;
}
```
* **Pitfall (Android TV STB):**
  Trên STB, các app VOD/Launcher thường gọi `startActivity` với custom Intent filter. Nếu Client truyền `options` chứa TV display ID không hợp lệ hoặc intent chứa URI trỏ tới Provider chưa mount sau boot, `Instrumentation.checkStartActivityResult()` sẽ ném runtime crash ngay trên UI thread của Launcher làm crash toàn bộ giao diện màn hình chính.

---

### Q2. Trong ATMS, `ActivityStarter` và `ActivityTaskSupervisor` phối hợp xử lý Task, Stack, và Launch Mode như thế nào trước khi quyết định hiển thị Activity?
**Trả lời:**
* **Cơ chế bên trong:**
  1. `ActivityTaskManagerService.startActivityAsUser()` lấy `GlobalLock` (`synchronized (mGlobalLock)`) và chuyển giao xử lý cho `ActivityStarter.execute()`.
  2. `ActivityStarter` phân giải Intent (thông qua `PackageManagerInternal.resolveIntent()`), kiểm tra callers, permission, appop, và launch flags (`FLAG_ACTIVITY_NEW_TASK`, `FLAG_ACTIVITY_CLEAR_TOP`,...).
  3. `ActivityStarter.startActivityUnchecked()` phối hợp với `ActivityTaskSupervisor` và `TaskDisplayArea`:
     - Xác định target `Task` dựa trên `taskAffinity` và launch mode (`singleTask`, `singleInstance`, `singleTop`).
     - Gọi `ActivityTaskSupervisor.findTaskLocked()` để xem task đã tồn tại chưa. Nếu có, di chuyển Task lên top (`mRootWindowContainer.moveTaskToFront()`); nếu chưa, tạo `Task` mới gán vào display phù hợp.
  4. Tạo đối tượng `ActivityRecord` đại diện cho Activity mới và đẩy vào Task tương ứng.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java`
  - `frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java`
  - `frameworks/base/services/core/java/com/android/server/wm/Task.java`
  - `frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java`
* **Pitfall (Android TV STB):**
  Android TV sử dụng Leanback Launcher (`singleTask`/`singleInstance`). Khi phát video DRM trong app playback, nếu app cấu hình sai `launchMode` kết hợp `FLAG_ACTIVITY_NEW_TASK`, Activity phát video có thể nuốt Task của Leanback Launcher hoặc tạo ra màn hình đen vô tận khi bấm phím Back do hệ thống không thể pop stack về Launcher.

---

### Q3. Nếu process đích chưa tồn tại, cơ chế System Server yêu cầu Zygote fork process mới diễn ra như thế nào qua Local Socket?
**Trả lời:**
* **Cơ chế bên trong:**
  1. `ActivityTaskSupervisor.startSpecificActivity()` kiểm tra `WindowProcessController wpc = mService.getProcessController(r.processName, r.info.applicationInfo.uid)`. Nếu process chưa chạy hoặc thread Binder là null, gọi `mService.mAmInternal.startProcess()`.
  2. `ActivityManagerService.startProcessLocked()` tạo `ProcessRecord` và gọi `Process.start("android.app.ActivityThread", ...)`.
  3. `ZygoteProcess` gửi chuỗi lệnh mở tiến trình qua Unix domain socket (`/dev/socket/zygote` hoặc `zygote_secondary` cho 32-bit).
  4. Server Zygote (`ZygoteServer.java`) lắng nghe qua `selectReadable()`, nhận kết nối và gọi `ZygoteConnection.processOneCommand()`.
  5. `Zygote.forkAndSpecialize()` thực thi native fork. Tiến trình con (App process mới) được thiết lập UID, GID, SELinux context (`seinfo`), cgroups và đóng socket descriptor của Zygote.
  6. Tiến trình con gọi `ZygoteInit.zygoteInit()` -> khởi tạo runtime Java, thiết lập Binder thread pool (`ProcessState::self()->startThreadPool()`) -> `RuntimeInit.applicationInit()` -> gọi phương thức static `ActivityThread.main()`.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`
  - `frameworks/base/core/java/android/os/ZygoteProcess.java`
  - `frameworks/base/core/java/com/android/internal/os/ZygoteServer.java`
  - `frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java`
  - `frameworks/base/core/java/com/android/internal/os/RuntimeInit.java`
* **Pitfall (Android TV STB):**
  Khi khởi động Zygote trên các chip SoC STB (Amlogic, MediaTek, Realtek), nếu driver GPU (Mali, PowerVR) hoặc module DRM khởi tạo socket IPC thất bại trong giai đoạn preload class của Zygote, mọi tiến trình con được fork sẽ crash hàng loạt (`SIGSEGV` trong driver shared library), gây bootloop Zygote.

---

### Q4. Quá trình bắt tay hai chiều giữa App process mới và System Server diễn ra ra sao (`attachApplication` vs `bindApplication`)?
**Trả lời:**
* **Cơ chế bên trong:**
  1. Trong `ActivityThread.main()`, khởi tạo `Looper.prepareMainLooper()`, tạo instance `ActivityThread` và gọi `thread.attach(false, 0)`.
  2. App IPC ngược lại System Server: `ActivityManager.getService().attachApplication(mAppThread, startSeq)`. Tại đây, `mAppThread` là một Binder stub (`ApplicationThread`).
  3. Phía System Server, AMS xử lý `attachApplicationLocked()`:
     - Gửi cấu hình app, package, instrumentation qua IPC: gọi `mAppThread.bindApplication(...)`.
     - Thông báo cho ATMS: `mActivityTaskManager.attachApplication(app.getWindowProcessController())`.
  4. Phía Client nhận `bindApplication`: `ApplicationThread` gửi message `H.BIND_APPLICATION` vào main thread queue -> `ActivityThread.handleBindApplication()` khởi tạo `LoadedApk`, `ContextImpl`, `Instrumentation`, gọi `LoadedApk.makeApplication()` và kích hoạt `Application.onCreate()`.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/ActivityThread.java`
  - `frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`
* **Pitfall (Android TV STB):**
  Không bao giờ thực hiện các lệnh network sync hoặc khởi tạo SDK nặng trong `Application.onCreate()`. Trên STB cấu hình RAM 1.5GB/2GB, nếu `Application.onCreate()` bị block quá lâu, Binder transaction timeout hoặc ANR startup sẽ ném ra trước khi Activity đầu tiên kịp render khung hình.

---

### Q5. Phân tích kiến trúc `ClientTransaction` và Transaction Executor trong Android 14. `LaunchActivityItem` chuyển hóa thành `performLaunchActivity()` như thế nào?
**Trả lời:**
* **Cơ chế bên trong:**
  1. Sau khi app attach thành công, `ActivityTaskSupervisor.realStartActivityLocked()` tạo một đối tượng `ClientTransaction`.
  2. Giao dịch này bao gồm:
     - **Callback Item:** `LaunchActivityItem` (chứa Intent, cấu hình, ActivityInfo, token).
     - **Lifecycle Request:** `ResumeActivityItem` (chỉ thị trạng thái mong muốn là Resumed).
  3. System Server gửi transaction qua IPC: `mClientLifecycleManager.scheduleTransaction(clientTransaction)`.
  4. Phía App, `TransactionExecutor.execute()` tiếp nhận:
     - Duyệt qua callback: gọi `LaunchActivityItem.execute()` -> chuyển tiếp vào `ActivityThread.handleLaunchActivity()`.
     - Chuyển tiếp tới `ActivityThread.performLaunchActivity()`:
       * Khởi tạo `LoadedApk`, lấy ClassLoader tạo instance Activity qua `AppComponentFactory.instantiateActivity()`.
       * Tạo `ContextImpl` gán token cửa sổ (`IBinder`).
       * Gọi `activity.attach(...)` để liên kết `Window` (`PhoneWindow`), `WindowManagerService`.
       * Gọi `mInstrumentation.callActivityOnCreate(activity, ...)` -> gọi `Activity.onCreate()`.
     - Duyệt qua lifecycle request: gọi `ResumeActivityItem.execute()` -> đưa Activity vào trạng thái `onResume`.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/servertransaction/ClientTransaction.java`
  - `frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java`
  - `frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java`
  - `frameworks/base/core/java/android/app/ActivityThread.java`
* **Pitfall:**
  Nếu một crash xảy ra trong `LaunchActivityItem.execute()` (ví dụ lỗi inflate view trong `onCreate`), `TransactionExecutor` sẽ bắt Exception nhưng không thể phục hồi trạng thái Task trong ATMS, dẫn đến việc System Server giữ `ActivityRecord` ở trạng thái lơ lửng, gây rò rỉ bộ nhớ phía WM.

---

## CHỦ ĐỀ 2: ActivityThread, ApplicationThread, H Handler, Instrumentation, LoadedApk

### Q6. `ActivityThread` có thực sự là một `java.lang.Thread` không? Vai trò chính của nó và cơ chế hoạt động của `Looper.prepareMainLooper()` trong `ActivityThread.main()` là gì?
**Trả lời:**
* **Cơ chế bên trong:**
  - `ActivityThread` **không** kế thừa từ `java.lang.Thread`. Nó là một lớp điều phối (management class) chạy trên thread mặc định được tạo ra bởi Linux sau khi Zygote fork (gọi là Main Thread hoặc UI Thread).
  - Phương thức `ActivityThread.main(String[] args)` là entry-point của ứng dụng:
    1. Khởi tạo tracing và cấu hình bảo mật môi trường.
    2. Gọi `Looper.prepareMainLooper()`: khởi tạo đối tượng `Looper` và `MessageQueue` liên kết vĩnh viễn với thread này (`sMainLooper`). Cấm gọi hàm này lần thứ hai bằng việc kiểm tra `sMainLooper != null`.
    3. Tạo instance của `ActivityThread`.
    4. Gọi `Looper.loop()`: kích hoạt vòng lặp vô tận xử lý message từ `MessageQueue`. Nếu vòng lặp này kết thúc, tiến trình app sẽ chết.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/ActivityThread.java`
  - `frameworks/base/core/java/android/os/Looper.java`
  - `frameworks/base/core/java/android/os/MessageQueue.java`
* **Code Framework:**
```java
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
    AndroidOs.install();
    CloseGuard.setEnabled(true);
    Environment.initForCurrentUser();

    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false, 0);

    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
* **Pitfall:**
  Nhiều kỹ sư nhầm lẫn việc bắt crash bằng cách wrap `try-catch` quanh `Looper.loop()` bên trong `ActivityThread.main()`. Làm như vậy sẽ phá vỡ tính đồng bộ của `TransactionExecutor` và khiến System Server coi tiến trình là zombie khi các Binder transaction không được phản hồi đúng thời hạn.

---

### Q7. Phân tích vai trò và mối quan hệ threading giữa `ApplicationThread` và Handler `H`. Tại sao `ApplicationThread` không gọi trực tiếp các phương thức vòng đời?
**Trả lời:**
* **Cơ chế bên trong:**
  - `ApplicationThread` là một private inner class của `ActivityThread`, kế thừa từ `IApplicationThread.Stub`. Do đó, nó là một Binder IPC Server component.
  - Khi System Server gọi các hàm IPC (ví dụ: `scheduleTransaction()`, `bindApplication()`), code của `ApplicationThread` thực thi trên **Binder Thread Pool** của tiến trình (`Binder:pid_x`), không phải trên Main Thread.
  - Để đảm bảo thread-safety cho mô hình UI đơn luồng (Single-thread UI model) của Android, `ApplicationThread` tuyệt đối không được truy cập trực tiếp vào Views hay Activity instances.
  - Thay vào đó, `ApplicationThread` đóng gói dữ liệu vào `Message` hoặc Runnable và dispatch sang Handler `H` (kế thừa từ `Handler`, gắn với Main Looper) thông qua `sendMessage()`.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/ActivityThread.java` (lớp `ApplicationThread` và lớp `H`)
* **Code cấu trúc:**
```java
// Trong ActivityThread.java
private class ApplicationThread extends IApplicationThread.Stub {
    @Override
    public void scheduleTransaction(ClientTransaction transaction) {
        ActivityThread.this.scheduleTransaction(transaction); // Gửi message EXECUTE_TRANSACTION tới H
    }
}

class H extends Handler {
    public static final int BIND_APPLICATION        = 110;
    public static final int EXIT_APPLICATION        = 111;
    public static final int RECEIVER                = 113;
    public static final int EXECUTE_TRANSACTION     = 159;
    
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BIND_APPLICATION:
                AppBindData data = (AppBindData) msg.obj;
                handleBindApplication(data);
                break;
            case EXECUTE_TRANSACTION:
                final ClientTransaction transaction = (ClientTransaction) msg.obj;
                mTransactionExecutor.execute(transaction);
                break;
        }
    }
}
```
* **Pitfall:**
  Nếu Main Thread bị nghẽn (do DB query, parsing JSON lớn), MessageQueue đầy. Binder Thread Pool vẫn tiếp nhận transaction từ System Server và đẩy thêm message vào `H`. Khi vượt quá thời gian timeout (5s cho Input, 10s cho Broadcast, 20s cho Service), System Server sẽ bắn ANR ngay lập tức.

---

### Q8. Lớp `LoadedApk` quản lý những tài nguyên nào trong client runtime? Rò rỉ BroadcastReceiver hoặc ServiceConnection thường xuất phát từ `LoadedApk` ra sao?
**Trả lời:**
* **Cơ chế bên trong:**
  - `LoadedApk` đại diện cho một file APK đã được nạp vào bộ nhớ tiến trình.
  - Các trách nhiệm cốt lõi:
    1. Quản lý `ClassLoader` (thường là `PathClassLoader`).
    2. Quản lý tài nguyên: `ResourcesManager`, `AssetManager`.
    3. Tạo và cache các dispatchers cho IPC: `mReceivers` (`ArrayMap<Context, ArrayMap<BroadcastReceiver, ReceiverDispatcher>>`) và `mServices` (`ArrayMap<Context, ArrayMap<ServiceConnection, ServiceDispatcher>>`).
  - Khi gọi `context.registerReceiver()` hoặc `context.bindService()`, hệ thống cần một Binder token để giao tiếp với AMS. `LoadedApk.getReceiverDispatcher()` hoặc `getServiceDispatcher()` tạo ra `IIntentReceiver.Stub` hoặc `IServiceConnection.Stub`.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/LoadedApk.java`
* **Cơ chế rò rỉ (Leak):**
  - Trong `LoadedApk`, `ReceiverDispatcher` giữ một Strong Reference (hoặc qua Context wrapper) tới `BroadcastReceiver`.
  - Nếu một `BroadcastReceiver` hoặc `ServiceConnection` được đăng ký qua một `Activity Context` mà không gọi `unregisterReceiver()` hoặc `unbindService()` trước khi Activity bị hủy:
  - `LoadedApk.mReceivers` vẫn giữ tham chiếu tới `ReceiverDispatcher`, dẫn tới `Activity` không thể được Garbage Collection thu hồi -> **Memory Leak**.
* **Pitfall (Android TV STB):**
  Trên TV STB chạy 24/7 không reboot, các app system/launcher thường lắng nghe các sự kiện phần cứng (`HDMI_PLUGGED`, `ETHERNET_STATE_CHANGED`). Nếu đăng ký broadcast trong custom Views mà quên unregister khi view detach khỏi Window, bộ nhớ RAM hạn hẹp của STB sẽ bị cạn kiệt trong vòng vài ngày.

---

### Q9. `Instrumentation` can thiệp vào những điểm nào của vòng đời App? Làm sao áp dụng để hook/monitor Activity launch time trong môi trường production/testing?
**Trả lời:**
* **Cơ chế can thiệp:**
  - `Instrumentation` kiểm soát việc tạo instance và gọi vòng đời của hầu hết core component:
    - `newActivity()`: Tạo instance Activity qua reflection.
    - `callActivityOnCreate()`, `callActivityOnStart()`, `callActivityOnResume()`, `callActivityOnPause()`, v.v.
    - `callApplicationOnCreate()`: Khởi chạy application lifecycle.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/Instrumentation.java`
  - `frameworks/base/core/java/android/app/ActivityThread.java` (`mInstrumentation`)
* **Kỹ thuật Hooking để đo lường launch time:**
```kotlin
class PerformanceInstrumentation(private val base: Instrumentation) : Instrumentation() {
    override fun callActivityOnCreate(activity: Activity?, icicle: Bundle?) {
        val start = SystemClock.elapsedRealtimeNanos()
        base.callActivityOnCreate(activity, icicle)
        val durationMs = (SystemClock.elapsedRealtimeNanos() - start) / 1_000_000
        Log.d("PerfHook", "Activity ${activity?.javaClass?.simpleName} onCreate: ${durationMs}ms")
    }
}

// Hooking via Reflection trong custom Application
fun hookInstrumentation() {
    val activityThreadClass = Class.forName("android.app.ActivityThread")
    val currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread")
    currentActivityThreadMethod.isAccessible = true
    val currentActivityThread = currentActivityThreadMethod.invoke(null)

    val mInstrumentationField = activityThreadClass.getDeclaredField("mInstrumentation")
    mInstrumentationField.isAccessible = true
    val originalInstrumentation = mInstrumentationField.get(currentActivityThread) as Instrumentation

    mInstrumentationField.set(currentActivityThread, PerformanceInstrumentation(originalInstrumentation))
}
```
* **Pitfall:**
  Kể từ Android 9 trở đi với Hidden API Enforcement, việc reflect trường `mInstrumentation` trong `ActivityThread` thuộc danh sách Blacklist/Greylist max-target-P. Nếu ứng dụng target Android 14 không có quyền platform/system, lệnh set reflection sẽ bị chặn (`NoSuchFieldException` hoặc runtime rejection).

---

## CHỦ ĐỀ 3: Quản lý Process, OomAdjuster, ProcessList, lmkd

### Q10. Trình bày chi tiết thang điểm `oom_score_adj` (-1000 đến 1000) và cách `OomAdjuster` ánh xạ `Process State` (ProcState) sang giá trị này trên Android 14.
**Trả lời:**
* **Cơ chế bên trong:**
  - Linux kernel sử dụng file `/proc/<pid>/oom_score_adj` để quyết định tiến trình bị kill khi thiếu RAM.
  - Phía AOSP, `OomAdjuster.java` tính toán trạng thái tiến trình dựa trên 2 yếu tố:
    1. **Process State (`ActivityManager.PROCESS_STATE_*`)**: Biểu thị trạng thái logic của ứng dụng trong hệ thống.
    2. **OOM Adj (`ProcessList.*_ADJ`)**: Mức độ ưu tiên kill, giá trị từ -1000 đến +1000.
* **Bảng phân loại AOSP (`ProcessList.java`):**
  - `-1000` (`SYSTEM_ADJ`): Native processes, init, Zygote, System Server.
  - `-900` (`PERSISTENT_PROC_ADJ`): Các system app có cờ `android:persistent="true"` trong Manifest (ví dụ: Phone, Bluetooth, TV Input HAL Service).
  - `-800` (`PERSISTENT_SERVICE_ADJ`): Service persistent.
  - `-700` (`FOREGROUND_APP_ADJ`): App đang tương tác trực tiếp với người dùng (đang có Activity ở Resume, hoặc có Provider đang được client foreground sử dụng).
  - `100` (`VISIBLE_APP_ADJ`): Activity đang visible nhưng không focus (ví dụ bị che một phần bởi Dialog hoặc chạy Picture-in-Picture / Live TV preview).
  - `200` (`PERCEPTIBLE_APP_ADJ`): App không thấy hình nhưng người dùng cảm nhận được (ví dụ: Foreground Service phát nhạc, STB Passthrough Audio).
  - `500` (`BACKUP_ADJ`): Đang chạy backup.
  - `600 - 700` (`SERVICE_ADJ` / `SERVICE_B_ADJ`): Các Background Services bình thường.
  - `900 - 999` (`CACHED_APP_MIN_ADJ` đến `CACHED_APP_MAX_ADJ`): Các tiến trình cached không còn component nào active, chia thành nhiều bậc để lmkd giải phóng dần theo LRU.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java`
  - `frameworks/base/services/core/java/com/android/server/am/ProcessList.java`
  - `frameworks/base/services/core/java/com/android/server/am/ProcessRecord.java`

---

### Q11. Giao thức IPC giữa AMS và daemon `lmkd` hoạt động như thế nào qua Local Socket? Sự khác biệt giữa cơ chế kernel `vmpressure` và Linux `PSI` (Pressure Stall Information)?
**Trả lời:**
* **Cơ chế IPC qua Socket:**
  1. `ProcessList.java` trong AMS mở một Unix domain stream socket kết nối tới `/dev/socket/lmkd`.
  2. Mỗi khi `OomAdjuster.updateOomAdjLocked()` hoàn tất tính toán điểm số cho một process, AMS gửi gói tin nhị phân qua socket này:
     - Command code: `LMK_PROCPRIO` (đăng ký PID và oom_score_adj).
     - Command code: `LMK_PROCREMOVE` (xóa PID khi tiến trình kết thúc).
  3. `lmkd` (viết bằng C trong `system/memory/lmkd/`) nhận packet và trực tiếp ghi giá trị vào `/proc/<pid>/oom_score_adj`.
* **vmpressure vs PSI (Pressure Stall Information):**
  - **vmpressure (Legacy):**
    Dựa trên sự kiện memory reclaim của kernel (tỷ lệ scan pages / reclaim pages). Rất thiếu chính xác trên Android TV vì bộ nhớ đồ họa cấp phát qua ION/DMA-BUF không kích hoạt scan pages của kernel slab/page cache, dẫn đến việc `lmkd` nhận biết tình trạng thiếu RAM quá muộn.
  - **PSI (Android 10+ và chuẩn trên Android 14):**
    Đo lường thời gian CPU, Memory, hoặc I/O bị "stall" (bị treo/chờ) do thiếu tài nguyên phần cứng thông qua file `/proc/pressure/memory`.
    `lmkd` đăng ký epoll vào kernel PSI trigger:
    - `some`: Tỷ lệ phần trăm thời gian mà ít nhất một task bị block vì thiếu memory.
    - `full`: Toàn bộ các CPU core đều phải chờ memory (hệ thống đứng hình).
    Khi ngưỡng PSI chạm mức cấu hình, `lmkd` chủ động quét cây tiến trình và SIGKILL tiến trình có `oom_score_adj` cao nhất.
* **File/Class AOSP:**
  - `system/memory/lmkd/lmkd.c`
  - `frameworks/base/services/core/java/com/android/server/am/ProcessList.java`
* **Pitfall (Android TV STB):**
  Khi chơi video 4K 60fps HDR, codec phần cứng ngốn hàng trăm MB DMA-BUF memory. Nếu kernel của nhà sản xuất STB không bật cấu hình `CONFIG_PSI=y`, `lmkd` buộc phải fallback về `vmpressure`, dẫn tới tình trạng kernel nhảy thẳng vào Kernel OOM Killer và kill luôn cả tiến trình `SurfaceFlinger` hoặc `system_server` thay vì kill cached app.

---

### Q12. "Cached Process" và "Empty Process" khác nhau như thế nào? Cơ chế LRU và quản lý giới hạn số lượng tiến trình cached trên thiết bị Android TV RAM thấp.
**Trả lời:**
* **Sự khác biệt:**
  - **Cached Process:** Là tiến trình không có component nào đang hoạt động (không Activity foreground, không Service, không active BroadcastReceiver), nhưng **vẫn giữ lại Activity instances** trong back-stack hoặc memory. Mục đích là để người dùng nhấn Back hoặc mở lại app có thể khôi phục tức thì (Hot/Warm launch).
  - **Empty Process:** Là tiến trình không chứa bất kỳ component nào, kể cả Activity history. Tiến trình này chỉ được giữ lại để cache mã bytecode (DEX code đã load, shared libraries đã link) nhằm mục đích tái sử dụng khi cần khởi động lại một component của package đó.
* **Cơ chế LRU & Giới hạn:**
  - AOSP sử dụng một danh sách `mProcessList.mProcessDependencies` được sắp xếp theo thuật toán LRU (Least Recently Used).
  - Trong `ProcessList.java`, có hai biến số quan trọng:
    - `MAX_CACHED_PROCESSES`: Thường mặc định là 32 trên điện thoại, nhưng trên TV STB thường được cấu hình giảm xuống 16 hoặc 8.
    - `EMPTY_APP_PERCENT`: Giới hạn tỷ lệ process empty tối đa trong danh sách cached.
  - Khi danh sách vượt ngưỡng, `trimApplicationsLocked()` sẽ kill các process rỗng hoặc cached cũ nhất theo thứ tự LRU bằng tín hiệu `Process.killProcessQuiet(app.getPid())`.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/am/ProcessList.java`
  - `frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`
* **Pitfall (Android TV STB):**
  Trên STB có RAM 1.5GB, nếu set `MAX_CACHED_PROCESSES` quá cao, `lmkd` liên tục kích hoạt (thrashing), gây giật lag khung hình khi xem phim. Ngược lại, nếu set quá thấp (ví dụ < 4), mỗi lần người dùng bấm Home ra màn hình chính rồi quay lại app VOD, app đều phải Cold Start từ đầu, làm tăng thời gian load kênh.

---

### Q13. Phân tích cơ chế "Priority/OOM Score Propagation" khi một Client bind vào một Remote Service với các flags: `BIND_AUTO_CREATE`, `BIND_IMPORTANT`, và `BIND_ALLOW_OOM_MANAGEMENT`.
**Trả lời:**
* **Cơ chế bên trong:**
  - Khi App A (Client) bind vào Service của App B (Host), `OomAdjuster.updateOomAdjLocked()` phải tính toán lại `oom_score_adj` của Process B dựa trên mức độ ưu tiên của Process A.
  - Quá trình tính toán thông qua cấu trúc liên kết `ConnectionRecord` -> `AppBindRecord` -> `ProcessRecord`.
* **Phân tích các cờ (Flags):**
  1. `Context.BIND_AUTO_CREATE`:
     - Tự động start process của Service nếu chưa chạy. Tuy nhiên, nó không đảm bảo Service có mức độ ưu tiên ngang bằng Client trừ khi đi kèm các flag khác.
  2. `Context.BIND_IMPORTANT`:
     - Nâng Service của B lên mức độ quan trọng ngang ngửa với Process A. Nếu A đang ở Foreground (`FOREGROUND_APP_ADJ`), B cũng được kéo lên mức tương tự, bảo vệ B khỏi bị kill bởi `lmkd`.
  3. `Context.BIND_ALLOW_OOM_MANAGEMENT`:
     - **Bảo lưu OOM của Service:** Bỏ qua sự liên đới ưu tiên từ Client A. Mức độ ưu tiên của Service B vẫn được tính toán như thể nó là một background process độc lập. Thường dùng cho các tiến trình phụ trợ (như sync dữ liệu, render nền) để khi thiếu RAM, hệ thống có thể kill B mà không làm sụp đổ A.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/am/ActiveServices.java`
  - `frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java`
  - `frameworks/base/services/core/java/com/android/server/am/ConnectionRecord.java`
* **Pitfall:**
  Một System App trên STB (ví dụ Custom TV Setup Wizard) bind vào một Service vendor xử lý Tuner mà quên truyền `BIND_AUTO_CREATE` hoặc dùng sai flag, khi người dùng mở video 4K, tiến trình Service bị coi là background thuần túy và bị kill giữa chừng, làm mất tín hiệu giải mã truyền hình cáp.

---

## CHỦ ĐỀ 4: ActiveServices, startService, bindService, ANR Service, Foreground Service

### Q14. Trình bày chi tiết luồng `startServiceLocked()` và `bringUpServiceLocked()` trong `ActiveServices.java`. Sự khác nhau trong cấu trúc lưu trữ giữa Start và Bind là gì?
**Trả lời:**
* **Cơ chế bên trong:**
  1. Client gọi `startService()` -> IPC tới `ActivityManagerService.startService()` -> chuyển tiếp vào `ActiveServices.startServiceLocked()`.
  2. `ActiveServices` gọi `retrieveServiceLocked()`: Phân giải Intent, kiểm tra permission, tìm kiếm hoặc tạo mới `ServiceRecord`.
  3. Gọi `startServiceInnerLocked()` -> `bringUpServiceLocked()`:
     - Kiểm tra process chứa Service đã chạy chưa (`mAm.getProcessRecordLocked()`).
     - Nếu chưa: gọi `mAm.startProcessLocked()` để fork tiến trình mới qua Zygote.
     - Nếu đã chạy: gọi `realStartServiceLocked()`.
  4. `realStartServiceLocked()`:
     - Gửi Binder call `app.thread.scheduleCreateService(r, r.serviceInfo, ...)` -> Client gọi `Service.onCreate()`.
     - Gọi `sendServiceArgsLocked()` -> gửi IPC `app.thread.scheduleServiceArgs(...)` -> Client gọi `Service.onStartCommand()`.
* **Cấu trúc dữ liệu lưu trữ:**
  - **Start Service:** Trạng thái được kiểm soát bằng `ServiceRecord.startRequested = true`. Dữ liệu các lần gọi `startService` được xếp hàng trong `ServiceRecord.pendingStarts` (danh sách các `ServiceRecord.StartItem`).
  - **Bind Service:** Quản lý bằng đồ thị liên kết:
    - `ServiceRecord.bindings`: Lưu danh sách `IntentBindRecord`.
    - `IntentBindRecord.apps`: Lưu `AppBindRecord` (mỗi tiến trình client bind vào là một record).
    - `AppBindRecord.connections`: Tập hợp các `ConnectionRecord` đại diện cho từng kết nối `ServiceConnection` cụ thể.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/am/ActiveServices.java`
  - `frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java`
  - `frameworks/base/services/core/java/com/android/server/am/ConnectionRecord.java`

---

### Q15. Cơ chế phát hiện và trigger ANR của Service hoạt động như thế nào? Timer được kích hoạt và hủy bỏ ở đâu trong AOSP?
**Trả lời:**
* **Cơ chế Timer:**
  - Thời gian timeout của Service được định nghĩa trong `ActiveServices.java`:
    - `SERVICE_TIMEOUT` = 20 giây (dành cho Foreground Service).
    - `SERVICE_BACKGROUND_TIMEOUT` = 200 giây (dành cho Background Service).
  - Khi hệ thống chuẩn bị gửi lệnh thực thi sang Client (trong `realStartServiceLocked()` hoặc `sendServiceArgsLocked()`), AMS kích hoạt timer:
    `bumpServiceExecutingLocked(r, anyTx, "start")` -> `scheduleServiceTimeoutLocked(r.app)`.
  - Timer này thực chất là một Message `SERVICE_TIMEOUT_MSG` được đẩy vào `ActivityManagerService.mHandler` với thời gian trễ bằng timeout tương ứng.
* **Cơ chế Hủy Timer:**
  - Sau khi Client hoàn tất thực thi trong Main Thread (`handleCreateService()` hoặc `handleServiceArgs()`), Client gửi IPC ngược lại System Server:
    `ActivityManager.getService().serviceDoneExecuting(token, ...)`.
  - Phía AMS: `ActiveServices.serviceDoneExecutingLocked()` được gọi. Hàm này gỡ bỏ `SERVICE_TIMEOUT_MSG` khỏi Handler queue nếu danh sách công việc của Service rỗng.
* **Trigger ANR:**
  - Nếu hết thời gian timeout mà Message chưa bị gỡ bỏ, Handler xử lý `SERVICE_TIMEOUT_MSG` -> gọi `ActiveServices.serviceTimeout()`.
  - AMS thu thập stack trace của process bị lỗi và các process liên quan (`/data/anr/traces.txt`), sau đó gọi `AppErrors.appNotResponding()`.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/am/ActiveServices.java`
  - `frameworks/base/services/core/java/com/android/server/am/AppErrors.java`
* **Pitfall:**
  Một lỗi phổ biến là thực hiện khởi tạo database SQLite hoặc gọi network đồng bộ trong `Service.onStartCommand()`. Vì `onStartCommand()` chạy trên UI Thread của Service process, thao tác này sẽ chặn việc gửi `serviceDoneExecuting` về AMS, gây ra Service ANR.

---

### Q16. Phân tích các ràng buộc mới của Foreground Service (FGS) trong Android 14. Điều gì xảy ra nếu vi phạm `foregroundServiceType` hoặc background start restrictions?
**Trả lời:**
* **Ràng buộc mới trong Android 14:**
  1. **Bắt buộc khai báo Type:** Mọi FGS đều phải khai báo thuộc tính `android:foregroundServiceType` cụ thể trong `AndroidManifest.xml` (ví dụ: `mediaPlayback`, `connectedDevice`, `location`, `dataSync`, `systemExempted`,...).
  2. **Kiểm tra khi gọi runtime:** Khi gọi `Service.startForeground(int id, Notification notification, int foregroundServiceType)`, type truyền vào runtime phải là tập con (hoặc khớp hoàn toàn) với type đã khai báo trong Manifest.
  3. **Yêu cầu Runtime Permissions tương ứng:** Mỗi type đi kèm một quyền runtime cụ thể (ví dụ: `FOREGROUND_SERVICE_MEDIA_PLAYBACK`).
* **Hậu quả vi phạm:**
  - Nếu gọi `startForeground()` mà không khai báo type trong manifest: Hệ thống ném ra `MissingForegroundServiceTypeException`.
  - Nếu app đang ở background và cố tình gọi `startForegroundService()` mà không thuộc danh sách ngoại lệ (ví dụ: có quyền SYSTEM_ALERT_WINDOW, nhận push high priority, companion device manager): Hệ thống ném ra `ForegroundServiceStartNotAllowedException`.
* **Code Framework Android 14:**
```java
// Trong Service.java
public final void startForeground(int id, Notification notification, int foregroundServiceType) {
    try {
        mActivityManager.setServiceForeground(
            new ComponentName(this, mClassName), mToken, id,
            notification, 0, foregroundServiceType);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
}
```
* **Pitfall (Android TV STB):**
  Trên Android TV, các app truyền hình thường duy trì một Service chạy ngầm để lắng nghe bản tin EPG (Electronic Program Guide) hoặc điều khiển HDMI-CEC. Nếu developer cấu hình `dataSync` mà không xử lý các trường hợp timeout theo chính sách mới của Google trên Android 14, Service sẽ bị hệ thống tự động stop và crash sau một khoảng thời gian ngắn chạy nền.

---

## CHỦ ĐỀ 5: BroadcastQueue (Android 14 BroadcastQueueModernImpl) & ANR Broadcast

### Q17. Android 14 giới thiệu `BroadcastQueueModernImpl` thay thế cho `BroadcastQueueImpl` (Legacy). Kiến trúc theo dõi theo tiến trình (`BroadcastProcessQueue`) giải quyết vấn đề cốt lõi nào?
**Trả lời:**
* **Vấn đề của Legacy Queue (`BroadcastQueueImpl`):**
  - Trước Android 14, BroadcastQueue quản lý công việc theo danh sách phẳng gồm hai hàng đợi: Foreground và Background.
  - Các broadcast được dispatch tuần tự tới từng receiver. Nếu một tiến trình chậm chạp hoặc bị treo (slow process) đang xử lý một broadcast, toàn bộ các broadcast phía sau gửi tới các ứng dụng khác đều bị dồn ứ (Head-of-Line Blocking).
* **Kiến trúc mới (`BroadcastQueueModernImpl`):**
  - Chuyển sang mô hình **Per-Process Queuing**: Mỗi tiến trình nhận broadcast sẽ có một hàng đợi riêng gọi là `BroadcastProcessQueue`.
  - Trạng thái phân phối được quản lý độc lập cho từng tiến trình:
    1. Một tiến trình bị lag hay chết chỉ làm chậm hàng đợi của chính nó, không ảnh hưởng đến các tiến trình khác trong hệ thống.
    2. Cho phép tối ưu hóa việc phân phát dựa trên trạng thái thực tế của tiến trình (Running, Cached, Frozen).
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/am/BroadcastQueueModernImpl.java`
  - `frameworks/base/services/core/java/com/android/server/am/BroadcastProcessQueue.java`
  - `frameworks/base/services/core/java/com/android/server/am/BroadcastRecord.java`
* **Lợi ích thực tế:**
  Giảm thiểu tối đa hiện tượng lag toàn hệ thống (system-wide ANR) khi có các đợt phát tán broadcast diện rộng (Broadcast Storm).

---

### Q18. Phân tích cơ chế "Process Freezing" và "Deferred Broadcasts" đối với các tiến trình cached trong Android 14. Cơ chế gộp (coalescing) Intent hoạt động ra sao?
**Trả lời:**
* **Cơ chế bên trong:**
  - Trong Android 14, hệ thống sử dụng Linux Cgroup v2 Freezer để đóng băng (freeze) các tiến trình ở trạng thái Cached nhằm tiết kiệm CPU và Pin.
  - Khi một Broadcast được gửi tới một tiến trình đang bị đóng băng (`isProcessFrozen() == true`):
    - `BroadcastQueueModernImpl` không đánh thức tiến trình ngay.
    - Thay vào đó, nó đưa broadcast vào hàng đợi trì hoãn (**Deferred Queue**) nằm trong `BroadcastProcessQueue`.
  - Khi tiến trình được unfreeze (ví dụ do người dùng mở lại app), toàn bộ deferred broadcasts mới được flush và dispatch đồng loạt.
* **Cơ chế Coalescing (Gộp Broadcast):**
  - Đối với các broadcast có tính chất cập nhật trạng thái liên tục (ví dụ: `Intent.ACTION_TIME_TICK`, `BATTERY_CHANGED`, các custom status updates):
  - `BroadcastProcessQueue` tự động kiểm tra các Intent trùng lặp trong hàng đợi chờ của tiến trình đó.
  - Hệ thống sẽ loại bỏ các Intent cũ và chỉ giữ lại Intent mới nhất -> Tiết kiệm chu kỳ đánh thức CPU và tránh overload message queue khi app rã đông.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/am/BroadcastProcessQueue.java`
  - `frameworks/base/services/core/java/com/android/server/am/CachedAppOptimizer.java`
* **Pitfall (Android TV STB):**
  Nếu viết system service trên STB gửi custom broadcast báo trạng thái mạng/kết nối tuner tới background app, nếu không gắn cờ `FLAG_RECEIVER_FOREGROUND` hoặc không cấu hình delivery group policies, broadcast có thể bị delay hàng chục phút cho tới khi người dùng tương tác mở app đó lên.

---

### Q19. Cơ chế tính toán Timeout và trigger Broadcast ANR trong Android 14 diễn ra như thế nào? Sự khác nhau giữa Ordered Broadcast và Normal Broadcast?
**Trả lời:**
* **Cơ chế bên trong:**
  - Timeout của Broadcast:
    - Foreground Broadcast: Thường là 10 - 20 giây.
    - Background Broadcast: 60 giây (trên Android 14 có thể lên tới 120 giây tùy cấu hình DeviceConfig).
  - Khi một broadcast được gửi tới một Client thông qua `app.thread.scheduleRegisteredReceiver()` hoặc `scheduleReceiver()`:
    Hệ thống tạo một token `DeliveryToken` và đặt timer tracking.
  - Phía Client: Sau khi `BroadcastReceiver.onReceive()` kết thúc, Client gọi `ActivityManager.getService().finishReceiver(token, ...)`.
  - Phía AMS: Nếu hết thời gian timeout mà `finishReceiver()` chưa được gọi, AMS xác định receiver đó bị treo và gọi `BroadcastQueue.deliveryTimeoutLocked()`.
* **Ordered vs Normal Broadcast:**
  - **Normal (Parallel) Broadcast:** Gửi đồng loạt tới tất cả các dynamic registered receivers cùng lúc. Trong `BroadcastQueueModernImpl`, mỗi process chạy song song.
  - **Ordered Broadcast:** Gửi tuần tự theo thứ tự priority (`android:priority`). Receiver sau chỉ nhận được intent khi receiver trước đã gọi xong `onReceive()` hoặc gọi `finishReceiver()`. Nếu một receiver trong chuỗi bị timeout, toàn bộ chuỗi phía sau bị block cho đến khi hệ thống trigger ANR và skip receiver lỗi.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/am/BroadcastQueueModernImpl.java`
* **Pitfall:**
  Gọi `goAsync()` trong `onReceive()` nhưng quên gọi `PendingResult.finish()`. Khi đó, timer ANR của Broadcast vẫn tiếp tục đếm trong System Server, và ANR sẽ xảy ra sau khi hết timeout dù UI thread của app hoàn toàn rảnh rỗi.

---

## CHỦ ĐỀ 6: ContentProvider publish & ContentProviderHolder

### Q20. Tại sao `ContentProvider.onCreate()` luôn chạy TRƯỚC `Application.onCreate()` của ứng dụng? Luồng thực thi chi tiết trong `ActivityThread.handleBindApplication()`.
**Trả lời:**
* **Cơ chế bên trong:**
  1. Trong `ActivityThread.handleBindApplication()`, tiến trình khởi tạo Context, LoadedApk và `Instrumentation`.
  2. Hệ thống gọi `LoadedApk.makeApplication()` để tạo instance của `Application`, nhưng **chưa gọi** `Application.onCreate()`.
  3. Ngay sau đó, hàm `installContentProviders(app, data.providers)` được gọi:
     - Duyệt danh sách `ProviderInfo` được cấu hình trong Manifest.
     - Với mỗi provider, gọi `installProvider(app, null, cpi, ...)`:
       * Dùng ClassLoader nạp class và khởi tạo instance `ContentProvider`.
       * Gắn Context vào provider: `localProvider.attachInfo(c, cpi)`.
       * Trong `attachInfo()`, phương thức `ContentProvider.onCreate()` được kích hoạt.
  4. Sau khi tất cả local providers khởi tạo xong và được publish lên AMS, `ActivityThread` mới gọi `mInstrumentation.callApplicationOnCreate(app)`.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/ActivityThread.java`
  - `frameworks/base/core/java/android/content/ContentProvider.java`
* **Đoạn code thực thi trong `ActivityThread.java`:**
```java
// ActivityThread.java -> handleBindApplication()
Application app = data.info.makeApplicationInner(data.restrictedBackupMode, null);
mInitialApplication = app;

if (!data.restrictedBackupMode) {
    if (!ArrayUtils.isEmpty(data.providers)) {
        installContentProviders(app, data.providers);
    }
}

// Application.onCreate() chỉ được gọi sau khi cài đặt xong Providers
mInstrumentation.callApplicationOnCreate(app);
```
* **Pitfall:**
  Rất nhiều thư viện (như Firebase, WorkManager, Jetpack App Startup) lợi dụng cơ chế này để tự khởi tạo bằng cách nhét một Provider rỗng vào Manifest. Nếu trong `ContentProvider.onCreate()` mà truy cập vào các biến singleton dự kiến được khởi tạo ở `Application.onCreate()`, app sẽ crash ngay lập tức vì `NullPointerException`.

---

### Q21. `ContentProviderHolder` là gì? Phân tích luồng `getContentProvider()` từ Client sang AMS và cơ chế Publish Provider ngược lại hệ thống.
**Trả lời:**
* **`ContentProviderHolder`:**
  - Là một đối tượng Parcelable (`android.app.ContentProviderHolder`) dùng để vận chuyển thông tin của một ContentProvider qua Binder IPC giữa System Server và Client.
  - Chứa: `ProviderInfo`, `IContentProvider` (Binder proxy interface để client query dữ liệu), và token theo dõi liên kết.
* **Luồng `getContentProvider()`:**
  1. Client gọi `context.getContentResolver().query(...)` -> dẫn tới `ActivityThread.acquireProvider()`.
  2. Nếu client chưa có instance của provider trong cache nội bộ (`mProviderMap`), nó gọi IPC sang AMS:
     `ActivityManager.getService().getContentProvider(mAppThread, name, userId, stable)`.
  3. Phía AMS (`ActivityManagerService.getContentProviderImpl()`):
     - Kiểm tra xem process chứa Provider đó đã chạy chưa.
     - Nếu đã chạy: Lấy `IContentProvider` từ `ContentProviderRecord` và đóng gói vào `ContentProviderHolder` trả về cho Client.
     - Nếu chưa chạy: AMS kích hoạt `startProcessLocked()` để fork tiến trình mới. Lúc này, thread của Client gọi Binder sẽ bị **block** (chờ đợi trên condition variable) cho tới khi tiến trình chứa provider khởi động xong.
  4. Tiến trình mới khởi động xong, trong `handleBindApplication()`, gọi `installContentProviders()` -> gọi IPC ngược lại AMS:
     `ActivityManager.getService().publishContentProviders(mAppThread, providersList)`.
  5. AMS đánh thức các Client đang chờ và trả về `ContentProviderHolder`.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/ContentProviderHolder.java`
  - `frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`
  - `frameworks/base/services/core/java/com/android/server/am/ContentProviderRecord.java`
* **Pitfall:**
  Nếu hai App A và App B có ContentProvider và trong quá trình khởi tạo (`onCreate`), Provider A thực hiện query sang Provider B, đồng thời Provider B cũng query ngược lại Provider A, hệ thống sẽ rơi vào trạng thái **Deadlock liên tiến trình**, làm treo vĩnh viễn cả hai app và gây nghẽn Binder threads trong System Server.

---

### Q22. Khái niệm "Unstable ContentProvider" (`acquireUnstableContentProviderClient`) giải quyết vấn đề gì trong kiến trúc IPC của Android?
**Trả lời:**
* **Vấn đề của Stable Provider:**
  - Mặc định, khi Client kết nối tới một ContentProvider qua `acquireContentProviderClient()`, kết nối này là **Stable**.
  - AMS sẽ tạo một mối liên kết chặt chẽ (`ContentProviderConnection` với cờ `stable = true`).
  - Nếu tiến trình chứa Provider bị crash hoặc bị `lmkd` kill, Linux kernel sẽ gửi thông báo chết qua Binder. AMS sẽ phát hiện và **lập tức kill luôn tiến trình Client** để đảm bảo tính nhất quán dữ liệu (tránh việc Client tiếp tục hoạt động với trạng thái hỏng).
* **Cơ chế của Unstable Provider:**
  - Sử dụng `acquireUnstableContentProviderClient()`: Liên kết được đánh dấu là `stable = false`.
  - Phía Client, `ActivityThread` bọc interface `IContentProvider` vào một proxy kiểm soát ngoại lệ.
  - Nếu tiến trình chứa Provider bị chết:
    - AMS **không** kill tiến trình Client.
    - Cuộc gọi Binder tiếp theo từ Client tới Provider sẽ ném ra `DeadObjectException`.
    - Client có cơ hội bắt exception này, giải phóng connection, và thử kết nối lại (kích hoạt AMS hồi sinh tiến trình Provider) mà không làm gián đoạn trải nghiệm người dùng.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/content/ContentResolver.java`
  - `frameworks/base/services/core/java/com/android/server/am/ContentProviderConnection.java`
* **Ứng dụng trên Android TV STB:**
  Các ứng dụng System UI / Leanback Launcher khi query danh sách kênh hoặc metadata video từ các ứng dụng streaming của bên thứ ba (qua TvProvider) bắt buộc phải sử dụng Unstable ContentProvider. Nếu không, chỉ cần một app bên thứ ba bị crash ngầm, Leanback Launcher của TV sẽ bị hệ thống kéo chết theo.

---

## CHỦ ĐỀ 7: PackageManagerService (Boot Scan, PackageInstaller, Dexopt, Packages.xml, Signatures)

### Q23. Trình bày chi tiết giai đoạn Boot Scan của `PackageManagerService` (PMS). Kiến trúc Android 14 đã tái cấu trúc PMS (Computer Engine Pattern) ra sao?
**Trả lời:**
* **Giai đoạn Boot Scan:**
  1. Khi System Server khởi động `PackageManagerService`, PMS quét các phân vùng theo thứ tự ưu tiên:
     - `/system/framework`
     - `/system/priv-app` (ứng dụng đặc quyền hệ thống)
     - `/system/app` (ứng dụng hệ thống thông thường)
     - `/vendor/priv-app`, `/vendor/app` (ứng dụng của nhà sản xuất SoC/phần cứng)
     - `/product/priv-app`, `/product/app`
     - Cuối cùng là `/data/app` (ứng dụng người dùng cài đặt).
  2. Với mỗi file APK, hệ thống sử dụng `ApkLiteUtils` / `ParsingPackageUtils` để parse `AndroidManifest.xml`, xác thực chữ ký (signatures) và lưu trữ metadata vào cấu trúc RAM.
* **Tái cấu trúc Computer Engine Pattern trong Android 14:**
  - Trước đây, PMS bị nghẽn cổ chai nghiêm trọng do sử dụng một khóa toàn cục duy nhất (`mLock`). Mọi thao tác đọc (query intent, check permission) và ghi (install, uninstall) đều tranh chấp lock này.
  - Android 14 tách biệt hoàn toàn giữa **State Mutation (Ghi)** và **Query (Đọc)**:
    - Sử dụng lớp `Computer` (hoặc `ComputerEngine`).
    - Mỗi khi trạng thái hệ thống thay đổi (ví dụ cài đặt xong app mới), PMS tạo ra một snapshot bất biến (immutable snapshot) của dữ liệu gói gọi là `Computer`.
    - Mọi lời gọi đọc (`queryIntentActivities`, `getPackageInfo`) đều được phân phối cho instance `Computer` hiện tại thực thi mà **hoàn toàn không cần giữ `mLock`**.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java`
  - `frameworks/base/services/core/java/com/android/server/pm/Computer.java`
  - `frameworks/base/services/core/java/com/android/server/pm/ComputerEngine.java`

---

### Q24. Phân tích vòng đời cài đặt ứng dụng qua `PackageInstallerSession`. Từ lúc ghi APK đến khi rename staging directory sang `/data/app/...`.
**Trả lời:**
* **Cơ chế bên trong:**
  1. **Khởi tạo Session:** App cài đặt gọi `PackageInstaller.createSession()` -> PMS tạo một `PackageInstallerSession` với một `sessionId` định danh, lưu trong `/data/app/vmdl<session_id>.tmp`.
  2. **Ghi dữ liệu (Write Streams):** Client mở stream qua `session.openWrite()` và truyền bytes của APK vào file staging descriptor.
  3. **Commit:** Client gọi `session.commit()`. Lúc này quyền điều khiển thuộc về PMS.
  4. **Validation & Parsing:**
     - PMS chuyển session vào hàng đợi cài đặt (`PackageInstallObserver`).
     - Dùng `Paging/PackageParser` parse APK từ thư mục staging.
     - Xác thực tính hợp lệ của Signature (chữ ký mới phải khớp chữ ký cũ nếu là cập nhật).
  5. **Dexopt (Biên dịch ART):** Kích hoạt `artd` (ART Service daemon) biên dịch DEX bytecode theo compiler filter cấu hình.
  6. **Reconciliation & Renaming:**
     - Sau khi verify thành công, PMS lock trạng thái.
     - Atomic rename thư mục staging từ `/data/app/vmdl<session_id>.tmp` sang đường dẫn chính thức: `/data/app/~~[random_hash]/[package_name]-[random_hash]/`.
     - Cập nhật thông tin vào `Settings` (ghi `packages.xml`).
  7. **Broadcast:** Bắn broadcast hệ thống `ACTION_PACKAGE_ADDED` hoặc `PACKAGE_REPLACED`.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java`
  - `frameworks/base/services/core/java/com/android/server/pm/InstallPackageHelper.java`
* **Pitfall (Android TV STB OTA):**
  Trên TV STB, bộ nhớ flash NAND/eMMC thường chỉ có 8GB. Khi thực hiện silent update các ứng dụng lớn (như YouTube, Netflix), nếu dung lượng trống của phân vùng `/data` nhỏ hơn kích thước staging file + kích thước file dexopt, quá trình rename hoặc dexopt sẽ thất bại giữa chừng, để lại các file rác `.tmp` nếu không có cơ chế rollback dọn dẹp tốt.

---

### Q25. Tệp `packages.xml` và `packages.list` lưu trữ những dữ liệu gì? Hậu quả và cơ chế phục hồi nếu `packages.xml` bị hỏng (corrupt) trong quá trình thiết bị mất điện đột ngột?
**Trả lời:**
* **Nội dung lưu trữ:**
  - **`packages.xml`** (Nằm tại `/data/system/packages.xml`): Là "trái tim" dữ liệu của PMS.
    - Danh sách tất cả các package đã cài đặt trên hệ thống.
    - Cấu hình UID phân bổ cho từng app (`userId="10045"`).
    - Quyền đã cấp (`<perms>`), trạng thái enabled/disabled của từng component.
    - Code path (`codePath="/data/app/..."`), signatures keyset.
  - **`packages.list`**: Tệp văn bản đơn giản phục vụ cho tiến trình Zygote và SELinux để map nhanh giữa `package_name`, `UID`, `debug_flag`, và `seinfo`.
* **Cơ chế ghi an toàn (Atomic File):**
  - AOSP sử dụng class `AtomicFile` để ghi `packages.xml`.
  - Quá trình ghi luôn thực hiện trên file tạm `packages.xml.new`, sau đó đồng bộ dữ liệu xuống đĩa (`fsync()`), rồi mới đổi tên đè lên `packages.xml`. Trước đó, file cũ được backup thành `packages.xml.bak`.
* **Hậu quả khi bị Corrupted & Phục hồi:**
  - Nếu STB bị rút điện đột ngột đúng thời điểm fsync thất bại và file `packages.xml` bị 0 byte hoặc sai định dạng XML:
  - Khi reboot, PMS phát hiện XML parse error -> lập tức tìm file dự phòng `packages.xml.bak` để khôi phục.
  - **Kịch bản tồi tệ nhất:** Nếu cả `.bak` cũng hỏng, PMS buộc phải kích hoạt chế độ **Re-scan toàn bộ hệ thống**. Toàn bộ UID của các app bên thứ 3 trong `/data/app` sẽ bị cấp phát lại từ đầu. Điều này dẫn đến lỗi nghiêm trọng: App không thể truy cập vào thư mục dữ liệu cũ của chính mình trong `/data/data/<pkg>` do sai lệch Linux UID ownership (Permission Denied).
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/pm/Settings.java`
  - `frameworks/base/core/java/android/util/AtomicFile.java`

---

### Q26. Phân tích sự tiến hóa của Android Signature Scheme (v1 đến v4). Tại sao `android:sharedUserId` bị khai tử (deprecated) trong Android hiện đại?
**Trả lời:**
* **Tiến hóa Signature Scheme:**
  - **v1 (JAR signing):** Xác thực dựa trên việc kiểm tra hash của từng file giải nén trong archive META-INF. Rất chậm và có lỗ hổng bảo mật (có thể thay đổi metadata ZIP mà không làm hỏng chữ ký).
  - **v2 (APK Signing Block):** Nhúng khối chữ ký vào giữa file ZIP (trước Central Directory). Kiểm tra toàn vẹn toàn bộ file nhị phân bằng cây băm Merkle Tree. Tốc độ verify cực nhanh.
  - **v3 (Key Rotation):** Bổ sung cấu trúc bằng chứng xoay vòng khóa (`Proof-of-rotation`). Cho phép app thay đổi signing key mới mà vẫn duy trì sự tin cậy với key cũ thông qua lịch sử ký chuỗi.
  - **v4 (Streaming / Incremental):** Chữ ký được lưu trong file riêng biệt `base.apk.idsig`. Dựa trên cấu trúc Merkle tree của Linux `fs-verity`, cho phép ứng dụng khởi chạy ngay lập tức khi mới chỉ tải được một phần APK (rất quan trọng cho cloud gaming/streaming app trên TV).
* **Tại sao `android:sharedUserId` bị deprecated?**
  - `sharedUserId` cho phép hai hoặc nhiều app ký cùng cert được chạy chung một Linux UID, dùng chung data sandbox.
  - **Lý do khai tử:**
    1. **Phá vỡ ranh giới bảo mật Sandbox:** Một app bị hack đồng nghĩa toàn bộ các app dùng chung UID bị lộ dữ liệu.
    2. **Ảnh hưởng nghiêm trọng đến PMS:** Việc gán chung UID khiến logic tính toán permission, package state và quản lý vòng đời trong PMS bị phân nhánh phức tạp, gây nhiều lỗ hổng privilege escalation.
    3. **Không thể đảo ngược:** Một app đã gán `sharedUserId` thì không thể gỡ bỏ nếu không xóa toàn bộ data cũ của người dùng.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/util/apk/ApkSignatureVerifier.java`
  - `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java`

---

## CHỦ ĐỀ 8: Permission Framework & Priv-App Allowlist

### Q27. Phân biệt các cấp độ bảo vệ quyền: `normal`, `dangerous`, `signature`, `privileged`, và `appop`. Kiến trúc `PermissionManagerService` trong Android 14 quản lý chúng ra sao?
**Trả lời:**
* **Phân loại Protection Levels:**
  - **`normal`:** Cấp tự động khi cài đặt (ví dụ: `INTERNET`, `BLUETOOTH`). Rủi ro thấp.
  - **`dangerous` (Runtime):** Yêu cầu người dùng cấp quyền tại runtime qua UI dialog (ví dụ: `RECORD_AUDIO`, `ACCESS_FINE_LOCATION`).
  - **`signature`:** Chỉ cấp nếu ứng dụng yêu cầu được ký bởi cùng certificate với package định nghĩa quyền đó (thường là platform key của OEM).
  - **`privileged` (trước đây là `signatureOrSystem`):** Dành riêng cho các ứng dụng hệ thống nằm trong phân vùng đặc quyền (`/system/priv-app`, `/vendor/priv-app`).
  - **`appop`:** Không phải là một permission truyền thống mà là một switch kiểm soát hành vi động trong `AppOpsService` (ví dụ: quyền vẽ đè màn hình `SYSTEM_ALERT_WINDOW`, quyền truy cập file đặc biệt).
* **Kiến trúc `PermissionManagerService` (Android 14):**
  - Đã được module hóa và tách biệt khỏi PMS core.
  - Quản lý trạng thái cấp quyền bằng `PermissionRegistry`.
  - Lưu trữ trạng thái runtime permissions độc lập trong file `/data/system/users/<user_id>/runtime-permissions.xml`.
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java`
  - `frameworks/base/services/core/java/com/android/server/pm/permission/PermissionRegistry.java`
* **Cơ chế cấp quyền nội bộ:**
```java
// Logic kiểm tra permission nội bộ
public int checkPermission(String permName, String pkgName, int userId) {
    AndroidPackage pkg = mPackageManagerInt.getPackage(pkgName);
    if (pkg == null) return PERMISSION_DENIED;
    
    // Kiểm tra permission state từ registry
    final int uid = UserHandle.getUid(userId, pkg.getUid());
    return checkUidPermission(uid, permName);
}
```

---

### Q28. Cơ chế "Priv-App Permissions Allowlist" (`privapp-permissions-*.xml`) hoạt động như thế nào? Chuyện gì xảy ra nếu thiết bị bật `ro.control_privapp_permissions=enforce` mà app thiếu cấu hình trong XML?
**Trả lời:**
* **Cơ chế bên trong:**
  - Các app đặt trong `/system/priv-app` hoặc `/vendor/priv-app` được cấp các quyền nguy hiểm cấp hệ thống (`privileged permissions`).
  - Để tránh việc app hệ thống tự ý dùng quyền tùy tiện, AOSP áp dụng cơ chế xác thực danh sách trắng (Allowlist).
  - OEM phải định nghĩa file XML đặt tại `/etc/permissions/privapp-permissions-[tên_hệ_thống].xml`:
```xml
<!-- /system/etc/permissions/privapp-permissions-tvlauncher.xml -->
<permissions>
    <privapp-permissions package="com.vendor.tv.launcher">
        <permission name="android.permission.MEDIA_CONTENT_CONTROL"/>
        <permission name="android.permission.STATUS_BAR_SERVICE"/>
    </privapp-permissions>
</permissions>
```
* **Hậu quả khi vi phạm (`ro.control_privapp_permissions=enforce`):**
  - Trong quá trình boot scan của PMS, lớp `SystemConfig.java` đọc toàn bộ file XML allowlist.
  - Khi PMS phân tích một priv-app, nó so khớp các quyền `privileged` trong Manifest của app với allowlist đã đọc:
    1. Nếu thiếu và hệ thống đang ở mode `log`: Hệ thống chỉ ghi log cảnh báo `WARNING`.
    2. Nếu thuộc tính hệ thống `ro.control_privapp_permissions=enforce` (mặc định trên mọi build `user` của Android TV thương mại):
       **PMS sẽ cố tình ném ra một Exception nghiêm trọng làm crash System Server ngay lập tức!**
       Hậu quả: **Thiết bị rơi vào vòng lặp Bootloop vĩnh viễn (System Server crash loop).**
* **File/Class AOSP:**
  - `frameworks/base/services/core/java/com/android/server/SystemConfig.java`
  - `frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java`
* **Pitfall (Android TV STB):**
  Khi một engineer tích hợp thêm một quyền mới (ví dụ `REBOOT` hoặc `CAPTURE_AUDIO_OUTPUT` cho tính năng điều khiển giọng nói TV) vào file Manifest của launcher mà quên tạo commit bổ sung file XML vào thư mục `vendor/etc/permissions/`, bản build firmware `user` tạo ra sẽ bị bootloop ngay lần khởi động đầu tiên.

---

### Q29. Phân biệt cơ chế kiểm tra quyền giữa Linux UID check, Framework Permission check, và `AppOpsManager`. Tại sao AppOps có thể chặn âm thầm (silent deny)?
**Trả lời:**
* **Kiến trúc bảo mật 3 lớp:**
  1. **Lớp 1: Linux Kernel/UID Check:**
     Dựa trên Linux GID/UID và kernel capabilities. Ví dụ: Socket network mở bởi tiến trình có GID `inet` (AID_INET = 3003). Nếu không có GID này, kernel trả về `EACCES` (Permission denied).
  2. **Lớp 2: Framework Permission Check (`checkCallingPermission`):**
     Được thực hiện trong Binder interfaces của các System Service. Dựa trên token UID của caller. Nếu app chưa được cấp quyền, ném ra `SecurityException`.
  3. **Lớp 3: AppOps (`AppOpsManager` / `AppOpsService`):**
     Theo dõi và điều chỉnh quyền truy cập vào các tính năng riêng tư theo thời gian thực (Microphone, Camera, Overlay, Location).
* **Cơ chế "Silent Deny" của AppOps:**
  - Khi một app bị thu hồi quyền qua AppOps (ví dụ người dùng tắt micro qua phím cứng trên remote TV, hoặc app bị đưa vào trạng thái background restricted):
  - Khác với Framework Permission (sẽ ném `SecurityException` làm crash app), AppOps cho phép thực hiện **Silent Denial**:
    - Khi app đọc audio record: Framework không ném lỗi, nhưng HAL sẽ trả về buffer toàn byte `0` (âm thanh im lặng).
    - Khi app query camera: Trả về frame đen hoặc dữ liệu rỗng.
    - Khi app lấy vị trí: Trả về `null` hoặc vị trí giả lập.
  - Điều này bảo vệ tính toàn vẹn của ứng dụng cũ (vốn không được thiết kế để xử lý việc bị mất quyền đột ngột) mà vẫn bảo vệ được quyền riêng tư của người dùng.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/AppOpsManager.java`
  - `frameworks/base/services/core/java/com/android/server/appop/AppOpsService.java`

---

## CHỦ ĐỀ 9: Context, ContextImpl, SystemServiceRegistry

### Q30. Trình bày cấu trúc phân tầng: `Context`, `ContextWrapper`, `ContextThemeWrapper`, và `ContextImpl`. Tại sao gọi `ContextImpl` là "ngựa kéo" (workhorse)?
**Trả lời:**
* **Mối quan hệ kế thừa & Decorator Pattern:**
  - `Context` là một abstract class định nghĩa các API trừu tượng (như `openFileInput()`, `getSystemService()`, `startActivity()`).
  - `ContextImpl` là class cài đặt thực tế (concrete implementation) của hầu hết mọi API được khai báo trong `Context`. Mọi giao tiếp với Binder, nạp file, quản lý thư mục data đều thực thi tại đây -> vì vậy nó được gọi là **Workhorse**.
  - `ContextWrapper` là một Proxy class kế thừa `Context`, chứa một biến tham chiếu ngầm:
    `Context mBase;`
    Mọi lời gọi hàm trên `ContextWrapper` chỉ đơn giản là ủy quyền tiếp: `mBase.startActivity(intent)`.
  - `ContextThemeWrapper` kế thừa `ContextWrapper`, bổ sung thêm logic quản lý giao diện (`Theme`, `Resources`).
* **Sơ đồ kế thừa component:**
  - `Application` kế thừa trực tiếp từ `ContextWrapper`.
  - `Service` kế thừa trực tiếp từ `ContextWrapper`.
  - `Activity` kế thừa từ `ContextThemeWrapper` (do Activity bắt buộc phải có Theme UI).
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/content/Context.java`
  - `frameworks/base/core/java/android/content/ContextWrapper.java`
  - `frameworks/base/core/java/android/view/ContextThemeWrapper.java`
  - `frameworks/base/core/java/android/app/ContextImpl.java`

---

### Q31. Phân biệt UI Context và Non-UI Context. Những lỗi hoặc rò rỉ nào sẽ xảy ra nếu truyền nhầm `Application Context` vào `LayoutInflater` hoặc tạo View trên Android 14?
**Trả lời:**
* **Sự khác biệt cốt lõi:**
  - **UI Context (`Activity`, `WindowContext`):** Được liên kết với một `Display` cụ thể và một cấu hình giao diện động (`Configuration`: density, orientation, font scale). Chứa token cửa sổ của Window Manager (`IBinder`).
  - **Non-UI Context (`Application`, `Service`, `ContextImpl` cơ bản):** Được tạo ra từ hệ thống toàn cục, không gắn với bất kỳ display logic nào và không có Window Token.
* **Các lỗi xảy ra khi dùng nhầm `Application Context` cho UI:**
  1. **Hiển thị sai Density / Cấu hình TV:**
     Trên TV STB hỗ trợ đa màn hình (hoặc xuất tín hiệu HDMI 4K trong khi UI render 1080p), `Application Context` chỉ mang metrics mặc định của hệ thống. Nếu dùng nó để `LayoutInflater.inflate()`, các kích thước `dp`, `sp` sẽ bị tính toán sai, dẫn tới giao diện bị vỡ, icon bị phóng quá to hoặc quá nhỏ.
  2. **Crash khi mở Dialog:**
     Gọi `new AlertDialog.Builder(applicationContext).show()` sẽ crash ngay lập tức với lỗi:
     `WindowManager$BadTokenException: Unable to add window -- token null is not valid; is your activity running?`
     (Do Application Context không có `windowToken` được cấp phép bởi WMS).
  3. **Vi phạm StrictMode trên Android 14:**
     Android 14 siết chặt việc truy cập Visual Components thông qua Non-UI context. Hệ thống sẽ bắn ra cảnh báo `IncorrectContextUseViolation`.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/view/LayoutInflater.java`
  - `frameworks/base/core/java/android/view/WindowManagerGlobal.java`

---

### Q32. Cơ chế hoạt động của `SystemServiceRegistry.java`. Phân tích 3 loại Service Fetchers (Cached, Static, Uncached).
**Trả lời:**
* **Cơ chế bên trong:**
  - Khi gọi `context.getSystemService(String name)`, `ContextImpl` không trực tiếp truy vấn Service qua ServiceManager mỗi lần. Thay vào đó, nó chuyển quyền qua `SystemServiceRegistry`.
  - `SystemServiceRegistry` lưu trữ một ánh xạ tĩnh:
    `Map<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS`
    Được đăng ký trong khối static khởi tạo (`registerService(...)`).
* **3 loại Service Fetchers:**
  1. **`StaticServiceFetcher`:**
     - Instance của Service được tạo **duy nhất một lần** cho toàn bộ tiến trình và được cache vĩnh viễn trong registry (ví dụ: `ConnectivityManager`). Mọi Context gọi hàm đều nhận chung một object.
  2. **`CachedServiceFetcher`:**
     - Instance được cache ở cấp độ **từng instance của `ContextImpl`**.
     - Dữ liệu lưu trong mảng `mServiceCache` của `ContextImpl`. Lần đầu gọi sẽ tạo mới, các lần sau lấy từ cache của chính context đó.
     - Ví dụ: `LayoutInflater` (mỗi Activity cần một LayoutInflater riêng mang Theme của Activity đó).
  3. **`StaticApplicationContextServiceFetcher`:**
     - Service cần Context của Application để khởi tạo một lần duy nhất trong toàn app.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/app/SystemServiceRegistry.java`
  - `frameworks/base/core/java/android/app/ContextImpl.java`
* **Code Framework:**
```java
// Đăng ký Service trong SystemServiceRegistry.java
registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
    new CachedServiceFetcher<LayoutInflater>() {
        @Override
        public LayoutInflater createService(ContextImpl ctx) {
            return new PhoneLayoutInflater(ctx.getOuterContext());
        }});
```
* **Pitfall:**
  Nếu lưu giữ tham chiếu (static reference) của một service thuộc loại `CachedServiceFetcher` (như `WindowManager` lấy từ Activity) vào một biến tĩnh toàn cục (Singleton), `Activity Context` tạo ra service đó sẽ không bao giờ được GC giải phóng -> Rò rỉ toàn bộ cây View và Window của Activity đó.

---

## CHỦ ĐỀ 10: SettingsProvider (System/Secure/Global) & Hidden API Enforcement

### Q33. Phân biệt bản chất lưu trữ, phạm vi ảnh hưởng và quyền truy cập giữa 3 bảng: `Settings.Global`, `Settings.Secure`, và `Settings.System`. Cơ chế cache phía client tối ưu IPC ra sao?
**Trả lời:**
* **Phân biệt 3 bảng:**
  1. **`Settings.Global`:**
     - Phạm vi: Toàn thiết bị, áp dụng chung cho mọi user profile.
     - Dữ liệu: Cấu hình phần cứng, mạng (ví dụ: `adb_enabled`, `wifi_on`, `hdmi_control_enabled`).
     - Quyền ghi: Yêu cầu quyền cấp hệ thống `android.permission.WRITE_SECURE_SETTINGS`.
     - File lưu: `/data/system/users/0/settings_global.xml`.
  2. **`Settings.Secure`:**
     - Phạm vi: Riêng biệt cho từng User (`userId`).
     - Dữ liệu: Cấu hình an toàn, riêng tư (ví dụ: `enabled_input_methods`, `location_mode`).
     - Quyền ghi: Yêu cầu quyền `WRITE_SECURE_SETTINGS` (App thường chỉ được đọc).
     - File lưu: `/data/system/users/<id>/settings_secure.xml`.
  3. **`Settings.System`:**
     - Phạm vi: Riêng biệt cho từng User (mang tính legacy).
     - Dữ liệu: Tùy chọn người dùng (ví dụ: âm lượng `volume_music`, độ sáng màn hình).
     - Quyền ghi: App bên thứ 3 có thể ghi nếu được cấp quyền `WRITE_SETTINGS` qua Settings toggle.
     - File lưu: `/data/system/users/<id>/settings_system.xml`.
* **Cơ chế Cache phía Client (`Settings.GenerationTracker` / `NameValueCache`):**
  - Việc query Settings diễn ra liên tục. Nếu mỗi lần đọc đều dùng Binder IPC gọi sang tiến trình `com.android.providers.settings`, hệ thống sẽ bị nghẽn IPC.
  - AOSP giải quyết bằng class `NameValueCache` bên trong `Settings.java`:
    1. Client giữ một memory cache cục bộ dạng `ArrayMap<String, String>`.
    2. Sử dụng cơ chế bộ nhớ chia sẻ hoặc generation tracker: Phía `SettingsProvider` duy trì một biến phiên bản (generation int) cho từng bảng.
    3. Khi Client gọi `getString()`, nó chỉ cần kiểm tra xem generation number của bảng có thay đổi không:
       - Nếu không đổi: Lấy trực tiếp giá trị từ RAM cache của client mà **không tốn một cuộc gọi IPC Binder nào**.
       - Nếu generation tăng lên (do có ai đó vừa ghi đè settings): Client thực hiện IPC một lần để fetch dữ liệu mới và cập nhật cache.
* **File/Class AOSP:**
  - `frameworks/base/core/java/android/provider/Settings.java`
  - `frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java`

---

### Q34. Trên Android TV STB, làm thế nào để cấu hình các giá trị default settings ngay từ khi xuất xưởng (First Boot) và cơ chế RRO (Runtime Resource Overlay) can thiệp vào đây thế nào?
**Trả lời:**
* **Cơ chế cấu hình mặc định (Defaults):**
  1. Khi khởi động lần đầu (First Boot), database của `SettingsProvider` chưa tồn tại.
  2. Lớp `DatabaseHelper.java` (hoặc `SettingsProvider.java`) sẽ kích hoạt logic nạp dữ liệu ban đầu từ tài nguyên XML mặc định được định nghĩa tại:
     `frameworks/base/packages/SettingsProvider/res/values/defaults.xml`.
  3. Các giá trị như: âm thanh mặc định, ngôn ngữ, sleep timeout, chế độ HDMI-CEC được định nghĩa tại đây:
```xml
<resources>
    <bool name="def_hdmi_control_enabled">true</bool>
    <integer name="def_sleep_timeout_entries">1800000</integer>
</resources>
```
* **Cơ chế can thiệp qua RRO (Runtime Resource Overlay):**
  - Nhà sản xuất STB không được sửa trực tiếp vào mã nguồn AOSP gốc để đảm bảo tính module hóa và dễ nâng cấp Android version.
  - Thay vào đó, OEM tạo một package RRO dạng APK overlay (ví dụ: `SettingsProviderOverlay.apk`) đặt tại `/vendor/overlay/` hoặc `/product/overlay/`.
  - Package này chứa một `defaults.xml` tùy biến:
```xml
<!-- device/vendor/stb/overlay/SettingsProvider/res/values/defaults.xml -->
<resources>
    <!-- Ghi đè: Mặc định bật HDMI CEC trên STB -->
    <bool name="def_hdmi_control_enabled">true</bool>
    <!-- Đổi thời gian sleep timeout riêng cho TV Box -->
    <integer name="def_sleep_timeout_entries">7200000</integer> 
</resources>
```
  - Khi hệ thống boot, `idmap2` sẽ liên kết các resource ID của overlay vào APK gốc của `SettingsProvider`, giúp nạp cấu hình của OEM mà không cần sửa một dòng code Java nào.
* **Pitfall:**
  Nếu cấu hình sai tên biến trong file overlay XML hoặc package overlay không khớp chữ ký (hoặc thiếu target package name), `SettingsProvider` sẽ âm thầm sử dụng giá trị mặc định của AOSP, dẫn đến việc thiết bị xuất xưởng bị sai cấu hình tiêu chuẩn của nhà mạng.

---

### Q35. Trình bày chi tiết cơ chế "Hidden API Enforcement" của ART Runtime và Framework. Làm sao runtime phát hiện việc gọi Private API qua Reflection?
**Trả lời:**
* **Phân loại danh sách API (API Lists):**
  Kể từ Android 9 và siết chặt tối đa trên Android 14, mọi class/field/method trong framework được gắn cờ (metadata flags trong file DEX/boot image):
  1. **Whitelist (Public API):** Được định nghĩa trong Android SDK công khai. Được phép gọi tự do.
  2. **Greylist (Unsupported):** Các API ẩn nhưng chưa có giải pháp thay thế, tạm thời vẫn cho phép app target SDK thấp gọi.
  3. **Blacklist / Max-Target-X (Blocked):** Bị cấm hoàn toàn đối với các app target SDK tương ứng. Nếu gọi qua reflection sẽ bị ném `NoSuchMethodException` hoặc trả về lỗi.
* **Cơ chế phát hiện trong ART Runtime (`hidden_api.cc`):**
  - Khi ứng dụng gọi một method qua Java Reflection:
    `Class.forName("android.app.ActivityThread").getDeclaredMethod(...)`
  - Cuộc gọi được chuyển giao xuống native layer của máy ảo ART:
    Hàm `art::hiddenapi::detail::GetMemberAction(...)` trong source file `art/runtime/hidden_api.cc` được kích hoạt.
  - **Kiểm tra Stack Walker:**
    1. ART thực hiện duyệt ngược stack trace (Stack Walk) để xác định danh tính của **Caller** (Tiến trình/Class nào đang gọi reflection).
    2. Nếu Caller thuộc về **Boot ClassLoader** (tức là mã của chính hệ điều hành Framework): Cho phép truy cập (Exempted).
    3. Nếu Caller thuộc về **App ClassLoader** (ClassLoader nạp APK người dùng):
       - ART kiểm tra `targetSdkVersion` của App.
       - So sánh với domain quy định của API mục tiêu trong file `hiddenapi-flags.csv`.
       - Nếu vi phạm chính sách: ART chặn lời gọi, log cảnh báo vào `logcat`, và ném ra Exception giả lập như thể hàm đó không hề tồn tại.
* **File/Class AOSP:**
  - `art/runtime/hidden_api.cc`
  - `art/runtime/native/java_lang_Class.cc`
  - `frameworks/base/core/java/android/content/pm/ApplicationInfo.java`

---

### Q36. Trong môi trường R&D thiết bị Android TV STB, làm thế nào để bypass cơ chế Hidden API Enforcement cho các ứng dụng hệ thống của bên thứ ba?
**Trả lời:**
* **Các phương pháp Bypass thực tế:**
  1. **Sử dụng lệnh ADB (Dành cho Dev/Testing trên userdebug build):**
     Ghi đè chính sách kiểm tra của hệ thống thông qua `Settings.Global`:
```bash
# Tắt hoàn toàn việc chặn Hidden API (0 = Bật enforcement, 1 = Tắt hoàn toàn / Cho phép tất cả)
adb shell settings put global hidden_api_policy 1
adb shell settings put global hidden_api_policy_p_apps 1
```
  2. **Sử dụng API ẩn của `VMRuntime` (Yêu cầu App có quyền Signature/Platform):**
     Ứng dụng hệ thống của OEM có thể gọi hàm exempt của máy ảo ART thông qua reflection đặc quyền:
```java
try {
    Method getRuntimeMethod = Class.forName("dalvik.system.VMRuntime").getDeclaredMethod("getRuntime");
    getRuntimeMethod.setAccessible(true);
    Object vmRuntime = getRuntimeMethod.invoke(null);
    
    Method setExemptionsMethod = vmRuntime.getClass().getDeclaredMethod("setHiddenApiExemptions", String[].class);
    setExemptionsMethod.setAccessible(true);
    // Truyền vào tiền tố 'L' để miễn trừ toàn bộ class signature trong hệ thống
    setExemptionsMethod.invoke(vmRuntime, new Object[]{new String[]{"L"}});
} catch (Exception e) {
    Log.e("Bypass", "Exemption failed", e);
}
```
  3. **Can thiệp cấp độ Hệ Điều Hành / Build ROM (Platform Engineer):**
     - Sửa file `build/make/core/dex_preopt_config.mk` hoặc cấu hình Makefile của sản phẩm:
       Bật cờ cho phép build target bỏ qua việc sinh metadata kiểm tra hidden API.
     - Sửa trực tiếp logic trong `art/runtime/hidden_api.cc`:
       Bắt hàm `ShouldDenyAccessToMember` luôn trả về `EnforcementPolicy::kDisabled`.
* **Pitfall:**
  Không bao giờ được sử dụng các thủ thuật "Double Reflection" hoặc sửa bộ nhớ runtime (`Unsafe`) trên các ứng dụng STB thương mại chuẩn bị chứng chỉ Google GMS (Android TV Certification). Bộ kiểm thử CTS (Compatibility Test Suite) và VTS của Google sẽ quét và đánh rớt ngay lập tức nếu phát hiện thiết bị mở lỗ hổng cho phép app thường bypass Hidden API.
