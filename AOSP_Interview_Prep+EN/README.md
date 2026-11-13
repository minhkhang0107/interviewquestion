# Android AOSP Framework Interview Prep (Senior) — Android 14 / Amlogic STB

> Study material organized by chapter, in **Question → Answer → Example** form, referencing the source code of the project
> `kpp_nctc_kstb2500` (Android 14, Amlogic S7D SoC, board `KSTB2500`, vendor `david`).
> English translation of `../AOSP_Interview_Prep+VN/`.

## Table of contents

| # | Chapter | Main content |
|---|--------|----------------|
| 01 | [Architecture overview & Treble](01_Architecture_Overview_Treble.md) | Android stack, partitions, Treble, VNDK, GSI |
| 02 | [Boot flow](02_Boot_flow.md) | ROM → BL → kernel → init (1st/2nd stage) → zygote → system_server → Launcher, init.rc, AVB, A/B |
| 03 | [Build system: Android.bp & Android.mk](03_Build_system_Android_bp_mk.md) | Soong, Make, namespaces, product makefiles, PRODUCT_PACKAGES |
| 04 | [Binder & AIDL](04_Binder_AIDL.md) | Binder driver, ServiceManager, oneway, death recipients, NDK/CPP/Java/Rust backends |
| 05 | [HAL: HIDL → AIDL HAL, VINTF](05_HAL_AIDL_HIDL_VINTF.md) | Stable AIDL, versioning/freeze, manifest, compatibility matrix |
| 06 | [System Service & Manager](06_System_service_Manager.md) | SystemServer, SystemServiceManager, SystemServiceRegistry, permissions |
| 07 | [SELinux](07_SELinux.md) | te, file_contexts, service_contexts, macros, neverallow, debugging denials |
| 08 | [Debug & Performance](08_Debug_Performance.md) | logcat, dumpsys, ANR, tombstones, watchdog, perfetto, lshal |
| 09 | [Case study: DavidDeviceControl end-to-end](09_Case_study_DavidDeviceControl.md) | Analysis of a real HAL in the project + senior-style code review |
| 10 | [Combined mock interview](10_Mock_interview.md) | Scenario questions, rapid-fire, system design |
| **11** | [**App layer**](11_App_layer.md) | Lifecycle, launch modes, Service/FGS A14, Broadcasts, Providers, Looper, Coroutines, Jetpack, rendering, Compose, ANR/leaks, Android TV (focus, Leanback, Launcher, TIF) |
| **12** | [**Framework: ATMS/AMS & Process**](12_Framework_AMS_ATMS_Process.md) | startActivity end-to-end, ActivityThread, oom_adj/lmkd, ActiveServices, BroadcastQueueModernImpl, ContentProvider |
| **13** | [**Framework: PMS, Permissions, Context**](13_Framework_PMS_Permission_Context.md) | Boot scan, install sessions, signatures, privapp/default permissions, **protectionLevel (`signature\|privileged`, `signatureOrSystem`, `vendorPrivileged`, `knownSigner`...)**, Context, Settings, RRO, hidden API |
| **14** | [**Framework: WMS, Graphics, Input**](14_Framework_WMS_Graphics_Input.md) | WindowContainer tree, addView, BLAST, SurfaceFlinger/HWC, input pipeline, global keys, `.kl`, native Looper |
| **15** | [**Framework: Power, Media, TV**](15_Framework_Power_Media_TV.md) | Standby/suspend, HDMI-CEC, Audio/Codec2/tunnel, TIF, Alarm/Job, multi-user, CE/DE |
| **16** | [**Discussion with Gemini**](16_Gemini_discussion.md) | Who was right/wrong, evidence in the source, fixed errors |
| **17** | [**Kotlin & Modern Android**](17_Kotlin_Modern_Android.md) | From personal notes (errors fixed): Kotlin core, generics, inline/reified, coroutines/Flow, Service/IPC, architecture & design patterns; new up to 11/2026: Kotlin 2.x/K2, Compose, Android 15/16 |
| **18** | [**DRM & Widevine**](18_DRM_Widevine.md) | MediaDrm/MediaCrypto, L1/L3, provisioning/keybox, secure video path, HDCP, AIDL DRM HAL, CAS, debugging "SD only" |
| **19** | [**OTA: A/B, Virtual A/B**](19_OTA_Update.md) | ota_from_target_files, payload.bin, update_engine, boot_control, rollback/anti-rollback, snapuserd/merge, OTA design for 1M boxes |
| **20** | [**Concurrency & Deadlock**](20_Concurrency_Deadlock.md) | Lock order, system_server deadlocks, binder re-entrancy/thread pool, Watchdog, reading traces, C++ HALs, coroutines |
| **21** | [**ART & Garbage Collector**](21_ART_Garbage_Collector.md) | CMS → CC → generational → CMC, heap spaces, read barrier, GC logs, heap props, OOM, reference types, memory tools |
| **22** | [**App Services & System Services (deep dive)**](22_Service_and_System_Service.md) | 5 kinds of "service", bindService end-to-end, BIND_*, restarts, FGS A8→A15, threading, RemoteCallbackList, security, OEM SDK, native services |
| — | [`gemini_raw/`](gemini_raw/) | Gemini's original output (uncorrected; read together with chapter 16) — machine-translated by Gemini |

> The **⚠️ Corrected** marker in chapters 11–15 flags places where Gemini was wrong; **⚠️ Notes fix** in chapter 17 flags places where the old personal notes were wrong/outdated.

## Study plan (suggested 7 days, focused on App + Framework)

1. **Day 1**: Chapter 11 (App layer): lifecycle, tasks, service/FGS A14, broadcasts, providers.
2. **Day 2**: Chapter 11 (cont.): Looper, coroutines, Jetpack, rendering, Compose, Android TV.
3. **Day 3**: Chapter 12: be able to redraw the `startActivity` flow from app A to app B's `onResume` (new process), explain the oom_adj table.
4. **Day 4**: Chapters 13 + 14: permissions/privapp, WMS addView, BLAST, input pipeline, global keys (David example).
5. **Day 5**: Chapters 15 + 04 + 06: binder, system services, power/CEC/media/TIF.
6. **Day 6**: Chapters 01–03, 05, 07, 09: boot, build, HAL, SELinux, the DavidDeviceControl case study.
7. **Day 7**: Chapters 08 + 10 + 16: debugging, mock interview, review the verified "traps".
8. **Day 8 (added)**: Chapters 18–22: DRM/Widevine, OTA, deadlock, GC, services deep dive — prioritize 18 and 20 for STB roles.
9. **30 minutes every day**: Chapter 17 (Kotlin): the 17.7 rapid-fire and the **⚠️ Notes fix** items.

## How to answer as a senior (formula)

1. **Short definition** (1 sentence).
2. **Internal mechanism** (how it works, which file/class).
3. **Real example** from a project you worked on (e.g. the `DavidDeviceControl` HAL driving the LED via sysfs).
4. **Trade-offs / pitfalls** (common bugs, how to debug).

## Reference files in the source tree

| Component | Path |
|------------|-----------|
| Vendor AIDL interface | `vendor/david/interfaces/daviddevicecontrol/aidl/` |
| AIDL HAL service (C++ NDK) | `vendor/david/hals/DavidDeviceControl/` |
| Java client (persistent app, system uid) | `vendor/david/services/framework/davidextensionsvc/` |
| Vendor SELinux | `vendor/david/sepolicy/vendor/` |
| Vendor product config | `vendor/david/david.mk` |
| Device config | `device/amlogic/KSTB2500/{KSTB2500.mk, device.mk, BoardConfig.mk}` |
| Board init rc | `device/amlogic/KSTB2500/init.amlogic.board.rc` |
| fstab | `device/amlogic/KSTB2500/fstab.ab_oem.amlogic` |
