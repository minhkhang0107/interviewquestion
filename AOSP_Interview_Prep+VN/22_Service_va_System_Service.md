# Chương 22 — Service (app) & System Service: cơ chế bên trong

> Chương này **đi sâu** và nối các phần đã có: Chương 06 (thêm system service, boot phase, permission), Chương 11 (Service/FGS ở tầng app), Chương 12 Q13–Q14 (`startService`, timer ANR). Dựa trên AOSP 14; tên hàm có thể khác nhẹ giữa các bản, kiểm tra lại trong tree trước khi trích dẫn.

## 22.1 Bản đồ: "service" trong Android có ít nhất 5 nghĩa

| Loại | Chạy ở đâu | Ai quản lý vòng đời | Đăng ký/tìm thế nào | Ví dụ |
|---|---|---|---|---|
| **App Service** (component) | process của app | AMS → `ActiveServices` | `<service>` trong manifest; `startService`/`bindService` | music player, sync |
| **System service (Java)** | process `system_server` | `SystemServiceManager` | `ServiceManager.addService` (qua `publishBinderService`) | `activity`, `window`, `power` |
| **Local service** | trong `system_server` | như trên | `LocalServices.addService` (không qua binder) | `ActivityManagerInternal` |
| **Native system service** | process riêng (C++), start bởi init | init (`.rc`) | `defaultServiceManager()->addService` | `surfaceflinger`, `media.codec`, `update_engine` |
| **HAL service** | process vendor | init (`.rc`) | servicemanager (AIDL) / hwservicemanager (HIDL) | `android.hardware.drm-service.widevine` |
| **Persistent app service** | app `persistent=true`, uid system | AMS (tự restart) | `ServiceManager.addService` từ app | `DavidExtension` (Chương 06 Q4) |

Câu trả lời senior: chỉ rõ **ai quản lý vòng đời**, **ranh giới bảo mật (uid/SELinux domain)** và **cách client tìm thấy** — đó là thứ quyết định chọn loại nào.

---

### Q1. `bindService()` đi thế nào từ app tới `onServiceConnected`? (end-to-end)

```
Client: ContextImpl.bindServiceCommon(intent, conn, flags)
  → LoadedApk.getServiceDispatcher(conn, ...)  → tạo ServiceDispatcher + InnerConnection (IServiceConnection.Stub)
  → ActivityManager.getService().bindServiceInstance(caller, token, intent, ..., IServiceConnection, flags, ...)
system_server: ActiveServices.bindServiceLocked
  → retrieveServiceLocked (resolve, check exported/permission, tạo/tìm ServiceRecord)
  → tạo ConnectionRecord, AppBindRecord, IntentBindRecord (theo Intent.FilterComparison)
  → BIND_AUTO_CREATE → bringUpServiceLocked (start process nếu chưa có → onCreate)
  → requestServiceBindingLocked → app.thread.scheduleBindService(...)
Service process: ActivityThread.handleBindService → service.onBind(intent) → IBinder
  → AMS.publishService(token, intent, binder)
system_server: ActiveServices.publishServiceLocked
  → với mọi ConnectionRecord cùng Intent: c.conn.connected(componentName, binder, dead=false)   [oneway]
Client: InnerConnection.connected → ServiceDispatcher.connected
  → post lên main thread (hoặc Executor) → linkToDeath(binder) → conn.onServiceConnected(name, binder)
```
Ý quan trọng:
- **`onBind()` chỉ được gọi một lần cho mỗi Intent khác nhau** (so sánh theo `Intent.filterEquals`, không tính extras). Client thứ hai bind cùng Intent nhận **binder đã cache** → extras của client thứ hai **không** tới service.
- Hết client → `onUnbind()`; trả `true` thì lần bind sau gọi `onRebind()` thay vì `onBind()`.
- Service process chết → client nhận `onServiceDisconnected()` (binder chết), AMS có thể restart service; khi chạy lại sẽ gọi `onServiceConnected()` lần nữa. `onBindingDied()` (API 26+) khi binding không thể phục hồi (vd. package bị update) → phải unbind rồi bind lại. `onNullBinding()` khi `onBind` trả null.

---

### Q2. Các cờ `BIND_*` ảnh hưởng gì tới process priority?

| Cờ | Tác dụng |
|---|---|
| `BIND_AUTO_CREATE` | tạo service nếu chưa có; giữ service sống chừng nào còn bind |
| `BIND_IMPORTANT` | service host được nâng gần ngang client foreground |
| `BIND_ABOVE_CLIENT` | service quan trọng hơn client (khi thiếu RAM kill client trước) |
| `BIND_NOT_FOREGROUND` | không kéo service lên mức foreground scheduling |
| `BIND_WAIVE_PRIORITY` | không ảnh hưởng oom_adj của service |
| `BIND_ALLOW_OOM_MANAGEMENT` | cho phép hệ thống kill service như app bình thường |
| `BIND_NOT_PERCEPTIBLE` (A10+) | không coi service là perceptible |
| `BIND_INCLUDE_CAPABILITIES` | truyền quyền "while-in-use" (location, camera...) cho service |
| `BIND_EXTERNAL_SERVICE` | bind tới isolated service chạy dưới danh tính app gọi |

Liên hệ Chương 12 Q12: `OomAdjuster` tính adj của service host từ client mạnh nhất, bị giới hạn bởi các cờ trên. Một app nền bind tới service của bạn → **không** kéo service lên foreground.

---

### Q3. `onStartCommand` trả gì, AMS restart service ra sao?

| Giá trị | Process bị kill khi đang chạy service | Intent khi restart |
|---|---|---|
| `START_STICKY` | restart service | `null` intent |
| `START_NOT_STICKY` | không restart (trừ khi còn start pending) | — |
| `START_REDELIVER_INTENT` | restart | gửi lại intent chưa `stopSelf(startId)` |
| `START_STICKY_COMPATIBILITY` | như sticky nhưng không đảm bảo gọi `onStartCommand` | |

- AMS lên lịch restart trong `ActiveServices.scheduleServiceRestartLocked` với **backoff tăng dần**. ✅ Hằng số A14 (`ActivityManagerConstants`): lần đầu chờ `SERVICE_RESTART_DURATION` = 1s, mỗi lần crash tiếp nhân `SERVICE_RESTART_DURATION_FACTOR` = 4, cách nhau tối thiểu `SERVICE_MIN_RESTART_TIME_BETWEEN` = 10s; chạy ổn `SERVICE_RESET_RUN_DURATION` = 60s thì reset backoff.
- `stopSelf(startId)`: chỉ dừng nếu `startId` là lần start mới nhất — tránh dừng nhầm khi có intent mới tới.
- `onDestroy()` **không được đảm bảo** khi process bị kill → không lưu dữ liệu quan trọng ở đó.
- `onTaskRemoved()`: người dùng vuốt app khỏi recents (hiếm trên TV).

---

### Q4. Background start & FGS: quy tắc hiện tại (A8 → A15)?

- **A8**: app ở background gọi `startService()` → `IllegalStateException` (background execution limits). Dùng `startForegroundService()` + gọi `startForeground()` kịp thời.
  - ✅ **Source A14 của project** (`ActivityManagerConstants.java`): `DEFAULT_SERVICE_START_FOREGROUND_TIMEOUT_MS = 30 * 1000` và `DEFAULT_SERVICE_START_FOREGROUND_ANR_DELAY_MS = 10 * 1000` (cả hai chỉnh được qua `DeviceConfig`). Hết 30s, `ActiveServices.serviceForegroundTimeout()` **dừng service** rồi sau 10s nữa báo **ANR** với annotation `Context.startForegroundService() did not then call Service.startForeground()`.
  - Con số "5 giây" trong tài liệu cũ/ghi chú là của các bản Android trước; khi phỏng vấn nên nói "phải gọi ngay trong `onStartCommand`, A14 cho tối đa 30s + 10s rồi ANR".
- **A12**: cấm start FGS từ background, trừ danh sách miễn trừ (exact alarm, FCM high priority, `BOOT_COMPLETED`, đang bind bởi app foreground, uid system...) → `ForegroundServiceStartNotAllowedException`.
- **A14**: bắt buộc khai báo `android:foregroundServiceType` + quyền `FOREGROUND_SERVICE_<TYPE>` (Chương 11 Q10).
- **A15**: `dataSync` (và `mediaProcessing`) có giới hạn thời gian chạy (khoảng 6 giờ/24h, gọi `Service.onTimeout()`); receiver `BOOT_COMPLETED` không được start FGS loại `dataSync`, `camera`, `mediaPlayback`, `phoneCall`, `mediaProjection`, `microphone`.
- System app/privileged trên STB: nhiều hạn chế được miễn (uid system, persistent), nhưng app OEM thường là app thường → vẫn bị áp dụng; đây là nguồn bug "service OEM không chạy sau boot" phổ biến.

---

### Q5. Threading model trong system service — chạy trên thread nào?

- Method AIDL của service chạy trên **binder thread** của system_server (✅ `sMaxBinderThreads = 31` trong `SystemServer.java`) → nhiều client gọi **song song** → service phải thread-safe.
- Các thread dùng chung (singleton trong `com.android.server`):
  | Thread | Dùng cho |
  |---|---|
  | `UiThread` (`android.ui`) | việc liên quan UI của system_server (dialog), độ ưu tiên cao |
  | `FgThread` (`android.fg`) | việc ngắn, ưu tiên foreground, cần hoàn tất nhanh |
  | `BackgroundThread` (`android.bg`) | việc nền không gấp |
  | `IoThread` (`android.io`) | I/O ngắn (network/file) |
  | `DisplayThread`, `AnimationThread` | display/WM, animation |
- Service có trạng thái phức tạp: tạo `ServiceThread`/`HandlerThread` riêng + **chuyển mọi thay đổi state sang Handler** (single-thread) → binder thread chỉ kiểm tra quyền rồi post message. Đăng ký thread này với `Watchdog.addThread` (Chương 20 Q4).
- **Không làm I/O dài, không gọi binder ra ngoài trên binder thread khi giữ lock** → nghẽn pool, watchdog.

---

### Q6. Service gửi sự kiện cho nhiều app thế nào cho an toàn?

```java
private final RemoteCallbackList<ILedListener> mListeners = new RemoteCallbackList<>();

public void registerListener(ILedListener l) {
    enforceLedPermission();
    mListeners.register(l, /* cookie */ Binder.getCallingUid());   // tự linkToDeath, tự gỡ khi client chết
}

private void notifyChanged(int state) {            // gọi trên Handler thread, KHÔNG giữ mLock
    int n = mListeners.beginBroadcast();
    try {
        for (int i = 0; i < n; i++) {
            try { mListeners.getBroadcastItem(i).onLedChanged(state); }   // ILedListener khai báo oneway
            catch (RemoteException ignored) { }
        }
    } finally { mListeners.finishBroadcast(); }
}
```
- Interface listener nên là `oneway interface ILedListener` → app chậm không chặn service.
- Phía Manager (SDK) hiện đại nhận `Executor`: `addListener(@CallbackExecutor Executor, Listener)` → app chọn thread nhận callback; Manager bọc listener thành một stub binder duy nhất.
- Giới hạn số listener mỗi uid (chống app spam đăng ký → leak trong system_server).

---

### Q7. Bảo mật trong system service: checklist

1. **Kiểm tra quyền** ở mỗi method public: `mContext.enforceCallingOrSelfPermission(...)`, hoặc annotation **`@EnforcePermission("...")`** trong AIDL (A14, sinh code kiểm tra tự động; thân method gọi `xxx_enforcePermission()`).
2. **Danh tính caller**: `Binder.getCallingUid()/getCallingPid()`; không tin `packageName` do client gửi — đối chiếu bằng `AppOpsManager.checkPackage(uid, pkg)` hoặc `PackageManagerInternal`.
3. **Multi-user**: `ActivityManager.handleIncomingUser(...)`/`UserHandle.getCallingUserId()`; quyền `INTERACT_ACROSS_USERS(_FULL)` khi thao tác user khác.
4. **`clearCallingIdentity()`/`restoreCallingIdentity()`** khi service gọi tiếp sang service khác bằng quyền của system (Chương 04 Q10) — luôn trong `try/finally`.
5. **Validate input**: kích thước mảng, giá trị enum, `Parcelable` từ app không tin cậy.
6. **SELinux**: `service_contexts` cho tên service, app domain nào được `find`.
7. **Không lộ thông tin qua `dump()`**: kiểm tra `DumpUtils.checkDumpPermission`.

---

### Q8. `PropertyInvalidatedCache` là gì? (câu hỏi phân loại senior)

- Nhiều API được gọi rất thường xuyên (`hasSystemFeature`, `getPackageInfo`, `isUserUnlocked`...) → mỗi lần là một binder call tới system_server.
- `PropertyInvalidatedCache` cache kết quả **trong process client**; khi dữ liệu ở server thay đổi, server tăng một **system property "nonce"** → mọi client thấy nonce đổi → xoá cache.
- Ưu: giảm hàng nghìn binder call/giây. Lưu ý: chỉ hợp với dữ liệu ít thay đổi, và server phải nhớ gọi invalidate.

---

### Q9. OEM muốn cung cấp API riêng (LED, serial, chế độ standby) cho app bên thứ ba — đặt ở đâu?

| Cách | Ưu | Nhược |
|---|---|---|
| Sửa `framework.jar` thêm `@SystemApi`/public API | dùng như API chuẩn | sửa framework → khó cập nhật, có thể vỡ CTS/không chạy với GSI; app bên thứ ba không gọi được `@SystemApi` |
| **`java_sdk_library` riêng** (vd. `com.vendor.tv.sdk`) ở `system_ext`/`product`, app khai báo `<uses-library>` | tách khỏi framework, có stub để app build, version hoá được | phải phát hành stub SDK cho đối tác |
| App service exported + AIDL + permission tự định nghĩa (`signature` hoặc `knownSigner`) | đơn giản, không đụng framework | vòng đời phụ thuộc app; phải xử lý restart, đối tác phải bind |
| Persistent app service (như `DavidExtension`) | luôn chạy, uid system | tốn RAM thường trực; vẫn cần SELinux cho `addService` |

Khuyến nghị cho STB: logic nghiệp vụ trong **service riêng (persistent app hoặc system_ext service)** + **`java_sdk_library`** làm Manager phía client + permission riêng (Chương 13 Q22). Tránh sửa `frameworks/base` nếu không bắt buộc.

---

### Q10. Native system service (C++) tối thiểu?

```cpp
// main.cpp — service riêng process, start bởi init
int main() {
    android::ProcessState::self()->setThreadPoolMaxThreadCount(4);
    android::ProcessState::self()->startThreadPool();
    android::sp<MyService> svc = android::sp<MyService>::make();
    android::defaultServiceManager()->addService(android::String16("vendor.myservice"), svc);
    android::IPCThreadState::self()->joinThreadPool();   // main thread cũng thành binder thread
}
```
```
# myservice.rc
service myservice /system_ext/bin/myservice
    class main
    user system
    group system
```
- Process trong `/system`/`/system_ext` dùng backend `cpp` (`libbinder`); process vendor phải dùng backend `ndk` (`libbinder_ndk`, `AServiceManager_addService`) — Chương 04/05.
- Cần: `file_contexts` cho binary, domain `.te` + `init_daemon_domain`, `service_contexts` + quyền `add`.
- **Lazy service**: `LazyServiceRegistrar` + `interface`/`disabled` trong rc → servicemanager tự start khi có client và tắt khi hết client (tiết kiệm RAM).

---

### Q11. Debug service

```bash
adb shell dumpsys activity services <package>       # ServiceRecord, connections, start/bind count, restart
adb shell dumpsys activity service <pkg>/<.Svc>      # gọi dump() của app service
adb shell service list | grep -i led                 # system/native service đã đăng ký
adb shell service check vendor.myservice
adb shell dumpsys -l                                  # danh sách service hỗ trợ dumpsys
adb shell cmd <service> help                          # ShellCommand (Chương 06 Q8)
adb shell dumpsys binder_calls_stats                  # thống kê binder call vào system_server (nếu bật)
adb logcat -s ActivityManager:* | grep -i "service"   # "Scheduling restart of crashed service", "Background start not allowed"
```

---

### Q12. Tình huống: "App OEM bind vào service hệ thống của bạn, thỉnh thoảng `onServiceConnected` không bao giờ được gọi." Kiểm tra gì?

1. `bindService` trả `false`? → không resolve được (package visibility A11 — thiếu `<queries>`, service không `exported`, thiếu permission).
2. Có trả `true` nhưng không connect: service process đang **khởi động chậm** (`onCreate` nặng, chờ HAL) → xem `dumpsys activity services` thấy `executing`/`startRequested`.
3. Main thread của client bị chặn → callback `onServiceConnected` post lên main nhưng không chạy được.
4. Process service bị **freeze** (cached app freezer A12+) hoặc liên tục crash → `Scheduling restart of crashed service` trong log.
5. Bind với cùng Intent nhưng mong extras khác → `onBind` không gọi lại (Q1).
6. Fix: dùng `bindService(intent, flags, executor, conn)` (API 29+) để nhận callback trên executor riêng; xử lý `onBindingDied`/`onNullBinding`; timeout + retry phía client.

---

## 22.2 Rapid-fire

1. **App Service chạy trên thread nào?** → Main thread của app (lifecycle callbacks); method AIDL chạy trên binder thread.
2. **`IntentService` còn dùng không?** → Deprecated (A11); thay bằng WorkManager hoặc coroutine trong Service.
3. **`publishBinderService` vs `ServiceManager.addService`?** → Cái đầu là helper của `SystemService`, gọi addService kèm cờ (allowIsolated, dumpPriority).
4. **Service trong system_server crash thì sao?** → Exception không bắt ở binder thread chỉ trả về client; exception trên thread của system_server → **system_server crash → soft reboot**.
5. **Bao nhiêu binder thread trong app thường?** → Tối đa 15 + main (cấu hình `ProcessState`), system_server nâng lên 31.
6. **`isolatedProcess=true`?** → Service chạy trong process uid riêng, gần như không có quyền — dùng cho xử lý dữ liệu không tin cậy (vd. render web, parse media).
7. **Làm sao để service OEM luôn chạy sau boot?** → Persistent app (privileged, uid system) hoặc system service; app thường dựa vào `BOOT_COMPLETED` sẽ bị giới hạn FGS (Q4).
