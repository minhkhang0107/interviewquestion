# Chapter 02 — Android Boot Flow (A14, Amlogic STB)

## 2.1 Overview

```
Power on
  │
  ▼
BootROM (mask ROM in the SoC) ── verify & load BL2 (secure boot, efuse key)
  │
  ▼
BL2 / BL31 (ATF, EL3) / BL32 (TEE: OP-TEE/Amlogic TDK) / BL33 (U-Boot)
  │   U-Boot: read misc (BCB), select A/B slot, verify vbmeta (libavb),
  │   load boot + init_boot + vendor_boot + DTB, pass cmdline/bootconfig
  ▼
Linux kernel (GKI) ── start_kernel() → rest_init() → kernel_init()
  │   runs /init from the ramdisk (PID 1)
  ▼
init — First stage (ramdisk)
  │   mount /dev, /proc, /sys; load vendor_boot modules; read first_stage fstab;
  │   create dm-linear (super) + dm-verity; mount /system, /vendor...; switch_root
  │   → exec /system/bin/init selinux_setup
  ▼
init — SELinux setup: load policy (split: plat + vendor + ...), setenforce
  │   → exec /system/bin/init second_stage
  ▼
init — Second stage
  │   property service, parse *.rc, triggers early-init→init→late-init→boot
  │   start ueventd, servicemanager, hwservicemanager, vndservicemanager, HALs,
  │   vold, logd, surfaceflinger, ... and zygote
  ▼
Zygote (app_process64) ── ZygoteInit: preload classes/resources, fork system_server
  ▼
system_server ── SystemServer.run(): startBootstrapServices → startCoreServices
  │   → startOtherServices → startApexServices → AMS.systemReady()
  ▼
Launcher (HOME intent) → BOOT_COMPLETED, sys.boot_completed=1
```

---

### Q1. Describe the boot flow from power-on until the Launcher appears.

Answer following the diagram above, stressing **4 senior-level points**:
1. **Chain of trust**: BootROM → BL2 → U-Boot → vbmeta → boot → dm-verity for system/vendor. Keys in efuse (Amlogic secure boot) + AVB key.
2. **init's 3 stages**: first stage (`FirstStageMain`), selinux setup (`SetupSelinux`), second stage (`SecondStageMain`) — source: `system/core/init/`.
3. **Zygote forks system_server** (`Zygote.forkSystemServer`) with uid 1000 and special capabilities.
4. **systemReady → start Home → BOOT_COMPLETED** — `sys.boot_completed=1` is set by AMS (`finishBooting`) → rc triggers fire (e.g. the project sets the CPU governor at this point).

---

### Q2. What does first stage init do? Why split first/second stage?

- Runs from the **ramdisk** (generic ramdisk from `init_boot` + vendor ramdisk from `vendor_boot`).
- Main work (`system/core/init/first_stage_init.cpp`):
  - mount tmpfs `/dev`, `/proc`, `/sys`, `/sys/fs/selinux`...
  - load kernel modules from `/lib/modules` (vendor ramdisk) according to `modules.load`.
  - read fstab (from DT or `/first_stage_ramdisk/fstab.*`), mount partitions flagged `first_stage_mount` (system, vendor, product, odm, metadata...).
  - set up **dm-verity** (AVB) + **dm-linear** for super.
  - `switch_root` to `/system` → exec `/system/bin/init selinux_setup`.
- **Why split**: the SELinux policy lives on /system & /vendor, so they must be mounted before it can be loaded; the ramdisk stays small and generic (GKI).

Example fstab from the project (`fstab.ab_oem.amlogic`):
```
vendor   /vendor   erofs   ro   wait,slotselect,avb=vbmeta,logical,first_stage_mount
/dev/block/by-name/userdata  /data  f2fs  ...  latemount,wait,check,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,...
```
- `slotselect`: adds the `_a/_b` suffix; `avb=vbmeta`: verified via vbmeta; `logical`: lives in super; `latemount`: mounted later (by `mount_all --late`); `/data` uses **FBE + metadata encryption**.

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
    class main          # group, started by class_start main
    user root
    oneshot             # runs once, not restarted
    disabled            # not started with its class, must be started explicitly
```
(Taken from `device/amlogic/KSTB2500/init.amlogic.board.rc`)

**Main trigger order:** `early-init` → `init` → `late-init` → (`early-fs` → `fs` → `post-fs` → `late-fs` → `post-fs-data` → `zygote-start` → `early-boot` → `boot`).

**Frequently asked service options:**
| Option | Meaning |
|--------|---------|
| `class core/main/late_start/hal` | startup group |
| `user/group/capabilities` | run-as privileges |
| `oneshot` | not restarted on exit |
| `disabled` | not started with its class |
| `critical` | crashes > 4 times in 4 minutes → reboot to bootloader |
| `onrestart restart xxx` | restart another service when this one restarts |
| `socket name type perm` | create socket `/dev/socket/name` |
| `writepid /dev/cpuset/...` / `task_profiles` | cgroup |
| `interface aidl <name>/<instance>` | declares the interface the service provides → used for lazy HALs / `ctl.interface_start` |
| `seclabel` | SELinux domain (usually domain transition via file_contexts is preferred over seclabel) |
| `updatable` | allows override by an APEX |

---

### Q4. How does the property system work? `ro.`, `persist.`, `ctl.`, `sys.`, `vendor.`?

- init creates the **property area** (`/dev/__properties__`) — shared memory, one file per SELinux context → processes read directly without IPC.
- **Writing** a property: via the socket `/dev/socket/property_service` → init checks SELinux `property_service set` → writes.
- Prefixes:
  - `ro.*`: set only once (read-only). `ro.boot.*` come from cmdline/bootconfig (`androidboot.xxx`).
  - `persist.*`: stored in `/data/property/persistent_properties`.
  - `ctl.start/ctl.stop/ctl.restart`: control init services.
  - `sys.*`, `vendor.*`: vendor must use the `vendor.`/`ro.vendor.`/`persist.vendor.` prefixes (Treble namespace).
- Build sources: `/system/build.prop`, `/vendor/build.prop`, `/product/etc/build.prop`… set with `PRODUCT_SYSTEM_PROPERTIES`, `PRODUCT_VENDOR_PROPERTIES`.

Project examples:
```make
# vendor/david/david.mk
PRODUCT_SYSTEM_PROPERTIES += ro.david.version=$(BUILD_DAVID_FIRMWARE_VERSION)
```
```rc
# copy a property from the bootloader into ro.*
on property:ro.boot.oem.key1=*
    setprop ro.oem.key1 ${ro.boot.oem.key1}
```
Trick question: *"Why should `ro.david.version` rather be `ro.vendor.david.*` or `ro.product.*`?"* → Treble property namespace: a property declared on system needs a matching `property_contexts` entry in `plat`/`system_ext`; vendor properties must use the vendor prefix so they don't collide with the system namespace (there is a test, `vts_treble_sys_prop_test`).

---

### Q5. What does ueventd do?

- It is the `init` binary itself running in `ueventd` mode (symlink).
- Receives **uevents** from the kernel (netlink), creates `/dev/*` nodes with permissions from `ueventd.rc` (+ `/vendor/etc/ueventd.rc`), loads firmware (`firmware_directories`), and does cold boot (walks `/sys` to create nodes for devices probed before ueventd started).
- The project has `ueventd.parallel.rc` → parallel restorecon / coldboot to speed things up.
- Example ueventd.rc lines:
```
/dev/amvideo   0666   media   system
/sys/class/leds/*  brightness  0664  system system
```

---

### Q6. What does Zygote do? Why fork instead of starting a fresh process?

- `service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server` (`init.zygote64.rc`).
- `ZygoteInit.main()`:
  1. Preload classes (`/system/etc/preloaded-classes`), resources, shared libs, OpenGL/Vulkan driver.
  2. `forkSystemServer()` → `SystemServer.main()`.
  3. `runSelectLoop()` waits for commands on the `/dev/socket/zygote` socket (AMS sends them via `ZygoteProcess`).
- **Fork + Copy-on-Write**: children share the preloaded pages → fast app startup, RAM savings.
- There is also the **USAP pool** (unspecialized app processes) to reduce latency, the **WebView zygote**, and the **app zygote** (isolated).

---

### Q7. How does SystemServer start its services?

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
    startOtherServices(t);       // WMS, InputMS, AudioService, Network..., finally calls AMS.systemReady()
    startApexServices(t);
    Looper.loop();
}
```
- Every service extends `SystemService` → `onStart()` (publishBinderService), `onBootPhase(phase)`.
- Boot phases: `PHASE_WAIT_FOR_DEFAULT_DISPLAY(100)` → `PHASE_WAIT_FOR_SENSOR_SERVICE(200)` → `PHASE_LOCK_SETTINGS_READY(480)` → `PHASE_SYSTEM_SERVICES_READY(500)` → `PHASE_DEVICE_SPECIFIC_SERVICES_READY(520)` → `PHASE_ACTIVITY_MANAGER_READY(550)` → `PHASE_THIRD_PARTY_APPS_CAN_START(600)` → `PHASE_BOOT_COMPLETED(1000)`.

---

### Q8. How do AVB / dm-verity work? What happens when the bootloader is unlocked?

- **AVB 2.0** (libavb in U-Boot): `vbmeta` holds hashes (for small partitions: boot, vendor_boot, dtbo) and **hashtree descriptors** (for large partitions: system, vendor...) + a signature.
- The kernel/init use **dm-verity** to verify each block on read (lazily) → runtime tampering is detected.
- `vbmeta_system` is chained with its own key (`avb_keys=/avb` in fstab) → the OEM can sign system independently of vendor.
- State: `ro.boot.verifiedbootstate` = green / yellow / orange (unlocked) / red.
- Rollback protection: `rollback_index` stored in TEE/RPMB.
- Disabling verity during development: `adb root; adb disable-verity; adb reboot` (userdebug/eng only, bootloader unlocked) → then `adb remount` with overlayfs.

BoardConfig example:
```make
BOARD_AVB_SYSTEM_ADD_HASHTREE_FOOTER_ARGS += --hash_algorithm sha256
BOARD_AVB_VENDOR_ADD_HASHTREE_FOOTER_ARGS += --hash_algorithm sha256
```

---

### Q9. How does bootconfig differ from the kernel cmdline?

- Since A12, `androidboot.*` parameters moved from the kernel cmdline to **bootconfig** (a file appended to the end of the initrd), readable at `/proc/bootconfig`.
- Why: the cmdline has a length limit, and the kernel doesn't need to know Android parameters.
- Project: `BOARD_KERNEL_CMDLINE += bootconfig`.
- init turns `androidboot.foo=bar` into `ro.boot.foo=bar`.

---

### Q10. How do you measure/optimize boot time?

- `adb shell dmesg | grep -i "init"`, `logcat -b events | grep boot_progress` (`boot_progress_start`, `boot_progress_system_run`, `boot_progress_pms_start`, `boot_progress_ams_ready`, `boot_progress_enable_screen`).
- `adb shell getprop | grep boottime` (`ro.boottime.init`, `ro.boottime.<service>`).
- **bootchart**: `adb shell touch /data/bootchart/enabled` → reboot → `system/core/init/grab-bootchart.sh`.
- **Perfetto boot trace** (A13+): config `persist.debug.perfetto.boottrace=1`.
- Optimizations: load modules in parallel / defer unneeded modules; use `erofs`; reduce preloaded classes; lazy HALs; avoid blocking `wait_for_prop`/`exec` in rc; full dexpreopt (avoid dex2oat at boot); parallel `ueventd`; fewer `class core` services.

---

### Q11. Recovery / fastbootd / BCB?

- **BCB** (Bootloader Control Block) is written in the `misc` partition: `boot-recovery`, args `--wipe_data`, `--update_package`...
- Recovery is a separate ramdisk (or inside boot with `BOARD_USES_RECOVERY_AS_BOOT`), running `/system/bin/recovery`.
- **fastbootd**: fastboot in userspace (inside recovery) to flash logical partitions in `super`.
- Commands: `adb reboot recovery`, `adb reboot fastboot` (fastbootd), `adb reboot bootloader` (U-Boot fastboot).

---

### Q12. When are HAL services started? If a HAL starts late, what does the client do?

- HALs are usually `class hal` → started by `class_start hal` in `on boot` of `init.rc`... (core HALs may use `class early_hal`).
- Clients use `AServiceManager_waitForService()` / `ServiceManager.waitForDeclaredService()` to **block and wait** (servicemanager knows the service is declared in the VINTF manifest).
- If the service is not declared in VINTF → `waitForDeclaredService` returns `null` immediately (no wait) — a very common bug.

Project example (`vendor.david.hardware.daviddevicecontrol-service.rc`):
```rc
on init
    start daviddevicecontrol

service daviddevicecontrol /vendor/bin/hw/vendor.david.hardware.daviddevicecontrol-service
    class hal
    user system
    group system
```
Senior remark: `on init start ...` starts earlier than `class hal` (because the LED must light up early to signal boot), but you should add `interface aidl vendor.david.hardware.daviddevicecontrol.IDavidDeviceControl/default` so init knows which interface this service serves (needed for lazy HALs & `ctl.interface_start`), and consider `class early_hal` instead of a manual trigger.
