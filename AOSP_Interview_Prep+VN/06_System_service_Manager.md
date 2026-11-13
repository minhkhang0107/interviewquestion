# Chương 06 — System Service & Manager

## 6.1 Mô hình Service – Manager

```
App
 │  ctx.getSystemService(Context.LED_SERVICE)  → LedManager (process app)
 │        │ (SystemServiceRegistry cache theo Context)
 │        ▼
 │  ILedManager.Stub.Proxy ── binder ──►  system_server: LedManagerService.BinderService (ILedManager.Stub)
 │                                                │ enforce permission, logic, state
 │                                                ▼
 │                                     IDavidDeviceControl (AIDL HAL, /vendor)  ──► sysfs / driver
```
- **Manager** (`android.app.*Manager`, `android.os.*Manager`): API public cho app, wrapper mỏng quanh binder proxy, xử lý `RemoteException` → `e.rethrowFromSystemServer()`.
- **Service** (`com.android.server.*`): chạy trong `system_server`, giữ state, kiểm tra quyền, nói chuyện với HAL.

---

### Q1. Các bước thêm một system service mới vào system_server (câu hỏi kinh điển)

Ví dụ: `LedManagerService` dùng HAL `IDavidDeviceControl` của project.

**1. AIDL** `frameworks/base/core/java/android/os/ILedManager.aidl`
```aidl
package android.os;
/** @hide */
interface ILedManager {
    void setColor(int color);
    int getColor();
}
```
(Thêm vào `filegroup` `framework-core-sources`/ hoặc tự động glob `core/java/**/*.aidl`.)

**2. Service** `frameworks/base/services/core/java/com/android/server/led/LedManagerService.java`
```java
public final class LedManagerService extends SystemService {
    private static final String TAG = "LedManagerService";
    private final Object mLock = new Object();
    private IDavidDeviceControl mHal;
    private int mColor;

    public LedManagerService(Context context) { super(context); }

    @Override
    public void onStart() {
        publishBinderService(Context.LED_SERVICE, new BinderService());
    }

    @Override
    public void onBootPhase(int phase) {
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            // KHÔNG block main thread của system_server: waitForDeclaredService có thể chờ vô hạn
            // nếu HAL crash-loop → watchdog → soft reboot. Chuyển sang background thread.
            BackgroundThread.getHandler().post(() -> {
                IBinder b = ServiceManager.waitForDeclaredService(
                        IDavidDeviceControl.DESCRIPTOR + "/default");
                if (b == null) return;                     // HAL không được khai báo trong VINTF
                try {
                    b.linkToDeath(() -> { synchronized (mLock) { mHal = null; } }, 0);
                } catch (RemoteException e) { return; }
                synchronized (mLock) { mHal = IDavidDeviceControl.Stub.asInterface(b); }
            });
        }
    }

    private final class BinderService extends ILedManager.Stub {
        @Override
        public void setColor(int color) {
            getContext().enforceCallingOrSelfPermission(
                    android.Manifest.permission.CONTROL_LED, "setColor");
            final IDavidDeviceControl hal;
            synchronized (mLock) { hal = mHal; }           // chỉ giữ lock để đọc reference
            if (hal == null) throw new IllegalStateException("LED HAL not ready");
            final long token = Binder.clearCallingIdentity();
            try {
                // AIDL enum @Backing(type="int") sinh ra Java là hằng int: LedColor.RED, ...
                hal.setLedColor(color);                    // gọi binder NGOÀI lock
                synchronized (mLock) { mColor = color; }
            } catch (RemoteException e) {
                Slog.e(TAG, "HAL died", e);
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
        @Override public int getColor() { synchronized (mLock) { return mColor; } }

        @Override
        protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
            if (!DumpUtils.checkDumpPermission(getContext(), TAG, pw)) return;
            pw.println("color=" + mColor + " hal=" + (mHal != null));
        }
    }
}
```

> **Bài học (Gemini review chỉ ra đúng 2 lỗi trong bản nháp đầu của code mẫu này):**
> 1. Không gọi API blocking (`waitForDeclaredService`, I/O) trong `onStart`/`onBootPhase` vì chúng chạy **tuần tự trên main thread của system_server**.
> 2. Không gọi binder ra process khác **khi đang giữ lock** của service. HAL treo sẽ kéo theo mọi thread chờ lock đó, rồi Watchdog kill system_server.
> Pattern chuẩn: copy reference trong lock, gọi IPC ngoài lock, cập nhật state lại trong lock.

**3. Start service** trong `SystemServer.startOtherServices()`:
```java
t.traceBegin("StartLedManagerService");
mSystemServiceManager.startService(LedManagerService.class);
t.traceEnd();
```

**4. Context constant** `Context.java`: `public static final String LED_SERVICE = "led";` (+ `@SystemApi`/`@hide`, thêm vào `@ServiceName`).

**5. Manager** `frameworks/base/core/java/android/os/LedManager.java`
```java
@SystemService(Context.LED_SERVICE)
public final class LedManager {
    private final ILedManager mService;
    /** @hide */ public LedManager(Context ctx, ILedManager service) { mService = service; }

    @RequiresPermission(android.Manifest.permission.CONTROL_LED)
    public void setColor(int color) {
        try { mService.setColor(color); }
        catch (RemoteException e) { throw e.rethrowFromSystemServer(); }
    }
}
```

**6. Đăng ký** trong `SystemServiceRegistry.java` (static block):
```java
registerService(Context.LED_SERVICE, LedManager.class,
        new CachedServiceFetcher<LedManager>() {
    @Override
    public LedManager createService(ContextImpl ctx) throws ServiceNotFoundException {
        IBinder b = ServiceManager.getServiceOrThrow(Context.LED_SERVICE);
        return new LedManager(ctx, ILedManager.Stub.asInterface(b));
    }});
```

**7. Permission** `frameworks/base/core/res/AndroidManifest.xml`:
```xml
<permission android:name="android.permission.CONTROL_LED"
    android:protectionLevel="signature|privileged" />
```

**8. SELinux** (system/sepolicy hoặc `SYSTEM_EXT_PRIVATE_SEPOLICY_DIRS`):
```
# service.te
type led_service, app_api_service, system_server_service, service_manager_type;
# service_contexts
led    u:object_r:led_service:s0
# system_server.te: gọi HAL
allow system_server daviddevicecontrol_service:service_manager find;
binder_call(system_server, daviddevicecontrol)
```

**9. API**: `m update-api` (nếu public/SystemApi), cập nhật `current.txt` / `system-current.txt`. Nếu `@hide` thì không cần.

**10. Test**: `adb shell service list | grep led`, `adb shell dumpsys led`, `adb shell cmd led ...` (nếu implement `onShellCommand`).

---

### Q2. `SystemService` lifecycle: `onStart`, `onBootPhase`, `onUserStarting`...?

- `onStart()`: publish binder (`publishBinderService`) + local service (`publishLocalService`). **Không** gọi service khác chưa sẵn sàng ở đây.
- `onBootPhase(int)`: các mốc 100 → 1000 (xem Chương 02 Q7). Ví dụ đọc Settings ở `PHASE_SYSTEM_SERVICES_READY`, bắt đầu gửi broadcast ở `PHASE_ACTIVITY_MANAGER_READY`, bind service app thứ ba ở `PHASE_THIRD_PARTY_APPS_CAN_START`.
- `onUserStarting/Unlocking/Switching/Stopping(TargetUser)`: hỗ trợ multi-user.

---

### Q3. `publishBinderService` vs `publishLocalService` / `LocalServices`?

- `publishBinderService(name, binder)` → `ServiceManager.addService` → process khác dùng qua binder.
- `publishLocalService(Class, impl)` → `LocalServices.addService` → **chỉ trong system_server**, gọi trực tiếp không qua binder, không cần permission check. Ví dụ `ActivityManagerInternal`, `PackageManagerInternal`, `PowerManagerInternal`.
- Pattern: `FooManagerInternal` (abstract class trong `com.android.server` hoặc `android.app`) để service khác gọi nội bộ.

---

### Q4. Service trong system_server vs native service vs "persistent app service" — khi nào chọn cái nào?

| Lựa chọn | Ưu | Nhược | Ví dụ |
|----------|----|-------|-------|
| Trong `system_server` | truy cập trực tiếp AMS/PMS, LocalServices; là API chuẩn | sửa frameworks/base, crash = **soft reboot** toàn hệ thống | PowerManagerService |
| Native daemon (`/system/bin`) | nhanh, C++, cô lập | không có Context/Java API | surfaceflinger, installd |
| Persistent app `sharedUserId=android.uid.system` | không đụng frameworks/base, dễ update | không vào được LocalServices; tên service cần sepolicy; có thể bị kill/restart | `DavidExtension` |
| APEX/mainline | update qua Play | phức tạp | — |

**Ví dụ project — `DavidExtension`** (`vendor/david/services/framework/davidextensionsvc`):
```xml
<manifest package="com.davidmedia.service.davidextension"
    android:sharedUserId="android.uid.system">
    <application android:name=".DavidExtApplication" android:persistent="true" />
</manifest>
```
```java
public class DavidExtApplication extends Application {
    private static final String SERVICE_NAME = "davidextension";
    @Override public void onCreate() {
        super.onCreate();
        DavidExtService service = new DavidExtService(this);   // kết nối HAL, quản lý LED/power
        ServiceManager.addService(SERVICE_NAME, service);    // hidden API → cần platform cert
    }
}
```
```java
// DavidExtManager.java (phía client)
this.mService = IDavidExt.Stub.asInterface(ServiceManager.getService(SERVICE_NAME));
```
- `android:persistent="true"` + privileged → AMS start lúc boot (sau `systemReady`) và tự restart nếu chết; oom_adj = `PERSISTENT_PROC_ADJ (-800)`.
- `android:sharedUserId` bị **deprecated từ API 29 (Android 10)**, nhưng app platform (Settings, SystemUI...) vẫn dùng `android.uid.system` ở A14. Khi phỏng vấn nên nêu thêm phương án thay thế: app privileged với UID riêng + privapp allowlist + SELinux domain riêng, hoặc đưa logic vào system_server.
- Câu hỏi follow-up: *"App uid system addService được không?"* → Được về mặt binder, nhưng **SELinux** sẽ check: process `system_app` phải được `allow system_app davidextension_service:service_manager add;` và tên `davidextension` phải có trong `service_contexts` (nếu không sẽ là `default_android_service` và bị `neverallow` cấm add trên build user).
- Nhận xét: interface `IDavidExt` đang rỗng → service chỉ dùng để "giữ" logic chạy nền; nếu có API thật nên thêm permission check (`enforceCallingPermission`) vì mọi app đều `getService` được nếu SELinux cho phép `find`.

---

### Q5. Context.getSystemService hoạt động thế nào? Cache ở đâu?

- `ContextImpl.getSystemService(name)` → `SystemServiceRegistry.getSystemService(ctx, name)` → `ServiceFetcher`:
  - `CachedServiceFetcher`: 1 instance mỗi `Context` (lưu trong `ctx.mServiceCache`).
  - `StaticServiceFetcher`: 1 instance mỗi process.
  - `StaticApplicationContextServiceFetcher`: gắn với application context.
- Lần đầu fetcher gọi `ServiceManager.getService(name)`. `ServiceManager.sCache` **chỉ** chứa các binder core được AMS truyền sẵn lúc `bindApplication` (`activity`, `package`, `window`...). `getService()` của service khác **không ghi vào `sCache`**. Phần cache Manager cho app là việc của `SystemServiceRegistry` (`ContextImpl.mServiceCache`). *(Gemini đã sửa giúp cách diễn đạt này.)*

---

### Q6. Kiểm tra permission trong system service — các cách?

```java
mContext.enforceCallingOrSelfPermission(PERM, msg);   // throw SecurityException
mContext.checkCallingPermission(PERM) == PERMISSION_GRANTED
Binder.getCallingUid() / getCallingPid()
UserHandle.getCallingUserId()
mAppOps.noteOp(...)                                   // AppOps
PermissionChecker / AttributionSource                 // A12+ attribution chain
```
- Kiểm tra theo uid đặc biệt: `Process.SYSTEM_UID (1000)`, `SHELL_UID (2000)`, `ROOT_UID (0)`.
- Không tin package name do client truyền → so với `PackageManager.getPackageUid(pkg)` == calling uid.

---

### Q7. Watchdog trong system_server là gì?

- `com.android.server.Watchdog` theo dõi các **HandlerChecker** (foreground thread, main thread, UI, IO, display, animation...) và **monitor** (lock của AMS, WMS, PowerMS...).
- Nếu một checker bị block > **60s** (30s thì dump stack lần 1) → dump trace (`/data/anr/`), kill system_server → zygote nhận SIGCHLD → `SigChldHandler` log `"Exit zygote because system server (pid N) has terminated"` và **tự `kill(getpid(), SIGKILL)`** (`com_android_internal_os_Zygote.cpp:465-468`; còn `RuntimeAbort` ở dòng 2448 chỉ dành cho trường hợp system_server chết ngay lúc fork) → init restart zygote (và các service `onrestart`) → **soft reboot**.
- Nguyên nhân hay gặp: gọi HAL/binder **đồng bộ trong lúc giữ lock** quan trọng, HAL treo, deadlock giữa 2 lock.
- Bài học: không gọi HAL chậm trong `synchronized(mLock)` lớn; dùng oneway hoặc thread riêng; set timeout.

---

### Q8. `ShellCommand` / `dumpsys` cho service của mình?

```java
@Override
public void onShellCommand(FileDescriptor in, FileDescriptor out, FileDescriptor err,
        String[] args, ShellCallback cb, ResultReceiver rr) {
    new LedShellCommand(this).exec(this, in, out, err, args, cb, rr);
}
```
→ `adb shell cmd led set-color 2`. `dump()` → `adb shell dumpsys led` (cần `DUMP` permission, nhớ `DumpUtils.checkDumpPermission`).

---

### Q9. Thêm một Manager vào SDK cho app bên thứ ba: @SystemApi, @hide, @TestApi khác gì?

- `@hide`: không xuất hiện trong SDK; chỉ code platform (`platform_apis: true`) dùng được; từ A9 bị **hidden API enforcement** chặn app thường (greylist/blocklist).
- `@SystemApi`: cho app system/privileged build với `sdk_version: "system_current"`.
- `@TestApi`: cho CTS.
- Public: cần API review, cập nhật `api/current.txt` (`m update-api`), lint `metalava`.

---

### Q10. Các service quan trọng nên biết rõ (hay bị hỏi sâu)

- **AMS/ATMS**: process lifecycle, ANR, `oom_adj`, activity stack (ATMS từ A10).
- **PMS**: scan package lúc boot (`/system/app`, `/system/priv-app`, `/product/app`…), permission, `packages.xml`.
- **WMS**: window, surface, input focus.
- **InputManagerService** + `InputReader/InputDispatcher` (native): key event từ remote STB (`.kl`, `.kcm` — project có `vendor/david/keylayout/`).
- **PowerManagerService**: wakelock, suspend; với STB còn có HDMI-CEC (`HdmiControlService`) và chế độ standby (project có `PowerStateManager`, `PowerLedManager` để đổi LED theo trạng thái power).
- **AudioService / AudioPolicy / AudioFlinger**, **MediaCodec / Codec2** (Amlogic `codec2`).
- **TvInputManagerService** (TIF) — quan trọng với STB (DTVKit, Live TV).
