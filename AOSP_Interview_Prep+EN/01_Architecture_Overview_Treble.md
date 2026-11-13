# Chapter 01 — Android Architecture Overview & Project Treble

## 1.1 Layer diagram

```
┌──────────────────────────────────────────────────────────────┐
│ Apps (system/priv-app, product/app, data/app)                │
├──────────────────────────────────────────────────────────────┤
│ Framework Java API (android.jar) — XxxManager                │  Context.getSystemService()
├──────────────────────────────────────────────────────────────┤
│ system_server: AMS/ATMS, PMS, WMS, PowerMS, InputMS, ...     │  Binder (/dev/binder)
│ Native services: surfaceflinger, audioserver, cameraserver   │
├──────────────────────────────────────────────────────────────┤
│ HAL (AIDL / HIDL) — separate processes in /vendor/bin/hw/    │  Binder (/dev/binder, /dev/hwbinder)
├──────────────────────────────────────────────────────────────┤
│ Linux kernel (GKI) + vendor modules (.ko) — drivers, sysfs   │
└──────────────────────────────────────────────────────────────┘
```

---

### Q1. What is Project Treble? Why did Google create it?

**Answer:**
- Treble (Android 8.0) separates the **framework (system)** from the **vendor implementation** through a **stable interface** (HIDL, now stable AIDL) called the *Vendor Interface*.
- Goal: OEMs can **update the system partition** (new Android version) **without the SoC vendor (Amlogic, Qualcomm...) rebuilding the vendor partition**.
- Main components:
  - **Separate partitions**: `system`, `system_ext`, `product` (Google/OEM) vs `vendor`, `odm` (SoC/ODM).
  - **Stable + versioned HAL interfaces** (HIDL `@1.0`, AIDL `V1, V2...`).
  - **VINTF** (Vendor Interface Object): manifest + compatibility matrix, checked against each other at build time and during OTA.
  - **VNDK** (Vendor NDK): the set of system libraries vendor code is allowed to link against.
  - **SELinux split policy**: `plat_*` vs `vendor_*`.
  - **GSI** (Generic System Image): runs on any Treble-compliant device → used for VTS.

**Example in this project:** the HAL `vendor.david.hardware.daviddevicecontrol` lives entirely on `/vendor` (`vendor: true` in Android.bp), while the client `DavidExtension` lives in `system_ext` → they talk through stable AIDL `@VintfStability`.

---

### Q2. Name the partitions and what goes in each.

| Partition | Owner | Contents |
|-----------|-------|----------|
| `boot` | Google (GKI) | kernel + (since A13) no generic ramdisk if `init_boot` exists |
| `init_boot` | Google | generic ramdisk (first stage init) — A13+ |
| `vendor_boot` | SoC vendor | vendor ramdisk (kernel modules, first-stage fstab), DTB |
| `vbmeta`, `vbmeta_system` | — | AVB metadata (hash/hashtree, public key) |
| `system` | Google/OEM | framework, `/system/bin`, `/system/framework` |
| `system_ext` | OEM | system extensions (OEM apps/services using hidden APIs) |
| `product` | OEM | apps, overlays, per-product config |
| `vendor` | SoC | HALs, vendor libs, vendor init rc, vendor sepolicy |
| `odm` | ODM | ODM customizations on top of vendor |
| `vendor_dlkm`, `system_dlkm`, `odm_dlkm` | | kernel modules (.ko) |
| `super` | | dynamic partition containing system/vendor/product/odm/... |
| `userdata`, `metadata`, `misc` | | data, metadata-encryption keys, BCB for recovery |

**Real example** `device/amlogic/KSTB2500/BoardConfig.mk`:
```make
BOARD_SUPER_PARTITION_GROUPS := amlogic_dynamic_partitions
BOARD_AMLOGIC_DYNAMIC_PARTITIONS_PARTITION_LIST := system vendor product odm
BOARD_AMLOGIC_DYNAMIC_PARTITIONS_PARTITION_LIST += system_ext
BOARD_AMLOGIC_DYNAMIC_PARTITIONS_PARTITION_LIST += vendor_dlkm
```

---

### Q3. What is a dynamic partition (super)? Benefits?

- Since Android 10: logical partitions (system, vendor, ...) live inside one physical `super` partition, managed by **liblp** (metadata at the start of `super`) and mapped with **device-mapper** (`dm-linear`).
- Benefits: partitions can be resized via OTA without changing the partition table; saves space (Virtual A/B needs only 1 copy + COW snapshot).
- Marked with the `logical` flag in fstab:
```
system  /system  erofs  ro  wait,slotselect,avb=vbmeta_system,logical,first_stage_mount
```
- Tools: `lpdump`, `lpmake`, `lpunpack`, `fastboot getvar all` (only userspace fastbootd can flash logical partitions).

---

### Q4. How do A/B and Virtual A/B differ?

| | A/B (seamless) | Virtual A/B (A11+) |
|--|----------------|--------------------|
| Storage | 2 copies of every partition (`_a`, `_b`) | 1 copy of dynamic partitions + **COW snapshot** |
| OTA | update_engine writes to the inactive slot | writes into COW (snapuserd), merged after a successful boot |
| Rollback | switch slot | discard snapshot |
| Compression | — | `virtual_ab_ota/compression.mk` (dm-user + snapuserd) |

The project uses compressed Virtual A/B:
```make
# device/amlogic/KSTB2500/KSTB2500.mk
$(call inherit-product, $(SRC_TARGET_DIR)/product/virtual_ab_ota/compression.mk)
$(call inherit-product, $(SRC_TARGET_DIR)/product/virtual_ab_ota/launch_with_vendor_ramdisk.mk)
```
Debug commands: `snapshotctl dump`, `update_engine_client --status`, `bootctl get-current-slot`.

---

### Q5. What is VNDK? Distinguish LL-NDK, VNDK, VNDK-SP, vendor_available, vendor: true

- **LL-NDK**: system libs vendor may link against, ABI-stable (libc, libm, libdl, liblog, libEGL, libvndksupport, `libbinder_ndk`...).
- **VNDK**: core system libs that have a separate copy for vendor (`/apex/com.android.vndk.vXX/`), e.g. `libbase`, `libutils`, `libcutils`...
- **VNDK-SP**: VNDK libs usable inside a *same-process HAL* (e.g. graphics mapper, vendor libEGL running inside the app process).
- `vendor_available: true`: the module is built as **2 variants** (core + vendor).
- `vendor: true`: built only for vendor, installed to `/vendor`.
- **Note for A14**: VNDK is **deprecated** (removed completely in A15). Vendor libs should be self-packaged or use LL-NDK/AIDL. Mentioning this in an interview earns points.

---

### Q6. How does system_ext differ from product? When does a module go where?

- `system_ext`: modules **tightly coupled to system** (may use hidden APIs/`platform_apis`, signed with the platform key, sharedUserId system). There is no stable interface with system → must be updated together with system.
- `product`: per-SKU apps/overlays/config, only **public/system APIs** (since A11, `PRODUCT_ENFORCE_PRODUCT_PARTITION_INTERFACE`).
- `vendor/odm`: SoC/HAL code only; **must not** use the Java framework directly.

**Example:** `DavidExtension` is `system_ext_specific: true`, `privileged: true`, `certificate: "platform"`, `platform_apis: true` → the right place, because it uses `android.os.ServiceManager` (hidden API).

---

### Q7. How do GSI and VTS/CTS relate to Treble?

- **CTS**: tests public APIs (app-level compatibility).
- **VTS**: tests HALs + kernel (the vendor interface) — run on a **GSI** to prove the vendor partition satisfies Treble.
- **CTS-on-GSI**: run CTS with system = GSI + the OEM's vendor.
- **STS** (security), **GTS** (Google Mobile Services), **TVTS** (Android TV — mentioned in `david.mk`: "energy modes - TVTS, CTSV fail").

---

### Q8. What is GKI (Generic Kernel Image)?

- Since A12: the core kernel is built by Google (**GKI**); SoC vendors only ship **vendor modules** (`.ko`) through a stable **KMI** (Kernel Module Interface).
- Modules are loaded from the `vendor_boot` ramdisk (first stage) or `vendor_dlkm` (second stage), listed in `modules.load`.
- The project also loads modules from init rc:
```
on init
    insmod /vendor/lib/modules/mali.ko
```
- Follow-up question: *"Why use `modules.load` instead of `insmod` in rc?"* → ordering/dependencies are handled by modprobe (modules.dep), modules load in parallel, boot time drops, and you avoid SELinux `module_load` denials.
