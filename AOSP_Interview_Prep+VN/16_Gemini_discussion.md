# Chương 16 — Nhật ký thảo luận Claude ↔ Gemini (kiểm chứng bằng source A14)

> **Quy trình**: Gemini (qua CLI `agy`) viết bộ Q&A tầng App và Framework, đồng thời review chương 04 và 06. Claude đối chiếu từng khẳng định với source Android 14 trong project này (`frameworks/base`, `frameworks/native`, `system/`, `packages/modules`). Vòng 2: Claude gửi lại 14 phản biện kèm bằng chứng, **Gemini đồng ý cả 14**.
> File gốc: `gemini_raw/01..05_*.md`.
>
> **Bài học khi đi phỏng vấn**: mọi con số (timeout, adj, kích thước buffer) phải nhớ kèm **nguồn trong code**. LLM, kể cả hai model trong cuộc thảo luận này, đều có lúc nói sai một cách rất tự tin.

## A. Gemini phản biện tài liệu của Claude: ĐÚNG → đã sửa vào tài liệu

| # | Gemini nói | Kiểm chứng | Đã sửa ở |
|---|-----------|-----------|----------|
| 1 | Code mẫu gọi `waitForDeclaredService()` trong `onBootPhase()` → block main thread system_server → watchdog | Đúng (lifecycle SystemService chạy tuần tự trên main thread) | Ch.06 Q1 |
| 2 | Code mẫu gọi HAL (binder) **trong** `synchronized(mLock)` | Đúng, trái với chính nguyên tắc nêu ở Ch.06 Q7 | Ch.06 Q1 |
| 3 | Java `ServiceManager.getService()` không chờ 5s; chỉ C++ shim chờ | Đúng: `IServiceManager.cpp:130` comment "5s delay"; Java trả `null` | Ch.04 Q5, Ch.10 #24 |
| 4 | `ServiceManager.sCache` chỉ pre-seed lúc bindApplication, `getService` không ghi vào | Đúng (`ServiceManager.java:141-153`) | Ch.06 Q5 |
| 5 | binderfs, driver gồm `binder.c` + `binder_alloc.c` | Đúng | Ch.04 |
| 6 | Buffer ~1MB là tổng dùng chung, oneway ≤ ½ | Đúng (diễn đạt lại cho rõ) | Ch.04 Q2 |
| 7 | `Map` không dùng được trong stable AIDL | Đúng về thực tế (Map chỉ có ở backend Java) | Ch.04 Q8 |

## B. Gemini phản biện: SAI → Claude giữ nguyên (Gemini đã đồng ý ở vòng 2)

| # | Gemini nói | Bằng chứng trong source |
|---|-----------|-------------------------|
| 1 | servicemanager A14 viết lại bằng Rust (`src/main.rs`) | `frameworks/native/cmds/servicemanager/` chỉ có `ServiceManager.cpp`, `main.cpp`, `Access.cpp` |
| 2 | `libbinder_ndk` không phải LL-NDK | `frameworks/native/libs/binder/ndk/Android.bp:41` → `llndk: { symbol_file: "libbinder_ndk.map.txt" }` |
| 3 | `sharedUserId` deprecated từ "Android 11 (API 29)", platform app nên bỏ | API 29 = Android 10; Settings/SystemUI A14 vẫn dùng `android.uid.system` |
| 4 | Watchdog: "init kill zygote" | `com_android_internal_os_Zygote.cpp:465-468`: `SigChldHandler` thấy pid == `gSystemServerPid` → `kill(getpid(), SIGKILL)` (zygote tự kết thúc, init restart zygote). Dòng 2448 (`RuntimeAbort`) chỉ là trường hợp system_server chết ngay khi vừa fork. Bản đầu của tài liệu trích dòng 2448, đã sửa lại cho đúng đường chính |

## C. Lỗi trong nội dung Gemini viết cho tầng App/Framework (đã sửa trong Ch.11–15)

| # | Gemini viết | Đúng theo source A14 | File/dòng |
|---|------------|----------------------|-----------|
| 1 | `FOREGROUND_APP_ADJ = -700`, `SYSTEM_ADJ = -1000`, `BACKUP=500`... | `FOREGROUND=0`, `SYSTEM=-900`, `PERSISTENT_PROC=-800`, `PERSISTENT_SERVICE=-700`, `BACKUP=300`, `SERVICE=500`, `HOME=600`, `PREVIOUS=700`, `SERVICE_B=800` | `ProcessList.java:190-282` |
| 2 | `MAX_CACHED_PROCESSES` mặc định 32 | `DEFAULT_MAX_CACHED_PROCESSES = 1024` | `ActivityManagerConstants.java:156` |
| 3 | FocusFinder: `13*major + minor` | `13 * major * major + minor * minor` | `FocusFinder.java:537` |
| 4 | FGS `dataSync` có timeout 6h trên A14 | Không có trong A14 (là của A15) | `ActiveServices.java` |
| 5 | `startForeground()` không truyền type → crash | Dùng `FOREGROUND_SERVICE_TYPE_MANIFEST`; chỉ lỗi nếu manifest không khai type | `ActiveServices.java:~2575` |
| 6 | CursorWindow 4MB | 2MB (`config_cursorWindowSize = 2048`) | `core/res/res/values/config.xml:2383` |
| 7 | Danh sách implicit broadcast miễn trừ ở `BroadcastConstants.java` | sysconfig `allow-implicit-broadcast` → `SystemConfig` → `mBackgroundLaunchBroadcasts` | `SystemConfig.java:1065`, `ActivityManagerService.java:2602` |
| 8 | Global key: PhoneWindowManager `startActivity` Netflix | `GlobalKeyManager` gửi broadcast `ACTION_GLOBAL_BUTTON` tới component trong `global_keys.xml` | `GlobalKeyManager.java`, `vendor/david/apps/DavidFrameworkOverlay` |
| 9 | Tunnel mode dùng `KEY_AUDIO_HW_SYNC` | `KEY_AUDIO_SESSION_ID` + `FEATURE_TunneledPlayback` | MediaFormat API |
| 10 | Broadcast timeout xử lý ở `deliveryTimeoutLocked`, "có thể 120s" | `deliveryTimeoutSoft/Hard`; fg 10s, bg 60s | `BroadcastQueueModernImpl.java:1145-1168`, `ActivityManagerService.java:573` |
| 11 | `runtime-permissions.xml` ở `/data/system/users/<id>/` | Permission mainline module, lưu trong apexdata (`ApexEnvironment`) | `packages/modules/Permission/.../RuntimePermissionsPersistenceImpl.java` |
| 12 | `hidden_api_policy`: "0 = bật enforcement" | `-1` default, `0` DISABLED, `1` JUST_WARN, `2` ENABLED | `ApplicationInfo.java:1542-1563` |
| 13 | Input ANR dựa vào `KEY_WAITING_FOR_EVENTS_TIMEOUT` (5s) | Hằng này là **500ms** (chờ event trước); ANR 5s là dispatching timeout | `InputDispatcher.cpp:105` |
| 14 | TIF overlay "attach vào SurfaceControl riêng" | `WindowManager.addView` với `TYPE_APPLICATION_MEDIA_OVERLAY` | `TvInputService.java:1930-1947` |
| 15 | `LEANBACK_LAUNCHER` "bắt buộc" để làm HOME | HOME chỉ cần `HOME`+`DEFAULT`; LEANBACK_LAUNCHER để hiện trong danh sách app | — |
| 16 | `abortBroadcast()` ở broadcast thường "ném exception" | Chỉ log lỗi, không crash | `BroadcastReceiver.checkSynchronousHint` |
| 17 | `ViewModelStore` "gửi lên system_server" | Chỉ nằm trong `ActivityClientRecord` của process app | `ActivityThread.java` |
| 18 | Wakelock xuống kernel qua `/dev/wakeup_source` | `system_suspend` dùng `/sys/power/wakeup_count` + `/sys/power/state` | `system/hardware/interfaces/suspend/.../SystemSuspend.cpp:253-331` |
| 19 | `fullScreenIntent` mở Activity từ background tuỳ ý | A14 cần `USE_FULL_SCREEN_INTENT`, chỉ app gọi/báo thức được cấp mặc định | — |
| 20 | Câu bổ sung vòng 2: "creator PendingIntent phải opt-in BAL ở A14" | A14: **sender** phải opt-in (`setPendingIntentBackgroundActivityStartMode`); creator opt-in mặc định là A15 | Ch.13 Q17 |

## D. Lỗi của chính Claude, phát hiện trong quá trình kiểm chứng (đã sửa)

| Chỗ | Sai | Đúng |
|-----|-----|------|
| Ch.08 Q2 | `startForeground` timeout 10s | **30s** trên A14 (`ActivityManagerConstants.java:252`) |
| Ch.13 Q16 (bản nháp) | `hidden_api_policy 1` = tắt chặn | `0` = DISABLED, `1` = JUST_WARN |
| Ch.12 (bản nháp) | `OomAdjuster.computeServiceHostOomAdjLSP` | Không có trong A14; logic nằm trong `computeOomAdjLSP` (`OomAdjuster.java:1710`) |

## E. Phát hiện thêm trong source project (rất hợp để kể khi phỏng vấn)

1. **Patch inputflinger của David**: `KeyboardInputMapper.cpp:166` và `CursorInputMapper.cpp:182` đổi `doNotWakeByDefault` mặc định thành `true` để bàn phím/chuột USB không đánh thức STB (chi tiết ở Ch.14 Q13).
2. **RRO global keys**: `DavidFrameworkOverlay` overlay `global_keys.xml` map F5–F12 tới `GlobalKeyReceiver`; remote khai `key usage 0x0c0075 F9` (HID usage) trong `.kl`.
3. **DavidLauncher** prebuilt `LOCAL_DEX_PREOPT := false` làm cold start/boot đầu chậm hơn; allowlist privapp liệt kê thừa quyền normal/dangerous.
4. **AIDL HAL DavidDeviceControl** chưa freeze API (xem Ch.09).

## F. Câu hỏi Gemini bổ sung ở vòng 2 (đã đưa vào tài liệu)
- Modern broadcast queue và process cached/frozen → Ch.12 Q15
- BAL với PendingIntent (A14) → Ch.13 Q17 (có chỉnh)
- Partial photo access `READ_MEDIA_VISUAL_USER_SELECTED` → Ch.13 Q18
- Predictive back (`enableOnBackInvokedCallback`, `OnBackInvokedDispatcher`) → ít liên quan TV (không có cử chỉ vuốt), chỉ cần biết khái niệm
- `SurfaceView` vs `TextureView` cho video TV → Ch.11 Q35

## G. Review lần 2 toàn bộ tài liệu (Claude tự rà, đối chiếu lại source)

| Chỗ | Trước | Sau khi kiểm tra source |
|-----|-------|-------------------------|
| Ch.08 Q4, Ch.06, bảng B#4 | "zygote tự `RuntimeAbort`" (trích dòng 2448) / "`onrestart restart zygote`" | Đường chính: `SigChldHandler` → `kill(getpid(), SIGKILL)` (`com_android_internal_os_Zygote.cpp:465-468`); dòng 2448 chỉ khi system_server chết ngay lúc fork |
| Ch.13 Q7 | privapp thiếu → "PMS ném exception" | Gom vi phạm trong lúc scan (`PermissionManagerServiceImpl.java:~3360`), ném `IllegalStateException` sau khi quét xong (dòng 4458); chỉ kiểm lúc boot |
| Ch.14 Q18 | chỉ nêu guideline 320dpi | Project dùng `ro.sf.lcd_density=240` (`KSTB2500.mk:38`) |
| Ch.15 Q4 | Low Power Standby "Android 14" | Có từ Android 13 |
| Ch.11 Q21 | `postSyncBarrier` là `@hide` | `@TestApi` + `@UnsupportedAppUsage` |
| Ch.11 Q8 | service timeout 20s cố định | `20s × Build.HW_TIMEOUT_MULTIPLIER` |
| Ch.02 Q7 | thiếu boot phase 200, 520 | bổ sung `PHASE_WAIT_FOR_SENSOR_SERVICE(200)`, `PHASE_DEVICE_SPECIFIC_SERVICES_READY(520)` |
| Ch.13 | chưa có câu riêng về `protectionLevel` | thêm mục 13.7 (Q19–Q23): `system` == `privileged` (0x10), `fixProtectionLevel`, logic cấp quyền signature/privileged, `vendorPrivileged`, bảng flag |
