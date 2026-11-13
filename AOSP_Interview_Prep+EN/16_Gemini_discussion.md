# Chapter 16 — Claude ↔ Gemini discussion log (verified against the A14 source)

> **Process**: Gemini (via the `agy` CLI) wrote the App and Framework layer Q&A and reviewed chapters 04 and 06. Claude cross-checked every claim against the Android 14 source in this project (`frameworks/base`, `frameworks/native`, `system/`, `packages/modules`). Round 2: Claude sent back 14 rebuttals with evidence, and **Gemini agreed with all 14**.
> Original files: `gemini_raw/01..05_*.md`.
>
> **Lesson for the interview**: every number (timeout, adj, buffer size) should be remembered together with **its source in the code**. LLMs, including both models in this discussion, sometimes state wrong things very confidently.

## A. Gemini's critiques of Claude's document: CORRECT → applied to the document

| # | Gemini said | Verification | Fixed in |
|---|-----------|-----------|----------|
| 1 | The sample code calls `waitForDeclaredService()` in `onBootPhase()` → blocks system_server's main thread → watchdog | Correct (the SystemService lifecycle runs sequentially on the main thread) | Ch.06 Q1 |
| 2 | The sample code calls the HAL (binder) **inside** `synchronized(mLock)` | Correct, contradicts the very principle stated in Ch.06 Q7 | Ch.06 Q1 |
| 3 | Java `ServiceManager.getService()` doesn't wait 5s; only the C++ shim waits | Correct: `IServiceManager.cpp:130` comment "5s delay"; Java returns `null` | Ch.04 Q5, Ch.10 #24 |
| 4 | `ServiceManager.sCache` is only pre-seeded at bindApplication; `getService` doesn't write into it | Correct (`ServiceManager.java:141-153`) | Ch.06 Q5 |
| 5 | binderfs; the driver consists of `binder.c` + `binder_alloc.c` | Correct | Ch.04 |
| 6 | The ~1MB buffer is a shared total; oneway ≤ ½ | Correct (reworded for clarity) | Ch.04 Q2 |
| 7 | `Map` can't be used in stable AIDL | Correct in practice (Map exists only in the Java backend) | Ch.04 Q8 |

## B. Gemini's critiques: WRONG → Claude kept the original (Gemini agreed in round 2)

| # | Gemini said | Evidence in the source |
|---|-----------|-------------------------|
| 1 | servicemanager in A14 was rewritten in Rust (`src/main.rs`) | `frameworks/native/cmds/servicemanager/` only has `ServiceManager.cpp`, `main.cpp`, `Access.cpp` |
| 2 | `libbinder_ndk` is not LL-NDK | `frameworks/native/libs/binder/ndk/Android.bp:41` → `llndk: { symbol_file: "libbinder_ndk.map.txt" }` |
| 3 | `sharedUserId` deprecated since "Android 11 (API 29)", platform apps should drop it | API 29 = Android 10; Settings/SystemUI on A14 still use `android.uid.system` |
| 4 | Watchdog: "init kills zygote" | `com_android_internal_os_Zygote.cpp:465-468`: `SigChldHandler` sees pid == `gSystemServerPid` → `kill(getpid(), SIGKILL)` (zygote terminates itself, init restarts zygote). Line 2448 (`RuntimeAbort`) is only the case where system_server dies right after fork. The first version of this document cited line 2448; corrected to the main path |

## C. Errors in Gemini's App/Framework content (fixed in Ch.11–15)

| # | Gemini wrote | Correct per the A14 source | File/line |
|---|------------|----------------------|-----------|
| 1 | `FOREGROUND_APP_ADJ = -700`, `SYSTEM_ADJ = -1000`, `BACKUP=500`... | `FOREGROUND=0`, `SYSTEM=-900`, `PERSISTENT_PROC=-800`, `PERSISTENT_SERVICE=-700`, `BACKUP=300`, `SERVICE=500`, `HOME=600`, `PREVIOUS=700`, `SERVICE_B=800` | `ProcessList.java:190-282` |
| 2 | `MAX_CACHED_PROCESSES` default 32 | `DEFAULT_MAX_CACHED_PROCESSES = 1024` | `ActivityManagerConstants.java:156` |
| 3 | FocusFinder: `13*major + minor` | `13 * major * major + minor * minor` | `FocusFinder.java:537` |
| 4 | FGS `dataSync` has a 6h timeout on A14 | Not in A14 (it's A15) | `ActiveServices.java` |
| 5 | `startForeground()` without a type → crash | Uses `FOREGROUND_SERVICE_TYPE_MANIFEST`; only fails if the manifest declares no type | `ActiveServices.java:~2575` |
| 6 | CursorWindow 4MB | 2MB (`config_cursorWindowSize = 2048`) | `core/res/res/values/config.xml:2383` |
| 7 | The implicit broadcast exemption list is in `BroadcastConstants.java` | sysconfig `allow-implicit-broadcast` → `SystemConfig` → `mBackgroundLaunchBroadcasts` | `SystemConfig.java:1065`, `ActivityManagerService.java:2602` |
| 8 | Global keys: PhoneWindowManager `startActivity`s Netflix | `GlobalKeyManager` sends the `ACTION_GLOBAL_BUTTON` broadcast to the component in `global_keys.xml` | `GlobalKeyManager.java`, `vendor/david/apps/DavidFrameworkOverlay` |
| 9 | Tunnel mode uses `KEY_AUDIO_HW_SYNC` | `KEY_AUDIO_SESSION_ID` + `FEATURE_TunneledPlayback` | MediaFormat API |
| 10 | Broadcast timeouts handled in `deliveryTimeoutLocked`, "can be 120s" | `deliveryTimeoutSoft/Hard`; fg 10s, bg 60s | `BroadcastQueueModernImpl.java:1145-1168`, `ActivityManagerService.java:573` |
| 11 | `runtime-permissions.xml` in `/data/system/users/<id>/` | Permission mainline module, stored in apexdata (`ApexEnvironment`) | `packages/modules/Permission/.../RuntimePermissionsPersistenceImpl.java` |
| 12 | `hidden_api_policy`: "0 = enforcement on" | `-1` default, `0` DISABLED, `1` JUST_WARN, `2` ENABLED | `ApplicationInfo.java:1542-1563` |
| 13 | Input ANR is based on `KEY_WAITING_FOR_EVENTS_TIMEOUT` (5s) | That constant is **500ms** (waiting for previous events); the 5s ANR is the dispatching timeout | `InputDispatcher.cpp:105` |
| 14 | TIF overlay "attached to a separate SurfaceControl" | `WindowManager.addView` with `TYPE_APPLICATION_MEDIA_OVERLAY` | `TvInputService.java:1930-1947` |
| 15 | `LEANBACK_LAUNCHER` is "required" to be HOME | HOME only needs `HOME`+`DEFAULT`; LEANBACK_LAUNCHER is for appearing in the app list | — |
| 16 | `abortBroadcast()` on a normal broadcast "throws an exception" | Only logs an error, no crash | `BroadcastReceiver.checkSynchronousHint` |
| 17 | `ViewModelStore` "is sent to system_server" | Lives only in the app process's `ActivityClientRecord` | `ActivityThread.java` |
| 18 | Wakelocks reach the kernel through `/dev/wakeup_source` | `system_suspend` uses `/sys/power/wakeup_count` + `/sys/power/state` | `system/hardware/interfaces/suspend/.../SystemSuspend.cpp:253-331` |
| 19 | `fullScreenIntent` can open an Activity from the background freely | A14 requires `USE_FULL_SCREEN_INTENT`, granted by default only to calling/alarm apps | — |
| 20 | Round 2 addition: "the PendingIntent creator must opt in to BAL on A14" | A14: the **sender** must opt in (`setPendingIntentBackgroundActivityStartMode`); creator opt-in by default is A15 | Ch.13 Q17 |

## D. Claude's own errors, found during verification (fixed)

| Where | Wrong | Right |
|-----|-----|-----|
| Ch.08 Q2 | `startForeground` timeout 10s | **30s** on A14 (`ActivityManagerConstants.java:252`) |
| Ch.13 Q16 (draft) | `hidden_api_policy 1` = blocking off | `0` = DISABLED, `1` = JUST_WARN |
| Ch.12 (draft) | `OomAdjuster.computeServiceHostOomAdjLSP` | Doesn't exist in A14; the logic is in `computeOomAdjLSP` (`OomAdjuster.java:1710`) |

## E. Extra findings in the project source (great to tell in an interview)

1. **David's inputflinger patch**: `KeyboardInputMapper.cpp:166` and `CursorInputMapper.cpp:182` change the default of `doNotWakeByDefault` to `true` so USB keyboards/mice don't wake the STB (details in Ch.14 Q13).
2. **Global keys RRO**: `DavidFrameworkOverlay` overlays `global_keys.xml`, mapping F5–F12 to `GlobalKeyReceiver`; the remote declares `key usage 0x0c0075 F9` (HID usage) in its `.kl`.
3. **DavidLauncher** prebuilt with `LOCAL_DEX_PREOPT := false` makes cold start/first boot slower; the privapp allowlist lists unnecessary normal/dangerous permissions.
4. **The DavidDeviceControl AIDL HAL** has no frozen API (see Ch.09).

## F. Questions Gemini added in round 2 (included in the document)
- Modern broadcast queue and cached/frozen processes → Ch.12 Q15
- BAL with PendingIntent (A14) → Ch.13 Q17 (adjusted)
- Partial photo access `READ_MEDIA_VISUAL_USER_SELECTED` → Ch.13 Q18
- Predictive back (`enableOnBackInvokedCallback`, `OnBackInvokedDispatcher`) → little relevance to TV (no swipe gestures), just know the concept
- `SurfaceView` vs `TextureView` for TV video → Ch.11 Q35

## G. Second full review of the document (Claude's own pass, re-checked against the source)

| Where | Before | After checking the source |
|-----|-------|-------------------------|
| Ch.08 Q4, Ch.06, table B#4 | "zygote calls `RuntimeAbort` on itself" (citing line 2448) / "`onrestart restart zygote`" | Main path: `SigChldHandler` → `kill(getpid(), SIGKILL)` (`com_android_internal_os_Zygote.cpp:465-468`); line 2448 applies only when system_server dies right at fork |
| Ch.13 Q7 | missing privapp → "PMS throws an exception" | Violations are collected during the scan (`PermissionManagerServiceImpl.java:~3360`), and `IllegalStateException` is thrown after the scan (line 4458); checked only at boot |
| Ch.14 Q18 | only the 320dpi guideline | The project uses `ro.sf.lcd_density=240` (`KSTB2500.mk:38`) |
| Ch.15 Q4 | Low Power Standby "Android 14" | Available since Android 13 |
| Ch.11 Q21 | `postSyncBarrier` is `@hide` | `@TestApi` + `@UnsupportedAppUsage` |
| Ch.11 Q8 | service timeout a fixed 20s | `20s × Build.HW_TIMEOUT_MULTIPLIER` |
| Ch.02 Q7 | boot phases 200, 520 missing | added `PHASE_WAIT_FOR_SENSOR_SERVICE(200)`, `PHASE_DEVICE_SPECIFIC_SERVICES_READY(520)` |
| Ch.13 | no dedicated question about `protectionLevel` | added section 13.7 (Q19–Q23): `system` == `privileged` (0x10), `fixProtectionLevel`, signature/privileged grant logic, `vendorPrivileged`, flag table |
