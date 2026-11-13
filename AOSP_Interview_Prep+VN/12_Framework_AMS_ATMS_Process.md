# Chương 12 — Framework: ATMS/AMS, ActivityThread, Process, Service, Broadcast, Provider

> Đã kiểm chứng với `frameworks/base` A14 trong project. Bản gốc Gemini: `gemini_raw/02_gemini_framework_part1.md`.

## 12.1 Luồng `startActivity()` end-to-end

```
App A: Activity.startActivity
  → Instrumentation.execStartActivity
  → ActivityTaskManager.getService().startActivity(...)   [IActivityTaskManager, binder]
system_server: ActivityTaskManagerService.startActivityAsUser
  → ActivityStartController.obtainStarter(...).execute()
  → ActivityStarter.executeRequest   (resolve intent qua PMS, check permission/exported/BAL)
  → ActivityStarter.startActivityUnchecked → startActivityInner
       (tính launch flags, tìm/ tạo Task, đặt ActivityRecord vào Task)
  → RootWindowContainer.resumeFocusedTasksTopActivities
  → TaskFragment.resumeTopActivity → pause activity hiện tại (PauseActivityItem)
  → ActivityTaskSupervisor.startSpecificActivity
       ├─ process đã có  → realStartActivityLocked
       └─ chưa có        → ATMS.startProcessAsync → AMS.startProcessLocked
                             → ProcessList.startProcess → ZygoteProcess.start (socket /dev/socket/zygote)
App B (process mới): ActivityThread.main → attach() → IActivityManager.attachApplication
system_server: AMS.attachApplicationLocked → thread.bindApplication(...)
             → ATMS.attachApplication → realStartActivityLocked
             → ClientTransaction{ LaunchActivityItem, ResumeActivityItem } → scheduleTransaction
App B: ApplicationThread.scheduleTransaction → H.EXECUTE_TRANSACTION
     → TransactionExecutor.execute → handleLaunchActivity → performLaunchActivity
     → Instrumentation.newActivity → activity.attach (PhoneWindow) → onCreate → onStart → onResume
     → ViewRootImpl/WMS addWindow → vẽ frame đầu → ATMS reportDrawn
```

### Q1. Vì sao phải đi qua `Instrumentation`?
Instrumentation là điểm duy nhất tạo Activity/gọi lifecycle → cho phép test framework (Espresso, `am instrument`) chặn/giám sát; `checkStartActivityResult()` chuyển mã lỗi trả về thành exception (`ActivityNotFoundException`, `SecurityException`...).

### Q2. `ActivityStarter` làm gì?
- `executeRequest`: resolve `ActivityInfo` (qua `PackageManagerInternal`), kiểm tra caller, permission, `exported`, **Background Activity Launch (BAL)** restrictions (A10+), interceptor (`ActivityStartInterceptor`: suspended app, work profile...).
- `startActivityInner`: tính `mLaunchFlags` từ launchMode + flags, `getReusableTask()`, `setTargetRootTaskIfNeeded`, `deliverToCurrentTopIfNeeded` (`singleTop` → `onNewIntent`), `complyActivityFlags` (`CLEAR_TOP`...).
- Toàn bộ chạy dưới `mGlobalLock` của WM — lý do vì sao startActivity chậm có thể gây ảnh hưởng toàn hệ thống.

### Q3. Zygote fork process như thế nào?
`ProcessList.startProcess` → `Process.start("android.app.ActivityThread", ...)` → `ZygoteProcess` gửi danh sách arg qua **Unix socket** `/dev/socket/zygote` (hoặc **USAP pool** nếu bật) → `ZygoteServer.runSelectLoop` → `ZygoteConnection.processCommand` → `Zygote.forkAndSpecialize` (setuid/gid, SELinux `seinfo`, mount namespace, cgroup) → con chạy `ZygoteInit.zygoteInit` → `RuntimeInit.applicationInit` → `ActivityThread.main`.

### Q4. `attachApplication` vs `bindApplication`?
- `attachApplication` (app → AMS): "process đã sẵn sàng, đây là `IApplicationThread` của tôi".
- `bindApplication` (AMS → app, oneway): gửi `ApplicationInfo`, providers, config, instrumentation, danh sách service core (pre-seed `ServiceManager.sCache`).
- App: `handleBindApplication` → tạo `LoadedApk`, `ContextImpl`, Application, **install providers**, `Application.onCreate`.

### Q5. `ClientTransaction` (A9+) giải quyết gì?
Gom callback (`LaunchActivityItem`, `NewIntentItem`, `ActivityResultItem`...) + **trạng thái đích** (`ResumeActivityItem`/`PauseActivityItem`/`StopActivityItem`) vào 1 transaction; `TransactionExecutor` tự tính các bước lifecycle trung gian (`TransactionExecutorHelper.getLifecyclePath`) → tránh sai thứ tự lifecycle.

---

## 12.2 ActivityThread, ApplicationThread, H, LoadedApk

### Q6. `ActivityThread` có phải là Thread?
Không. Nó là class quản lý chạy trên main thread; `main()` gọi `Looper.prepareMainLooper()` → `new ActivityThread().attach(false, startSeq)` → `Looper.loop()`; nếu loop thoát → `RuntimeException("Main thread loop unexpectedly exited")`.

### Q7. Vì sao `ApplicationThread` không gọi lifecycle trực tiếp?
`ApplicationThread extends IApplicationThread.Stub` chạy trên **binder thread**; UI single-thread → phải post sang handler `H` của main thread (`EXECUTE_TRANSACTION`, `BIND_APPLICATION`, `RECEIVER`, `CREATE_SERVICE`...).

### Q8. LoadedApk giữ gì? Leak receiver/service connection từ đâu?
ClassLoader, resources, `mReceivers` (`ReceiverDispatcher`), `mServices` (`ServiceDispatcher`) theo từng Context. Không `unregisterReceiver`/`unbindService` → dispatcher giữ Context → leak; framework log `IntentReceiverLeaked`/`ServiceConnectionLeaked` khi Activity destroy.

---

## 12.3 Quản lý process: OomAdjuster, ProcessList, lmkd

### Q9. Bảng `oom_score_adj` (A14, `ProcessList.java`)

| Hằng số | Giá trị | Ý nghĩa |
|---------|---------|---------|
| `NATIVE_ADJ` | -1000 | native daemons (init, ...) |
| `SYSTEM_ADJ` | -900 | system_server |
| `PERSISTENT_PROC_ADJ` | -800 | app `persistent=true` (vd `DavidExtension`) |
| `PERSISTENT_SERVICE_ADJ` | -700 | service được system/persistent bind |
| `FOREGROUND_APP_ADJ` | 0 | app foreground |
| `PERCEPTIBLE_RECENT_FOREGROUND_APP_ADJ` | 50 | vừa rời foreground |
| `VISIBLE_APP_ADJ` | 100 | visible |
| `PERCEPTIBLE_APP_ADJ` | 200 | FGS, phát nhạc |
| `PERCEPTIBLE_MEDIUM/LOW_APP_ADJ` | 225 / 250 | |
| `BACKUP_APP_ADJ` | 300 | đang backup |
| `HEAVY_WEIGHT_APP_ADJ` | 400 | |
| `SERVICE_ADJ` | 500 | started service |
| `HOME_APP_ADJ` | 600 | launcher |
| `PREVIOUS_APP_ADJ` | 700 | app trước đó |
| `SERVICE_B_ADJ` | 800 | service cũ |
| `CACHED_APP_MIN..MAX_ADJ` | 900..999 | cached (`CACHED_APP_LMK_FIRST_ADJ = 950`) |

**⚠️ Đã sửa**: Gemini ghi `FOREGROUND_APP_ADJ = -700`, `SYSTEM_ADJ = -1000`, `BACKUP = 500`... đều sai so với source.

Kiểm tra trên máy: `adb shell cat /proc/<pid>/oom_score_adj`, `adb shell dumpsys activity o` (OOM levels), `dumpsys activity processes`.

### Q10. AMS ↔ lmkd, PSI vs vmpressure?
- `ProcessList` kết nối socket `/dev/socket/lmkd`, gửi `LMK_PROCPRIO` (pid, uid, oom_adj), `LMK_PROCREMOVE`, `LMK_TARGET`...; lmkd (`system/memory/lmkd/lmkd.cpp`) ghi `/proc/<pid>/oom_score_adj` và quyết định kill.
- lmkd hiện đại dùng **PSI** (`/proc/pressure/memory`, trigger `some`/`full`) thay `vmpressure` (cũ, kém chính xác). Tuning: `ro.lmk.*` (`ro.lmk.psi_partial_stall_ms`, `ro.lmk.swap_free_low_percentage`, `ro.config.low_ram`...).
- STB RAM thấp + zram (project: `zramsize=33%`).

### Q11. Cached vs empty process? Giới hạn số process cached?
- Cached: không còn component active nhưng còn activity trong history; empty: không còn gì, chỉ giữ để start nhanh.
- `ProcessList.mLruProcesses` (LRU). Giới hạn `ActivityManagerConstants.MAX_CACHED_PROCESSES`: **A14 mặc định 1024** (`DEFAULT_MAX_CACHED_PROCESSES = 1024`, cấu hình qua DeviceConfig `max_cached_processes`) — thực tế lmkd + freezer quyết định kill. **⚠️ Đã sửa**: Gemini nói 32 (đó là giá trị cũ).
- A14 **freezer** (`CachedAppOptimizer`, cgroup v2 freezer) đóng băng process cached để không tốn CPU.

### Q12. Bind service ảnh hưởng oom_adj của service host?
`OomAdjuster.computeOomAdjLSP` (`OomAdjuster.java:1710`, vòng lặp qua `ConnectionRecord` của các service mà process đang host) lan độ ưu tiên client → service:
- mặc định: service nhận tối đa mức của client (bị giới hạn ở mức visible/perceptible tuỳ flag),
- `BIND_IMPORTANT`: có thể lên foreground,
- `BIND_NOT_FOREGROUND`, `BIND_WAIVE_PRIORITY`: không lan,
- `BIND_ALLOW_OOM_MANAGEMENT`: cho phép host bị quản lý như bình thường (có thể bị kill).

---

## 12.4 ActiveServices

### Q13. `startService` đi thế nào?
`AMS.startService` → `ActiveServices.startServiceLocked` → `retrieveServiceLocked` (resolve, permission, tạo `ServiceRecord`) → kiểm tra background start → `startServiceInnerLocked` → `bringUpServiceLocked` → (chưa có process → `startProcessLocked`, đưa vào `mPendingServices`) → `realStartServiceLocked` → `app.thread.scheduleCreateService` → `sendServiceArgsLocked` → `scheduleServiceArgs` (`onStartCommand`).

### Q14. Timer ANR service đặt/huỷ ở đâu?
`bumpServiceExecutingLocked` → `scheduleServiceTimeoutLocked` (post `SERVICE_TIMEOUT_MSG`, 20s/200s) → client xong → `serviceDoneExecuting` → `serviceDoneExecutingLocked` gỡ message → nếu hết giờ → `serviceTimeout()` → `mAm.mAnrHelper.appNotResponding(...)`.

---

## 12.5 Broadcast (A14 BroadcastQueueModernImpl)

### Q15. Modern queue khác legacy?
- Legacy: 2–3 hàng đợi toàn cục (fg/bg/offload), receiver chậm → chặn cả hàng (head-of-line blocking).
- Modern (`BroadcastQueueModernImpl`, mặc định bật ở A14): **mỗi process một hàng** (`BroadcastProcessQueue`), dispatch song song giữa các process, **trì hoãn** broadcast tới process cached/frozen, **gộp** theo delivery group policy (`BroadcastOptions.setDeliveryGroupPolicy(DELIVERY_GROUP_POLICY_MOST_RECENT)`).
- Timeout: fg 10s, bg 60s (`ActivityManagerService.java:573-574`); xử lý trong `deliveryTimeoutSoft/Hard` (**⚠️ Đã sửa**: Gemini gọi là `deliveryTimeoutLocked` và nói "A14 có thể tới 120s" — không có trong source).
- `goAsync()` mà quên `PendingResult.finish()` → ANR broadcast.

### Q16. System service OEM gửi broadcast tới app background bị chậm?
Do bị trì hoãn tới process cached. Giải pháp: `FLAG_RECEIVER_FOREGROUND`, gửi explicit, hoặc thay broadcast bằng binder callback (`RemoteCallbackList`) cho dữ liệu trạng thái liên tục.

---

## 12.6 ContentProvider

### Q17. `getContentProvider` + publish?
Client `ActivityThread.acquireProvider` → AMS `ContentProviderHelper.getContentProviderImpl` → nếu provider chưa chạy: start process và **block caller** (timeout 10s publish) → process mới `installContentProviders` → `publishContentProviders` → AMS đánh thức caller, trả `ContentProviderHolder` (chứa `IContentProvider`).

### Q18. Stable vs unstable provider?
- Stable (mặc định với `query`): provider process chết → AMS **kill luôn client** (trừ client persistent).
- Unstable (`acquireUnstableContentProviderClient`): client nhận `DeadObjectException`, tự xử lý/kết nối lại. Launcher TV/System UI nên dùng unstable khi đọc provider bên thứ ba.

---

## 12.7 Câu hỏi tình huống (framework)

1. **Cold start launcher STB mất 4s, tìm nguyên nhân thế nào?** → `adb shell am start -W`, `logcat -b events | grep am_activity_launch_time`, Perfetto (`bindApplication`, `activityStart`, `inflate`, `Choreographer#doFrame`), kiểm tra content provider khởi tạo sớm, dexopt (`LOCAL_DEX_PREOPT := false` của `DavidLauncher`!), `dumpsys package dexopt`.
2. **App bị kill liên tục khi xem phim 4K?** → `dumpsys activity processes`, `logcat -b events | grep am_kill`, `lmkd` log `Kill 'pkg' (pid), uid, oom_score_adj ... to free ...`, DMA-BUF video ngốn RAM; tune `ro.lmk.*`, giảm cached, dùng FGS `mediaPlayback` cho service phát nhạc.
3. **Persistent app crash liên tục làm gì?** → AMS restart ngay (persistent), nếu crash quá nhiều trong thời gian ngắn → bị `AppErrors` coi là bad process; system_server không reboot, nhưng log `am_crash` dày đặc; với system_server thì RescueParty.
