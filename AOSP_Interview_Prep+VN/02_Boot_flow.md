# Chương 02 — Boot flow Android (A14, Amlogic STB)

## 2.1 Tổng quan

```
Power on
  │
  ▼
BootROM (mask ROM trong SoC) ── verify & load BL2 (secure boot, efuse key)
  │
  ▼
BL2 / BL31 (ATF, EL3) / BL32 (TEE: OP-TEE/Amlogic TDK) / BL33 (U-Boot)
  │   U-Boot: đọc misc (BCB), chọn slot A/B, verify vbmeta (libavb),
  │   nạp boot + init_boot + vendor_boot + DTB, truyền cmdline/bootconfig
  ▼
Linux kernel (GKI) ── start_kernel() → rest_init() → kernel_init()
  │   chạy /init trong ramdisk (PID 1)
  ▼
init — First stage (ramdisk)
  │   mount /dev, /proc, /sys; load module vendor_boot; đọc fstab first_stage;
  │   tạo dm-linear (super) + dm-verity; mount /system, /vendor...; switch_root
  │   → exec /system/bin/init selinux_setup
  ▼
init — SELinux setup: load policy (split: plat + vendor + ...), setenforce
  │   → exec /system/bin/init second_stage
  ▼
init — Second stage
  │   property service, parse *.rc, triggers early-init→init→late-init→boot
  │   start ueventd, servicemanager, hwservicemanager, vndservicemanager, HALs,
  │   vold, logd, surfaceflinger, ... và zygote
  ▼
Zygote (app_process64) ── ZygoteInit: preload classes/resources, fork system_server
  ▼
system_server ── SystemServer.run(): startBootstrapServices → startCoreServices
  │   → startOtherServices → startApexServices → AMS.systemReady()
  ▼
Launcher (HOME intent) → BOOT_COMPLETED, sys.boot_completed=1
```

---

### Q1. Mô tả boot flow từ lúc bật nguồn đến lúc thấy Launcher.

Trả lời theo sơ đồ trên, nhấn mạnh **4 điểm senior**:
1. **Chain of trust**: BootROM → BL2 → U-Boot → vbmeta → boot → dm-verity cho system/vendor. Key ở efuse (Amlogic secure boot) + AVB key.
2. **init 3 giai đoạn**: first stage (`FirstStageMain`), selinux setup (`SetupSelinux`), second stage (`SecondStageMain`) — source: `system/core/init/`.
3. **Zygote fork system_server** (`Zygote.forkSystemServer`) với uid 1000, capabilities đặc biệt.
4. **systemReady → start Home → BOOT_COMPLETED** — `sys.boot_completed=1` được set bởi AMS (`finishBooting`) → trigger trong rc chạy (ví dụ project set CPU governor lúc này).

---

### Q2. First stage init làm gì? Tại sao cần tách first/second stage?

- Chạy từ **ramdisk** (generic ramdisk của `init_boot` + vendor ramdisk của `vendor_boot`).
- Việc chính (`system/core/init/first_stage_init.cpp`):
  - mount tmpfs `/dev`, `/proc`, `/sys`, `/sys/fs/selinux`...
  - load kernel modules từ `/lib/modules` (vendor ramdisk) theo `modules.load`.
  - đọc fstab (tìm trong DT hoặc `/first_stage_ramdisk/fstab.*`), mount partition có cờ `first_stage_mount` (system, vendor, product, odm, metadata...).
  - setup **dm-verity** (AVB) + **dm-linear** cho super.
  - `switch_root` sang `/system` → exec `/system/bin/init selinux_setup`.
- **Lý do tách**: SELinux policy nằm trên /system & /vendor, phải mount xong mới load được; ramdisk nhỏ, generic (GKI).

Ví dụ fstab của project (`fstab.ab_oem.amlogic`):
```
vendor   /vendor   erofs   ro   wait,slotselect,avb=vbmeta,logical,first_stage_mount
/dev/block/by-name/userdata  /data  f2fs  ...  latemount,wait,check,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,...
```
- `slotselect`: thêm suffix `_a/_b`; `avb=vbmeta`: verify bằng vbmeta; `logical`: nằm trong super; `latemount`: mount sau (bởi `mount_all --late`); `/data` dùng **FBE + metadata encryption**.

---

### Q3. init.rc language: Action, Service, Trigger, Command, Option?

```rc
# Action: on <trigger> + commands
on early-init
    loglevel 3
    start led

on property:sys.boot_completed=1           # property trigger
    write /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor schedutil
    start factory_mount

# Service: service <name> <path> [args] + options
service factory_mount /vendor/bin/factory_mount.sh
    class main          # nhóm, start bằng class_start main
    user root
    oneshot             # chạy 1 lần, không restart
    disabled            # không tự start theo class, phải start tay
```
(Trích từ `device/amlogic/KSTB2500/init.amlogic.board.rc`)

**Thứ tự trigger chính:** `early-init` → `init` → `late-init` → (`early-fs` → `fs` → `post-fs` → `late-fs` → `post-fs-data` → `zygote-start` → `early-boot` → `boot`).

**Service options hay hỏi:**
| Option | Ý nghĩa |
|--------|---------|
| `class core/main/late_start/hal` | nhóm khởi động |
| `user/group/capabilities` | quyền chạy |
| `oneshot` | không restart khi exit |
| `disabled` | không start theo class |
| `critical` | crash > 4 lần/4 phút → reboot bootloader |
| `onrestart restart xxx` | restart service khác khi service này restart |
| `socket name type perm` | tạo socket `/dev/socket/name` |
| `writepid /dev/cpuset/...` / `task_profiles` | cgroup |
| `interface aidl <name>/<instance>` | khai báo service cung cấp interface → dùng cho lazy HAL / `ctl.interface_start` |
| `seclabel` | domain SELinux (thường dùng transition qua file_contexts thay vì seclabel) |
| `updatable` | cho phép override bởi APEX |

---

### Q4. Property system hoạt động thế nào? `ro.`, `persist.`, `ctl.`, `sys.`, `vendor.`?

- init tạo **property area** (`/dev/__properties__`) — shared memory, mỗi context SELinux 1 file → process đọc trực tiếp không qua IPC.
- **Ghi** property: qua socket `/dev/socket/property_service` → init kiểm tra SELinux `property_service set` → ghi.
- Prefix:
  - `ro.*`: chỉ set 1 lần (read-only). `ro.boot.*` từ cmdline/bootconfig (`androidboot.xxx`).
  - `persist.*`: lưu vào `/data/property/persistent_properties`.
  - `ctl.start/ctl.stop/ctl.restart`: điều khiển service init.
  - `sys.*`, `vendor.*`: vendor phải dùng prefix `vendor.`/`ro.vendor.`/`persist.vendor.` (Treble namespace).
- File nguồn build: `/system/build.prop`, `/vendor/build.prop`, `/product/etc/build.prop`… set bằng `PRODUCT_SYSTEM_PROPERTIES`, `PRODUCT_VENDOR_PROPERTIES`.

Ví dụ project:
```make
# vendor/david/david.mk
PRODUCT_SYSTEM_PROPERTIES += ro.david.version=$(BUILD_DAVID_FIRMWARE_VERSION)
```
```rc
# chuyển property từ bootloader sang ro.*
on property:ro.boot.oem.key1=*
    setprop ro.oem.key1 ${ro.boot.oem.key1}
```
Câu bẫy: *"Tại sao `ro.david.version` lẽ ra nên là `ro.vendor.david.*` hoặc `ro.product.*`?"* → Treble property namespace: property khai báo ở system phải có `property_contexts` tương ứng ở `plat`/`system_ext`; property vendor phải có prefix vendor để không đụng namespace của system (có test `vts_treble_sys_prop_test`).

---

### Q5. ueventd làm gì?

- Là chính binary `init` chạy mode `ueventd` (symlink).
- Nhận **uevent** từ kernel (netlink), tạo node `/dev/*` với permission theo `ueventd.rc` (+ `/vendor/etc/ueventd.rc`), load firmware (`firmware_directories`), cold boot (duyệt `/sys` tạo node cho device đã probe trước khi ueventd chạy).
- Project có `ueventd.parallel.rc` → parallel restorecon / coldboot để tăng tốc.
- Ví dụ dòng ueventd.rc:
```
/dev/amvideo   0666   media   system
/sys/class/leds/*  brightness  0664  system system
```

---

### Q6. Zygote làm gì? Tại sao fork thay vì start process mới?

- `service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server` (`init.zygote64.rc`).
- `ZygoteInit.main()`:
  1. Preload classes (`/system/etc/preloaded-classes`), resources, shared libs, OpenGL/Vulkan driver.
  2. `forkSystemServer()` → `SystemServer.main()`.
  3. `runSelectLoop()` chờ lệnh qua socket `/dev/socket/zygote` (AMS gửi qua `ZygoteProcess`).
- **Fork + Copy-on-Write**: app con chia sẻ trang nhớ preload → khởi động app nhanh, tiết kiệm RAM.
- Có **USAP pool** (unspecialized app process) để giảm latency, **WebView zygote**, **app zygote** (isolated).

---

### Q7. SystemServer khởi động service thế nào?

`frameworks/base/services/java/com/android/server/SystemServer.java`:
```java
private void run() {
    ...
    Looper.prepareMainLooper();
    System.loadLibrary("android_servers");
    createSystemContext();
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    startBootstrapServices(t);   // Installer, AMS/ATMS, PowerMS, PMS, DisplayMS, ...
    startCoreServices(t);        // BatteryService, UsageStats, WebViewUpdate, ...
    startOtherServices(t);       // WMS, InputMS, AudioService, Network..., cuối gọi AMS.systemReady()
    startApexServices(t);
    Looper.loop();
}
```
- Mỗi service kế thừa `SystemService` → `onStart()` (publishBinderService), `onBootPhase(phase)`.
- Boot phase: `PHASE_WAIT_FOR_DEFAULT_DISPLAY(100)` → `PHASE_WAIT_FOR_SENSOR_SERVICE(200)` → `PHASE_LOCK_SETTINGS_READY(480)` → `PHASE_SYSTEM_SERVICES_READY(500)` → `PHASE_DEVICE_SPECIFIC_SERVICES_READY(520)` → `PHASE_ACTIVITY_MANAGER_READY(550)` → `PHASE_THIRD_PARTY_APPS_CAN_START(600)` → `PHASE_BOOT_COMPLETED(1000)`.

---

### Q8. AVB / dm-verity hoạt động thế nào? Unlock bootloader thì sao?

- **AVB 2.0** (libavb trong U-Boot): `vbmeta` chứa hash (cho partition nhỏ: boot, vendor_boot, dtbo) và **hashtree descriptor** (cho partition lớn: system, vendor...) + chữ ký.
- Kernel/init dùng **dm-verity** để verify từng block khi đọc (lazy) → phát hiện sửa đổi lúc runtime.
- `vbmeta_system` được chain bằng key riêng (`avb_keys=/avb` trong fstab) → OEM ký system độc lập vendor.
- Trạng thái: `ro.boot.verifiedbootstate` = green / yellow / orange (unlocked) / red.
- Rollback protection: `rollback_index` lưu trong TEE/RPMB.
- Disable verity khi dev: `adb root; adb disable-verity; adb reboot` (chỉ userdebug/eng, bootloader unlocked) → dùng `adb remount` với overlayfs.

Ví dụ BoardConfig:
```make
BOARD_AVB_SYSTEM_ADD_HASHTREE_FOOTER_ARGS += --hash_algorithm sha256
BOARD_AVB_VENDOR_ADD_HASHTREE_FOOTER_ARGS += --hash_algorithm sha256
```

---

### Q9. bootconfig khác kernel cmdline thế nào?

- Từ A12, các tham số `androidboot.*` chuyển từ kernel cmdline sang **bootconfig** (file append cuối initrd), đọc ở `/proc/bootconfig`.
- Lý do: cmdline giới hạn độ dài, kernel không cần biết param Android.
- Project: `BOARD_KERNEL_CMDLINE += bootconfig`.
- init chuyển `androidboot.foo=bar` thành `ro.boot.foo=bar`.

---

### Q10. Làm sao đo/tối ưu boot time?

- `adb shell dmesg | grep -i "init"`, `logcat -b events | grep boot_progress` (`boot_progress_start`, `boot_progress_system_run`, `boot_progress_pms_start`, `boot_progress_ams_ready`, `boot_progress_enable_screen`).
- `adb shell getprop | grep boottime` (`ro.boottime.init`, `ro.boottime.<service>`).
- **bootchart**: `adb shell touch /data/bootchart/enabled` → reboot → `system/core/init/grab-bootchart.sh`.
- **Perfetto boot trace** (A13+): config `persist.debug.perfetto.boottrace=1`.
- Tối ưu: load module song song / defer module không cần; dùng `erofs`; giảm preload class; lazy HAL; tránh `wait_for_prop`/`exec` blocking trong rc; dexpreopt đầy đủ (tránh dex2oat lúc boot); `ueventd` parallel; giảm service `class core`.

---

### Q11. Recovery / fastbootd / BCB?

- **BCB** (Bootloader Control Block) ghi trong partition `misc`: `boot-recovery`, args `--wipe_data`, `--update_package`...
- Recovery là ramdisk riêng (hoặc trong boot với `BOARD_USES_RECOVERY_AS_BOOT`), chạy `/system/bin/recovery`.
- **fastbootd**: fastboot ở userspace (trong recovery) để flash logical partitions trong `super`.
- Lệnh: `adb reboot recovery`, `adb reboot fastboot` (fastbootd), `adb reboot bootloader` (U-Boot fastboot).

---

### Q12. Service HAL được start lúc nào? Nếu HAL start muộn thì client làm sao?

- HAL thường `class hal` → được start bởi `class_start hal` trong `on boot` của `init.rc`... (core HALs có thể `class early_hal`).
- Client dùng `AServiceManager_waitForService()` / `ServiceManager.waitForDeclaredService()` để **block chờ** (servicemanager biết service được declare trong VINTF manifest).
- Nếu service chưa declare trong VINTF → `waitForDeclaredService` trả `null` ngay (không chờ) — lỗi rất hay gặp.

Ví dụ project (`vendor.david.hardware.daviddevicecontrol-service.rc`):
```rc
on init
    start daviddevicecontrol

service daviddevicecontrol /vendor/bin/hw/vendor.david.hardware.daviddevicecontrol-service
    class hal
    user system
    group system
```
Nhận xét senior: `on init start ...` start sớm hơn `class hal` (vì LED cần sáng sớm để báo boot), nhưng nên thêm `interface aidl vendor.david.hardware.daviddevicecontrol.IDavidDeviceControl/default` để init biết service này phục vụ interface nào (cần cho lazy HAL & `ctl.interface_start`), và cân nhắc dùng `class early_hal` thay vì trigger thủ công.
