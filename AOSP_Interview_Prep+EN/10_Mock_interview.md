# Chapter 10 — Combined Mock Interview

## Part A — Rapid-fire (answer in ≤ 30 seconds)

| # | Question | Short answer |
|---|---------|--------------|
| 1 | PID of init? | 1 |
| 2 | Parent process of system_server? | zygote (zygote64) |
| 3 | UID of system_server / shell / root? | 1000 / 2000 / 0 |
| 4 | First app after boot? | Launcher (HOME); on Android TV, the TV Launcher/DavidLauncher |
| 5 | Property that signals boot completion? | `sys.boot_completed=1`, `dev.bootcomplete=1` |
| 6 | Binder buffer limit? | ~1MB/process (oneway uses ≤ ½) |
| 7 | servicemanager's handle? | 0 |
| 8 | The 3 binder devices? | `/dev/binder`, `/dev/hwbinder`, `/dev/vndbinder` |
| 9 | Commands to list HALs? | `lshal`, `service list`, `dumpsys -l` |
| 10 | AIDL HAL instance name format? | `<package>.<IFace>/<instance>`, e.g. `vendor.david.hardware.daviddevicecontrol.IDavidDeviceControl/default` |
| 11 | File declaring the HALs vendor provides? | VINTF device manifest (`/vendor/etc/vintf/manifest*`) |
| 12 | Can `ro.*` be set again? | No (except before it is set for the first time) |
| 13 | Vendor property prefixes? | `vendor.`, `ro.vendor.`, `persist.vendor.` |
| 14 | Command to freeze AIDL? | `m <name>-freeze-api` |
| 15 | Domain of a platform-signed app with uid system? | `system_app` |
| 16 | Macro for a HAL daemon started by init? | `init_daemon_domain(domain)` |
| 17 | system_server watchdog timeout? | 60s (dump at 30s) |
| 18 | Input ANR timeout? | 5s |
| 19 | Partition holding the generic ramdisk on A13+? | `init_boot` |
| 20 | The project's `PRODUCT_SHIPPING_API_LEVEL`? | 34 (Android 14) |
| 21 | Converting Android.mk to bp? | the `androidmk` tool |
| 22 | Variable to add vendor sepolicy? | `BOARD_SEPOLICY_DIRS` |
| 23 | Variable to put a Soong namespace into the image? | `PRODUCT_SOONG_NAMESPACES` |
| 24 | `getService` vs `checkService`? | C++ `getService` waits ~5s, `checkService` returns immediately; **Java** `ServiceManager.getService` does not wait (returns null), use `waitForService` to wait |
| 25 | `waitForService` vs `waitForDeclaredService`? | Declared: waits only if the service is declared in VINTF, otherwise returns null immediately |

---

## Part B — Scenario questions

### B1. "The device bootloops after you added a new HAL. What do you do?"
1. Connect UART (STBs always have one) / `adb logcat` if adb comes up before the crash.
2. Look at init logs: is a service crashing repeatedly? Is it `critical`? → `init: Service 'xxx' (pid N) received signal 6`.
3. If system_server crashes: look for `FATAL EXCEPTION IN SYSTEM PROCESS` — maybe a client got null from the HAL and hit an NPE.
4. Check `checkvintf` at boot: `init: ... VINTF compatibility check failed` → **the framework refuses to boot** on a manifest/matrix mismatch (with A/B, the slot can be rolled back).
5. SELinux: an `avc: denied` makes an important service fail. Try a userdebug build + `androidboot.selinux=permissive` (via bootconfig) to narrow it down.
6. Fix, rebuild, reflash `vendor` (fastbootd) or `adb sync`.

### B2. "Apps calling your Manager occasionally get a 5s ANR."
- Get the ANR trace: the main thread is in `BinderProxy.transactNative` → a synchronous call into system_server.
- In system_server: the binder thread is waiting for `mLock` or calling a slow HAL (slow sysfs/I2C writes).
- Fix: make the API async (oneway + callback), shrink lock scope, cache state in system_server so `get*()` doesn't go down to the HAL, add timeouts in the HAL, move heavy work to a dedicated `Handler`.

### B3. "After an OTA, the old vendor HAL is incompatible with the new framework."
- Explain VINTF: the new framework has a higher FCM version; if the vendor `target-level` is too old (deprecated) → the OTA is blocked.
- AIDL: the new framework uses `-V2` but the HAL only has V1 → the framework must handle `getInterfaceVersion()` and fall back; the FCM declares `<version>1-2</version>`.

### B4. "Adding a new remote key (e.g. a Netflix key): which layers does it go through?"
1. Kernel: the IR/BT HID driver emits `EV_KEY` with a scancode → `/dev/input/eventX`.
2. `InputReader` (native, inside system_server) reads through EventHub and maps scancode → keycode using the **`.kl`** (`/vendor/usr/keylayout/<Vendor>_<Product>.kl`, project: `vendor/david/keylayout/`), `.kcm` for characters.
3. If the keycode doesn't exist in Android → add it to `KeyEvent.java`, `attrs.xml`, `InputEventLabels.h` (or reuse an existing one like `KEYCODE_PROG_RED`...).
4. `InputDispatcher` → `PhoneWindowManager.interceptKeyBeforeQueueing/Dispatching` (global keys) → focused app.
5. Android TV global keys: `com.android.internal.R.xml.global_keys` (the project has `DavidFrameworkOverlay` — "Global key") → sends the `GLOBAL_BUTTON` broadcast to the specified component.

### B5. "The product manager wants a property `persist.david.first_start_oem_app` that the OEM app can read/write. How do you do it the Treble way?"
- A property set from `PRODUCT_SYSTEM_PROPERTIES` → lives in `/system/build.prop`.
- Needs `property_contexts` for the `persist.david.` prefix in **system_ext** sepolicy (since it's a system property), its own type, `set_prop(system_app, david_oem_prop)`.
- Otherwise the default label is `default_prop` → the app can't write it (neverallow), and `vts_treble_sys_prop_test` may fail because of the unknown namespace.

### B6. "You need to run a shell script at boot as root (e.g. `led.sh`, `wifi.sh` in the project). How do you assess it?"
- It works (`user root`, `oneshot`, `disabled`, property trigger), but:
  - It needs its own SELinux domain for the script (`vendor_shell` / `*_exec` + `init_daemon_domain`), otherwise it gets blocked.
  - Security risk (root), hard to control; better to replace it with `write` in rc (init does it directly), or a HAL/daemon running as a restricted user — exactly why `DavidDeviceControl` exists.

### B7. "Feature design: a system service that counts TV viewing time per channel and uploads it to a server daily."
- Data source: TIF (`TvInputManager` session events) / `MediaSession` → a service in system_server or a privileged app (prefer an app to avoid touching the framework).
- Storage: Room/SQLite in `/data/system_ce/` or the app's data; per user.
- Upload: `JobScheduler` (network constraint, idle), backoff, no unbounded wakelocks.
- Permissions: a signature permission for the API that reads statistics; privacy (opt-in).
- Debugging: `dumpsys`, `cmd` shell commands.

---

## Part C — Deep-dive questions (senior/lead)

1. **How does binder priority inheritance work?** → The driver raises the server thread's priority (nice/RT) to match the client (`FLAT_BINDER_FLAG_INHERIT_RT`, the node's `min_priority`), avoiding priority inversion (important for audio/real-time HALs).
2. **Why does system_server use big locks (AMS `this`, WMS `mGlobalLock`)? Consequences?** → Simplifies correctness but causes contention; Google is gradually splitting them (ATMS out of AMS, `mProcLock`), using lock-free `ProcessRecord` state.
3. **What is an APEX, how does it differ from an APK?** → A native+Java module package mounted at `/apex/<name>` (loop + dm-verity), updated via Play (Mainline): ART, Conscrypt, media, tzdata, VNDK...
4. **What is a linker namespace?** → The Bionic linker isolates libraries between system/vendor/apex (`ld.config.txt` generated by `linkerconfig`): vendor processes don't see system libs outside LL-NDK/VNDK → `library "libfoo.so" not found` even though the file exists.
5. **RRO (Runtime Resource Overlay)?** → Overrides framework/app resources without changing source; `runtime_resource_overlay` in bp, installed in `/product/overlay` or `/vendor/overlay`; project examples `DroidATVTvSettingsResOverlay`, `DavidFrameworkOverlay`.
6. **`privapp-permissions` vs `default-permissions`?** → privapp: allowlist of **privileged permissions** for priv-apps (mandatory, otherwise the device won't boot in enforce mode). default-permissions: **auto-grants runtime permissions** to apps at install (project: `default-launcher-permissions.xml`, `default-oad-permissions.xml`).
7. **Why must vendor HALs not use `libbinder` (cpp backend)?** → libbinder has no stable ABI; vendor must use `libbinder_ndk` (LL-NDK) so system updates don't break vendor.
8. **Explain `dexpreopt`.** → AOT-compiles dex → odex/vdex/art at build time (`/system/framework/oat`, `<app>/oat/`), reducing boot/first-launch time; configured by `WITH_DEXPREOPT`, `PRODUCT_DEX_PREOPT_*`; the boot image (`boot.art`) for framework classes.
9. **system_server crash loop after an update: what rescue mechanisms exist?** → `RescueParty`: reset settings → ... → boot into recovery offering a factory reset; `PackageWatchdog` rolls back APEX/mainline.
10. **Describe how AMS starts an Activity in a process that doesn't exist yet.** → `startActivity` → ATMS → `ActivityStarter` → no process → `AMS.startProcessLocked` → `ZygoteProcess.start` (socket) → zygote forks → `ActivityThread.main` → `attachApplication` → AMS `bindApplication` → ATMS `realStartActivityLocked` → `ClientTransaction` (`LaunchActivityItem`) → `onCreate`.

---

## Part D — Self-assessment checklist before the interview

- [ ] Can draw the boot flow and name the source file for each step (`first_stage_init.cpp`, `init.cpp`, `ZygoteInit.java`, `SystemServer.java`).
- [ ] Can hand-write `aidl_interface`, the HAL `cc_binary`, rc, manifest, FCM, sepolicy for a new HAL (Chapter 05 Q3).
- [ ] Can hand-write the steps to add a system service + Manager (Chapter 06 Q1).
- [ ] Can explain one binder transaction from proxy → driver → stub (Chapter 04 Q2).
- [ ] Can read an avc denial and propose a minimal fix (Chapter 07 Q3).
- [ ] Can tell the DavidDeviceControl case study + 3 things you'd improve (Chapter 09).
- [ ] Clearly distinguish: HIDL/AIDL, servicemanager/hwservicemanager/vndservicemanager, system/system_ext/product/vendor/odm, user/userdebug/eng.
