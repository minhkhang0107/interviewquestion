# Chương 01 — Kiến trúc tổng quan Android & Project Treble

## 1.1 Sơ đồ tầng

```
┌──────────────────────────────────────────────────────────────┐
│ Apps (system/priv-app, product/app, data/app)                │
├──────────────────────────────────────────────────────────────┤
│ Framework Java API (android.jar) — XxxManager                │  Context.getSystemService()
├──────────────────────────────────────────────────────────────┤
│ system_server: AMS/ATMS, PMS, WMS, PowerMS, InputMS, ...     │  Binder (/dev/binder)
│ Native services: surfaceflinger, audioserver, cameraserver   │
├──────────────────────────────────────────────────────────────┤
│ HAL (AIDL / HIDL) — process riêng trên /vendor/bin/hw/       │  Binder (/dev/binder, /dev/hwbinder)
├──────────────────────────────────────────────────────────────┤
│ Linux kernel (GKI) + vendor modules (.ko) — driver, sysfs     │
└──────────────────────────────────────────────────────────────┘
```

---

### Q1. Project Treble là gì? Tại sao Google làm Treble?

**Trả lời:**
- Treble (Android 8.0) tách **framework (system)** khỏi **vendor implementation** bằng một **interface ổn định** (HIDL, nay là stable AIDL) gọi là *Vendor Interface*.
- Mục tiêu: OEM có thể **update system partition** (Android version mới) mà **không cần SoC vendor (Amlogic, Qualcomm...) build lại vendor partition**.
- Thành phần chính:
  - **Partition tách biệt**: `system`, `system_ext`, `product` (Google/OEM) vs `vendor`, `odm` (SoC/ODM).
  - **HAL interface ổn định + versioned** (HIDL `@1.0`, AIDL `V1, V2...`).
  - **VINTF** (Vendor Interface Object): manifest + compatibility matrix để kiểm tra khớp nhau lúc build và lúc OTA.
  - **VNDK** (Vendor NDK): tập thư viện system mà vendor được phép link.
  - **SELinux split policy**: `plat_*` vs `vendor_*`.
  - **GSI** (Generic System Image): chạy được trên mọi thiết bị Treble-compliant → dùng cho VTS.

**Ví dụ trong project:** HAL `vendor.david.hardware.daviddevicecontrol` nằm hoàn toàn trên `/vendor` (`vendor: true` trong Android.bp), còn client `DavidExtension` nằm ở `system_ext` → giao tiếp qua stable AIDL `@VintfStability`.

---

### Q2. Kể tên các partition và cái gì nằm trong đó?

| Partition | Chủ sở hữu | Nội dung |
|-----------|-----------|----------|
| `boot` | Google (GKI) | kernel + (từ A13) không còn ramdisk generic nếu có `init_boot` |
| `init_boot` | Google | generic ramdisk (first stage init) — A13+ |
| `vendor_boot` | SoC vendor | vendor ramdisk (kernel modules, fstab first stage), DTB |
| `vbmeta`, `vbmeta_system` | — | metadata AVB (hash/hashtree, public key) |
| `system` | Google/OEM | framework, `/system/bin`, `/system/framework` |
| `system_ext` | OEM | extension của system (app/service OEM dùng hidden API) |
| `product` | OEM | app, overlay, config theo sản phẩm |
| `vendor` | SoC | HAL, vendor libs, vendor init rc, sepolicy vendor |
| `odm` | ODM | customize của ODM trên nền vendor |
| `vendor_dlkm`, `system_dlkm`, `odm_dlkm` | | kernel modules (.ko) |
| `super` | | dynamic partition chứa system/vendor/product/odm/... |
| `userdata`, `metadata`, `misc` | | data, key metadata encryption, BCB cho recovery |

**Ví dụ thực tế** `device/amlogic/KSTB2500/BoardConfig.mk`:
```make
BOARD_SUPER_PARTITION_GROUPS := amlogic_dynamic_partitions
BOARD_AMLOGIC_DYNAMIC_PARTITIONS_PARTITION_LIST := system vendor product odm
BOARD_AMLOGIC_DYNAMIC_PARTITIONS_PARTITION_LIST += system_ext
BOARD_AMLOGIC_DYNAMIC_PARTITIONS_PARTITION_LIST += vendor_dlkm
```

---

### Q3. Dynamic partition (super) là gì? Lợi ích?

- Từ Android 10: các partition logic (system, vendor, ...) nằm trong một partition vật lý `super`, quản lý bằng **liblp** (metadata ở đầu `super`), map bằng **device-mapper** (`dm-linear`).
- Lợi ích: resize partition qua OTA, không phải sửa partition table; tiết kiệm dung lượng (Virtual A/B chỉ cần 1 bản + snapshot COW).
- Trong fstab đánh dấu cờ `logical`:
```
system  /system  erofs  ro  wait,slotselect,avb=vbmeta_system,logical,first_stage_mount
```
- Tool: `lpdump`, `lpmake`, `lpunpack`, `fastboot getvar all` (fastbootd ở userspace mới flash được logical partition).

---

### Q4. A/B và Virtual A/B khác nhau thế nào?

| | A/B (seamless) | Virtual A/B (A11+) |
|--|----------------|--------------------|
| Bộ nhớ | 2 bản mọi partition (`_a`, `_b`) | 1 bản partition dynamic + **snapshot COW** |
| OTA | update_engine ghi vào slot không active | ghi vào COW (snapuserd), merge sau khi boot thành công |
| Rollback | đổi slot | hủy snapshot |
| Compression | — | `virtual_ab_ota/compression.mk` (dm-user + snapuserd) |

Project dùng Virtual A/B nén:
```make
# device/amlogic/KSTB2500/KSTB2500.mk
$(call inherit-product, $(SRC_TARGET_DIR)/product/virtual_ab_ota/compression.mk)
$(call inherit-product, $(SRC_TARGET_DIR)/product/virtual_ab_ota/launch_with_vendor_ramdisk.mk)
```
Lệnh debug: `snapshotctl dump`, `update_engine_client --status`, `bootctl get-current-slot`.

---

### Q5. VNDK là gì? Phân biệt LL-NDK, VNDK, VNDK-SP, vendor_available, vendor: true

- **LL-NDK**: libs system mà vendor được link, ổn định ABI (libc, libm, libdl, liblog, libEGL, libvndksupport, `libbinder_ndk`...).
- **VNDK**: libs core system có bản riêng cho vendor (`/apex/com.android.vndk.vXX/`), ví dụ `libbase`, `libutils`, `libcutils`...
- **VNDK-SP**: VNDK dùng được trong *same-process HAL* (ví dụ graphics mapper, libEGL vendor chạy trong process app).
- `vendor_available: true`: module build ra **2 variant** (core + vendor).
- `vendor: true`: chỉ build cho vendor, cài vào `/vendor`.
- **Lưu ý A14**: VNDK đã **deprecated** (A15 bỏ hẳn). Các lib vendor nên tự đóng gói hoặc dùng LL-NDK/AIDL. Đây là điểm cộng nếu nói được trong phỏng vấn.

---

### Q6. system_ext khác product thế nào? Khi nào đặt module vào đâu?

- `system_ext`: module **gắn chặt với system** (được dùng hidden API/`platform_apis`, ký platform key, sharedUserId system). Không có interface ổn định với system → phải update cùng system.
- `product`: app/overlay/config theo SKU, chỉ dùng **public/system API** (từ A11 `PRODUCT_ENFORCE_PRODUCT_PARTITION_INTERFACE`).
- `vendor/odm`: chỉ code SoC/HAL; **không được** dùng framework Java trực tiếp.

**Ví dụ:** `DavidExtension` là `system_ext_specific: true`, `privileged: true`, `certificate: "platform"`, `platform_apis: true` → đúng chỗ vì nó dùng `android.os.ServiceManager` (hidden API).

---

### Q7. GSI và VTS/CTS liên quan gì đến Treble?

- **CTS**: test API public (app-level compatibility).
- **VTS**: test HAL + kernel (vendor interface) — chạy trên **GSI** để chứng minh vendor partition đáp ứng Treble.
- **CTS-on-GSI**: chạy CTS với system = GSI + vendor của OEM.
- **STS** (security), **GTS** (Google Mobile Services), **TVTS** (Android TV – nhắc trong `david.mk`: "energy modes - TVTS, CTSV fail").

---

### Q8. GKI (Generic Kernel Image) là gì?

- Từ A12: kernel core do Google build (**GKI**), SoC vendor chỉ cung cấp **vendor modules** (`.ko`) qua **KMI** (Kernel Module Interface) ổn định.
- Module load ở `vendor_boot` ramdisk (first stage) hoặc `vendor_dlkm` (second stage), danh sách trong `modules.load`.
- Project load module cả trong init rc:
```
on init
    insmod /vendor/lib/modules/mali.ko
```
- Câu hỏi follow-up: *"Tại sao không dùng `insmod` trong rc mà nên dùng `modules.load`?"* → thứ tự/phụ thuộc được modprobe xử lý (modules.dep), load song song, giảm thời gian boot, tránh bị SELinux denial `module_load`.
