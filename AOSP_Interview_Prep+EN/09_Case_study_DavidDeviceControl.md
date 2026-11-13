# Chapter 09 — Case study: `DavidDeviceControl` end-to-end (from App → sysfs)

> This is the strongest "project story" for an interview: a real feature that goes through **every layer**
> (build, init, VINTF, SELinux, binder, HAL, kernel). Ticket: **KG25040-165 AIDL - DavidDeviceControl**.

## 9.1 Flow diagram

```
[system_ext/priv-app] DavidExtension (uid system, domain system_app, persistent)
   DavidExtApplication.onCreate()
     └─ DavidExtService
          ├─ NativeWrapperImpl.connectToHal()
          │     ServiceManager.waitForDeclaredService("vendor.david.hardware.daviddevicecontrol.IDavidDeviceControl/default")
          │     linkToDeath(...)
          └─ PowerLedManager / PowerStateManager → nativeWrapper.setLedColor(...)
                         │  Java proxy (vendor.david.hardware.daviddevicecontrol-V1-java)
                         ▼  /dev/binder  (VINTF stability checked by servicemanager)
[vendor] /vendor/bin/hw/vendor.david.hardware.daviddevicecontrol-service  (domain daviddevicecontrol, user system)
   BnDavidDeviceControl ← DavidDeviceControl::setLedColor(LedColor)
     └─ DeviceControl::setLedColor(ledColorIndex) → writeSysfs(SYSFS_LED_*_ONOFF, "...")
                         ▼
[kernel] LED class driver (/sys/class/leds/...), IR trigger
```

## 9.2 File map

| Layer | File |
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

## 9.3 Interview script (STAR, ~2 minutes)

- **Situation**: the STB needed to control its front LED (red = standby, white = active, blinking on IR receive) from the framework; previously a `led.sh` script ran as root from init.
- **Task**: design a Treble-stable interface between `system_ext` and `vendor`.
- **Action**: defined a stable AIDL `@VintfStability` interface + `@Backing(int)` enums; an NDK-backend HAL running as user `system`; declared the VINTF manifest + FCM; wrote a dedicated SELinux domain; a Java client in a persistent system app with a death recipient to reconnect.
- **Result**: the framework no longer depends on sysfs paths; changing board/driver only touches the HAL; passes VTS/`checkvintf`; root privileges removed from the LED path.

---

## 9.4 Senior-style code review — improvement points (frequently asked: "if you did it again, what would you change?")

| # | Issue | Location | Proposal |
|---|--------|--------|---------|
| 1 | API **not frozen** (only `aidl_api/.../current/`) while the manifest declares version 1 | `interfaces/daviddevicecontrol/aidl` | `m vendor.david.hardware.daviddevicecontrol-freeze-api` before release; every later change → V2 |
| 2 | Method names don't follow AIDL style (`DC_Status`); `setLedOnOff` takes arguments but **does nothing** | `.aidl`, `DavidDeviceControl.cpp` | rename to `getStatus()` returning a `parcelable LedState`; or remove the stub method |
| 3 | Every method returns `ok()`; `writeSysfs` errors are not reported | `DavidDeviceControl.cpp` | return `fromServiceSpecificError` / `EX_ILLEGAL_ARGUMENT` |
| 4 | Enums from binder can carry out-of-range values (int backing) → `switch` without `default` → the `ledColor` variable is **uninitialized** | `setLedColor`, `setLedRedBlink` | add `default: return fromExceptionCode(EX_ILLEGAL_ARGUMENT)` |
| 5 | Thread pool of 8 but no mutex; several threads write sysfs concurrently (the `red off` → `white on` sequence isn't atomic) | `Service.cpp`, `DeviceControl.cpp` | `std::mutex` in `DeviceControl`, or a pool of 1 |
| 6 | `DeviceControl` has a singleton `getInstance()`, yet `DavidDeviceControl` does `new DeviceControl()` + a raw pointer | `DavidDeviceControl.cpp` | `std::unique_ptr<DeviceControl>`, drop the redundant singleton |
| 7 | `AIBinder_setRequestingSid(true)` but the SID is never used for permission checks | `Service.cpp` | either use `AIBinder_getCallingSid()` to check the client, or remove it |
| 8 | Unneeded links to `android.hardware.common-V2-ndk`, `libaidlcommonsupport` | `Android.bp` | remove to reduce size/deps |
| 9 | rc: started `on init` but has no `interface aidl ...` line | `.rc` | add `interface aidl vendor.david.hardware.daviddevicecontrol.IDavidDeviceControl/default`; consider `class early_hal` |
| 10 | SELinux `allow daviddevicecontrol sysfs:file { read write open }` is too broad | `daviddevicecontrol.te` | label the node via `genfs_contexts`, use the `binder_use`, `add_service`, `rw_file_perms` macros |
| 11 | Client reconnect runs on the **main looper** and calls a blocking API | `NativeWrapperImpl.binderDied()` | dedicated `HandlerThread`; set `mIDavidDeviceControl = null` on death; use `IServiceCallback` |
| 12 | `david.mk` adds `-V1-ndk`, `-V1-java` and the interface name to PRODUCT_PACKAGES | `david.mk` | only the service binary + app are needed (deps follow automatically) |
| 13 | The `SYS_LOGx` log macro calls `property_get` **on every log** | `common.h` | read the property once (or use `__android_log_is_loggable` / `LOG_NDEBUG`) |
| 14 | `IDavidExt` is empty; the `davidextension` service is added by an app; needs `service_contexts` + permission checks | `davidextensionsvc` | declare `davidextension u:object_r:davidext_service:s0`, `enforceCallingPermission` for future APIs |

### HAL refactor example (points 3, 4, 5, 6)

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
    return std::nullopt;               // out-of-enum value sent over binder
}

::ndk::ScopedAStatus DavidDeviceControl::setLedColor(LedColor in_color) {
    auto native = toNative(in_color);
    if (!native) {
        return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_ARGUMENT);
    }
    std::lock_guard<std::mutex> lk(mLock);
    if (!mDeviceControl->setLedColor(*native)) {          // returns bool from writeSysfs
        return ndk::ScopedAStatus::fromServiceSpecificError(IDavidDeviceControl::ERROR_IO);
    }
    return ndk::ScopedAStatus::ok();
}
```
(`ERROR_IO` needs `const int ERROR_IO = 1;` declared in the `.aidl` → an API change → must be **V2** if V1 is already frozen.)

### Example: bumping to V2 the right way

```aidl
@VintfStability
interface IDavidDeviceControl {
    // ---- V1, order unchanged ----
    void DC_Status();
    void setLedOnOff(in LedColor color, in LedOnOff onOff);
    void setLedColor(in LedColor color);
    void setLedBlink();
    void setLedRedBlink(in LedOnOff onOff);
    void setLedWhiteBlink(in LedOnOff onOff);
    // ---- V2: only append at the end ----
    const int ERROR_IO = 1;
    LedColor getLedColor();
    void registerCallback(in IDavidDeviceControlCallback cb);
}
```
```bash
m vendor.david.hardware.daviddevicecontrol-update-api
m vendor.david.hardware.daviddevicecontrol-freeze-api      # creates aidl_api/.../2/
```
- Switch libs: `vendor.david.hardware.daviddevicecontrol-V2-ndk` (HAL), `-V2-java` (client).
- Manifest `<version>2</version>`; FCM `<version>1-2</version>` so the framework accepts older vendors.
- Old (V1) clients still work with a V2 HAL. New clients call `getInterfaceVersion()` before calling `getLedColor()` if they may meet a V1 HAL.

## 9.5 Interview questions drilling into the case study

1. *Why AIDL and not HIDL?* → HIDL is deprecated since A13; AIDL unifies IPC, supports Java/NDK/Rust, simpler versioning.
2. *Why does the HAL use the NDK backend while the client uses the Java backend? Are they compatible?* → Yes. The wire format is the same; a backend is just generated code for a language.
3. *Why is the HAL in `/vendor/bin/hw` and the client in `system_ext`?* → Treble: the HAL is a vendor implementation; the client uses hidden APIs, so it belongs in system_ext.
4. *What if `vintf_fragments` is removed?* → servicemanager rejects `addService` for a VINTF-stability binder; `waitForDeclaredService` returns null immediately; `checkvintf` fails if the FCM requires that HAL.
5. *What if the FCM is removed?* → `checkvintf` build error: a HAL in the device manifest is not in the framework compatibility matrix (non-`android.*` HALs must be declared via `DEVICE_PRODUCT_COMPATIBILITY_MATRIX_FILE`).
6. *Can a third-party app call this HAL?* → No: SELinux only grants `find` + `binder_call` to `system_server` and `system_app`; `untrusted_app` is blocked at servicemanager.
7. *What happens to the LED if the HAL crashes? Does the framework know?* → init restarts the HAL (not `oneshot`); the client gets `binderDied` → reconnects → it should **re-apply the last state** (the current code doesn't).
8. *How do you test this HAL?* → VTS-style gtest (`cc_test` with `test_suites: ["vts"]`, `AServiceManager_waitForService`, parameterized by instances from `getAidlHalInstanceNames(IDavidDeviceControl::descriptor)`), Java instrumentation tests for the client, `atest`.
