# Chapter 04 — Binder IPC & AIDL

## 4.1 Binder architecture

```
 Client process                                   Server process
┌───────────────────────┐                      ┌─────────────────────────┐
│ IFoo.Stub.Proxy (Java)│                      │ IFoo.Stub (Java)        │
│  / BpFoo (cpp,ndk)    │                      │  / BnFoo (cpp,ndk)      │
│        │ transact()   │                      │   onTransact() ▲        │
│   BinderProxy/BpBinder│                      │   BBinder      │        │
│   IPCThreadState      │                      │   IPCThreadState (pool) │
└────────┬──────────────┘                      └────────────────┼────────┘
         │ ioctl(BINDER_WRITE_READ)                              │
─────────▼───────────────────── kernel ──────────────────────────┴─────────
                      binder driver (drivers/android/binder.c)
     - copies data once (mmap'ed server buffer), manages refs/handles,
       passes fds, credentials (uid/pid/SELinux sid), thread scheduling, death notices
```
- The driver lives in the kernel's `drivers/android/`: `binder.c` (protocol, nodes/refs, threads), `binder_alloc.c` (mmap buffer management).
- Since A11 with GKI, nodes are created dynamically through **binderfs** (`/dev/binderfs/`); `/dev/binder`, `/dev/hwbinder`, `/dev/vndbinder` are symlinks into it.
- 3 device nodes: `/dev/binder` (framework ↔ app, and AIDL HALs), `/dev/hwbinder` (HIDL, legacy), `/dev/vndbinder` (vendor ↔ vendor internal IPC, interfaces built with vendor stability).
- 3 matching service managers: `servicemanager` (still written in **C++** in A14: `frameworks/native/cmds/servicemanager/ServiceManager.cpp`), `hwservicemanager`, `vndservicemanager`.

---

### Q1. Why does Android use Binder instead of sockets/pipes/shared memory?

1. **Performance**: only **1 copy** (sender → kernel buffer that is mmap'ed into the receiver) vs 2 copies for sockets/pipes.
2. **Security**: the kernel attaches the caller's real **UID/PID/SID** (`Binder.getCallingUid()`), which cannot be spoofed → the basis for permission checks.
3. **Object model**: remote object references (handles), reference counting, **death notification**.
4. **Thread pool + priority inheritance**, passing **file descriptors**, oneway async.
5. Easy synchronous RPC — looks like a local function call.

---

### Q2. How does a Binder transaction work? (deep dive)

1. The client calls `proxy.foo(x)` → the Proxy writes the interface token + arguments into a `Parcel` → `mRemote.transact(TRANSACTION_foo, data, reply, flags)`.
2. `IPCThreadState::transact` → `ioctl(fd, BINDER_WRITE_READ)` with the `BC_TRANSACTION` command.
3. Driver: finds the target `binder_node` by handle, allocates a buffer in the server's mmap area, copies the data **once**, translates handles/fds into the server's space, picks an idle thread in the server's pool and wakes it with `BR_TRANSACTION`.
4. Server thread: `BBinder::transact` → `onTransact` → the Stub reads the Parcel, checks the interface token (`enforceInterface`), calls the implementation.
5. The reply is written into a Parcel → `BC_REPLY` → the client receives `BR_REPLY` and reads the result. The client thread **blocks** throughout (except for oneway).
- Buffer limit: each process mmaps **~1MB** (`BINDER_VM_SIZE = 1MB − 2×PAGE_SIZE`, i.e. 1016KB with 4KB pages). This is the **total shared by all transactions being processed concurrently** for that process, not a per-transaction limit. **Oneway** transactions may use at most **half** of it (`free_async_space`) to leave room for synchronous calls. Exceeding it → `TransactionTooLargeException` / `FAILED_TRANSACTION`. *(Clarified after Gemini's feedback.)*

---

### Q3. What is `oneway`? When to use it? Pitfalls?

- `oneway` method/interface: the client **does not wait** for a reply, returns immediately, no return value/out params.
- Oneway calls to the **same binder object** are queued by the driver & **executed sequentially** (async_todo).
- Used for: callbacks/listeners from a service back to apps (so a hung app can't block the service), event notifications.
- Pitfalls:
  - Spamming oneway → the async buffer (½ of 1MB) fills up → `FAILED_TRANSACTION`, log `binder: ... oneway spamming`.
  - Server-side exceptions are swallowed.
  - A oneway call to a **local binder (same process)** still runs synchronously.

```aidl
oneway interface IStatusCallback {
    void onLedChanged(int color);
}
```

---

### Q4. Death recipient — why does it matter? Write an example.

- When the server process dies, the driver sends `BR_DEAD_BINDER` to every client that called `linkToDeath` → the client cleans up/reconnects.
- Conversely the server uses `RemoteCallbackList` (Java) to automatically drop callbacks of dead clients.

Real example from the project (`davidextensionsvc/.../NativeWrapperImpl.java`):
```java
mIDavidDeviceControl = IDavidDeviceControl.Stub.asInterface(
        ServiceManager.waitForDeclaredService(serviceAIDL));
mIDavidDeviceControl.asBinder().linkToDeath(this, 0);
...
@Override
public void binderDied() {
    Handler handler = new Handler(Looper.getMainLooper());
    handler.postDelayed(runnable /* calls connectToHal() again, retries at most 5 times */, 1000);
}
```
**Senior review (great story for an interview):**
- `connectToHal()` uses `waitForDeclaredService()` → it **blocks** until the HAL comes back; and it runs on the persistent app's **main looper** → can cause an **ANR** or hang the main thread if the HAL never comes back. Use a dedicated `HandlerThread`, or `ServiceManager.registerForNotifications()` / `IServiceCallback`.
- The retry is pointless because `waitForDeclaredService` already waits; set `mIDavidDeviceControl = null` in `binderDied()` so callers don't call into a dead binder (`DeadObjectException`).
- Native NDK side: `AIBinder_DeathRecipient_new(onDied)` + `AIBinder_linkToDeath(binder, recipient, cookie)`.

---

### Q5. How does ServiceManager work? What is handle 0?

- `servicemanager` (`frameworks/native/cmds/servicemanager`) is the **context manager**: it registers with the driver via `BINDER_SET_CONTEXT_MGR`; every process reaches it via **handle 0**.
- API: `addService(name, binder)`, `getService`, `checkService` (no wait), `waitForService`, `isDeclared`, `registerForNotifications`.
- **Note the Java vs C++ difference** (spotted by Gemini, verified):
  - C++ `defaultServiceManager()->getService()` (`ServiceManagerShim`, `frameworks/native/libs/binder/IServiceManager.cpp:130`) **waits up to ~5s** (a `sleep(1)` loop), logging `Waiting for service ...` (deprecated; use `waitForService`/`checkService`).
  - Java `android.os.ServiceManager.getService()` **does not wait**: it returns `null` immediately if the service isn't there (except binders pre-seeded in `sCache`). To wait, use `waitForService()`/`waitForDeclaredService()`.
- Every `addService`/`find` is checked by **SELinux** `service_manager { add find list }` based on `service_contexts`.
- For vendor AIDL HALs (`@VintfStability`), servicemanager also checks **whether the service is declared in the VINTF manifest** (`vintf_fragments`) — if not, addService is rejected: `Could not find ... in the VINTF manifest`.

---

### Q6. AIDL backends: Java, CPP, NDK, Rust — which to choose?

| Backend | Library | Used where | ABI-stable |
|---------|----------|-----------|-------------|
| Java | `framework.jar` binder | framework, apps | — |
| CPP | `libbinder` | system native (not for vendor/HAL) | ✘ |
| NDK | `libbinder_ndk` | **vendor HALs**, APEX, mainline | ✔ (LL-NDK — `ndk/Android.bp` has an `llndk {}` stanza; Gemini disputed this but was wrong) |
| Rust | `libbinder_rs` | new Rust code | ✔ |

In the project, `aidl_interface` disables CPP and enables Java (for the `DavidExtension` client) and NDK (default, for the HAL service):
```bp
aidl_interface {
    name: "vendor.david.hardware.daviddevicecontrol",
    vendor_available: true,
    srcs: ["vendor/david/hardware/daviddevicecontrol/*.aidl"],
    stability: "vintf",
    backend: {
        cpp: { enabled: false },
        java: { enabled: true, sdk_version: "module_current" },
    },
}
```
→ generates `vendor.david.hardware.daviddevicecontrol-V1-ndk` and `-V1-java`.

---

### Q7. What does AIDL-generated code consist of?

For `IDavidDeviceControl.aidl`:
- **NDK**: `aidl/vendor/david/hardware/daviddevicecontrol/IDavidDeviceControl.h` (interface), `BnDavidDeviceControl.h` (server stub), `BpDavidDeviceControl.h` (client proxy), enum `LedColor.h`.
- **Java**: `IDavidDeviceControl` with `Stub` (server, extends `Binder`) and `Stub.Proxy` (client), `DESCRIPTOR`, `getInterfaceVersion()`, `getInterfaceHash()`.
- See `out/soong/.intermediates/vendor/david/interfaces/daviddevicecontrol/aidl/...-ndk-source/gen/`.

---

### Q8. Directional tags `in`, `out`, `inout` and supported AIDL data types?

- `in`: client → server (default for primitives). `out`: server writes, client receives. `inout`: both (marshaled twice).
- Primitives, `String`, `byte[]`, `List<T>`, `T[]`, `Map` (not for stable), `ParcelFileDescriptor`, `IBinder`, other interfaces, `parcelable`, `enum` (`@Backing(type="int")`), `union`, `@nullable`, `const`.
- `Map` is supported only by the **Java backend**. Stable/VINTF interfaces must generate NDK/CPP/Rust, so in practice **`Map` cannot be used**; replace it with `List<parcelable Entry{key,value}>`. Hand-written Java parcelables (declared as `parcelable Foo;` without structure) are not allowed either; stable AIDL only allows **structured parcelables**.

```aidl
@VintfStability
parcelable LedState {
    LedColor color = LedColor.OFF;
    boolean blinking;
    int periodMs;
}
```

---

### Q9. Binder thread pool: how many threads by default? The "binder thread pool exhausted" error?

- Apps/system_server: max **15** binder threads + main (system_server sets 31). Native services set their own: `ABinderProcess_setThreadPoolMaxThreadCount(n)` + `ABinderProcess_startThreadPool()` / `joinThreadPool()`.
- Project HAL: `ABinderProcess_setThreadPoolMaxThreadCount(8); ... ABinderProcess_joinThreadPool();` — for a HAL that only drives LEDs through sysfs, 8 threads is overkill, 1-2 would do; but the implementation must be **thread-safe** (several threads may write sysfs at once).
- When all threads are busy (deadlock, slow calls) → clients block, log `binder thread pool (15 threads) starved`, easily leading to **ANR / watchdog**.

---

### Q10. When do you use `Binder.clearCallingIdentity()`?

- In a system service, while handling a call from an app (calling uid = app) when you need to call another API **with system_server's own privileges** (e.g. `ContentResolver`, `PackageManager` calls that need system permission):
```java
final long token = Binder.clearCallingIdentity();
try {
    mContext.getContentResolver().notifyChange(...);
} finally {
    Binder.restoreCallingIdentity(token);
}
```
- Check permissions **before** clearing the identity, otherwise you have a *confused deputy* vulnerability.

---

### Q11. AIDL vs Messenger vs ContentProvider vs Broadcast?

- AIDL: multi-threaded RPC, strongly typed, many methods → for complex services.
- Messenger: Handler-based, **sequential on 1 thread**, simple.
- ContentProvider: shares tabular/URI data, per-URI permissions.
- Broadcast: 1-to-n, loosely coupled, slow, subject to background restrictions (implicit broadcasts).

---

### Q12. How are exceptions/status propagated in AIDL?

- Java: `RemoteException`, plus "special" exceptions that get parceled: `SecurityException`, `IllegalArgumentException`, `IllegalStateException`, `NullPointerException`, `UnsupportedOperationException`, `ServiceSpecificException`.
- NDK: `ndk::ScopedAStatus::ok()`, `fromExceptionCode(EX_ILLEGAL_ARGUMENT)`, `fromServiceSpecificError(code)`, `fromStatus(STATUS_...)`.
- A HAL should return meaningful errors rather than always `ok()`:
```cpp
::ndk::ScopedAStatus DavidDeviceControl::setLedColor(LedColor c) {
    if (!isValid(c)) return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_ARGUMENT);
    if (!mDeviceControl->setLedColor(toNative(c)))
        return ndk::ScopedAStatus::fromServiceSpecificError(ERROR_SYSFS_WRITE);
    return ndk::ScopedAStatus::ok();
}
```
