# Chương 13 — Framework: PackageManager, Permission, Context, Settings, Hidden API

> Đã kiểm chứng với source A14. Bản gốc Gemini: `gemini_raw/02_gemini_framework_part1.md` (Q23–Q36).

## 13.1 PackageManagerService

### Q1. PMS boot scan làm gì? Thứ tự quét?
- `PackageManagerService.main()` (bootstrap service trong `SystemServer.startBootstrapServices`) → `InitAppsHelper` quét các partition: `/system/framework`, `/system/priv-app`, `/system/app`, `/system_ext/{priv-app,app}`, `/product/{priv-app,app}`, `/vendor/{priv-app,app}`, `/odm/...`, `/apex`, rồi `/data/app`.
- Parse bằng `ParsingPackageUtils`, verify chữ ký, so với `packages.xml`, gán UID, cập nhật permission, quyết định dexopt.
- Boot đầu / sau OTA chậm vì phải parse lại & dexopt. Đo: `logcat -b events | grep boot_progress_pms`.

### Q2. "Computer" snapshot trong PMS (A13+)?
Truy vấn đọc (`getPackageInfo`, `queryIntentActivities`...) chạy trên snapshot bất biến (`Computer`/`ComputerEngine`) mà không cần giữ `mLock` → giảm contention; ghi (install/uninstall) tạo snapshot mới.

### Q3. Luồng cài APK qua `PackageInstaller` session?
`createSession` → staging `/data/app/vmdl<sessionId>.tmp` → `openWrite` ghi APK → `commit` → `PackageInstallerSession` verify (chữ ký, package verifier, rollback) → `InstallPackageHelper` (prepare → scan → reconcile → commit) → chuyển sang `/data/app/~~<rand>/<pkg>-<rand>/` → dexopt (A14 qua **ART Service / artd**) → ghi `packages.xml` → broadcast `PACKAGE_ADDED/REPLACED`.
- Lệnh: `adb install`, `pm install-create/-write/-commit`, `cmd package compile -m speed -f <pkg>`.

### Q4. `packages.xml` / `packages.list`?
- `/data/system/packages.xml`: package, UID (`userId`), codePath, chữ ký, flag, shared user, trạng thái component. Ghi bằng `AtomicFile`/`ResilientAtomicFile` (có backup).
- `packages.list`: text nhẹ (pkg, uid, debuggable, dataDir, seinfo, gids) cho `installd`, `run-as`, ...
- Mất `packages.xml` → PMS scan lại, có thể cấp UID khác → app không đọc được data cũ.

### Q5. APK Signature scheme v1→v4? Vì sao `sharedUserId` deprecated?
- v1 JAR (từng file), v2 (APK Signing Block, verify toàn file, A7), v3 (key rotation – proof of rotation, A9), v3.1 (A13, rotation theo SDK), v4 (`.idsig` cho incremental install qua fs-verity, A11).
- `sharedUserId` **deprecated ở API 29 (Android 10)** — chia sẻ UID phá sandbox, không thể gỡ sau khi phát hành. Nhưng app platform AOSP (Settings, SystemUI...) và app OEM như `DavidExtension` vẫn dùng `android.uid.system` ở A14. A13 thêm `android:sharedUserMaxSdkVersion` để app mới thoát khỏi shared UID.
- **⚠️ Đã sửa**: Gemini ghi "deprecated từ Android 11 (API 29)" và khuyên platform app bỏ; vòng 2 Gemini đã đồng ý sửa lại.

---

## 13.2 Permission framework

### Q6. Protection level?
| Level | Cách cấp |
|-------|---------|
| `normal` | tự cấp lúc cài |
| `dangerous` | runtime, user cấp (dialog) |
| `signature` | cùng chữ ký với package khai báo permission (thường là platform cert) |
| `signature\|privileged` | chữ ký **hoặc** app priv-app **và** có trong allowlist |
| flag `appop` | quyền + trạng thái AppOps (vd `SYSTEM_ALERT_WINDOW`, `PACKAGE_USAGE_STATS`) |
| flag `development`, `role`, `installer`, `preinstalled`... | các trường hợp đặc biệt |

- Quản lý: `PermissionManagerServiceImpl` (`services/core/java/com/android/server/pm/permission/`). Trạng thái runtime permission ở A14 do **Permission mainline module** quản lý: `/data/misc_de/<user>/apexdata/com.android.permission/runtime-permissions.xml`. **⚠️ Đã sửa**: Gemini ghi `/data/system/users/<id>/runtime-permissions.xml` (đường dẫn của bản cũ).

### Q7. Priv-app allowlist — thiếu thì sao?
- App ở `priv-app` xin quyền `privileged` phải có trong `/<partition>/etc/permissions/privapp-permissions-*.xml` (cùng partition với app).
- `ro.control_privapp_permissions=enforce` + thiếu → trong lúc boot scan, `PermissionManagerServiceImpl` log `Privileged permission X for package Y ... not in privapp-permissions allowlist` và gom vào `mPrivappPermissionsViolations` (dòng ~3360), quét xong thì ném `IllegalStateException("Signature|privileged permissions not in privapp-permissions allowlist: ...")` (dòng 4458) → system_server crash → **bootloop**. Mode `log` chỉ cảnh báo và vẫn cấp quyền. Chỉ kiểm tra lúc boot (`!mSystemReady`).
- **Ví dụ project**: `vendor/david/apps/DavidLauncher/privapp-permissions-launcher.xml` liệt kê cả quyền `normal`/`dangerous` (INTERNET, ACCESS_FINE_LOCATION...) → không cần thiết (allowlist chỉ áp dụng cho quyền privileged) nhưng vô hại; quyền thực sự cần là `REBOOT`, `MASTER_CLEAR`, `PACKAGE_USAGE_STATS`, `SET/GET_SYSTEM_PROPERTY`... Câu hỏi hay: *tìm quyền thiếu thế nào?* → `development/tools/privapp_permissions/privapp_permissions.py` hoặc log `Privileged permission ... for package ... not in privapp-permissions allowlist`.

### Q8. `default-permissions` vs privapp?
- `default-permissions/*.xml`: **tự cấp runtime permission** (dangerous) cho app hệ thống khi cài/boot đầu (`DefaultPermissionGrantPolicy`), có thể `fixed="true"` để user không thu hồi được.
- Ví dụ project: `default-launcher-permissions.xml` cấp `ACCESS_FINE_LOCATION`, `BLUETOOTH_CONNECT`... cho `com.davidmedia.app.launcher`.

### Q9. UID check vs permission check vs AppOps?
- Kernel: UID/GID (vd `AID_INET` 3003 để mở socket, group `audio`...).
- Framework: `checkCallingPermission` → `SecurityException`.
- AppOps: chặn "mềm" theo thời gian thực — có op trả lỗi, có op trả dữ liệu rỗng/im lặng (mic bị tắt → buffer toàn 0).

---

## 13.3 Context

### Q10. `Context`, `ContextWrapper`, `ContextThemeWrapper`, `ContextImpl`?
Decorator pattern: `ContextImpl` làm thật (file, binder, resources, getSystemService); `ContextWrapper` uỷ quyền cho `mBase`; `Application`/`Service` extends `ContextWrapper`; `Activity` extends `ContextThemeWrapper` (theme, configuration override).

### Q11. Dùng Application Context để tạo UI?
- Dialog → `BadTokenException` (không có window token).
- Inflate bằng Application context: không có theme của Activity, sai configuration theo display.
- A11+ StrictMode `IncorrectContextUseViolation` khi lấy `WindowManager`/display metrics từ non-UI context. Dùng `Activity` hoặc `createWindowContext()` (A11+) cho overlay từ Service.

### Q12. `SystemServiceRegistry` fetcher?
`CachedServiceFetcher` (cache theo `ContextImpl.mServiceCache`, vd `LayoutInflater`, `WindowManager`), `StaticServiceFetcher` (1 instance/process), `StaticApplicationContextServiceFetcher` (tạo bằng application context). Giữ static manager lấy từ Activity → leak Activity.

---

## 13.4 Settings & RRO

### Q13. `Settings.Global/Secure/System`?
| | Global | Secure | System |
|--|-------|--------|--------|
| Phạm vi | toàn thiết bị | theo user | theo user |
| Ghi | `WRITE_SECURE_SETTINGS` | `WRITE_SECURE_SETTINGS` | `WRITE_SETTINGS` (app thường xin qua UI) |
| File | `/data/system/users/0/settings_global.xml` | `.../users/<id>/settings_secure.xml` | `.../settings_system.xml` |
- Client cache: `Settings.NameValueCache` + **generation tracker** (shared memory `MemoryIntArray`) → chỉ IPC lại khi generation tăng.
- Lệnh: `adb shell settings get/put/list global|secure|system`.

### Q14. Default settings xuất xưởng cho STB?
- Giá trị mặc định ở `frameworks/base/packages/SettingsProvider/res/values/defaults.xml`, nạp lần đầu bởi `DatabaseHelper`/`SettingsProvider` upgrade.
- OEM override bằng **RRO** nhắm vào `com.android.providers.settings` (đặt ở `/product/overlay` hoặc `/vendor/overlay`), không sửa AOSP.
- Ví dụ RRO thật trong project: `vendor/david/apps/DavidFrameworkOverlay` (`runtime_resource_overlay`, `targetPackage="android"`, `isStatic="true"`, `priority=1000`) override `res/xml/global_keys.xml` của framework.
```bp
runtime_resource_overlay {
    name: "DavidFrameworkOverlay",
    resource_dirs: ["res"],
    sdk_version: "current",
    manifest: "AndroidManifest.xml",
    certificate: "platform",
}
```
- Debug RRO: `adb shell cmd overlay list`, `cmd overlay enable/disable <pkg>`, `idmap2 dump --idmap-path ...`.

---

## 13.5 Hidden API

### Q15. Hidden API enforcement hoạt động thế nào?
- Lúc build: `hiddenapi` tool gắn flag cho từng member trong dex của framework (sdk / unsupported / max-target-X / blocked) dựa vào `hiddenapi-flags.csv`.
- Lúc chạy: ART (`art/runtime/hidden_api.cc`) kiểm tra khi link/reflection/JNI, xác định "domain" của caller (app / platform / core-platform); app bị chặn → `NoSuchMethodException`/`NoSuchFieldError`, log `Accessing hidden method ... (blocked)`.
- App **ký platform key** / có `platform_apis: true` (như `DavidExtension`) thuộc domain platform → **không bị chặn** — đây là cách đúng cho app hệ thống OEM, không cần trick reflection.

### Q16. Tắt enforcement khi debug?
```bash
adb shell settings put global hidden_api_policy 0     # 0 = DISABLED: không chặn (chỉ dùng khi dev/test)
adb shell settings delete global hidden_api_policy    # quay về mặc định
```
Giá trị theo `ApplicationInfo.java:1542-1563` (A14): `-1` DEFAULT, `0` DISABLED, `1` JUST_WARN (cho gọi nhưng log cảnh báo), `2` ENABLED (chặn).
**⚠️ Đã sửa**: Gemini chú thích "0 = bật enforcement", sai so với source. Bản nháp đầu của tài liệu này cũng ghi nhầm, đã sửa lại sau khi tra source. Không dùng `VMRuntime.setHiddenApiExemptions` trong sản phẩm.

---

## 13.6 Bổ sung từ vòng 2 với Gemini

### Q17. A14 hạn chế Background Activity Launch (BAL) với `PendingIntent` thế nào?
- A14: khi app **gửi** PendingIntent (sender) target 34, quyền BAL của sender **không tự động** được dùng; sender phải opt-in bằng `ActivityOptions.setPendingIntentBackgroundActivityStartMode(MODE_BACKGROUND_ACTIVITY_START_ALLOWED)`.
- Phía **tạo** PendingIntent (creator) có API riêng `setPendingIntentCreatorBackgroundActivityStartMode` (A15 mới bắt buộc opt-in mặc định).
- **⚠️ Chú thích**: Gemini nói "creator phải opt-in" và "bị chặn hoàn toàn" — chưa chính xác; nhớ phân biệt creator và sender.

### Q18. `READ_MEDIA_VISUAL_USER_SELECTED` (A14)?
User chọn "Select photos and videos" → app chỉ đọc được các URI đã chọn qua MediaStore; quyền này là partial, có thể bị thu hồi. (Ít gặp trên STB.)

---

## 13.7 `android:protectionLevel` chi tiết (`signature|system`, `signatureOrSystem`, `privileged`...)

> Nguồn A14: `frameworks/base/core/res/res/values/attrs_manifest.xml:194-314` (định nghĩa flag), `core/java/android/content/pm/PermissionInfo.java` (hằng số, `fixProtectionLevel`), `services/core/java/com/android/server/pm/permission/PermissionManagerServiceImpl.java:3420-3530` (logic cấp quyền).

### Q19. `signature|system`, `signatureOrSystem`, `signature|privileged` khác nhau không?
**Hiện tại là một.** Trong `attrs_manifest.xml`:
```xml
<flag name="signature"          value="2" />
<flag name="signatureOrSystem"  value="3" />   <!-- deprecated -->
<flag name="privileged"         value="0x10" />
<flag name="system"             value="0x10" />   <!-- tên cũ của privileged, CÙNG giá trị -->
```
- `system` và `privileged` cùng giá trị `0x10`, nên `signature|system` == `signature|privileged`.
- `signatureOrSystem` (= 3) được PMS chuẩn hoá thành `signature|privileged` khi parse:
```java
// PermissionInfo.fixProtectionLevel()
if (level == PROTECTION_SIGNATURE_OR_SYSTEM) {
    level = PROTECTION_SIGNATURE | PROTECTION_FLAG_PRIVILEGED;
}
```
- Lịch sử: trước Android 4.4, "system" nghĩa là mọi app nằm trên `/system`. Từ 4.4 có `/system/priv-app`, chỉ app ở thư mục `priv-app` mới được "privileged", nên đổi tên thành `privileged` cho đúng nghĩa. **Khi phỏng vấn nên dùng `signature|privileged`.**

### Q20. PMS quyết định cấp một quyền `signature|privileged` như thế nào?
Quyền được cấp nếu **một trong hai** điều kiện đúng:
1. **Signature** (`shouldGrantPermissionBySignature`): app có chung chứng chỉ với package **định nghĩa** quyền, **hoặc** app được ký bằng chứng chỉ của package `android` (platform key). Nghĩa là: app ký platform nhận được **mọi** quyền `signature`, kể cả quyền do app khác định nghĩa.
2. **Privileged** (`shouldGrantPermissionByProtectionFlags`): app là system app nằm trong `priv-app` (`pkgSetting.isPrivileged()`), **và** quyền có trong allowlist `privapp-permissions-*.xml` (thiếu → bootloop ở mode enforce, xem Q7).

Các điểm phụ hay bị hỏi:
- App **priv-app trên `/vendor`** (hoặc `/odm`) chỉ được quyền privileged nếu quyền có thêm flag **`vendorPrivileged`**. Nếu không, log: `Permission X cannot be granted to privileged vendor apk Y because it isn't a 'vendorPrivileged' permission`. → Lý do app OEM cần quyền hệ thống nên đặt ở `system_ext`/`product`, không đặt ở `vendor`.
- **Updated system app** (cài bản cập nhật vào `/data`): chỉ giữ quyền privileged nếu bản gốc trên system image đã xin quyền đó.
- `fixProtectionLevel` cũng bỏ flag `vendorPrivileged` nếu thiếu `privileged`.

### Q21. Các flag khác của `protectionLevel` (A14)?
| Flag | Ai được cấp |
|------|-------------|
| `normal` (0) | mọi app, tự cấp lúc cài |
| `dangerous` (1) | runtime, user bấm cho phép |
| `signature` (2) | cùng chữ ký với app định nghĩa, hoặc ký platform |
| `internal` (4) | chỉ cấp qua các flag bổ sung (vd `internal\|role`) |
| `privileged` / `system` (0x10) | priv-app + allowlist |
| `development` (0x20) | cấp bằng `pm grant` (vd `WRITE_SECURE_SETTINGS` cho shell/adb) |
| `appop` (0x40) | quyền đi kèm AppOps, user bật/tắt trong Settings (`SYSTEM_ALERT_WINDOW`, `PACKAGE_USAGE_STATS`, `REQUEST_INSTALL_PACKAGES`) |
| `pre23` (0x80) | tự cấp cho app `targetSdk < 23` |
| `installer`, `verifier`, `setup`, `configurator`, `textClassifier`, `recents`, `companion`, `retailDemo`... | cấp cho đúng package giữ vai trò đó (`KnownPackages`) |
| `preinstalled` (0x400) | mọi system app (không cần priv-app) |
| `oem` (0x4000) | system app được OEM cho phép qua `<oem-permissions>` trong sysconfig |
| `vendorPrivileged` (0x8000) | cho phép priv-app trên vendor/odm nhận quyền privileged |
| `role` (0x4000000) | app giữ role tương ứng (`RoleManager`, vd role HOME, DIALER) |
| `knownSigner` (0x8000000, A12+) | app ký bằng một trong các chứng chỉ khai ở `android:knownCerts` |
| `module` | APK nằm trong APEX (mainline) |

### Q22. Định nghĩa và kiểm tra một permission riêng cho service OEM (ví dụ thật)
Giả sử `DavidExtension` mở một binder API cho các app khác điều khiển LED:
```xml
<!-- AndroidManifest.xml của app định nghĩa -->
<permission android:name="com.davidmedia.permission.CONTROL_LED"
    android:protectionLevel="signature|privileged" />
```
Kiểm tra ở phía service, **không tin** vào kiểm tra phía client:
```java
@Override public void setLedColor(int color) {
    mContext.enforceCallingOrSelfPermission(
            "com.davidmedia.permission.CONTROL_LED", "setLedColor");
    final long token = Binder.clearCallingIdentity();   // gọi HAL bằng danh tính của mình
    try { mHal.setLedColor(color); }
    finally { Binder.restoreCallingIdentity(token); }
}
```
Các cách bảo vệ khác:
- Khai trên component: `<service android:permission="...">`, `<receiver android:permission="...">`, `<provider android:readPermission/writePermission>`.
- Gửi broadcast chỉ cho app có quyền: `sendBroadcast(intent, "com.davidmedia.permission.CONTROL_LED")`.
- A14: annotation `@EnforcePermission("...")` trong file `.aidl` (vd `core/java/android/app/INotificationManager.aidl:239`). Trình biên dịch AIDL sinh code kiểm tra, lint kiểm tra service có gọi đúng hàm `..._enforcePermission()`.

Câu bẫy:
- *Định nghĩa trùng tên permission ở 2 app khác chữ ký?* → app cài sau bị từ chối `INSTALL_FAILED_DUPLICATE_PERMISSION` (chống chiếm quyền).
- *Dùng `checkCallingPermission` hay `checkCallingOrSelfPermission`?* → `checkCallingPermission` trả DENIED nếu không ở trong binder call (tránh vô tình tự cấp quyền cho chính mình). Dùng `...OrSelf` khi hàm có thể được gọi nội bộ.
- *Chỉ kiểm tra UID (`Binder.getCallingUid() == SYSTEM_UID`) được không?* → được cho API nội bộ, nhưng permission linh hoạt hơn (cấp cho app ký platform mà không cần `sharedUserId`).

### Q23. Kiểm tra trạng thái quyền trên thiết bị?
```bash
adb shell dumpsys package com.davidmedia.app.launcher | grep -A50 "install permissions"
adb shell dumpsys package permission com.davidmedia.permission.CONTROL_LED   # ai định nghĩa, protectionLevel
adb shell pm list permissions -f -g
adb shell pm grant <pkg> android.permission.WRITE_SECURE_SETTINGS           # chỉ được với quyền development
adb shell cmd appops get <pkg> SYSTEM_ALERT_WINDOW
```
