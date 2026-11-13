# Chương 05 — HAL: Legacy HAL → HIDL → AIDL HAL, VINTF

## 5.1 Lịch sử HAL

| Thời kỳ | Dạng | Đặc điểm |
|---------|------|----------|
| ≤ Android 7 | **Legacy HAL** (`hw_module_t`, `hw_get_module()`, `.so` trong `/system/lib/hw`) | load `dlopen` trực tiếp vào process framework → không tách system/vendor |
| Android 8–10 | **HIDL** (`.hal`, `android.hardware.foo@1.0`) | binderized (`/dev/hwbinder`, hwservicemanager) hoặc passthrough |
| Android 11+ | **Stable AIDL HAL** (`@VintfStability`) | dùng `/dev/binder` + `servicemanager`, 1 ngôn ngữ IPC cho cả framework & HAL |
| Android 13+ | HIDL **deprecated** cho HAL mới | mọi HAL mới bắt buộc AIDL |

---

### Q1. So sánh HIDL và AIDL HAL

| | HIDL | AIDL HAL |
|--|------|----------|
| File | `.hal` | `.aidl` |
| Driver | `/dev/hwbinder` | `/dev/binder` |
| Service manager | `hwservicemanager` | `servicemanager` |
| Versioning | package version `@1.0`, `@1.1` (kế thừa interface) | số version `V1`, `V2` trong cùng package, **frozen API dump** |
| Instance name | `IFoo/default` qua `registerAsService()` | `"<descriptor>/default"` qua `AServiceManager_addService()` |
| Backend | C++ / Java | NDK / CPP / Java / Rust |
| Tool | `lshal` | `service list`, `dumpsys -l`, `lshal` (vẫn liệt kê AIDL) |
| Passthrough | có (`-impl.so` + `HIDL_FETCH_IFoo`) | không (chỉ binderized) |
| Trạng thái | deprecated | chuẩn hiện tại |

---

### Q2. Passthrough vs Binderized HAL?

- **Binderized**: HAL chạy process riêng (`/vendor/bin/hw/...-service`), giao tiếp IPC → cô lập lỗi, SELinux domain riêng.
- **Passthrough** (HIDL): wrapper quanh legacy HAL `.so`, client `dlopen` (`getService(true)`) → cùng process. Còn tồn tại cho **same-process HAL (SP-HAL)** như `android.hardware.graphics.mapper` (vì hiệu năng).
- AIDL không có passthrough; code muốn in-process thì dùng thư viện thường (hoặc HAL mapper A14 dùng `IMapper` stable-C).

---

### Q3. Các bước tạo một AIDL HAL mới từ đầu (câu hỏi kinh điển)

Lấy đúng ví dụ project `vendor.david.hardware.daviddevicecontrol`:

**Bước 1 — Định nghĩa interface** `vendor/david/interfaces/daviddevicecontrol/aidl/vendor/david/hardware/daviddevicecontrol/IDavidDeviceControl.aidl`
```aidl
package vendor.david.hardware.daviddevicecontrol;

import vendor.david.hardware.daviddevicecontrol.LedColor;
import vendor.david.hardware.daviddevicecontrol.LedOnOff;

@VintfStability
interface IDavidDeviceControl {
    void DC_Status();
    void setLedOnOff(in LedColor color, in LedOnOff onOff);
    void setLedColor(in LedColor color);
    void setLedBlink();
    void setLedRedBlink(in LedOnOff onOff);
    void setLedWhiteBlink(in LedOnOff onOff);
}
```
```aidl
@VintfStability
@Backing(type="int")
enum LedColor { OFF, RED, WHITE, BOTH, }
```

**Bước 2 — `aidl_interface`** (Android.bp cùng thư mục, xem Chương 04 Q6), `stability: "vintf"`.

**Bước 3 — Sinh/đóng băng API**
```bash
m vendor.david.hardware.daviddevicecontrol-update-api   # tạo aidl_api/<name>/current/
m vendor.david.hardware.daviddevicecontrol-freeze-api   # tạo aidl_api/<name>/1/ + .hash, thêm versions_with_info vào bp
```

**Bước 4 — Implement service (NDK)**
```cpp
// DavidDeviceControl.h
struct DavidDeviceControl : public BnDavidDeviceControl {
    ::ndk::ScopedAStatus setLedColor(LedColor in_color) override;
    ...
};
// Service.cpp
int main() {
    ABinderProcess_setThreadPoolMaxThreadCount(8);
    auto svc = ::ndk::SharedRefBase::make<DavidDeviceControl>();
    const std::string instance = std::string() + DavidDeviceControl::descriptor + "/default";
    binder_status_t status = AServiceManager_addService(svc->asBinder().get(), instance.c_str());
    CHECK(status == STATUS_OK);
    ABinderProcess_joinThreadPool();
}
```

**Bước 5 — Build binary** `cc_binary` với `vendor: true`, `relative_install_path: "hw"`, `init_rc`, `vintf_fragments` (Chương 03 Q3).

**Bước 6 — init rc** `vendor.david.hardware.daviddevicecontrol-service.rc` (Chương 02 Q12).

**Bước 7 — VINTF device manifest fragment**
```xml
<manifest version="1.0" type="device" target-level="8">
    <hal format="aidl">
        <name>vendor.david.hardware.daviddevicecontrol</name>
        <version>1</version>
        <interface><name>IDavidDeviceControl</name><instance>default</instance></interface>
    </hal>
</manifest>
```
(Từ A12, AIDL HAL có thể dùng `<fqname>IDavidDeviceControl/default</fqname>` thay cho khối `<interface>`.)

**Bước 8 — Framework compatibility matrix (FCM) cho HAL vendor-defined**
```xml
<compatibility-matrix version="1.0" type="framework">
    <hal format="aidl">
        <name>vendor.david.hardware.daviddevicecontrol</name>
        <version>1</version>
        <interface><name>IDavidDeviceControl</name><instance>default</instance></interface>
    </hal>
</compatibility-matrix>
```
```make
DEVICE_PRODUCT_COMPATIBILITY_MATRIX_FILE += vendor/david/hals/DavidDeviceControl/device_matrix_product_david_daviddevicecontrol.xml
```

**Bước 9 — SELinux** (Chương 07): domain, exec type, file_contexts, service.te, service_contexts, quyền sysfs, quyền client `find` + `binder_call`.

**Bước 10 — PRODUCT_PACKAGES**
```make
PRODUCT_PACKAGES += vendor.david.hardware.daviddevicecontrol-service
```
(Thực tế `david.mk` còn add cả `-V1-ndk`, `-V1-java` và tên interface — không cần thiết vì đã là dependency của service/app.)

**Bước 11 — Client**
- Java: `IDavidDeviceControl.Stub.asInterface(ServiceManager.waitForDeclaredService(IDavidDeviceControl.DESCRIPTOR + "/default"))`.
- NDK: `AServiceManager_waitForService(instance)` → `IDavidDeviceControl::fromBinder(ndk::SpAIBinder(...))`.

**Bước 12 — Verify**
```bash
adb shell service list | grep david
adb shell dumpsys -l | grep david
adb shell lshal | grep david
adb shell ps -AZ | grep daviddevicecontrol      # kiểm tra đúng domain u:r:daviddevicecontrol:s0
adb shell cat /vendor/etc/vintf/manifest/*daviddevicecontrol*
adb logcat -s daviddevicecontrol DeviceControl
```

---

### Q4. VINTF là gì? Device manifest vs Framework compatibility matrix?

- **VINTF object** = thông tin để đảm bảo system và vendor tương thích:
  - **Device manifest** (`/vendor/etc/vintf/manifest.xml` + fragments trong `manifest/`): vendor **cung cấp** HAL nào, version nào, kernel version, sepolicy version, `target-level` (FCM version).
  - **Framework compatibility matrix (FCM)** (`/system/etc/vintf/compatibility_matrix.xml`): framework **yêu cầu** HAL nào (optional/required, range version).
  - Ngược lại: **Framework manifest** (system cung cấp HAL/service gì) và **Device compatibility matrix** (vendor yêu cầu gì từ framework, ví dụ VNDK/sepolicy version).
- Kiểm tra: lúc build (`checkvintf`, `assemble_vintf`), lúc OTA (`update_engine` kiểm tra), lúc runtime (`vintf` tool, `VintfObject`).
- `target-level="8"` ↔ FCM level 8 = Android 14 (7 = A13, 6 = A12, 5 = A11).

```bash
adb shell vintf                    # dump runtime VINTF, check compatibility
adb shell cat /vendor/etc/vintf/manifest.xml
```

---

### Q5. Versioning của stable AIDL: frozen, current, hash — quy tắc thay đổi?

- Khi **freeze** version N: thư mục `aidl_api/<name>/N/` chứa bản dump + `.hash` → **không được sửa** nữa.
- `current/` là bản đang phát triển (version N+1 nếu có thay đổi).
- Quy tắc thay đổi tương thích ngược (đối với version mới):
  - ✅ Thêm method **ở cuối** interface, thêm field **ở cuối** parcelable (có default), thêm enumerator, thêm const.
  - ❌ Xóa/đổi tên/đổi thứ tự method, đổi kiểu tham số, xóa field.
- Client gọi method mới trên server version cũ → nhận `UNKNOWN_TRANSACTION` → dùng `getInterfaceVersion()` để check.
- Server phải implement version cao nhất nó khai báo trong manifest.
- **Quan sát project**: `aidl_api/vendor.david.hardware.daviddevicecontrol/` chỉ có `current/`, **chưa có `1/`** (chưa freeze), trong khi manifest khai báo `<version>1</version>`. Interface chưa freeze vẫn build (V1 = current) nhưng **không đảm bảo ổn định**; với release chính thức nên chạy `-freeze-api` để mọi thay đổi sau này bị build check bắt lại.

---

### Q6. Tại sao lệnh `AServiceManager_addService` của HAL vendor có thể fail dù code đúng?

Checklist debug:
1. **Không có trong VINTF manifest** → log `servicemanager: Could not find vendor.david...IDavidDeviceControl/default in the VINTF manifest`.
2. **SELinux**: thiếu `add_service`/`service_contexts` → `avc: denied { add } for service=... scontext=u:r:daviddevicecontrol:s0 tcontext=u:object_r:default_android_service:s0`.
3. **Stability**: interface không `@VintfStability` hoặc build bằng backend sai → `Cannot do a user transaction on a ... stability binder`.
4. Version trong manifest khác version thực tế của lib (`-V1-ndk` vs `-V2-ndk`).
5. Process chạy sai domain (thiếu file_contexts → chạy dưới `init` domain hoặc bị chặn exec: `avc denied { execute_no_trans }`).

---

### Q7. Lazy HAL là gì?

- HAL chỉ chạy khi có client, tự thoát khi không còn client → tiết kiệm RAM.
- AIDL: dùng `AServiceManager_registerLazyService()` thay `addService`; rc có `interface aidl <name>/<instance>`, `oneshot`, `disabled`.
- servicemanager khi có `getService` mà chưa chạy → set `ctl.interface_start aidl/<name>/<instance>` → init start service.
```rc
service vendor.foo-hal /vendor/bin/hw/android.hardware.foo-service.lazy
    interface aidl android.hardware.foo.IFoo/default
    class hal
    user system
    group system
    oneshot
    disabled
```

---

### Q8. HAL nên chạy với user nào? Nên truy cập phần cứng thế nào?

- Nguyên tắc **least privilege**: tránh `root`; dùng user riêng hoặc `system`, group theo device node (`audio`, `media`...), cấp `capabilities` cụ thể nếu cần.
- Quyền file `/dev`, `/sys` cấp qua `ueventd.rc` (mode/owner) + SELinux label qua `genfs_contexts`/`file_contexts`.
- Project: HAL chạy `user system` ghi sysfs LED (`/sys/class/leds/...`) — đúng hướng; nhưng script `led.sh`, `wifi.sh` trong `init.amlogic.board.rc` chạy `user root` → điểm có thể cải thiện.

---

### Q9. HAL phải thread-safe không? Callback từ HAL lên framework làm thế nào?

- Có — binder thread pool có thể gọi đồng thời → dùng `std::mutex` bảo vệ state/sysfs write.
- Callback: định nghĩa `oneway interface IFooCallback` trong cùng package; framework truyền callback vào `registerCallback(in IFooCallback cb)`; HAL lưu `std::shared_ptr<IFooCallback>`, link death để tự xóa khi client chết.
```cpp
::ndk::ScopedAStatus Hal::registerCallback(const std::shared_ptr<IFooCallback>& cb) {
    std::lock_guard<std::mutex> lk(mLock);
    AIBinder_linkToDeath(cb->asBinder().get(), mDeathRecipient.get(), this);
    mCallbacks.push_back(cb);
    return ndk::ScopedAStatus::ok();
}
```

---

### Q10. Cách chuyển một HIDL HAL sang AIDL?

1. Viết `.aidl` tương đương (tool `hidl2aidl -o out -r vendor.amlogic:vendor/amlogic/... vendor.amlogic.hardware.systemcontrol@1.1`).
2. `aidl_interface` với `stability: "vintf"`, freeze V1.
3. Implement NDK backend, đăng ký servicemanager.
4. Đổi manifest `format="hidl"` → `format="aidl"`, FCM tương ứng.
5. SELinux: `hal_attribute`, `hwservice_contexts` → `service_contexts`, `hal_foo_hwservice` → `hal_foo_service`.
6. Client Java chuyển từ `IFoo.getService()` (HIDL) sang `ServiceManager.waitForDeclaredService()`.
- Ví dụ trong project: `DavidExtension` vẫn dùng HIDL `vendor.amlogic.hardware.systemcontrol-V1.0/1.1-java` song song với AIDL HAL mới của David → minh chứng cho giai đoạn chuyển tiếp.
