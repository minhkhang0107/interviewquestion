# Chapter 07 — SELinux in Android

## 7.1 Concepts

- **MAC** (Mandatory Access Control) on top of DAC (uid/gid/mode). Default is **deny all**; only what an `allow` rule permits is allowed.
- Each process has a **domain** (`u:r:system_server:s0`), each object has a **type** (`u:object_r:system_file:s0`).
- Rule: `allow <source_domain> <target_type>:<class> { permissions };`
- Modes: `enforcing` (block + log) / `permissive` (log only). Android user builds must be enforcing.
- Split policy (Treble): `system/sepolicy/public` (API for vendor), `private` (platform only), `vendor` (`BOARD_SEPOLICY_DIRS`), `system_ext`/`product` (`SYSTEM_EXT_PUBLIC/PRIVATE_SEPOLICY_DIRS`, `PRODUCT_*`).

---

### Q1. Important sepolicy files and their roles?

| File | Role | Example |
|------|---------|-------|
| `*.te` | declares types/domains and allow rules | `daviddevicecontrol.te` |
| `file_contexts` | labels for files on the filesystem (ro partitions labeled at build time) | `/vendor/bin/hw/vendor.david...-service u:object_r:daviddevicecontrol_exec:s0` |
| `genfs_contexts` | labels for `/sys`, `/proc` (filesystems without xattr support) | `genfscon sysfs /devices/platform/leds u:object_r:sysfs_leds:s0` |
| `service_contexts` | labels for service names in servicemanager | `vendor.david...IDavidDeviceControl/default u:object_r:daviddevicecontrol_service:s0` |
| `hwservice_contexts` | labels for HIDL services | `vendor.amlogic.hardware.systemcontrol::ISystemControl u:object_r:hal_systemcontrol_hwservice:s0` |
| `vndservice_contexts` | labels for services on vndbinder | |
| `property_contexts` | labels for properties | `persist.vendor.sc.log.level u:object_r:vendor_sc_prop:s0` |
| `seapp_contexts` | assigns domains to apps by uid/seinfo/package | `user=system seinfo=platform domain=system_app type=system_app_data_file` |
| `mac_permissions.xml` | maps certificate → seinfo | platform cert → `seinfo=platform` |

---

### Q2. Write a complete policy for a new AIDL HAL (real project example)

`vendor/david/sepolicy/vendor/daviddevicecontrol.te`:
```
type daviddevicecontrol, domain;
type daviddevicecontrol_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(daviddevicecontrol)             # init execs the *_exec file → transitions into this domain

allow daviddevicecontrol servicemanager:binder { call transfer };
allow daviddevicecontrol daviddevicecontrol_service:service_manager { add };

allow daviddevicecontrol sysfs_leds:dir { search };
allow daviddevicecontrol sysfs_leds:file { getattr open read write };
allow daviddevicecontrol sysfs_leds:lnk_file { read };
allow daviddevicecontrol sysfs_remote:dir { search };
allow daviddevicecontrol sysfs_remote:file { getattr open read write };
allow daviddevicecontrol sysfs_remote:lnk_file { read };
allow daviddevicecontrol sysfs:file { read write open };
```
`service.te`:
```
type daviddevicecontrol_service, hal_service_type, service_manager_type;
```
`service_contexts`, `file_contexts`: as in the Q1 table.

`system_server.te` (client):
```
allow system_server daviddevicecontrol_service:service_manager { find };
binder_call(system_server, daviddevicecontrol)
allow system_app daviddevicecontrol_service:service_manager { find };
binder_call(system_app, daviddevicecontrol)
```
And in `david.mk`: `BOARD_SEPOLICY_DIRS += vendor/david/sepolicy/vendor`.

**"Senior" version (macros, least privilege):**
```
type daviddevicecontrol, domain;
type daviddevicecontrol_exec, exec_type, vendor_file_type, file_type;
init_daemon_domain(daviddevicecontrol)

binder_use(daviddevicecontrol)                                   # call servicemanager + use binder
add_service(daviddevicecontrol, daviddevicecontrol_service)       # add + neverallow check
allow daviddevicecontrol sysfs_leds:dir r_dir_perms;
allow daviddevicecontrol sysfs_leds:file rw_file_perms;
# REMOVE: allow daviddevicecontrol sysfs:file { read write open };   → too broad
```
→ The `sysfs:file` rule allows writing **any** sysfs node without its own label → violates least privilege; the right way is to label the specific node via `genfs_contexts` and allow only that type.

Client: `binder_call(system_app, daviddevicecontrol)` is enough instead of something broader; consider creating **attributes** `hal_daviddevicecontrol_client/server` with `hal_attribute(daviddevicecontrol)` + `hal_server_domain` / `hal_client_domain` like AOSP HALs.

---

### Q3. Reading and handling an AVC denial

```
avc: denied { write } for comm="vendor.david.har" name="brightness" dev="sysfs" ino=1234
  scontext=u:r:daviddevicecontrol:s0 tcontext=u:object_r:sysfs:s0 tclass=file permissive=0
```
Read it as: **who** (`scontext`, domain `daviddevicecontrol`) wants to **do what** (`write`) on **what** (`tcontext` type `sysfs`, class `file`, name `brightness`).

Correct handling steps:
1. Ask: **should** this permission exist at all? (don't run audit2allow blindly)
2. If the target is a generic type (`sysfs`, `device`, `default_prop`, `default_android_service`) → **create a dedicated type** and label correctly:
   ```
   # vendor/david/sepolicy/vendor/file.te
   type sysfs_david_led, sysfs_type, fs_type;
   # genfs_contexts
   genfscon sysfs /devices/platform/leds/leds/red_led u:object_r:sysfs_david_led:s0
   ```
3. Add the minimal rule: `allow daviddevicecontrol sysfs_david_led:file rw_file_perms;`
4. Rebuild with `m selinux_policy` or `m vendorimage`, check `ls -Z /sys/...`.
- Useful commands:
```bash
adb shell dmesg | grep avc ; adb logcat -b all | grep avc
audit2allow -p out/target/product/KSTB2500/root/sepolicy -i avc.txt   # suggestions (reference only)
adb shell getenforce ; adb shell setenforce 0    # userdebug only, for diagnosis only
adb shell ls -Z /vendor/bin/hw/ ; adb shell ps -AZ | grep david
adb shell restorecon -R /data/vendor/xxx
```

---

### Q4. What is neverallow? Can it be bypassed?

- `neverallow` is a compile-time assertion: if any policy allows something forbidden → **the build fails**. It is also in CTS (`SELinuxNeverallowRulesTest`) → cannot be bypassed on a certified device.
- Famous neverallows:
  - Vendor domains may not `binder_call` into arbitrary system domains except via HALs/permitted paths (`coredomain` vs non-coredomain).
  - Apps may not `add` services to servicemanager.
  - Nobody except init may write `default_prop`...
  - Vendor processes may not `execute` files on `/system` (with a few exceptions).
- How to handle: redesign (change types, use a HAL as a bridge, use a property with its own label).

---

### Q5. coredomain vs vendor domain? Why couldn't vendor use `/dev/binder` freely (before A11)?

- `coredomain`: domains belonging to the platform (system, system_ext). Vendor domains do not have this attribute.
- Treble neverallow: vendor ↔ core may only talk via **stable HAL interfaces**. Before A11 vendor used `/dev/vndbinder` for internal vendor IPC and `/dev/hwbinder` for HALs. Since A11, AIDL HALs with `@VintfStability` let vendor use `/dev/binder`, but servicemanager **checks the binder's stability**.

---

### Q6. How are app domains assigned? (`system_app`, `platform_app`, `priv_app`, `untrusted_app`)

- Based on `seapp_contexts`: uid + seinfo (from the cert in `mac_permissions.xml`) + isPrivApp + targetSdk.
- `DavidExtension` (`sharedUserId=android.uid.system`, platform-signed) → domain **`system_app`** → that's why the project grants `allow system_app daviddevicecontrol_service:service_manager find`.
- Platform-signed but not `uid.system` → `platform_app`. Privileged but not platform-signed → `priv_app`.

---

### Q7. Adding a new vendor property with its own label?

```
# property.te
vendor_internal_prop(vendor_david_led_prop)     # A12+ macros: vendor_internal/restricted/public
# property_contexts
persist.vendor.david.led.   u:object_r:vendor_david_led_prop:s0 prefix string
# allow
set_prop(daviddevicecontrol, vendor_david_led_prop)
get_prop(system_app, vendor_david_led_prop)     # needs vendor_restricted_prop if system reads it
```
- `vendor_internal_prop`: vendor-only; `vendor_restricted_prop`: system can read; `vendor_public_prop`: system can read/write. (Matching `system_internal/restricted/public_prop` on the platform side.)

---

### Q8. Debugging a process that won't start because of SELinux?

- init logs: `init: cannot execv('/vendor/bin/hw/xxx'): Permission denied` or `Could not get process context ...`.
- Missing `file_contexts` → the file is labeled `vendor_file` → init can't transition → `avc: denied { execute_no_trans }`, or the service runs in the `init`/`vendor_init` domain (blocked by neverallow in A14: init refuses to start a service without a domain: *"Service xxx does not have a SELinux domain defined"*).
- Check: `ls -Z /vendor/bin/hw/<binary>` must show `daviddevicecontrol_exec`.
