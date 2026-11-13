# Chapter 03 — Build System: Soong (Android.bp), Make (Android.mk), Product Config

## 3.1 Build pipeline overview

```
source build/envsetup.sh
lunch KSTB2500-userdebug           # TARGET_PRODUCT=KSTB2500, TARGET_BUILD_VARIANT=userdebug
m / make -j
   │
   ├─ Kati: reads product config (*.mk, BoardConfig.mk, Android.mk) → build-<product>.ninja
   ├─ Soong (Go, Blueprint): reads Android.bp → build.ninja
   │     (androidmk bridge: Soong exports modules to Make via out/soong/Android-<product>.mk)
   └─ Ninja: executes the combined ninja → out/target/product/KSTB2500/*.img
```
(Bazel/`BUILD` is gradually taking over, but A14 is still mostly Soong + Kati.)

---

### Q1. What's the difference between Android.bp and Android.mk?

| Criterion | Android.bp (Soong) | Android.mk (Make/Kati) |
|----------|--------------------|------------------------|
| Language | declarative (JSON-like), **no if/else** | Makefile, has conditional logic |
| Conditionals | `arch`, `target`, `product_variables`, `soong_config_module_type`, `select()` (A15) | `ifeq`, `$(shell ...)` |
| Speed | fast, parsed in parallel | slow |
| Status | **recommended** — mandatory for new modules | legacy, being migrated |
| Namespace | `soong_namespace` | global |
| Conversion | `androidmk Android.mk > Android.bp` | — |

**Equivalent example:**
```make
# Android.mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := hello
LOCAL_SRC_FILES := hello.cpp
LOCAL_SHARED_LIBRARIES := liblog libutils
LOCAL_CFLAGS := -Wall -Werror
LOCAL_VENDOR_MODULE := true
include $(BUILD_EXECUTABLE)
```
```bp
// Android.bp
cc_binary {
    name: "hello",
    srcs: ["hello.cpp"],
    shared_libs: ["liblog", "libutils"],
    cflags: ["-Wall", "-Werror"],
    vendor: true,
}
```

---

### Q2. Commonly used module types in Android.bp?

| Module type | Used for |
|-------------|----------|
| `cc_binary`, `cc_library`, `cc_library_shared/static`, `cc_defaults` | native |
| `cc_prebuilt_library_shared`, `prebuilt_etc`, `prebuilt_firmware` | prebuilts |
| `java_library`, `java_sdk_library`, `android_library`, `android_app`, `android_app_import`, `runtime_resource_overlay` | Java/APK |
| `aidl_interface` | stable AIDL (generates `-V1-ndk`, `-V1-java`, `-V1-cpp`, `-V1-rust`) |
| `hidl_interface` | HIDL (legacy) |
| `filegroup`, `genrule` | grouping files, code generation |
| `sh_binary`, `python_binary_host` | scripts |
| `apex` | APEX modules |
| `soong_namespace`, `soong_config_module_type` | namespaces, per-product conditionals |

---

### Q3. Explain each property in the DavidDeviceControl HAL's Android.bp

```bp
cc_binary {
    name: "vendor.david.hardware.daviddevicecontrol-service",
    init_rc: ["vendor.david.hardware.daviddevicecontrol-service.rc"],   // installed to /vendor/etc/init/
    vintf_fragments: ["manifest_vendor.david.hardware.daviddevicecontrol-service.xml"], // merged into the vendor VINTF manifest
    vendor: true,                         // installed to /vendor, links vendor variants
    relative_install_path: "hw",          // → /vendor/bin/hw/
    srcs: ["DavidDeviceControl.cpp", "DeviceControl.cpp", "Service.cpp"],
    cflags: ["-Wall", "-Werror"],
    shared_libs: [
        "libbase", "liblog", "libutils", "libcutils",
        "vendor.david.hardware.daviddevicecontrol-V1-ndk",   // lib generated from aidl_interface (NDK backend)
        "libbinder_ndk",                                   // LL-NDK, stable ABI
    ],
    static_libs: ["android.hardware.common-V2-ndk", "libaidlcommonsupport"],
}
```
Interview points:
- `init_rc` + `vintf_fragments` make the module **self-contained** — no need to edit `device.mk` to copy rc/manifest (replaces `PRODUCT_COPY_FILES`).
- Why use the **NDK backend** (`-ndk`, `libbinder_ndk`) for a vendor HAL? → `libbinder` (cpp) has **no stable ABI** and is not available to vendor (since A12 libbinder for vendor is restricted). The NDK backend is the standard choice for AIDL HALs.
- `android.hardware.common-V2-ndk` + `libaidlcommonsupport` are actually unused here (the `NativeHandle.h` include is commented out) → can be removed.

---

### Q4. What is `soong_namespace`? How do you fix "module not found / ambiguous" errors?

- By default all Soong modules share one global namespace → duplicate names fail.
- `soong_namespace { imports: [...] }` in an Android.bp → modules in that directory belong to their own namespace; they only see modules from imported namespaces + global.
- A namespace is **exported to Make** (installed into images) only if listed in `PRODUCT_SOONG_NAMESPACES`.

Project example:
```bp
// device/amlogic/KSTB2500/Android.bp
soong_namespace {
    imports: ["device/amlogic/common", "vendor/amlogic/common", "vendor/amlogic/KSTB2500"],
}
```
```make
# vendor/david/david.mk
PRODUCT_SOONG_NAMESPACES += \
    vendor/david/apps/DavidRecoveryQrTrigger \
    vendor/david/services/framework/davidextensionsvc
```
- Referencing a module in another namespace: `"//vendor/amlogic/common:droidlogic.software.core"`.
- Common mistake: adding `soong_namespace` but forgetting `PRODUCT_SOONG_NAMESPACES` → the module builds but **does not end up in the image**, or `PRODUCT_PACKAGES` reports it can't be found.

---

### Q5. Product makefiles: how do AndroidProducts.mk, `<product>.mk`, BoardConfig.mk, device.mk differ?

| File | Role |
|------|------|
| `AndroidProducts.mk` | declares `PRODUCT_MAKEFILES`, `COMMON_LUNCH_CHOICES` (shown in `lunch`) |
| `KSTB2500.mk` (product) | `PRODUCT_NAME/DEVICE/BRAND/MODEL`, `inherit-product`, PRODUCT_PACKAGES… |
| `device.mk` | common hardware config (HALs, manifest, overlays, properties) |
| `BoardConfig.mk` | `BOARD_*`/`TARGET_*` variables: arch, kernel, partition sizes, AVB, sepolicy dirs |
| `vendor/david/david.mk` | the OEM customization layer (`include`d from KSTB2500.mk) |

Project example:
```make
# AndroidProducts.mk
PRODUCT_MAKEFILES := $(LOCAL_DIR)/KSTB2500.mk
COMMON_LUNCH_CHOICES := KSTB2500-eng KSTB2500-user KSTB2500-userdebug

# KSTB2500.mk
$(call inherit-product, device/amlogic/common/products/mbox/product_mbox.mk)
$(call inherit-product, device/amlogic/$(PRODUCT_DIR)/device.mk)
PRODUCT_NAME := V000KM
PRODUCT_DEVICE := KSTB2500
PRODUCT_BRAND := DavidMedia
include vendor/david/david.mk
```
Follow-up: *`inherit-product` vs `include`?* → `inherit-product` handles `PRODUCT_*` variables via inheritance (append + records the source, avoids duplicates, allows override by priority order); `include` just pastes Makefile text, so a `:=` assignment can wipe out earlier values.

---

### Q6. PRODUCT_PACKAGES, PRODUCT_COPY_FILES, PRODUCT_*_PROPERTIES?

```make
PRODUCT_PACKAGES += vendor.david.hardware.daviddevicecontrol-service DavidExtension   # modules to install
PRODUCT_COPY_FILES += src:dst                                                     # raw copy (not through a module)
PRODUCT_COPY_FILES += \
    vendor/david/apps/DavidLauncher/operator_flags.xml:/system/etc/sysconfig/operator_flags.xml
PRODUCT_SYSTEM_PROPERTIES += ro.david.version=...     # → /system/build.prop
PRODUCT_VENDOR_PROPERTIES += ro.vendor.xxx=...       # → /vendor/build.prop
PRODUCT_PRODUCT_PROPERTIES += ...                    # → /product/etc/build.prop
DEVICE_MANIFEST_FILE += ...xml                       # VINTF device manifest
DEVICE_PRODUCT_COMPATIBILITY_MATRIX_FILE += ...xml   # FCM for vendor/product-defined HALs
```
- Prefer `prebuilt_etc` + `PRODUCT_PACKAGES` over `PRODUCT_COPY_FILES` (dependency tracking, ELF checks, cannot silently overwrite a module).
- **Do not** use `PRODUCT_COPY_FILES` to copy `.apk`/`.so` — the build errors out (`Prebuilt elf/apk files cannot be installed with PRODUCT_COPY_FILES`).
- Note in the project: `vendor/david/apps/DavidLauncher/operator_flags.xml:/system/etc/...` has a leading `/` in the destination — the correct form is `$(TARGET_COPY_OUT_SYSTEM)/etc/...` or `system/etc/...`.

---

### Q7. How do the build variants user / userdebug / eng differ?

| | eng | userdebug | user |
|--|-----|-----------|------|
| `ro.debuggable` | 1 | 1 | 0 |
| `adb root` | ✔ | ✔ | ✘ |
| SELinux | enforcing (may be permissive) | enforcing | enforcing |
| dexpreopt | may be disabled | yes | yes |
| Module `tags`/`*_DEBUG` | installs eng+debug modules | installs debug | official modules only |
| Used for | development | test/QA | release |

In `.bp`: `product_variables: { debuggable: { cflags: ["-DDEBUG"] } }`; in `.mk`: `PRODUCT_PACKAGES_DEBUG`, `PRODUCT_PACKAGES_ENG`.

---

### Q8. `vendor: true`, `vendor_available`, `product_specific`, `system_ext_specific`, `odm_available`... where do they install?

| Property | Installs to |
|------------|---------|
| (default) | `/system` |
| `system_ext_specific: true` | `/system_ext` |
| `product_specific: true` | `/product` |
| `vendor: true` (= `soc_specific`) | `/vendor` |
| `device_specific: true` | `/odm` |
| `vendor_available: true` | both core + vendor variants |
| `recovery_available` / `ramdisk_available` / `vendor_ramdisk_available` | extra recovery/ramdisk variants |

Example: `DavidExtension` uses `system_ext_specific: true` + `privileged: true` → `/system_ext/priv-app/DavidExtension/`. When `privileged`, the signature|privileged permissions it uses must be allowlisted in `privapp-permissions-*.xml`, otherwise **the device won't boot** (`Privileged permission ... not in privapp-permissions allowlist`) when `ro.control_privapp_permissions=enforce`.

---

### Q9. Building a single module quickly & useful commands?

```bash
m vendor.david.hardware.daviddevicecontrol-service    # build 1 module + deps
mm / mma / mmm vendor/david/hals/DavidDeviceControl   # build by directory
m vendorimage / systemimage / snod                   # build images
m <iface>-update-api / m <iface>-freeze-api          # AIDL API
m nothing                                            # only run Soong/Kati to check syntax
adb root && adb remount && adb sync vendor           # push quickly to the device (userdebug, verity off)
adb push out/target/product/KSTB2500/vendor/bin/hw/vendor.david.hardware.daviddevicecontrol-service /vendor/bin/hw/
get_build_var TARGET_PRODUCT; allmod | grep david; pathmod DavidExtension; outmod DavidExtension
bpfmt -w Android.bp                                  # format bp
```

---

### Q10. Common build errors & how to fix them

| Error | Cause | Fix |
|-----|-------------|------------|
| `depends on undefined module` | wrong name / namespace not imported | check `soong_namespace`, `PRODUCT_SOONG_NAMESPACES` |
| `module "x" variant "vendor" depends on non-vendor` | vendor module links a system lib | use a lib with `vendor_available` / LL-NDK |
| `API dump for the current version of AIDL interface ... does not exist` / `has changed` | .aidl edited without updating the API | `m <name>-update-api` (unfrozen) or create a new version |
| `Missing dependencies` / `ELF prebuilt` | .so copied with COPY_FILES | use `cc_prebuilt_library_shared` |
| `neverallow` violation | sepolicy violation | fix the policy, cannot be bypassed |
| `checkvintf` fail | HAL in manifest not in the compatibility matrix (or vice versa) | add to `DEVICE_PRODUCT_COMPATIBILITY_MATRIX_FILE`/manifest |
| `hidden API` / `platform_apis` | app calls hidden APIs | `platform_apis: true` + `certificate: "platform"`, or use `sdk_version: "system_current"` |
