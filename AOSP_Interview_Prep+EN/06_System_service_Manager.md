# Chapter 06 — System Service & Manager

## 6.1 The Service – Manager model

```
App
 │  ctx.getSystemService(Context.LED_SERVICE)  → LedManager (app process)
 │        │ (SystemServiceRegistry caches per Context)
 │        ▼
 │  ILedManager.Stub.Proxy ── binder ──►  system_server: LedManagerService.BinderService (ILedManager.Stub)
 │                                                │ enforce permission, logic, state
 │                                                ▼
 │                                     IDavidDeviceControl (AIDL HAL, /vendor)  ──► sysfs / driver
```
- **Manager** (`android.app.*Manager`, `android.os.*Manager`): the public API for apps, a thin wrapper around the binder proxy, handles `RemoteException` → `e.rethrowFromSystemServer()`.
- **Service** (`com.android.server.*`): runs in `system_server`, holds state, checks permissions, talks to the HAL.

---

### Q1. Steps to add a new system service to system_server (classic question)

Example: `LedManagerService` using the project's `IDavidDeviceControl` HAL.

**1. AIDL** `frameworks/base/core/java/android/os/ILedManager.aidl`
```aidl
package android.os;
/** @hide */
interface ILedManager {
    void setColor(int color);
    int getColor();
}
```
(Add it to the `framework-core-sources` `filegroup`, or rely on the automatic glob of `core/java/**/*.aidl`.)

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
            // DO NOT block system_server's main thread: waitForDeclaredService can wait forever
            // if the HAL crash-loops → watchdog → soft reboot. Move it to a background thread.
            BackgroundThread.getHandler().post(() -> {
                IBinder b = ServiceManager.waitForDeclaredService(
                        IDavidDeviceControl.DESCRIPTOR + "/default");
                if (b == null) return;                     // HAL not declared in VINTF
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
            synchronized (mLock) { hal = mHal; }           // hold the lock only to read the reference
            if (hal == null) throw new IllegalStateException("LED HAL not ready");
            final long token = Binder.clearCallingIdentity();
            try {
                // An AIDL enum with @Backing(type="int") becomes int constants in Java: LedColor.RED, ...
                hal.setLedColor(color);                    // binder call OUTSIDE the lock
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

> **Lessons (Gemini's review correctly found 2 bugs in the first draft of this sample):**
> 1. Don't call blocking APIs (`waitForDeclaredService`, I/O) in `onStart`/`onBootPhase`, because they run **sequentially on system_server's main thread**.
> 2. Don't make a binder call into another process **while holding the service's lock**. A hung HAL drags down every thread waiting for that lock, and then the Watchdog kills system_server.
> Standard pattern: copy the reference inside the lock, make the IPC outside the lock, update state inside the lock again.

**3. Start the service** in `SystemServer.startOtherServices()`:
```java
t.traceBegin("StartLedManagerService");
mSystemServiceManager.startService(LedManagerService.class);
t.traceEnd();
```

**4. Context constant** `Context.java`: `public static final String LED_SERVICE = "led";` (+ `@SystemApi`/`@hide`, add it to `@ServiceName`).

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

**6. Register** in `SystemServiceRegistry.java` (static block):
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

**8. SELinux** (system/sepolicy or `SYSTEM_EXT_PRIVATE_SEPOLICY_DIRS`):
```
# service.te
type led_service, app_api_service, system_server_service, service_manager_type;
# service_contexts
led    u:object_r:led_service:s0
# system_server.te: call the HAL
allow system_server daviddevicecontrol_service:service_manager find;
binder_call(system_server, daviddevicecontrol)
```

**9. API**: `m update-api` (if public/SystemApi), update `current.txt` / `system-current.txt`. Not needed if `@hide`.

**10. Test**: `adb shell service list | grep led`, `adb shell dumpsys led`, `adb shell cmd led ...` (if `onShellCommand` is implemented).

---

### Q2. `SystemService` lifecycle: `onStart`, `onBootPhase`, `onUserStarting`...?

- `onStart()`: publish the binder (`publishBinderService`) + local service (`publishLocalService`). **Do not** call other services that aren't ready yet here.
- `onBootPhase(int)`: milestones 100 → 1000 (see Chapter 02 Q7). E.g. read Settings at `PHASE_SYSTEM_SERVICES_READY`, start sending broadcasts at `PHASE_ACTIVITY_MANAGER_READY`, bind third-party app services at `PHASE_THIRD_PARTY_APPS_CAN_START`.
- `onUserStarting/Unlocking/Switching/Stopping(TargetUser)`: multi-user support.

---

### Q3. `publishBinderService` vs `publishLocalService` / `LocalServices`?

- `publishBinderService(name, binder)` → `ServiceManager.addService` → other processes use it over binder.
- `publishLocalService(Class, impl)` → `LocalServices.addService` → **only inside system_server**, called directly without binder, no permission check needed. E.g. `ActivityManagerInternal`, `PackageManagerInternal`, `PowerManagerInternal`.
- Pattern: `FooManagerInternal` (abstract class in `com.android.server` or `android.app`) so other services can call it internally.

---

### Q4. Service in system_server vs native service vs "persistent app service" — when to choose which?

| Option | Pros | Cons | Example |
|----------|----|-------|-------|
| Inside `system_server` | direct access to AMS/PMS, LocalServices; standard API | requires modifying frameworks/base, a crash = **soft reboot** of the whole system | PowerManagerService |
| Native daemon (`/system/bin`) | fast, C++, isolated | no Context/Java APIs | surfaceflinger, installd |
| Persistent app with `sharedUserId=android.uid.system` | doesn't touch frameworks/base, easy to update | no access to LocalServices; the service name needs sepolicy; can be killed/restarted | `DavidExtension` |
| APEX/mainline | updatable via Play | complex | — |

**Project example — `DavidExtension`** (`vendor/david/services/framework/davidextensionsvc`):
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
        DavidExtService service = new DavidExtService(this);   // connects to the HAL, manages LED/power
        ServiceManager.addService(SERVICE_NAME, service);    // hidden API → needs the platform cert
    }
}
```
```java
// DavidExtManager.java (client side)
this.mService = IDavidExt.Stub.asInterface(ServiceManager.getService(SERVICE_NAME));
```
- `android:persistent="true"` + privileged → AMS starts it at boot (after `systemReady`) and restarts it automatically if it dies; oom_adj = `PERSISTENT_PROC_ADJ (-800)`.
- `android:sharedUserId` has been **deprecated since API 29 (Android 10)**, but platform apps (Settings, SystemUI...) still use `android.uid.system` in A14. In an interview, also mention alternatives: a privileged app with its own UID + privapp allowlist + its own SELinux domain, or moving the logic into system_server.
- Follow-up: *"Can an app with uid system call addService?"* → Yes as far as binder is concerned, but **SELinux** checks it: the `system_app` process needs `allow system_app davidextension_service:service_manager add;` and the name `davidextension` must be in `service_contexts` (otherwise it is `default_android_service`, and adding it is forbidden by a `neverallow` on user builds).
- Remark: the `IDavidExt` interface is empty → the service only exists to "keep" logic running in the background; if it gets real APIs, add permission checks (`enforceCallingPermission`), because any app can `getService` it if SELinux allows `find`.

---

### Q5. How does Context.getSystemService work? Where is the cache?

- `ContextImpl.getSystemService(name)` → `SystemServiceRegistry.getSystemService(ctx, name)` → `ServiceFetcher`:
  - `CachedServiceFetcher`: 1 instance per `Context` (stored in `ctx.mServiceCache`).
  - `StaticServiceFetcher`: 1 instance per process.
  - `StaticApplicationContextServiceFetcher`: tied to the application context.
- The first time, the fetcher calls `ServiceManager.getService(name)`. `ServiceManager.sCache` **only** contains the core binders AMS hands over during `bindApplication` (`activity`, `package`, `window`...). `getService()` for other services **does not write into `sCache`**. Caching Managers for apps is the job of `SystemServiceRegistry` (`ContextImpl.mServiceCache`). *(Gemini helped correct this wording.)*

---

### Q6. Permission checks in a system service — which ways?

```java
mContext.enforceCallingOrSelfPermission(PERM, msg);   // throws SecurityException
mContext.checkCallingPermission(PERM) == PERMISSION_GRANTED
Binder.getCallingUid() / getCallingPid()
UserHandle.getCallingUserId()
mAppOps.noteOp(...)                                   // AppOps
PermissionChecker / AttributionSource                 // A12+ attribution chain
```
- Checks by special uid: `Process.SYSTEM_UID (1000)`, `SHELL_UID (2000)`, `ROOT_UID (0)`.
- Never trust a package name passed by the client → compare `PackageManager.getPackageUid(pkg)` with the calling uid.

---

### Q7. What is the Watchdog in system_server?

- `com.android.server.Watchdog` monitors **HandlerCheckers** (foreground thread, main thread, UI, IO, display, animation...) and **monitors** (locks of AMS, WMS, PowerMS...).
- If a checker is blocked > **60s** (first stack dump at 30s) → dump traces (`/data/anr/`), kill system_server → zygote receives SIGCHLD → `SigChldHandler` logs `"Exit zygote because system server (pid N) has terminated"` and **calls `kill(getpid(), SIGKILL)` on itself** (`com_android_internal_os_Zygote.cpp:465-468`; the `RuntimeAbort` at line 2448 is only for the case where system_server dies right at fork time) → init restarts zygote (and the `onrestart` services) → **soft reboot**.
- Common causes: **synchronous** HAL/binder calls while holding an important lock, a hung HAL, a deadlock between 2 locks.
- Lesson: don't call slow HALs inside a big `synchronized(mLock)`; use oneway or a separate thread; set timeouts.

---

### Q8. `ShellCommand` / `dumpsys` for your own service?

```java
@Override
public void onShellCommand(FileDescriptor in, FileDescriptor out, FileDescriptor err,
        String[] args, ShellCallback cb, ResultReceiver rr) {
    new LedShellCommand(this).exec(this, in, out, err, args, cb, rr);
}
```
→ `adb shell cmd led set-color 2`. `dump()` → `adb shell dumpsys led` (requires the `DUMP` permission; remember `DumpUtils.checkDumpPermission`).

---

### Q9. Adding a Manager to the SDK for third-party apps: how do @SystemApi, @hide, @TestApi differ?

- `@hide`: not in the SDK; only platform code (`platform_apis: true`) can use it; since A9 **hidden API enforcement** blocks normal apps (greylist/blocklist).
- `@SystemApi`: for system/privileged apps built with `sdk_version: "system_current"`.
- `@TestApi`: for CTS.
- Public: requires API review, updating `api/current.txt` (`m update-api`), `metalava` lint.

---

### Q10. Important services you should know well (frequently probed in depth)

- **AMS/ATMS**: process lifecycle, ANR, `oom_adj`, activity stack (ATMS since A10).
- **PMS**: package scanning at boot (`/system/app`, `/system/priv-app`, `/product/app`…), permissions, `packages.xml`.
- **WMS**: windows, surfaces, input focus.
- **InputManagerService** + `InputReader/InputDispatcher` (native): key events from the STB remote (`.kl`, `.kcm` — the project has `vendor/david/keylayout/`).
- **PowerManagerService**: wakelocks, suspend; on an STB also HDMI-CEC (`HdmiControlService`) and standby mode (the project has `PowerStateManager`, `PowerLedManager` to change the LED according to power state).
- **AudioService / AudioPolicy / AudioFlinger**, **MediaCodec / Codec2** (Amlogic `codec2`).
- **TvInputManagerService** (TIF) — important for STBs (DTVKit, Live TV).
