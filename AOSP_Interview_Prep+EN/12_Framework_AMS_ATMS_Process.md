# Chapter 12 — Framework: ATMS/AMS, ActivityThread, Process, Service, Broadcast, Provider

> Verified against `frameworks/base` A14 in the project. Gemini's original: `gemini_raw/02_gemini_framework_part1.md`.

## 12.1 The `startActivity()` flow end-to-end

```
App A: Activity.startActivity
  → Instrumentation.execStartActivity
  → ActivityTaskManager.getService().startActivity(...)   [IActivityTaskManager, binder]
system_server: ActivityTaskManagerService.startActivityAsUser
  → ActivityStartController.obtainStarter(...).execute()
  → ActivityStarter.executeRequest   (resolve intent via PMS, check permission/exported/BAL)
  → ActivityStarter.startActivityUnchecked → startActivityInner
       (compute launch flags, find/create the Task, put the ActivityRecord in the Task)
  → RootWindowContainer.resumeFocusedTasksTopActivities
  → TaskFragment.resumeTopActivity → pause the current activity (PauseActivityItem)
  → ActivityTaskSupervisor.startSpecificActivity
       ├─ process exists   → realStartActivityLocked
       └─ no process yet   → ATMS.startProcessAsync → AMS.startProcessLocked
                             → ProcessList.startProcess → ZygoteProcess.start (socket /dev/socket/zygote)
App B (new process): ActivityThread.main → attach() → IActivityManager.attachApplication
system_server: AMS.attachApplicationLocked → thread.bindApplication(...)
             → ATMS.attachApplication → realStartActivityLocked
             → ClientTransaction{ LaunchActivityItem, ResumeActivityItem } → scheduleTransaction
App B: ApplicationThread.scheduleTransaction → H.EXECUTE_TRANSACTION
     → TransactionExecutor.execute → handleLaunchActivity → performLaunchActivity
     → Instrumentation.newActivity → activity.attach (PhoneWindow) → onCreate → onStart → onResume
     → ViewRootImpl/WMS addWindow → draw the first frame → ATMS reportDrawn
```

### Q1. Why go through `Instrumentation`?
Instrumentation is the single point that creates Activities/calls lifecycle methods → lets test frameworks (Espresso, `am instrument`) intercept/monitor; `checkStartActivityResult()` converts returned error codes into exceptions (`ActivityNotFoundException`, `SecurityException`...).

### Q2. What does `ActivityStarter` do?
- `executeRequest`: resolves the `ActivityInfo` (via `PackageManagerInternal`), checks the caller, permissions, `exported`, **Background Activity Launch (BAL)** restrictions (A10+), interceptors (`ActivityStartInterceptor`: suspended apps, work profile...).
- `startActivityInner`: computes `mLaunchFlags` from launchMode + flags, `getReusableTask()`, `setTargetRootTaskIfNeeded`, `deliverToCurrentTopIfNeeded` (`singleTop` → `onNewIntent`), `complyActivityFlags` (`CLEAR_TOP`...).
- All of this runs under WM's `mGlobalLock` — which is why a slow startActivity can affect the whole system.

### Q3. How does Zygote fork a process?
`ProcessList.startProcess` → `Process.start("android.app.ActivityThread", ...)` → `ZygoteProcess` sends the argument list over the **Unix socket** `/dev/socket/zygote` (or the **USAP pool** if enabled) → `ZygoteServer.runSelectLoop` → `ZygoteConnection.processCommand` → `Zygote.forkAndSpecialize` (setuid/gid, SELinux `seinfo`, mount namespace, cgroup) → the child runs `ZygoteInit.zygoteInit` → `RuntimeInit.applicationInit` → `ActivityThread.main`.

### Q4. `attachApplication` vs `bindApplication`?
- `attachApplication` (app → AMS): "the process is ready, here is my `IApplicationThread`".
- `bindApplication` (AMS → app, oneway): sends the `ApplicationInfo`, providers, config, instrumentation, the list of core services (pre-seeding `ServiceManager.sCache`).
- App: `handleBindApplication` → creates `LoadedApk`, `ContextImpl`, Application, **installs providers**, `Application.onCreate`.

### Q5. What problem does `ClientTransaction` (A9+) solve?
It bundles callbacks (`LaunchActivityItem`, `NewIntentItem`, `ActivityResultItem`...) + the **target state** (`ResumeActivityItem`/`PauseActivityItem`/`StopActivityItem`) into one transaction; `TransactionExecutor` computes the intermediate lifecycle steps itself (`TransactionExecutorHelper.getLifecyclePath`) → avoids out-of-order lifecycle calls.

---

## 12.2 ActivityThread, ApplicationThread, H, LoadedApk

### Q6. Is `ActivityThread` a Thread?
No. It's a management class running on the main thread; `main()` calls `Looper.prepareMainLooper()` → `new ActivityThread().attach(false, startSeq)` → `Looper.loop()`; if the loop exits → `RuntimeException("Main thread loop unexpectedly exited")`.

### Q7. Why doesn't `ApplicationThread` call lifecycle methods directly?
`ApplicationThread extends IApplicationThread.Stub` runs on a **binder thread**; UI is single-threaded → it must post to the main thread's `H` handler (`EXECUTE_TRANSACTION`, `BIND_APPLICATION`, `RECEIVER`, `CREATE_SERVICE`...).

### Q8. What does LoadedApk hold? Where do receiver/service connection leaks come from?
The ClassLoader, resources, `mReceivers` (`ReceiverDispatcher`), `mServices` (`ServiceDispatcher`) per Context. Without `unregisterReceiver`/`unbindService` → the dispatcher holds the Context → leak; the framework logs `IntentReceiverLeaked`/`ServiceConnectionLeaked` when the Activity is destroyed.

---

## 12.3 Process management: OomAdjuster, ProcessList, lmkd

### Q9. The `oom_score_adj` table (A14, `ProcessList.java`)

| Constant | Value | Meaning |
|---------|---------|---------|
| `NATIVE_ADJ` | -1000 | native daemons (init, ...) |
| `SYSTEM_ADJ` | -900 | system_server |
| `PERSISTENT_PROC_ADJ` | -800 | apps with `persistent=true` (e.g. `DavidExtension`) |
| `PERSISTENT_SERVICE_ADJ` | -700 | services bound by system/persistent processes |
| `FOREGROUND_APP_ADJ` | 0 | foreground app |
| `PERCEPTIBLE_RECENT_FOREGROUND_APP_ADJ` | 50 | just left the foreground |
| `VISIBLE_APP_ADJ` | 100 | visible |
| `PERCEPTIBLE_APP_ADJ` | 200 | FGS, playing music |
| `PERCEPTIBLE_MEDIUM/LOW_APP_ADJ` | 225 / 250 | |
| `BACKUP_APP_ADJ` | 300 | performing a backup |
| `HEAVY_WEIGHT_APP_ADJ` | 400 | |
| `SERVICE_ADJ` | 500 | started service |
| `HOME_APP_ADJ` | 600 | launcher |
| `PREVIOUS_APP_ADJ` | 700 | previous app |
| `SERVICE_B_ADJ` | 800 | old services |
| `CACHED_APP_MIN..MAX_ADJ` | 900..999 | cached (`CACHED_APP_LMK_FIRST_ADJ = 950`) |

**⚠️ Corrected**: Gemini wrote `FOREGROUND_APP_ADJ = -700`, `SYSTEM_ADJ = -1000`, `BACKUP = 500`... all wrong compared to the source.

Check on device: `adb shell cat /proc/<pid>/oom_score_adj`, `adb shell dumpsys activity o` (OOM levels), `dumpsys activity processes`.

### Q10. AMS ↔ lmkd, PSI vs vmpressure?
- `ProcessList` connects to the `/dev/socket/lmkd` socket and sends `LMK_PROCPRIO` (pid, uid, oom_adj), `LMK_PROCREMOVE`, `LMK_TARGET`...; lmkd (`system/memory/lmkd/lmkd.cpp`) writes `/proc/<pid>/oom_score_adj` and decides what to kill.
- Modern lmkd uses **PSI** (`/proc/pressure/memory`, `some`/`full` triggers) instead of `vmpressure` (old, less accurate). Tuning: `ro.lmk.*` (`ro.lmk.psi_partial_stall_ms`, `ro.lmk.swap_free_low_percentage`, `ro.config.low_ram`...).
- Low-RAM STB + zram (project: `zramsize=33%`).

### Q11. Cached vs empty processes? Limit on the number of cached processes?
- Cached: no active components anymore but still has activities in history; empty: nothing left, kept only for a fast restart.
- `ProcessList.mLruProcesses` (LRU). Limit `ActivityManagerConstants.MAX_CACHED_PROCESSES`: **A14 default 1024** (`DEFAULT_MAX_CACHED_PROCESSES = 1024`, configurable via DeviceConfig `max_cached_processes`) — in practice lmkd + the freezer decide the kills. **⚠️ Corrected**: Gemini said 32 (an old value).
- A14 **freezer** (`CachedAppOptimizer`, cgroup v2 freezer) freezes cached processes so they don't consume CPU.

### Q12. How does binding a service affect the service host's oom_adj?
`OomAdjuster.computeOomAdjLSP` (`OomAdjuster.java:1710`, looping over the `ConnectionRecord`s of the services the process hosts) propagates client priority → service:
- default: the service gets at most the client's level (capped at visible/perceptible depending on flags),
- `BIND_IMPORTANT`: can reach foreground,
- `BIND_NOT_FOREGROUND`, `BIND_WAIVE_PRIORITY`: no propagation,
- `BIND_ALLOW_OOM_MANAGEMENT`: lets the host be managed normally (may be killed).

---

## 12.4 ActiveServices

### Q13. How does `startService` flow?
`AMS.startService` → `ActiveServices.startServiceLocked` → `retrieveServiceLocked` (resolve, permission, create `ServiceRecord`) → background start checks → `startServiceInnerLocked` → `bringUpServiceLocked` → (no process → `startProcessLocked`, added to `mPendingServices`) → `realStartServiceLocked` → `app.thread.scheduleCreateService` → `sendServiceArgsLocked` → `scheduleServiceArgs` (`onStartCommand`).

### Q14. Where is the service ANR timer set/cancelled?
`bumpServiceExecutingLocked` → `scheduleServiceTimeoutLocked` (posts `SERVICE_TIMEOUT_MSG`, 20s/200s) → client finishes → `serviceDoneExecuting` → `serviceDoneExecutingLocked` removes the message → if time runs out → `serviceTimeout()` → `mAm.mAnrHelper.appNotResponding(...)`.

---

## 12.5 Broadcast (A14 BroadcastQueueModernImpl)

### Q15. How does the modern queue differ from the legacy one?
- Legacy: 2–3 global queues (fg/bg/offload); a slow receiver → blocks the whole queue (head-of-line blocking).
- Modern (`BroadcastQueueModernImpl`, on by default in A14): **one queue per process** (`BroadcastProcessQueue`), parallel dispatch across processes, **defers** broadcasts to cached/frozen processes, **merges** them according to delivery group policy (`BroadcastOptions.setDeliveryGroupPolicy(DELIVERY_GROUP_POLICY_MOST_RECENT)`).
- Timeouts: fg 10s, bg 60s (`ActivityManagerService.java:573-574`); handled in `deliveryTimeoutSoft/Hard` (**⚠️ Corrected**: Gemini called it `deliveryTimeoutLocked` and said "A14 can go up to 120s" — not in the source).
- `goAsync()` without calling `PendingResult.finish()` → broadcast ANR.

### Q16. An OEM system service sends broadcasts to background apps and they arrive late?
Because they're deferred for cached processes. Solutions: `FLAG_RECEIVER_FOREGROUND`, send explicitly, or replace the broadcast with a binder callback (`RemoteCallbackList`) for continuous state data.

---

## 12.6 ContentProvider

### Q17. `getContentProvider` + publish?
Client `ActivityThread.acquireProvider` → AMS `ContentProviderHelper.getContentProviderImpl` → if the provider isn't running: start the process and **block the caller** (10s publish timeout) → the new process runs `installContentProviders` → `publishContentProviders` → AMS wakes the caller and returns a `ContentProviderHolder` (containing the `IContentProvider`).

### Q18. Stable vs unstable providers?
- Stable (default for `query`): if the provider process dies → AMS **kills the client too** (unless the client is persistent).
- Unstable (`acquireUnstableContentProviderClient`): the client gets a `DeadObjectException` and handles/reconnects itself. A TV Launcher/System UI should use unstable when reading third-party providers.

---

## 12.7 Scenario questions (framework)

1. **The STB launcher's cold start takes 4s — how do you find the cause?** → `adb shell am start -W`, `logcat -b events | grep am_activity_launch_time`, Perfetto (`bindApplication`, `activityStart`, `inflate`, `Choreographer#doFrame`), check content providers initialized early, dexopt (`DavidLauncher`'s `LOCAL_DEX_PREOPT := false`!), `dumpsys package dexopt`.
2. **An app keeps getting killed while playing 4K video?** → `dumpsys activity processes`, `logcat -b events | grep am_kill`, lmkd log `Kill 'pkg' (pid), uid, oom_score_adj ... to free ...`, DMA-BUF video buffers eating RAM; tune `ro.lmk.*`, reduce cached processes, use an FGS of type `mediaPlayback` for the playback service.
3. **A persistent app keeps crashing — what then?** → AMS restarts it immediately (persistent); if it crashes too often in a short time → `AppErrors` treats it as a bad process; system_server doesn't reboot, but `am_crash` logs pile up; for system_server itself, RescueParty.
