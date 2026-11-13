# Chapter 05 — HAL: Legacy HAL → HIDL → AIDL HAL, VINTF

## 5.1 HAL history

| Era | Form | Characteristics |
|---------|------|----------|
| ≤ Android 7 | **Legacy HAL** (`hw_module_t`, `hw_get_module()`, `.so` in `/system/lib/hw`) | `dlopen`ed directly into the framework process → no system/vendor separation |
| Android 8–10 | **HIDL** (`.hal`, `android.hardware.foo@1.0`) | binderized (`/dev/hwbinder`, hwservicemanager) or passthrough |
| Android 11+ | **Stable AIDL HAL** (`@VintfStability`) | uses `/dev/binder` + `servicemanager`, one IPC language for both framework & HALs |
| Android 13+ | HIDL **deprecated** for new HALs | every new HAL must be AIDL |

---

### Q1. Compare HIDL and AIDL HALs

| | HIDL | AIDL HAL |
|--|------|----------|
| File | `.hal` | `.aidl` |
| Driver | `/dev/hwbinder` | `/dev/binder` |
| Service manager | `hwservicemanager` | `servicemanager` |
| Versioning | package version `@1.0`, `@1.1` (interface inheritance) | version numbers `V1`, `V2` in the same package, **frozen API dumps** |
| Instance name | `IFoo/default` via `registerAsService()` | `"<descriptor>/default"` via `AServiceManager_addService()` |
| Backend | C++ / Java | NDK / CPP / Java / Rust |
| Tool | `lshal` | `service list`, `dumpsys -l`, `lshal` (still lists AIDL) |
| Passthrough | yes (`-impl.so` + `HIDL_FETCH_IFoo`) | no (binderized only) |
| Status | deprecated | current standard |

---

### Q2. Passthrough vs Binderized HAL?

- **Binderized**: the HAL runs in its own process (`/vendor/bin/hw/...-service`), communicates over IPC → fault isolation, its own SELinux domain.
- **Passthrough** (HIDL): a wrapper around a legacy HAL `.so`; the client `dlopen`s it (`getService(true)`) → same process. Still exists for **same-process HALs (SP-HAL)** such as `android.hardware.graphics.mapper` (for performance).
- AIDL has no passthrough; code that must be in-process uses a normal library (or, for the A14 mapper HAL, the stable-C `IMapper`).

---

### Q3. Steps to create a new AIDL HAL from scratch (classic question)

Using the project's own `vendor.david.hardware.daviddevicecontrol`:

**Step 1 — Define the interface** `vendor/david/interfaces/daviddevicecontrol/aidl/vendor/david/hardware/daviddevicecontrol/IDavidDeviceControl.aidl`
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

**Step 2 — `aidl_interface`** (Android.bp in the same directory, see Chapter 04 Q6), `stability: "vintf"`.

**Step 3 — Generate/freeze the API**
```bash
m vendor.david.hardware.daviddevicecontrol-update-api   # creates aidl_api/<name>/current/
m vendor.david.hardware.daviddevicecontrol-freeze-api   # creates aidl_api/<name>/1/ + .hash, adds versions_with_info to bp
```

**Step 4 — Implement the service (NDK)**
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

**Step 5 — Build the binary**: `cc_binary` with `vendor: true`, `relative_install_path: "hw"`, `init_rc`, `vintf_fragments` (Chapter 03 Q3).

**Step 6 — init rc** `vendor.david.hardware.daviddevicecontrol-service.rc` (Chapter 02 Q12).

**Step 7 — VINTF device manifest fragment**
```xml
<manifest version="1.0" type="device" target-level="8">
    <hal format="aidl">
        <name>vendor.david.hardware.daviddevicecontrol</name>
        <version>1</version>
        <interface><name>IDavidDeviceControl</name><instance>default</instance></interface>
    </hal>
</manifest>
```
(Since A12, AIDL HALs may use `<fqname>IDavidDeviceControl/default</fqname>` instead of the `<interface>` block.)

**Step 8 — Framework compatibility matrix (FCM) for a vendor-defined HAL**
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

**Step 9 — SELinux** (Chapter 07): domain, exec type, file_contexts, service.te, service_contexts, sysfs permissions, client `find` + `binder_call` permissions.

**Step 10 — PRODUCT_PACKAGES**
```make
PRODUCT_PACKAGES += vendor.david.hardware.daviddevicecontrol-service
```
(In reality `david.mk` also adds `-V1-ndk`, `-V1-java` and the interface name — unnecessary because they are already dependencies of the service/app.)

**Step 11 — Client**
- Java: `IDavidDeviceControl.Stub.asInterface(ServiceManager.waitForDeclaredService(IDavidDeviceControl.DESCRIPTOR + "/default"))`.
- NDK: `AServiceManager_waitForService(instance)` → `IDavidDeviceControl::fromBinder(ndk::SpAIBinder(...))`.

**Step 12 — Verify**
```bash
adb shell service list | grep david
adb shell dumpsys -l | grep david
adb shell lshal | grep david
adb shell ps -AZ | grep daviddevicecontrol      # check the domain is u:r:daviddevicecontrol:s0
adb shell cat /vendor/etc/vintf/manifest/*daviddevicecontrol*
adb logcat -s daviddevicecontrol DeviceControl
```

---

### Q4. What is VINTF? Device manifest vs Framework compatibility matrix?

- The **VINTF object** = information that guarantees system and vendor are compatible:
  - **Device manifest** (`/vendor/etc/vintf/manifest.xml` + fragments in `manifest/`): which HALs/versions the vendor **provides**, kernel version, sepolicy version, `target-level` (FCM version).
  - **Framework compatibility matrix (FCM)** (`/system/etc/vintf/compatibility_matrix.xml`): which HALs the framework **requires** (optional/required, version ranges).
  - The reverse direction: **Framework manifest** (HALs/services system provides) and **Device compatibility matrix** (what vendor requires from the framework, e.g. VNDK/sepolicy version).
- Checked: at build time (`checkvintf`, `assemble_vintf`), during OTA (`update_engine` checks), at runtime (`vintf` tool, `VintfObject`).
- `target-level="8"` ↔ FCM level 8 = Android 14 (7 = A13, 6 = A12, 5 = A11).

```bash
adb shell vintf                    # dump runtime VINTF, check compatibility
adb shell cat /vendor/etc/vintf/manifest.xml
```

---

### Q5. Stable AIDL versioning: frozen, current, hash — rules for changes?

- When version N is **frozen**: `aidl_api/<name>/N/` contains the dump + `.hash` → **must not be edited** anymore.
- `current/` is the in-development version (N+1 if there are changes).
- Backward-compatible change rules (for a new version):
  - ✅ Add methods **at the end** of the interface, add fields **at the end** of a parcelable (with defaults), add enumerators, add constants.
  - ❌ Delete/rename/reorder methods, change parameter types, remove fields.
- A client calling a new method on an old-version server → gets `UNKNOWN_TRANSACTION` → use `getInterfaceVersion()` to check.
- The server must implement the highest version it declares in the manifest.
- **Project observation**: `aidl_api/vendor.david.hardware.daviddevicecontrol/` only has `current/`, **no `1/`** (never frozen), while the manifest declares `<version>1</version>`. An unfrozen interface still builds (V1 = current) but stability is **not guaranteed**; for an official release run `-freeze-api` so every later change is caught by the build check.

---

### Q6. Why might a vendor HAL's `AServiceManager_addService` fail even though the code is correct?

Debug checklist:
1. **Not in the VINTF manifest** → log `servicemanager: Could not find vendor.david...IDavidDeviceControl/default in the VINTF manifest`.
2. **SELinux**: missing `add_service`/`service_contexts` → `avc: denied { add } for service=... scontext=u:r:daviddevicecontrol:s0 tcontext=u:object_r:default_android_service:s0`.
3. **Stability**: the interface isn't `@VintfStability` or was built with the wrong backend → `Cannot do a user transaction on a ... stability binder`.
4. The manifest version differs from the lib's actual version (`-V1-ndk` vs `-V2-ndk`).
5. The process runs in the wrong domain (missing file_contexts → runs under the `init` domain or exec is blocked: `avc denied { execute_no_trans }`).

---

### Q7. What is a lazy HAL?

- The HAL runs only while it has clients and exits when it has none → saves RAM.
- AIDL: use `AServiceManager_registerLazyService()` instead of `addService`; the rc has `interface aidl <name>/<instance>`, `oneshot`, `disabled`.
- When servicemanager gets a `getService` for a service that isn't running → sets `ctl.interface_start aidl/<name>/<instance>` → init starts the service.
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

### Q8. Which user should a HAL run as? How should it access hardware?

- **Least privilege**: avoid `root`; use a dedicated user or `system`, groups matching the device nodes (`audio`, `media`...), grant specific `capabilities` only when needed.
- `/dev`, `/sys` file permissions come from `ueventd.rc` (mode/owner) + SELinux labels via `genfs_contexts`/`file_contexts`.
- Project: the HAL runs as `user system` and writes the LED sysfs (`/sys/class/leds/...`) — the right approach; but the `led.sh`, `wifi.sh` scripts in `init.amlogic.board.rc` run as `user root` → room for improvement.

---

### Q9. Must a HAL be thread-safe? How do callbacks from the HAL to the framework work?

- Yes — the binder thread pool may call concurrently → protect state/sysfs writes with a `std::mutex`.
- Callbacks: define `oneway interface IFooCallback` in the same package; the framework passes the callback via `registerCallback(in IFooCallback cb)`; the HAL stores a `std::shared_ptr<IFooCallback>` and links to death to remove it automatically when the client dies.
```cpp
::ndk::ScopedAStatus Hal::registerCallback(const std::shared_ptr<IFooCallback>& cb) {
    std::lock_guard<std::mutex> lk(mLock);
    AIBinder_linkToDeath(cb->asBinder().get(), mDeathRecipient.get(), this);
    mCallbacks.push_back(cb);
    return ndk::ScopedAStatus::ok();
}
```

---

### Q10. How do you migrate a HIDL HAL to AIDL?

1. Write the equivalent `.aidl` (tool `hidl2aidl -o out -r vendor.amlogic:vendor/amlogic/... vendor.amlogic.hardware.systemcontrol@1.1`).
2. `aidl_interface` with `stability: "vintf"`, freeze V1.
3. Implement the NDK backend, register with servicemanager.
4. Change the manifest `format="hidl"` → `format="aidl"`, and the FCM accordingly.
5. SELinux: `hal_attribute`, `hwservice_contexts` → `service_contexts`, `hal_foo_hwservice` → `hal_foo_service`.
6. Java clients switch from `IFoo.getService()` (HIDL) to `ServiceManager.waitForDeclaredService()`.
- Example in the project: `DavidExtension` still uses HIDL `vendor.amlogic.hardware.systemcontrol-V1.0/1.1-java` alongside David's new AIDL HAL → evidence of the transition period.
