# Chapter 22 — App Services & System Services: internals

> This chapter **goes deeper** and connects existing material: Chapter 06 (adding a system service, boot phases, permissions), Chapter 11 (Service/FGS at the app layer), Chapter 12 Q13–Q14 (`startService`, ANR timer). Based on AOSP 14; method names may differ slightly between releases, verify in your tree before quoting.

## 22.1 Map: "service" has at least 5 meanings in Android

| Kind | Runs in | Lifecycle managed by | Registered/found via | Example |
|---|---|---|---|---|
| **App Service** (component) | the app's process | AMS → `ActiveServices` | `<service>` in the manifest; `startService`/`bindService` | music player, sync |
| **System service (Java)** | the `system_server` process | `SystemServiceManager` | `ServiceManager.addService` (via `publishBinderService`) | `activity`, `window`, `power` |
| **Local service** | inside `system_server` | as above | `LocalServices.addService` (no binder) | `ActivityManagerInternal` |
| **Native system service** | its own C++ process, started by init | init (`.rc`) | `defaultServiceManager()->addService` | `surfaceflinger`, `media.codec`, `update_engine` |
| **HAL service** | a vendor process | init (`.rc`) | servicemanager (AIDL) / hwservicemanager (HIDL) | `android.hardware.drm-service.widevine` |
| **Persistent app service** | a `persistent=true` app, system uid | AMS (auto-restart) | `ServiceManager.addService` from the app | `DavidExtension` (Chapter 06 Q4) |

Senior answer: state clearly **who manages the lifecycle**, the **security boundary (uid/SELinux domain)** and **how clients find it** — those decide which kind to pick.

---

### Q1. How does `bindService()` travel from the app to `onServiceConnected`? (end-to-end)

```
Client: ContextImpl.bindServiceCommon(intent, conn, flags)
  → LoadedApk.getServiceDispatcher(conn, ...)  → creates ServiceDispatcher + InnerConnection (IServiceConnection.Stub)
  → ActivityManager.getService().bindServiceInstance(caller, token, intent, ..., IServiceConnection, flags, ...)
system_server: ActiveServices.bindServiceLocked
  → retrieveServiceLocked (resolve, check exported/permission, create/find ServiceRecord)
  → create ConnectionRecord, AppBindRecord, IntentBindRecord (keyed by Intent.FilterComparison)
  → BIND_AUTO_CREATE → bringUpServiceLocked (start the process if needed → onCreate)
  → requestServiceBindingLocked → app.thread.scheduleBindService(...)
Service process: ActivityThread.handleBindService → service.onBind(intent) → IBinder
  → AMS.publishService(token, intent, binder)
system_server: ActiveServices.publishServiceLocked
  → for every ConnectionRecord with the same Intent: c.conn.connected(componentName, binder, dead=false)   [oneway]
Client: InnerConnection.connected → ServiceDispatcher.connected
  → posts to the main thread (or Executor) → linkToDeath(binder) → conn.onServiceConnected(name, binder)
```
Key points:
- **`onBind()` is called only once per distinct Intent** (compared with `Intent.filterEquals`, extras ignored). A second client binding with the same Intent gets the **cached binder** → its extras **never** reach the service.
- No clients left → `onUnbind()`; returning `true` makes the next bind call `onRebind()` instead of `onBind()`.
- Service process dies → clients get `onServiceDisconnected()` (binder died); AMS may restart the service, and `onServiceConnected()` fires again when it is back. `onBindingDied()` (API 26+) when the binding cannot recover (e.g. package updated) → unbind and bind again. `onNullBinding()` when `onBind` returns null.

---

### Q2. How do the `BIND_*` flags affect process priority?

| Flag | Effect |
|---|---|
| `BIND_AUTO_CREATE` | create the service if needed; keep it alive while bound |
| `BIND_IMPORTANT` | raise the service host close to a foreground client |
| `BIND_ABOVE_CLIENT` | the service matters more than the client (client killed first under memory pressure) |
| `BIND_NOT_FOREGROUND` | do not raise the service to foreground scheduling |
| `BIND_WAIVE_PRIORITY` | do not affect the service's oom_adj |
| `BIND_ALLOW_OOM_MANAGEMENT` | allow the system to kill the service like a normal app |
| `BIND_NOT_PERCEPTIBLE` (A10+) | do not treat the service as perceptible |
| `BIND_INCLUDE_CAPABILITIES` | pass "while-in-use" capabilities (location, camera...) to the service |
| `BIND_EXTERNAL_SERVICE` | bind to an isolated service running under the caller's identity |

Link to Chapter 12 Q12: `OomAdjuster` computes the service host's adj from its strongest client, capped by these flags. A background app binding to your service does **not** lift it to foreground.

---

### Q3. What does `onStartCommand` return, and how does AMS restart services?

| Value | Process killed while the service runs | Intent on restart |
|---|---|---|
| `START_STICKY` | service restarted | `null` intent |
| `START_NOT_STICKY` | not restarted (unless starts are pending) | — |
| `START_REDELIVER_INTENT` | restarted | redelivers intents not yet `stopSelf(startId)`'d |
| `START_STICKY_COMPATIBILITY` | like sticky but `onStartCommand` is not guaranteed | |

- AMS schedules restarts in `ActiveServices.scheduleServiceRestartLocked` with **increasing backoff**. ✅ A14 constants (`ActivityManagerConstants`): first wait `SERVICE_RESTART_DURATION` = 1s, each further crash multiplies by `SERVICE_RESTART_DURATION_FACTOR` = 4, at least `SERVICE_MIN_RESTART_TIME_BETWEEN` = 10s apart; running fine for `SERVICE_RESET_RUN_DURATION` = 60s resets the backoff.
- `stopSelf(startId)`: only stops if `startId` is the latest start — avoids stopping while a new intent has arrived.
- `onDestroy()` is **not guaranteed** when the process is killed → never persist important data there.
- `onTaskRemoved()`: user swiped the app from recents (rare on TV).

---

### Q4. Background starts & FGS: the current rules (A8 → A15)?

- **A8**: a background app calling `startService()` → `IllegalStateException` (background execution limits). Use `startForegroundService()` + call `startForeground()` in time.
  - ✅ **The project's A14 source** (`ActivityManagerConstants.java`): `DEFAULT_SERVICE_START_FOREGROUND_TIMEOUT_MS = 30 * 1000` and `DEFAULT_SERVICE_START_FOREGROUND_ANR_DELAY_MS = 10 * 1000` (both tunable via `DeviceConfig`). After 30s, `ActiveServices.serviceForegroundTimeout()` **stops the service**, then 10s later raises an **ANR** with the annotation `Context.startForegroundService() did not then call Service.startForeground()`.
  - The "5 seconds" in older docs/notes comes from earlier Android releases; in an interview say "call it right away in `onStartCommand`; A14 allows at most 30s + 10s before the ANR".
- **A12**: starting an FGS from background is forbidden except for exemptions (exact alarm, high-priority FCM, `BOOT_COMPLETED`, bound by a foreground app, system uid...) → `ForegroundServiceStartNotAllowedException`.
- **A14**: `android:foregroundServiceType` + `FOREGROUND_SERVICE_<TYPE>` permission are mandatory (Chapter 11 Q10).
- **A15**: `dataSync` (and `mediaProcessing`) get a runtime limit (about 6h per 24h, `Service.onTimeout()` is called); `BOOT_COMPLETED` receivers may not start FGS of type `dataSync`, `camera`, `mediaPlayback`, `phoneCall`, `mediaProjection`, `microphone`.
- System/privileged apps on an STB: many restrictions are waived (system uid, persistent), but OEM apps are often regular apps → still subject to them; a common source of "OEM service doesn't run after boot" bugs.

---

### Q5. Threading model in a system service — which thread does it run on?

- AIDL methods of a service run on system_server **binder threads** (✅ `sMaxBinderThreads = 31` in `SystemServer.java`) → many clients call **concurrently** → the service must be thread-safe.
- Shared threads (singletons in `com.android.server`):
  | Thread | Used for |
  |---|---|
  | `UiThread` (`android.ui`) | system_server UI work (dialogs), high priority |
  | `FgThread` (`android.fg`) | short, foreground-priority work that must finish quickly |
  | `BackgroundThread` (`android.bg`) | non-urgent background work |
  | `IoThread` (`android.io`) | short I/O (network/file) |
  | `DisplayThread`, `AnimationThread` | display/WM, animations |
- Services with complex state: create a dedicated `ServiceThread`/`HandlerThread` and **move all state changes to the Handler** (single-threaded) → binder threads only check permissions and post messages. Register this thread with `Watchdog.addThread` (Chapter 20 Q4).
- **No long I/O, no outgoing binder calls on a binder thread while holding a lock** → pool congestion, watchdog.

---

### Q6. How does a service safely deliver events to many apps?

```java
private final RemoteCallbackList<ILedListener> mListeners = new RemoteCallbackList<>();

public void registerListener(ILedListener l) {
    enforceLedPermission();
    mListeners.register(l, /* cookie */ Binder.getCallingUid());   // links to death, auto-removed when the client dies
}

private void notifyChanged(int state) {            // runs on the Handler thread, NOT holding mLock
    int n = mListeners.beginBroadcast();
    try {
        for (int i = 0; i < n; i++) {
            try { mListeners.getBroadcastItem(i).onLedChanged(state); }   // ILedListener is declared oneway
            catch (RemoteException ignored) { }
        }
    } finally { mListeners.finishBroadcast(); }
}
```
- The listener interface should be `oneway interface ILedListener` → slow apps cannot block the service.
- Modern Manager (SDK) APIs take an `Executor`: `addListener(@CallbackExecutor Executor, Listener)` → the app picks the callback thread; the Manager wraps listeners into a single binder stub.
- Cap the number of listeners per uid (prevents apps from spamming registrations → leaks in system_server).

---

### Q7. Security in a system service: checklist

1. **Check permissions** in every public method: `mContext.enforceCallingOrSelfPermission(...)`, or the AIDL **`@EnforcePermission("...")`** annotation (A14, generates the check; the method body calls `xxx_enforcePermission()`).
2. **Caller identity**: `Binder.getCallingUid()/getCallingPid()`; never trust a `packageName` sent by the client — verify with `AppOpsManager.checkPackage(uid, pkg)` or `PackageManagerInternal`.
3. **Multi-user**: `ActivityManager.handleIncomingUser(...)`/`UserHandle.getCallingUserId()`; `INTERACT_ACROSS_USERS(_FULL)` to act on other users.
4. **`clearCallingIdentity()`/`restoreCallingIdentity()`** when calling other services with system identity (Chapter 04 Q10) — always in `try/finally`.
5. **Validate input**: array sizes, enum values, `Parcelable`s from untrusted apps.
6. **SELinux**: `service_contexts` for the service name, which app domains may `find` it.
7. **Don't leak data through `dump()`**: check `DumpUtils.checkDumpPermission`.

---

### Q8. What is `PropertyInvalidatedCache`? (a senior-level differentiator)

- Many APIs are called very often (`hasSystemFeature`, `getPackageInfo`, `isUserUnlocked`...) → each one is a binder call into system_server.
- `PropertyInvalidatedCache` caches results **in the client process**; when the server-side data changes, the server bumps a **system property "nonce"** → every client sees the nonce change → cache cleared.
- Pros: removes thousands of binder calls per second. Caveats: only suits rarely changing data, and the server must remember to invalidate.

---

### Q9. The OEM wants to expose its own API (LED, serial number, standby mode) to third-party apps — where should it live?

| Approach | Pros | Cons |
|---|---|---|
| Modify `framework.jar`, add `@SystemApi`/public API | feels like a standard API | framework changes → harder updates, can break CTS/GSI; third-party apps can't call `@SystemApi` |
| **A separate `java_sdk_library`** (e.g. `com.vendor.tv.sdk`) in `system_ext`/`product`, apps declare `<uses-library>` | decoupled from the framework, stubs for app builds, versionable | you must ship the stub SDK to partners |
| Exported app service + AIDL + a custom permission (`signature` or `knownSigner`) | simple, no framework changes | lifecycle tied to the app; restarts to handle; partners must bind |
| Persistent app service (like `DavidExtension`) | always running, system uid | permanent RAM cost; still needs SELinux for `addService` |

Recommendation for an STB: business logic in a **dedicated service (persistent app or system_ext service)** + a **`java_sdk_library`** as the client-side Manager + a custom permission (Chapter 13 Q22). Avoid modifying `frameworks/base` unless required.

---

### Q10. A minimal native system service (C++)?

```cpp
// main.cpp — its own process, started by init
int main() {
    android::ProcessState::self()->setThreadPoolMaxThreadCount(4);
    android::ProcessState::self()->startThreadPool();
    android::sp<MyService> svc = android::sp<MyService>::make();
    android::defaultServiceManager()->addService(android::String16("vendor.myservice"), svc);
    android::IPCThreadState::self()->joinThreadPool();   // main thread becomes a binder thread too
}
```
```
# myservice.rc
service myservice /system_ext/bin/myservice
    class main
    user system
    group system
```
- Processes in `/system`/`/system_ext` use the `cpp` backend (`libbinder`); vendor processes must use the `ndk` backend (`libbinder_ndk`, `AServiceManager_addService`) — Chapters 04/05.
- Requires: `file_contexts` for the binary, a `.te` domain + `init_daemon_domain`, `service_contexts` + `add` permission.
- **Lazy service**: `LazyServiceRegistrar` + `interface`/`disabled` in the rc → servicemanager starts it when a client appears and stops it when none remain (saves RAM).

---

### Q11. Debugging services

```bash
adb shell dumpsys activity services <package>       # ServiceRecord, connections, start/bind counts, restarts
adb shell dumpsys activity service <pkg>/<.Svc>      # calls the app service's dump()
adb shell service list | grep -i led                 # registered system/native services
adb shell service check vendor.myservice
adb shell dumpsys -l                                  # services supporting dumpsys
adb shell cmd <service> help                          # ShellCommand (Chapter 06 Q8)
adb shell dumpsys binder_calls_stats                  # binder call stats into system_server (if enabled)
adb logcat -s ActivityManager:* | grep -i "service"   # "Scheduling restart of crashed service", "Background start not allowed"
```

---

### Q12. Scenario: "An OEM app binds to your system service and sometimes `onServiceConnected` is never called." What do you check?

1. Did `bindService` return `false`? → resolution failed (A11 package visibility — missing `<queries>`, service not `exported`, missing permission).
2. Returned `true` but never connects: the service process **starts slowly** (heavy `onCreate`, waiting on a HAL) → `dumpsys activity services` shows `executing`/`startRequested`.
3. The client's main thread is blocked → `onServiceConnected` is posted to main but cannot run.
4. The service process is **frozen** (cached app freezer, A12+) or keeps crashing → `Scheduling restart of crashed service` in the log.
5. Binding with the same Intent but expecting different extras → `onBind` is not called again (Q1).
6. Fix: use `bindService(intent, flags, executor, conn)` (API 29+) to get callbacks on a dedicated executor; handle `onBindingDied`/`onNullBinding`; client-side timeout + retry.

---

## 22.2 Rapid-fire

1. **Which thread does an app Service run on?** → The app's main thread (lifecycle callbacks); AIDL methods run on binder threads.
2. **Is `IntentService` still used?** → Deprecated (A11); replace with WorkManager or coroutines inside a Service.
3. **`publishBinderService` vs `ServiceManager.addService`?** → The former is a `SystemService` helper that calls addService with flags (allowIsolated, dumpPriority).
4. **What happens if a service inside system_server crashes?** → An uncaught exception on a binder thread is just returned to the client; an exception on a system_server thread → **system_server crashes → soft reboot**.
5. **How many binder threads in a normal app?** → Up to 15 + main (`ProcessState` config); system_server raises it to 31.
6. **`isolatedProcess=true`?** → The service runs in its own uid with almost no permissions — for processing untrusted data (e.g. web rendering, media parsing).
7. **How do you keep an OEM service always running after boot?** → A persistent app (privileged, system uid) or a system service; normal apps relying on `BOOT_COMPLETED` hit FGS limits (Q4).
