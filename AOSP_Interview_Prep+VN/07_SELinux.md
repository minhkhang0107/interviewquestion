# Chương 07 — SELinux trong Android

## 7.1 Khái niệm

- **MAC** (Mandatory Access Control) bổ sung cho DAC (uid/gid/mode). Mặc định **deny all**, chỉ cho phép những gì có rule `allow`.
- Mỗi process có **domain** (`u:r:system_server:s0`), mỗi object có **type** (`u:object_r:system_file:s0`).
- Rule: `allow <source_domain> <target_type>:<class> { permissions };`
- Mode: `enforcing` (chặn + log) / `permissive` (chỉ log). Android user build bắt buộc enforcing.
- Policy tách (Treble): `system/sepolicy/public` (API cho vendor dùng), `private` (chỉ platform), `vendor` (`BOARD_SEPOLICY_DIRS`), `system_ext`/`product` (`SYSTEM_EXT_PUBLIC/PRIVATE_SEPOLICY_DIRS`, `PRODUCT_*`).

---

### Q1. Những file sepolicy quan trọng và vai trò?

| File | Vai trò | Ví dụ |
|------|---------|-------|
| `*.te` | khai báo type/domain và rule allow | `daviddevicecontrol.te` |
| `file_contexts` | label cho file trên filesystem (ro partition gán lúc build) | `/vendor/bin/hw/vendor.david...-service u:object_r:daviddevicecontrol_exec:s0` |
| `genfs_contexts` | label cho `/sys`, `/proc` (filesystem không hỗ trợ xattr) | `genfscon sysfs /devices/platform/leds u:object_r:sysfs_leds:s0` |
| `service_contexts` | label tên service trong servicemanager | `vendor.david...IDavidDeviceControl/default u:object_r:daviddevicecontrol_service:s0` |
| `hwservice_contexts` | label HIDL service | `vendor.amlogic.hardware.systemcontrol::ISystemControl u:object_r:hal_systemcontrol_hwservice:s0` |
| `vndservice_contexts` | label service trên vndbinder | |
| `property_contexts` | label property | `persist.vendor.sc.log.level u:object_r:vendor_sc_prop:s0` |
| `seapp_contexts` | gán domain cho app theo uid/seinfo/package | `user=system seinfo=platform domain=system_app type=system_app_data_file` |
| `mac_permissions.xml` | map certificate → seinfo | platform cert → `seinfo=platform` |

---

### Q2. Viết policy đầy đủ cho một AIDL HAL mới (ví dụ thật trong project)

`vendor/david/sepolicy/vendor/daviddevicecontrol.te`:
```
type daviddevicecontrol, domain;
type daviddevicecontrol_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(daviddevicecontrol)             # init exec file *_exec → transition sang domain này

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
`service_contexts`, `file_contexts`: như bảng Q1.

`system_server.te` (client):
```
allow system_server daviddevicecontrol_service:service_manager { find };
binder_call(system_server, daviddevicecontrol)
allow system_app daviddevicecontrol_service:service_manager { find };
binder_call(system_app, daviddevicecontrol)
```
Và trong `david.mk`: `BOARD_SEPOLICY_DIRS += vendor/david/sepolicy/vendor`.

**Phiên bản "senior" (dùng macro, tối thiểu quyền):**
```
type daviddevicecontrol, domain;
type daviddevicecontrol_exec, exec_type, vendor_file_type, file_type;
init_daemon_domain(daviddevicecontrol)

binder_use(daviddevicecontrol)                                   # call servicemanager + dùng binder
add_service(daviddevicecontrol, daviddevicecontrol_service)       # add + neverallow check
allow daviddevicecontrol sysfs_leds:dir r_dir_perms;
allow daviddevicecontrol sysfs_leds:file rw_file_perms;
# BỎ: allow daviddevicecontrol sysfs:file { read write open };   → quá rộng
```
→ Rule `sysfs:file` cho phép ghi **mọi** node sysfs chưa có label riêng → vi phạm least privilege; cách đúng là label node cụ thể bằng `genfs_contexts` rồi chỉ cho phép type đó.

Client: `binder_call(system_app, daviddevicecontrol)` là đủ thay vì cho rộng; nên cân nhắc tạo **attribute** `hal_daviddevicecontrol_client/server` bằng `hal_attribute(daviddevicecontrol)` + `hal_server_domain` / `hal_client_domain` như HAL AOSP.

---

### Q3. Đọc và xử lý một AVC denial

```
avc: denied { write } for comm="vendor.david.har" name="brightness" dev="sysfs" ino=1234
  scontext=u:r:daviddevicecontrol:s0 tcontext=u:object_r:sysfs:s0 tclass=file permissive=0
```
Đọc: **ai** (`scontext` domain `daviddevicecontrol`) muốn **làm gì** (`write`) trên **cái gì** (`tcontext` type `sysfs`, class `file`, tên `brightness`).

Các bước xử lý đúng:
1. Hỏi: quyền này **có nên** có không? (đừng audit2allow mù quáng)
2. Nếu target là type chung chung (`sysfs`, `device`, `default_prop`, `default_android_service`) → **tạo type riêng** và label đúng:
   ```
   # vendor/david/sepolicy/vendor/file.te
   type sysfs_david_led, sysfs_type, fs_type;
   # genfs_contexts
   genfscon sysfs /devices/platform/leds/leds/red_led u:object_r:sysfs_david_led:s0
   ```
3. Thêm rule tối thiểu: `allow daviddevicecontrol sysfs_david_led:file rw_file_perms;`
4. Build lại `m selinux_policy` hoặc `m vendorimage`, kiểm tra `ls -Z /sys/...`.
- Lệnh hữu ích:
```bash
adb shell dmesg | grep avc ; adb logcat -b all | grep avc
audit2allow -p out/target/product/KSTB2500/root/sepolicy -i avc.txt   # gợi ý (chỉ tham khảo)
adb shell getenforce ; adb shell setenforce 0    # userdebug only, chỉ để chẩn đoán
adb shell ls -Z /vendor/bin/hw/ ; adb shell ps -AZ | grep david
adb shell restorecon -R /data/vendor/xxx
```

---

### Q4. neverallow là gì? Có bypass được không?

- `neverallow` là assertion compile-time: nếu policy nào cho phép điều bị cấm → **build fail**. Có cả trong CTS (`SELinuxNeverallowRulesTest`) → không thể bypass trên thiết bị certified.
- Ví dụ neverallow nổi tiếng:
  - Vendor domain không được `binder_call` vào domain system bất kỳ trừ qua HAL/được phép (`coredomain` vs non-coredomain).
  - App không được `add` service vào servicemanager.
  - Không ai ngoài init được ghi `default_prop`... 
  - Vendor process không được `execute` file trên `/system` (trừ vài exception).
- Cách xử lý: thiết kế lại (đổi type, dùng HAL làm cầu nối, dùng property có label riêng).

---

### Q5. coredomain vs vendor domain? Tại sao vendor không được dùng `/dev/binder` tùy ý (trước A11)?

- `coredomain`: các domain thuộc platform (system, system_ext). Vendor domain không có attribute này.
- Treble neverallow: vendor ↔ core chỉ giao tiếp qua **HAL interface ổn định**. Trước A11 vendor dùng `/dev/vndbinder` cho IPC nội bộ vendor, `/dev/hwbinder` cho HAL. Từ A11, AIDL HAL `@VintfStability` cho phép vendor dùng `/dev/binder` nhưng servicemanager **kiểm tra stability** của binder.

---

### Q6. App domain được gán thế nào? (`system_app`, `platform_app`, `priv_app`, `untrusted_app`)

- Dựa trên `seapp_contexts`: uid + seinfo (từ cert trong `mac_permissions.xml`) + isPrivApp + targetSdk.
- `DavidExtension` (`sharedUserId=android.uid.system`, ký platform) → domain **`system_app`** → vì vậy project cấp `allow system_app daviddevicecontrol_service:service_manager find`.
- App ký platform nhưng không `uid.system` → `platform_app`. App privileged không ký platform → `priv_app`.

---

### Q7. Thêm property vendor mới có label riêng?

```
# property.te
vendor_internal_prop(vendor_david_led_prop)     # macro A12+: vendor_internal/restricted/public
# property_contexts
persist.vendor.david.led.   u:object_r:vendor_david_led_prop:s0 prefix string
# allow
set_prop(daviddevicecontrol, vendor_david_led_prop)
get_prop(system_app, vendor_david_led_prop)     # cần vendor_restricted_prop nếu system đọc
```
- `vendor_internal_prop`: chỉ vendor dùng; `vendor_restricted_prop`: system đọc được; `vendor_public_prop`: system đọc/ghi. (tương ứng `system_internal/restricted/public_prop` ở platform).

---

### Q8. Debug khi process không start do SELinux?

- Log init: `init: cannot execv('/vendor/bin/hw/xxx'): Permission denied` hoặc `Could not get process context ...`.
- Thiếu `file_contexts` → file có label `vendor_file` → init không thể transition → `avc: denied { execute_no_trans }` hoặc service chạy trong domain `init`/`vendor_init` (bị neverallow chặn ở A14: init từ chối start service không có domain: *"Service xxx does not have a SELinux domain defined"*).
- Kiểm tra: `ls -Z /vendor/bin/hw/<binary>` phải ra `daviddevicecontrol_exec`.
