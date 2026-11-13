# Chương 10 — Mock interview tổng hợp

## Phần A — Rapid-fire (trả lời ≤ 30 giây)

| # | Câu hỏi | Trả lời ngắn |
|---|---------|--------------|
| 1 | PID của init? | 1 |
| 2 | Process cha của system_server? | zygote (zygote64) |
| 3 | UID của system_server / shell / root? | 1000 / 2000 / 0 |
| 4 | App đầu tiên sau boot? | Launcher (HOME), với Android TV là TV Launcher/DavidLauncher |
| 5 | Property báo boot xong? | `sys.boot_completed=1`, `dev.bootcomplete=1` |
| 6 | Giới hạn binder buffer? | ~1MB/process (oneway dùng ≤ ½) |
| 7 | Handle của servicemanager? | 0 |
| 8 | 3 binder device? | `/dev/binder`, `/dev/hwbinder`, `/dev/vndbinder` |
| 9 | Lệnh liệt kê HAL? | `lshal`, `service list`, `dumpsys -l` |
| 10 | AIDL HAL instance name format? | `<package>.<IFace>/<instance>` ví dụ `vendor.david.hardware.daviddevicecontrol.IDavidDeviceControl/default` |
| 11 | File khai báo HAL vendor cung cấp? | VINTF device manifest (`/vendor/etc/vintf/manifest*`) |
| 12 | `ro.*` set lại được không? | Không (trừ trước khi được set lần đầu) |
| 13 | Vendor property prefix? | `vendor.`, `ro.vendor.`, `persist.vendor.` |
| 14 | Lệnh freeze AIDL? | `m <name>-freeze-api` |
| 15 | Domain của app ký platform + uid system? | `system_app` |
| 16 | Macro cho HAL daemon được init start? | `init_daemon_domain(domain)` |
| 17 | Timeout watchdog system_server? | 60s (dump ở 30s) |
| 18 | ANR input timeout? | 5s |
| 19 | Partition chứa generic ramdisk A13+? | `init_boot` |
| 20 | `PRODUCT_SHIPPING_API_LEVEL` của project? | 34 (Android 14) |
| 21 | Đổi Android.mk sang bp? | tool `androidmk` |
| 22 | Biến thêm sepolicy vendor? | `BOARD_SEPOLICY_DIRS` |
| 23 | Biến thêm namespace Soong vào image? | `PRODUCT_SOONG_NAMESPACES` |
| 24 | Khác `getService` vs `checkService`? | C++ `getService` chờ ~5s, `checkService` trả ngay; **Java** `ServiceManager.getService` không chờ (trả null), muốn chờ dùng `waitForService` |
| 25 | Khác `waitForService` vs `waitForDeclaredService`? | Declared: chỉ chờ nếu service được khai báo VINTF, ngược lại trả null ngay |

---

## Phần B — Câu hỏi tình huống (scenario)

### B1. "Thiết bị bị bootloop sau khi bạn thêm một HAL mới. Bạn làm gì?"
1. Kết nối UART (STB luôn có) / `adb logcat` nếu adb lên được trước khi crash.
2. Xem log init: service crash liên tục? Có `critical`? → `init: Service 'xxx' (pid N) received signal 6`.
3. Nếu rơi vào `system_server` crash: tìm `FATAL EXCEPTION IN SYSTEM PROCESS` — có thể do client gọi HAL trả null rồi NPE.
4. Check `checkvintf` lúc boot: `init: ... VINTF compatibility check failed` → **framework từ chối boot** nếu manifest/matrix mismatch (trong A/B có thể rollback slot).
5. SELinux: `avc: denied` khiến service quan trọng fail. Thử build userdebug + `androidboot.selinux=permissive` (qua bootconfig) để khoanh vùng.
6. Fix, rebuild, flash lại `vendor` (fastbootd) hoặc `adb sync`.

### B2. "App gọi Manager của bạn thỉnh thoảng bị ANR 5s."
- Lấy ANR trace: main thread ở `BinderProxy.transactNative` → call đồng bộ vào system_server.
- Trong system_server: binder thread đang chờ `mLock` hoặc đang gọi HAL chậm (ghi sysfs/I2C chậm).
- Fix: làm API async (oneway + callback), giảm phạm vi lock, cache state ở system_server để `get*()` không cần xuống HAL, timeout ở HAL, chuyển việc nặng ra `Handler` riêng.

### B3. "Sau OTA, HAL vendor cũ không tương thích framework mới."
- Giải thích VINTF: framework mới có FCM version cao hơn; nếu `target-level` vendor quá cũ (deprecated) → OTA bị chặn.
- AIDL: framework mới dùng `-V2` nhưng HAL chỉ V1 → framework phải xử lý `getInterfaceVersion()` và fallback; FCM khai báo `<version>1-2</version>`.

### B4. "Thêm key mới trên remote (ví dụ phím Netflix) đi qua những tầng nào?"
1. Kernel: driver IR/BT HID gửi `EV_KEY` với scancode → `/dev/input/eventX`.
2. `InputReader` (native trong system_server) đọc qua EventHub, map scancode → keycode bằng **`.kl`** (`/vendor/usr/keylayout/<Vendor>_<Product>.kl`, project: `vendor/david/keylayout/`), `.kcm` cho ký tự.
3. Nếu keycode chưa có trong Android → thêm vào `KeyEvent.java`, `attrs.xml`, `InputEventLabels.h` (hoặc dùng sẵn `KEYCODE_PROG_RED`...).
4. `InputDispatcher` → `PhoneWindowManager.interceptKeyBeforeQueueing/Dispatching` (global key) → app focus.
5. Global key Android TV: `com.android.internal.R.xml.global_keys` (project có `DavidFrameworkOverlay` — "Global key") → gửi broadcast `GLOBAL_BUTTON` tới component chỉ định.

### B5. "Product manager muốn một property `persist.david.first_start_oem_app` để app OEM đọc/ghi. Làm thế nào cho đúng Treble?"
- Property set từ `PRODUCT_SYSTEM_PROPERTIES` → nằm `/system/build.prop`.
- Cần `property_contexts` cho prefix `persist.david.` trong **system_ext** sepolicy (vì system property), type riêng, `set_prop(system_app, david_oem_prop)`.
- Nếu không, label mặc định `default_prop` → app không ghi được (neverallow), test `vts_treble_sys_prop_test` có thể fail do namespace lạ.

### B6. "Bạn cần chạy một script shell lúc boot với quyền root (ví dụ `led.sh`, `wifi.sh` trong project). Bạn đánh giá thế nào?"
- Hoạt động được (`user root`, `oneshot`, `disabled`, trigger property) nhưng:
  - Cần SELinux domain riêng cho script (`vendor_shell` / `*_exec` + `init_daemon_domain`), nếu không bị chặn.
  - Rủi ro bảo mật (root), khó kiểm soát; nên thay bằng `write` trong rc (init làm trực tiếp), hoặc HAL/daemon chạy user giới hạn — chính là lý do có `DavidDeviceControl`.

### B7. "Thiết kế tính năng: system service đếm thời gian xem TV theo kênh, gửi về server mỗi ngày."
- Nguồn dữ liệu: TIF (`TvInputManager` session events) / `MediaSession` → service trong system_server hoặc privileged app (ưu tiên app để không đụng framework).
- Lưu: Room/SQLite trong `/data/system_ce/` hoặc data app; theo user.
- Upload: `JobScheduler` (network constraint, idle), backoff, không wakelock vô hạn.
- Quyền: permission signature cho API đọc thống kê; privacy (opt-in).
- Debug: `dumpsys`, `cmd` shell command.

---

## Phần C — Câu hỏi chiều sâu (senior/lead)

1. **Binder priority inheritance hoạt động thế nào?** → Driver nâng priority (nice/RT) của thread server lên bằng client (`FLAT_BINDER_FLAG_INHERIT_RT`, `min_priority` của node), tránh priority inversion (quan trọng cho audio/HAL real-time).
2. **Tại sao system_server dùng nhiều lock lớn (AMS `this`, WMS `mGlobalLock`)? Hậu quả?** → Đơn giản hóa tính đúng đắn nhưng gây contention; Google tách dần (ATMS khỏi AMS, `mProcLock`), dùng lock-free `ProcessRecord` state.
3. **APEX là gì, khác APK?** → Gói module native+Java mount tại `/apex/<name>` (loop + dm-verity), update qua Play (Mainline): ART, Conscrypt, media, tzdata, VNDK...
4. **Linker namespace là gì?** → Bionic linker cô lập thư viện giữa system/vendor/apex (`ld.config.txt` sinh bởi `linkerconfig`): vendor process không thấy lib system ngoài LL-NDK/VNDK → lỗi `library "libfoo.so" not found` dù file tồn tại.
5. **RRO (Runtime Resource Overlay)?** → Ghi đè resource của framework/app không sửa source; `runtime_resource_overlay` trong bp, cài `/product/overlay` hoặc `/vendor/overlay`; ví dụ project `DroidATVTvSettingsResOverlay`, `DavidFrameworkOverlay`.
6. **`privapp-permissions` và `default-permissions` khác nhau?** → privapp: whitelist **privileged permission** cho priv-app (bắt buộc nếu không device không boot ở mode enforce). default-permissions: **tự grant runtime permission** cho app khi cài (project: `default-launcher-permissions.xml`, `default-oad-permissions.xml`).
7. **Tại sao HAL không được dùng `libbinder` (cpp backend) trên vendor?** → libbinder không có ABI ổn định; vendor phải dùng `libbinder_ndk` (LL-NDK) để system update không làm vỡ vendor.
8. **Explain `dexpreopt`.** → Biên dịch AOT dex → odex/vdex/art lúc build (`/system/framework/oat`, `<app>/oat/`), giảm thời gian boot/first-launch; cấu hình `WITH_DEXPREOPT`, `PRODUCT_DEX_PREOPT_*`; boot image (`boot.art`) cho framework classes.
9. **System server crash loop sau update: cơ chế cứu hộ?** → `RescueParty`: reset settings → ... → đưa vào recovery đề nghị factory reset; `PackageWatchdog` rollback APEX/mainline.
10. **Mô tả cách AMS start một Activity ở process chưa có.** → `startActivity` → ATMS → `ActivityStarter` → process chưa có → `AMS.startProcessLocked` → `ZygoteProcess.start` (socket) → zygote fork → `ActivityThread.main` → `attachApplication` → AMS `bindApplication` → ATMS `realStartActivityLocked` → `ClientTransaction` (`LaunchActivityItem`) → `onCreate`.

---

## Phần D — Checklist tự đánh giá trước phỏng vấn

- [ ] Vẽ được boot flow và nói tên file source của từng bước (`first_stage_init.cpp`, `init.cpp`, `ZygoteInit.java`, `SystemServer.java`).
- [ ] Viết tay được `aidl_interface`, `cc_binary` HAL, rc, manifest, FCM, sepolicy cho 1 HAL mới (Chương 05 Q3).
- [ ] Viết tay được các bước thêm system service + Manager (Chương 06 Q1).
- [ ] Giải thích 1 binder transaction từ proxy → driver → stub (Chương 04 Q2).
- [ ] Đọc được 1 avc denial và đề xuất fix tối thiểu (Chương 07 Q3).
- [ ] Kể được case study DavidDeviceControl + 3 điểm sẽ cải thiện (Chương 09).
- [ ] Phân biệt rõ: HIDL/AIDL, servicemanager/hwservicemanager/vndservicemanager, system/system_ext/product/vendor/odm, user/userdebug/eng.
