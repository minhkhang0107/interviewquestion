# Ôn phỏng vấn Android AOSP Framework (Senior) — Android 14 / STB Amlogic

> Tài liệu ôn tập theo chương, dạng **Câu hỏi → Trả lời → Ví dụ**, tham chiếu trực tiếp source code của project
> `kpp_nctc_kstb2500` (Android 14, SoC Amlogic S7D, board `KSTB2500`, vendor `david`).

## Mục lục

| # | Chương | Nội dung chính |
|---|--------|----------------|
| 01 | [Kiến trúc tổng quan & Treble](01_Kien_truc_tong_quan_Treble.md) | Stack Android, partition, Treble, VNDK, GSI |
| 02 | [Boot flow](02_Boot_flow.md) | ROM → BL → kernel → init (1st/2nd stage) → zygote → system_server → Launcher, init.rc, AVB, A/B |
| 03 | [Build system: Android.bp & Android.mk](03_Build_system_Android_bp_mk.md) | Soong, Make, namespace, product makefile, PRODUCT_PACKAGES |
| 04 | [Binder & AIDL](04_Binder_AIDL.md) | Binder driver, ServiceManager, oneway, death recipient, backend NDK/CPP/Java/Rust |
| 05 | [HAL: HIDL → AIDL HAL, VINTF](05_HAL_AIDL_HIDL_VINTF.md) | Stable AIDL, versioning/freeze, manifest, compatibility matrix |
| 06 | [System Service & Manager](06_System_service_Manager.md) | SystemServer, SystemServiceManager, SystemServiceRegistry, permission |
| 07 | [SELinux](07_SELinux.md) | te, file_contexts, service_contexts, macros, neverallow, debug denial |
| 08 | [Debug & Performance](08_Debug_Performance.md) | logcat, dumpsys, ANR, tombstone, watchdog, perfetto, lshal |
| 09 | [Case study: DavidDeviceControl end-to-end](09_Case_study_DavidDeviceControl.md) | Phân tích HAL thật trong project + code review như senior |
| 10 | [Mock interview tổng hợp](10_Mock_interview.md) | Câu hỏi tình huống, rapid-fire, system design |
| **11** | [**Tầng App**](11_App_layer.md) | Lifecycle, launch mode, Service/FGS A14, Broadcast, Provider, Looper, Coroutines, Jetpack, rendering, Compose, ANR/leak, Android TV (focus, Leanback, Launcher, TIF) |
| **12** | [**Framework: ATMS/AMS & Process**](12_Framework_AMS_ATMS_Process.md) | startActivity end-to-end, ActivityThread, oom_adj/lmkd, ActiveServices, BroadcastQueueModernImpl, ContentProvider |
| **13** | [**Framework: PMS, Permission, Context**](13_Framework_PMS_Permission_Context.md) | Boot scan, install session, chữ ký, privapp/default permissions, **protectionLevel (`signature\|privileged`, `signatureOrSystem`, `vendorPrivileged`, `knownSigner`...)**, Context, Settings, RRO, hidden API |
| **14** | [**Framework: WMS, Graphics, Input**](14_Framework_WMS_Graphics_Input.md) | Cây WindowContainer, addView, BLAST, SurfaceFlinger/HWC, input pipeline, global keys, `.kl`, Looper native |
| **15** | [**Framework: Power, Media, TV**](15_Framework_Power_Media_TV.md) | Standby/suspend, HDMI-CEC, Audio/Codec2/tunnel, TIF, Alarm/Job, multi-user, CE/DE |
| **16** | [**Thảo luận với Gemini**](16_Gemini_discussion.md) | Ai đúng ai sai, bằng chứng trong source, lỗi đã sửa |
| **17** | [**Kotlin & Android hiện đại**](17_Kotlin_Android_hien_dai.md) | Từ ghi chú cá nhân (đã sửa chỗ sai): Kotlin core, generics, inline/reified, coroutines/Flow, Service/IPC, kiến trúc & design pattern; mới tới 11/2026: Kotlin 2.x/K2, Compose, Android 15/16 |
| — | [`gemini_raw/`](gemini_raw/) | Bản gốc output của Gemini (chưa sửa; đọc kèm chương 16) |

> Ký hiệu **⚠️ Đã sửa** trong chương 11–15 đánh dấu chỗ Gemini viết sai; **⚠️ Sửa ghi chú** trong chương 17 đánh dấu chỗ ghi chú cá nhân cũ sai/lỗi thời.


## Lộ trình ôn (gợi ý 7 ngày, trọng tâm App + Framework)

1. **Ngày 1**: Chương 11 (tầng App): lifecycle, task, service/FGS A14, broadcast, provider.
2. **Ngày 2**: Chương 11 (tiếp): Looper, coroutines, Jetpack, rendering, Compose, Android TV.
3. **Ngày 3**: Chương 12: vẽ lại được luồng `startActivity` từ app A tới `onResume` của app B (process mới), giải thích bảng oom_adj.
4. **Ngày 4**: Chương 13 + 14: permission/privapp, WMS addView, BLAST, input pipeline, global key (ví dụ David).
5. **Ngày 5**: Chương 15 + 04 + 06: binder, system service, power/CEC/media/TIF.
6. **Ngày 6**: Chương 01–03, 05, 07, 09: boot, build, HAL, SELinux, case study DavidDeviceControl.
7. **Ngày 7**: Chương 08 + 10 + 16: debug, mock interview, rà lại các "bẫy" đã được kiểm chứng.
8. **Xen kẽ mỗi ngày 30 phút**: Chương 17 (Kotlin): rapid-fire 17.7 và các mục **⚠️ Sửa ghi chú**.

## Cách trả lời phỏng vấn senior (công thức)

1. **Định nghĩa ngắn** (1 câu).
2. **Cơ chế bên trong** (nó hoạt động thế nào, file/class nào).
3. **Ví dụ thực tế** từ project đã làm (ví dụ: `DavidDeviceControl` HAL điều khiển LED qua sysfs).
4. **Trade-off / pitfall** (lỗi hay gặp, cách debug).

## File tham chiếu trong source tree

| Thành phần | Đường dẫn |
|------------|-----------|
| AIDL interface vendor | `vendor/david/interfaces/daviddevicecontrol/aidl/` |
| AIDL HAL service (C++ NDK) | `vendor/david/hals/DavidDeviceControl/` |
| Client Java (persistent app, system uid) | `vendor/david/services/framework/davidextensionsvc/` |
| SELinux vendor | `vendor/david/sepolicy/vendor/` |
| Product config vendor | `vendor/david/david.mk` |
| Device config | `device/amlogic/KSTB2500/{KSTB2500.mk, device.mk, BoardConfig.mk}` |
| init rc board | `device/amlogic/KSTB2500/init.amlogic.board.rc` |
| fstab | `device/amlogic/KSTB2500/fstab.ab_oem.amlogic` |
