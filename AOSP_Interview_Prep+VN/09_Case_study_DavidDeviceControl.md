# Chương 09 — Case study: `DavidDeviceControl` end-to-end (từ App → sysfs)

> Đây là ví dụ "kể chuyện dự án" mạnh nhất khi phỏng vấn: một tính năng thật đi xuyên qua **mọi tầng**
> (build, init, VINTF, SELinux, binder, HAL, kernel). Ticket: **KG25040-165 AIDL - DavidDeviceControl**.

## 9.1 Sơ đồ luồng

```
[system_ext/priv-app] DavidExtension (uid system, domain system_app, persistent)
   DavidExtApplication.onCreate()
     └─ DavidExtService
          ├─ NativeWrapperImpl.connectToHal()
          │     ServiceManager.waitForDeclaredService("vendor.david.hardware.daviddevicecontrol.IDavidDeviceControl/default")
          │     linkToDeath(...)
          └─ PowerLedManager / PowerStateManager → nativeWrapper.setLedColor(...)
                         │  Java proxy (vendor.david.hardware.daviddevicecontrol-V1-java)
                         ▼  /dev/binder  (stability VINTF được servicemanager kiểm tra)
[vendor] /vendor/bin/hw/vendor.david.hardware.daviddevicecontrol-service  (domain daviddevicecontrol, user system)
   BnDavidDeviceControl ← DavidDeviceControl::setLedColor(LedColor)
     └─ DeviceControl::setLedColor(ledColorIndex) → writeSysfs(SYSFS_LED_*_ONOFF, "...")
                         ▼
[kernel] LED class driver (/sys/class/leds/...), trigger IR
```

## 9.2 Bản đồ file

| Tầng | File |
|------|------|
| Interface | `vendor/david/interfaces/daviddevicecontrol/aidl/Android.bp`, `.../IDavidDeviceControl.aidl`, `LedColor.aidl`, `LedOnOff.aidl` |
| API dump | `vendor/david/interfaces/daviddevicecontrol/aidl/aidl_api/vendor.david.hardware.daviddevicecontrol/current/` |
| HAL | `vendor/david/hals/DavidDeviceControl/{Service.cpp, DavidDeviceControl.cpp/.h, DeviceControl.cpp/.h, common.h}` |
| Build | `vendor/david/hals/DavidDeviceControl/Android.bp` |
| init | `vendor.david.hardware.daviddevicecontrol-service.rc` |
| VINTF | `manifest_vendor.david...-service.xml` (device manifest), `device_matrix_product_david_daviddevicecontrol.xml` (FCM) |
| SELinux | `vendor/david/sepolicy/vendor/{daviddevicecontrol.te, service.te, service_contexts, file_contexts, system_server.te}` |
| Product | `vendor/david/david.mk` (PRODUCT_PACKAGES, BOARD_SEPOLICY_DIRS, DEVICE_PRODUCT_COMPATIBILITY_MATRIX_FILE) |
| Client | `vendor/david/services/framework/davidextensionsvc/` |

## 9.3 Kịch bản kể trong phỏng vấn (STAR, ~2 phút)

- **Situation**: STB cần điều khiển LED trước (đỏ = standby, trắng = active, nháy khi nhận IR) từ framework; trước đây dùng script `led.sh` chạy root trong init.
- **Task**: thiết kế interface ổn định Treble giữa `system_ext` và `vendor`.
- **Action**: định nghĩa stable AIDL `@VintfStability` + enum `@Backing(int)`; HAL NDK backend chạy user `system`; khai báo VINTF manifest + FCM; viết SELinux domain riêng; client Java trong persistent system app, có death recipient để reconnect.
- **Result**: framework không phụ thuộc sysfs path; khi đổi board/driver chỉ sửa HAL; pass VTS/`checkvintf`; bỏ được quyền root cho phần LED.

---

## 9.4 Code review như senior — các điểm cải thiện (rất hay bị hỏi "nếu làm lại bạn sửa gì?")

| # | Vấn đề | Vị trí | Đề xuất |
|---|--------|--------|---------|
| 1 | API **chưa freeze** (chỉ có `aidl_api/.../current/`) nhưng manifest khai báo version 1 | `interfaces/daviddevicecontrol/aidl` | `m vendor.david.hardware.daviddevicecontrol-freeze-api` trước release; sau đó mọi thay đổi → V2 |
| 2 | Tên method không theo AIDL style (`DC_Status`), `setLedOnOff` nhận tham số nhưng **không làm gì** | `.aidl`, `DavidDeviceControl.cpp` | đặt tên `getStatus()`, trả về `parcelable LedState`; hoặc bỏ method stub |
| 3 | Mọi method đều trả `ok()`; `writeSysfs` lỗi không báo lên | `DavidDeviceControl.cpp` | trả `fromServiceSpecificError` / `EX_ILLEGAL_ARGUMENT` |
| 4 | Enum từ binder có thể mang giá trị ngoài range (backing int) → `switch` không `default` → biến `ledColor` **chưa khởi tạo** | `setLedColor`, `setLedRedBlink` | thêm `default: return fromExceptionCode(EX_ILLEGAL_ARGUMENT)` |
| 5 | Thread pool 8 thread nhưng không có mutex; nhiều thread ghi sysfs đồng thời (chuỗi ghi `red off` → `white on` không atomic) | `Service.cpp`, `DeviceControl.cpp` | `std::mutex` trong `DeviceControl`, hoặc thread pool 1 |
| 6 | `DeviceControl` có singleton `getInstance()` nhưng `DavidDeviceControl` lại `new DeviceControl()` + raw pointer | `DavidDeviceControl.cpp` | `std::unique_ptr<DeviceControl>`, bỏ singleton thừa |
| 7 | `AIBinder_setRequestingSid(true)` nhưng không dùng SID để check quyền | `Service.cpp` | hoặc dùng `AIBinder_getCallingSid()` để kiểm tra client, hoặc bỏ |
| 8 | Link thừa `android.hardware.common-V2-ndk`, `libaidlcommonsupport` | `Android.bp` | bỏ để giảm size/deps |
| 9 | rc: start ở `on init` nhưng không có dòng `interface aidl ...` | `.rc` | thêm `interface aidl vendor.david.hardware.daviddevicecontrol.IDavidDeviceControl/default`; cân nhắc `class early_hal` |
| 10 | SELinux `allow daviddevicecontrol sysfs:file { read write open }` quá rộng | `daviddevicecontrol.te` | label node bằng `genfs_contexts`, dùng macro `binder_use`, `add_service`, `rw_file_perms` |
| 11 | Client reconnect chạy trên **main looper** và gọi API blocking | `NativeWrapperImpl.binderDied()` | `HandlerThread` riêng; set `mIDavidDeviceControl = null` khi chết; dùng `IServiceCallback` |
| 12 | `david.mk` add cả `-V1-ndk`, `-V1-java`, tên interface vào PRODUCT_PACKAGES | `david.mk` | chỉ cần service binary + app (deps tự kéo theo) |
| 13 | Log macro `SYS_LOGx` gọi `property_get` **mỗi lần log** | `common.h` | đọc property 1 lần (hoặc dùng `__android_log_is_loggable` / `LOG_NDEBUG`) |
| 14 | `IDavidExt` rỗng, service `davidextension` add bằng app; cần `service_contexts` + permission check | `davidextensionsvc` | khai báo `davidextension u:object_r:davidext_service:s0`, `enforceCallingPermission` cho API sau này |

### Ví dụ refactor HAL (điểm 3, 4, 5, 6)

```cpp
// DavidDeviceControl.h
struct DavidDeviceControl : public BnDavidDeviceControl {
    ::ndk::ScopedAStatus setLedColor(LedColor in_color) override;
    ...
  private:
    std::mutex mLock;
    std::unique_ptr<DeviceControl> mDeviceControl = std::make_unique<DeviceControl>();
};

// DavidDeviceControl.cpp
static std::optional<ledColorIndex> toNative(LedColor c) {
    switch (c) {
        case LedColor::OFF:   return LED_OFF;
        case LedColor::RED:   return LED_RED;
        case LedColor::WHITE: return LED_WHITE;
        case LedColor::BOTH:  return LED_BOTH;
    }
    return std::nullopt;               // giá trị ngoài enum gửi qua binder
}

::ndk::ScopedAStatus DavidDeviceControl::setLedColor(LedColor in_color) {
    auto native = toNative(in_color);
    if (!native) {
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_ARGUMENT);
    }
    std::lock_guard<std::mutex> lk(mLock);
    if (!mDeviceControl->setLedColor(*native)) {          // trả bool từ writeSysfs
        return ndk::ScopedAStatus::fromServiceSpecificError(IDavidDeviceControl::ERROR_IO);
    }
    return ndk::ScopedAStatus::ok();
}
```
(`ERROR_IO` cần khai báo `const int ERROR_IO = 1;` trong `.aidl` → thay đổi API → phải là **V2** nếu V1 đã freeze.)

### Ví dụ nâng version lên V2 đúng quy trình

```aidl
@VintfStability
interface IDavidDeviceControl {
    // ---- V1 giữ nguyên thứ tự ----
    void DC_Status();
    void setLedOnOff(in LedColor color, in LedOnOff onOff);
    void setLedColor(in LedColor color);
    void setLedBlink();
    void setLedRedBlink(in LedOnOff onOff);
    void setLedWhiteBlink(in LedOnOff onOff);
    // ---- V2: chỉ thêm ở cuối ----
    const int ERROR_IO = 1;
    LedColor getLedColor();
    void registerCallback(in IDavidDeviceControlCallback cb);
}
```
```bash
m vendor.david.hardware.daviddevicecontrol-update-api
m vendor.david.hardware.daviddevicecontrol-freeze-api      # tạo aidl_api/.../2/
```
- Đổi lib: `vendor.david.hardware.daviddevicecontrol-V2-ndk` (HAL), `-V2-java` (client).
- Manifest `<version>2</version>`; FCM `<version>1-2</version>` để framework chấp nhận vendor cũ.
- Client cũ (V1) vẫn chạy với HAL V2. Client mới gọi `getInterfaceVersion()` trước khi gọi `getLedColor()` nếu có thể gặp HAL V1.

## 9.5 Câu hỏi phỏng vấn xoáy vào case study

1. *Tại sao chọn AIDL mà không phải HIDL?* → HIDL deprecated từ A13; AIDL thống nhất IPC, hỗ trợ Java/NDK/Rust, versioning đơn giản.
2. *Tại sao HAL dùng NDK backend còn client dùng Java backend? Có tương thích không?* → Có. Wire format giống nhau; backend chỉ là code sinh ra cho ngôn ngữ.
3. *Tại sao HAL đặt ở `/vendor/bin/hw` mà client ở `system_ext`?* → Treble: HAL là vendor implementation; client dùng hidden API nên thuộc system_ext.
4. *Nếu bỏ `vintf_fragments` thì sao?* → servicemanager từ chối `addService` của VINTF-stability binder; `waitForDeclaredService` trả null ngay; `checkvintf` fail nếu FCM yêu cầu HAL đó.
5. *Nếu bỏ FCM thì sao?* → build lỗi `checkvintf`: HAL trong device manifest không có trong framework compatibility matrix (với HAL không phải `android.*` phải khai báo qua `DEVICE_PRODUCT_COMPATIBILITY_MATRIX_FILE`).
6. *Một app bên thứ ba có gọi được HAL này không?* → Không: SELinux chỉ cho `system_server` và `system_app` `find` + `binder_call`; app `untrusted_app` bị chặn ở servicemanager.
7. *HAL crash thì LED thế nào? framework biết không?* → init restart HAL (không `oneshot`); client nhận `binderDied` → reconnect → nên **re-apply state** cuối cùng (hiện code chưa làm).
8. *Làm thế nào để test HAL này?* → VTS-style gtest (`cc_test` với `test_suites: ["vts"]`, `AServiceManager_waitForService`, parametrized theo instance từ `getAidlHalInstanceNames(IDavidDeviceControl::descriptor)`), test Java instrumentation cho client, `atest`.
