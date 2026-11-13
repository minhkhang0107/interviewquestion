# Chương 03 — Build system: Soong (Android.bp), Make (Android.mk), Product config

## 3.1 Tổng quan pipeline build

```
source build/envsetup.sh
lunch KSTB2500-userdebug           # TARGET_PRODUCT=KSTB2500, TARGET_BUILD_VARIANT=userdebug
m / make -j
   │
   ├─ Kati: đọc product config (*.mk, BoardConfig.mk, Android.mk) → build-<product>.ninja
   ├─ Soong (Go, Blueprint): đọc Android.bp → build.ninja
   │     (androidmk bridge: Soong export module sang Make qua out/soong/Android-<product>.mk)
   └─ Ninja: thực thi combined ninja → out/target/product/KSTB2500/*.img
```
(Bazel/`BUILD` đang dần thay thế nhưng A14 vẫn chủ yếu Soong + Kati.)

---

### Q1. Khác nhau giữa Android.bp và Android.mk?

| Tiêu chí | Android.bp (Soong) | Android.mk (Make/Kati) |
|----------|--------------------|------------------------|
| Ngôn ngữ | khai báo (JSON-like), **không có if/else** | Makefile, có logic điều kiện |
| Điều kiện | `arch`, `target`, `product_variables`, `soong_config_module_type`, `select()` (A15) | `ifeq`, `$(shell ...)` |
| Tốc độ | nhanh, parse song song | chậm |
| Tình trạng | **khuyến nghị** — bắt buộc cho module mới | legacy, đang migrate |
| Namespace | `soong_namespace` | global |
| Convert | `androidmk Android.mk > Android.bp` | — |

**Ví dụ tương đương:**
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

### Q2. Các module type hay dùng trong Android.bp?

| Module type | Dùng cho |
|-------------|----------|
| `cc_binary`, `cc_library`, `cc_library_shared/static`, `cc_defaults` | native |
| `cc_prebuilt_library_shared`, `prebuilt_etc`, `prebuilt_firmware` | prebuilt |
| `java_library`, `java_sdk_library`, `android_library`, `android_app`, `android_app_import`, `runtime_resource_overlay` | Java/APK |
| `aidl_interface` | stable AIDL (sinh ra `-V1-ndk`, `-V1-java`, `-V1-cpp`, `-V1-rust`) |
| `hidl_interface` | HIDL (legacy) |
| `filegroup`, `genrule` | gom file, sinh code |
| `sh_binary`, `python_binary_host` | script |
| `apex` | module APEX |
| `soong_namespace`, `soong_config_module_type` | namespace, điều kiện theo product |

---

### Q3. Giải thích từng thuộc tính trong Android.bp của HAL DavidDeviceControl

```bp
cc_binary {
    name: "vendor.david.hardware.daviddevicecontrol-service",
    init_rc: ["vendor.david.hardware.daviddevicecontrol-service.rc"],   // cài vào /vendor/etc/init/
    vintf_fragments: ["manifest_vendor.david.hardware.daviddevicecontrol-service.xml"], // gộp vào VINTF manifest vendor
    vendor: true,                         // cài vào /vendor, link vendor variant
    relative_install_path: "hw",          // → /vendor/bin/hw/
    srcs: ["DavidDeviceControl.cpp", "DeviceControl.cpp", "Service.cpp"],
    cflags: ["-Wall", "-Werror"],
    shared_libs: [
        "libbase", "liblog", "libutils", "libcutils",
        "vendor.david.hardware.daviddevicecontrol-V1-ndk",   // lib sinh từ aidl_interface (NDK backend)
        "libbinder_ndk",                                   // LL-NDK, ABI ổn định
    ],
    static_libs: ["android.hardware.common-V2-ndk", "libaidlcommonsupport"],
}
```
Điểm phỏng vấn:
- `init_rc` + `vintf_fragments` giúp **module tự chứa** — không cần sửa `device.mk` để copy rc/manifest (thay cho `PRODUCT_COPY_FILES`).
- Tại sao dùng **NDK backend** (`-ndk`, `libbinder_ndk`) cho vendor HAL? → `libbinder` (cpp) **không ổn định ABI** và không có trong vendor (từ A12 libbinder cho vendor bị hạn chế). NDK backend là lựa chọn chuẩn cho AIDL HAL.
- `android.hardware.common-V2-ndk` + `libaidlcommonsupport` ở đây thực tế không dùng (include `NativeHandle.h` đã comment) → có thể bỏ.

---

### Q4. `soong_namespace` là gì? Lỗi "module not found / ambiguous" xử lý sao?

- Mặc định mọi module Soong cùng global namespace → trùng tên sẽ lỗi.
- `soong_namespace { imports: [...] }` trong Android.bp → module trong thư mục đó thuộc namespace riêng; chỉ thấy module của namespace import + global.
- Namespace chỉ được **export sang Make** (cài vào image) nếu có trong `PRODUCT_SOONG_NAMESPACES`.

Ví dụ project:
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
- Tham chiếu module ở namespace khác: `"//vendor/amlogic/common:droidlogic.software.core"`.
- Lỗi hay gặp: thêm `soong_namespace` nhưng quên `PRODUCT_SOONG_NAMESPACES` → module build được nhưng **không vào image**, hoặc `PRODUCT_PACKAGES` báo không tìm thấy.

---

### Q5. Product makefile: AndroidProducts.mk, `<product>.mk`, BoardConfig.mk, device.mk khác nhau thế nào?

| File | Vai trò |
|------|---------|
| `AndroidProducts.mk` | khai báo `PRODUCT_MAKEFILES`, `COMMON_LUNCH_CHOICES` (hiện trong `lunch`) |
| `KSTB2500.mk` (product) | `PRODUCT_NAME/DEVICE/BRAND/MODEL`, `inherit-product`, PRODUCT_PACKAGES… |
| `device.mk` | cấu hình phần cứng chung (HAL, manifest, overlay, properties) |
| `BoardConfig.mk` | biến `BOARD_*`/`TARGET_*`: arch, kernel, partition size, AVB, sepolicy dirs |
| `vendor/david/david.mk` | lớp customize của OEM (được `include` trong KSTB2500.mk) |

Ví dụ project:
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
Câu hỏi follow-up: *`inherit-product` vs `include` khác gì?* → `inherit-product` xử lý biến `PRODUCT_*` theo cơ chế kế thừa (append + ghi nhận nguồn, tránh trùng, cho phép override theo thứ tự ưu tiên); `include` chỉ chèn text Makefile, biến bị gán đè `:=` có thể mất giá trị trước đó.

---

### Q6. PRODUCT_PACKAGES, PRODUCT_COPY_FILES, PRODUCT_*_PROPERTIES?

```make
PRODUCT_PACKAGES += vendor.david.hardware.daviddevicecontrol-service DavidExtension   # module cần cài
PRODUCT_COPY_FILES += src:dst                                                     # copy thô (không qua module)
PRODUCT_COPY_FILES += \
    vendor/david/apps/DavidLauncher/operator_flags.xml:/system/etc/sysconfig/operator_flags.xml
PRODUCT_SYSTEM_PROPERTIES += ro.david.version=...     # → /system/build.prop
PRODUCT_VENDOR_PROPERTIES += ro.vendor.xxx=...       # → /vendor/build.prop
PRODUCT_PRODUCT_PROPERTIES += ...                    # → /product/etc/build.prop
DEVICE_MANIFEST_FILE += ...xml                       # VINTF device manifest
DEVICE_PRODUCT_COMPATIBILITY_MATRIX_FILE += ...xml   # FCM cho HAL vendor/product tự định nghĩa
```
- Ưu tiên `prebuilt_etc` + `PRODUCT_PACKAGES` thay cho `PRODUCT_COPY_FILES` (có dependency tracking, check ELF, không thể copy đè module).
- **Không** dùng `PRODUCT_COPY_FILES` để copy `.apk`/`.so` — build sẽ báo lỗi (`Prebuilt elf/apk files cannot be installed with PRODUCT_COPY_FILES`).
- Lưu ý trong project: `vendor/david/apps/DavidLauncher/operator_flags.xml:/system/etc/...` có `/` đầu đường dẫn đích — cách đúng là `$(TARGET_COPY_OUT_SYSTEM)/etc/...` hoặc `system/etc/...`.

---

### Q7. Build variant user / userdebug / eng khác gì?

| | eng | userdebug | user |
|--|-----|-----------|------|
| `ro.debuggable` | 1 | 1 | 0 |
| `adb root` | ✔ | ✔ | ✘ |
| SELinux | enforcing (có thể permissive) | enforcing | enforcing |
| dexpreopt | có thể tắt | có | có |
| Module `tags`/`*_DEBUG` | cài module eng+debug | cài debug | chỉ module chính thức |
| Dùng cho | dev | test/QA | release |

Trong `.bp`: `product_variables: { debuggable: { cflags: ["-DDEBUG"] } }`; trong `.mk`: `PRODUCT_PACKAGES_DEBUG`, `PRODUCT_PACKAGES_ENG`.

---

### Q8. `vendor: true`, `vendor_available`, `product_specific`, `system_ext_specific`, `odm_available`... cài vào đâu?

| Thuộc tính | Cài vào |
|------------|---------|
| (mặc định) | `/system` |
| `system_ext_specific: true` | `/system_ext` |
| `product_specific: true` | `/product` |
| `vendor: true` (= `soc_specific`) | `/vendor` |
| `device_specific: true` | `/odm` |
| `vendor_available: true` | có cả core + vendor variant |
| `recovery_available` / `ramdisk_available` / `vendor_ramdisk_available` | thêm variant recovery/ramdisk |

Ví dụ: `DavidExtension` dùng `system_ext_specific: true` + `privileged: true` → `/system_ext/priv-app/DavidExtension/`. Khi `privileged`, cần whitelist trong `privapp-permissions-*.xml` các permission signature|privileged nó dùng, nếu không **device không boot** (`Privileged permission ... not in privapp-permissions allowlist`) khi `ro.control_privapp_permissions=enforce`.

---

### Q9. Build nhanh 1 module & các lệnh hữu ích?

```bash
m vendor.david.hardware.daviddevicecontrol-service    # build 1 module + deps
mm / mma / mmm vendor/david/hals/DavidDeviceControl   # build theo thư mục
m vendorimage / systemimage / snod                   # build image
m <iface>-update-api / m <iface>-freeze-api          # AIDL API
m nothing                                            # chỉ chạy Soong/Kati để check cú pháp
adb root && adb remount && adb sync vendor           # đẩy nhanh lên máy (userdebug, verity off)
adb push out/target/product/KSTB2500/vendor/bin/hw/vendor.david.hardware.daviddevicecontrol-service /vendor/bin/hw/
get_build_var TARGET_PRODUCT; allmod | grep david; pathmod DavidExtension; outmod DavidExtension
bpfmt -w Android.bp                                  # format bp
```

---

### Q10. Lỗi build hay gặp & cách xử lý

| Lỗi | Nguyên nhân | Cách xử lý |
|-----|-------------|------------|
| `depends on undefined module` | sai tên / namespace chưa import | kiểm tra `soong_namespace`, `PRODUCT_SOONG_NAMESPACES` |
| `module "x" variant "vendor" depends on non-vendor` | vendor module link lib system | dùng lib có `vendor_available` / LL-NDK |
| `API dump for the current version of AIDL interface ... does not exist` / `has changed` | sửa .aidl mà không update API | `m <name>-update-api` (unfrozen) hoặc tạo version mới |
| `Missing dependencies` / `ELF prebuilt` | copy file .so bằng COPY_FILES | dùng `cc_prebuilt_library_shared` |
| `neverallow` violation | sepolicy vi phạm | sửa policy, không thể bypass |
| `checkvintf` fail | HAL trong manifest không có trong compatibility matrix (hoặc ngược lại) | thêm vào `DEVICE_PRODUCT_COMPATIBILITY_MATRIX_FILE`/manifest |
| `hidden API` / `platform_apis` | app gọi hidden API | `platform_apis: true` + `certificate: "platform"` hoặc dùng `sdk_version: "system_current"` |
