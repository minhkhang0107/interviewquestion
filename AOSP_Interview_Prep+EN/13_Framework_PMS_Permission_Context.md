# Chapter 13 — Framework: PackageManager, Permissions, Context, Settings, Hidden API

> Verified against the A14 source. Gemini's original: `gemini_raw/02_gemini_framework_part1.md` (Q23–Q36).

## 13.1 PackageManagerService

### Q1. What does the PMS boot scan do? In what order?
- `PackageManagerService.main()` (a bootstrap service in `SystemServer.startBootstrapServices`) → `InitAppsHelper` scans the partitions: `/system/framework`, `/system/priv-app`, `/system/app`, `/system_ext/{priv-app,app}`, `/product/{priv-app,app}`, `/vendor/{priv-app,app}`, `/odm/...`, `/apex`, then `/data/app`.
- Parses with `ParsingPackageUtils`, verifies signatures, compares with `packages.xml`, assigns UIDs, updates permissions, decides on dexopt.
- The first boot / boot after OTA is slow because everything must be re-parsed & dexopted. Measure: `logcat -b events | grep boot_progress_pms`.

### Q2. The "Computer" snapshot in PMS (A13+)?
Read queries (`getPackageInfo`, `queryIntentActivities`...) run against an immutable snapshot (`Computer`/`ComputerEngine`) without holding `mLock` → less contention; writes (install/uninstall) produce a new snapshot.

### Q3. The APK install flow through a `PackageInstaller` session?
`createSession` → staging in `/data/app/vmdl<sessionId>.tmp` → `openWrite` writes the APK → `commit` → `PackageInstallerSession` verifies (signature, package verifier, rollback) → `InstallPackageHelper` (prepare → scan → reconcile → commit) → moved to `/data/app/~~<rand>/<pkg>-<rand>/` → dexopt (on A14 via **ART Service / artd**) → writes `packages.xml` → broadcasts `PACKAGE_ADDED/REPLACED`.
- Commands: `adb install`, `pm install-create/-write/-commit`, `cmd package compile -m speed -f <pkg>`.

### Q4. `packages.xml` / `packages.list`?
- `/data/system/packages.xml`: packages, UID (`userId`), codePath, signatures, flags, shared users, component state. Written with `AtomicFile`/`ResilientAtomicFile` (with backup).
- `packages.list`: lightweight text (pkg, uid, debuggable, dataDir, seinfo, gids) for `installd`, `run-as`, ...
- Losing `packages.xml` → PMS rescans and may assign different UIDs → apps can't read their old data.

### Q5. APK Signature schemes v1→v4? Why is `sharedUserId` deprecated?
- v1 JAR (per file), v2 (APK Signing Block, verifies the whole file, A7), v3 (key rotation – proof of rotation, A9), v3.1 (A13, rotation targeted by SDK), v4 (`.idsig` for incremental install via fs-verity, A11).
- `sharedUserId` is **deprecated since API 29 (Android 10)** — sharing a UID breaks the sandbox and cannot be removed after release. But AOSP platform apps (Settings, SystemUI...) and OEM apps like `DavidExtension` still use `android.uid.system` on A14. A13 added `android:sharedUserMaxSdkVersion` so new apps can leave a shared UID.
- **⚠️ Corrected**: Gemini wrote "deprecated since Android 11 (API 29)" and advised platform apps to drop it; in round 2 Gemini agreed to correct it.

---

## 13.2 The permission framework

### Q6. Protection levels?
| Level | How it's granted |
|-------|---------|
| `normal` | automatically at install |
| `dangerous` | at runtime, granted by the user (dialog) |
| `signature` | same signature as the package declaring the permission (usually the platform cert) |
| `signature\|privileged` | signature **or** a priv-app **that is** in the allowlist |
| `appop` flag | permission + AppOps state (e.g. `SYSTEM_ALERT_WINDOW`, `PACKAGE_USAGE_STATS`) |
| flags `development`, `role`, `installer`, `preinstalled`... | special cases |

- Managed by: `PermissionManagerServiceImpl` (`services/core/java/com/android/server/pm/permission/`). On A14, runtime permission state is managed by the **Permission mainline module**: `/data/misc_de/<user>/apexdata/com.android.permission/runtime-permissions.xml`. **⚠️ Corrected**: Gemini wrote `/data/system/users/<id>/runtime-permissions.xml` (the path from older versions).

### Q7. Priv-app allowlist — what if it's missing?
- A `priv-app` requesting a `privileged` permission must be listed in `/<partition>/etc/permissions/privapp-permissions-*.xml` (same partition as the app).
- `ro.control_privapp_permissions=enforce` + missing → during the boot scan, `PermissionManagerServiceImpl` logs `Privileged permission X for package Y ... not in privapp-permissions allowlist` and collects it into `mPrivappPermissionsViolations` (line ~3360); after the scan it throws `IllegalStateException("Signature|privileged permissions not in privapp-permissions allowlist: ...")` (line 4458) → system_server crashes → **bootloop**. `log` mode only warns and still grants. Checked only at boot (`!mSystemReady`).
- **Project example**: `vendor/david/apps/DavidLauncher/privapp-permissions-launcher.xml` also lists `normal`/`dangerous` permissions (INTERNET, ACCESS_FINE_LOCATION...) → unnecessary (the allowlist only applies to privileged permissions) but harmless; the permissions actually needed are `REBOOT`, `MASTER_CLEAR`, `PACKAGE_USAGE_STATS`, `SET/GET_SYSTEM_PROPERTY`... Good question: *how do you find missing permissions?* → `development/tools/privapp_permissions/privapp_permissions.py` or the log `Privileged permission ... for package ... not in privapp-permissions allowlist`.

### Q8. `default-permissions` vs privapp?
- `default-permissions/*.xml`: **auto-grants runtime (dangerous) permissions** to system apps at install/first boot (`DefaultPermissionGrantPolicy`); `fixed="true"` prevents the user from revoking them.
- Project example: `default-launcher-permissions.xml` grants `ACCESS_FINE_LOCATION`, `BLUETOOTH_CONNECT`... to `com.davidmedia.app.launcher`.

### Q9. UID checks vs permission checks vs AppOps?
- Kernel: UID/GID (e.g. `AID_INET` 3003 to open sockets, the `audio` group...).
- Framework: `checkCallingPermission` → `SecurityException`.
- AppOps: "soft" real-time blocking — some ops return errors, some return empty/silent data (mic disabled → a buffer of zeros).

---

## 13.3 Context

### Q10. `Context`, `ContextWrapper`, `ContextThemeWrapper`, `ContextImpl`?
Decorator pattern: `ContextImpl` does the real work (files, binder, resources, getSystemService); `ContextWrapper` delegates to `mBase`; `Application`/`Service` extend `ContextWrapper`; `Activity` extends `ContextThemeWrapper` (theme, configuration override).

### Q11. Using the Application Context to create UI?
- Dialog → `BadTokenException` (no window token).
- Inflating with the Application context: no Activity theme, wrong per-display configuration.
- A11+ StrictMode `IncorrectContextUseViolation` when getting `WindowManager`/display metrics from a non-UI context. Use the `Activity`, or `createWindowContext()` (A11+) for overlays from a Service.

### Q12. `SystemServiceRegistry` fetchers?
`CachedServiceFetcher` (cached in `ContextImpl.mServiceCache`, e.g. `LayoutInflater`, `WindowManager`), `StaticServiceFetcher` (1 instance/process), `StaticApplicationContextServiceFetcher` (created with the application context). Holding a static manager obtained from an Activity → leaks the Activity.

---

## 13.4 Settings & RRO

### Q13. `Settings.Global/Secure/System`?
| | Global | Secure | System |
|--|-------|--------|--------|
| Scope | whole device | per user | per user |
| Write | `WRITE_SECURE_SETTINGS` | `WRITE_SECURE_SETTINGS` | `WRITE_SETTINGS` (normal apps request it via UI) |
| File | `/data/system/users/0/settings_global.xml` | `.../users/<id>/settings_secure.xml` | `.../settings_system.xml` |
- Client cache: `Settings.NameValueCache` + a **generation tracker** (shared memory `MemoryIntArray`) → IPC happens again only when the generation increases.
- Commands: `adb shell settings get/put/list global|secure|system`.

### Q14. Factory default settings for an STB?
- Defaults are in `frameworks/base/packages/SettingsProvider/res/values/defaults.xml`, loaded the first time by `DatabaseHelper`/`SettingsProvider` upgrade.
- OEMs override them with an **RRO** targeting `com.android.providers.settings` (placed in `/product/overlay` or `/vendor/overlay`), without modifying AOSP.
- A real RRO in the project: `vendor/david/apps/DavidFrameworkOverlay` (`runtime_resource_overlay`, `targetPackage="android"`, `isStatic="true"`, `priority=1000`) overrides the framework's `res/xml/global_keys.xml`.
```bp
runtime_resource_overlay {
    name: "DavidFrameworkOverlay",
    resource_dirs: ["res"],
    sdk_version: "current",
    manifest: "AndroidManifest.xml",
    certificate: "platform",
}
```
- Debugging RROs: `adb shell cmd overlay list`, `cmd overlay enable/disable <pkg>`, `idmap2 dump --idmap-path ...`.

---

## 13.5 Hidden API

### Q15. How does hidden API enforcement work?
- At build time: the `hiddenapi` tool tags every member in the framework's dex (sdk / unsupported / max-target-X / blocked) based on `hiddenapi-flags.csv`.
- At run time: ART (`art/runtime/hidden_api.cc`) checks on linking/reflection/JNI and determines the caller's "domain" (app / platform / core-platform); a blocked app → `NoSuchMethodException`/`NoSuchFieldError`, log `Accessing hidden method ... (blocked)`.
- Apps **signed with the platform key** / with `platform_apis: true` (like `DavidExtension`) are in the platform domain → **not blocked** — the proper way for OEM system apps, no reflection tricks needed.

### Q16. Disabling enforcement for debugging?
```bash
adb shell settings put global hidden_api_policy 0     # 0 = DISABLED: no blocking (dev/test only)
adb shell settings delete global hidden_api_policy    # back to default
```
Values per `ApplicationInfo.java:1542-1563` (A14): `-1` DEFAULT, `0` DISABLED, `1` JUST_WARN (allowed but logs a warning), `2` ENABLED (blocked).
**⚠️ Corrected**: Gemini annotated "0 = enforcement on", which contradicts the source. The first draft of this document was also wrong and was fixed after checking the source. Do not use `VMRuntime.setHiddenApiExemptions` in products.

---

## 13.6 Additions from round 2 with Gemini

### Q17. How does A14 restrict Background Activity Launch (BAL) with `PendingIntent`?
- A14: when an app **sends** a PendingIntent (the sender) targeting 34, the sender's BAL privilege is **not used automatically**; the sender must opt in with `ActivityOptions.setPendingIntentBackgroundActivityStartMode(MODE_BACKGROUND_ACTIVITY_START_ALLOWED)`.
- The side that **creates** the PendingIntent (the creator) has its own API `setPendingIntentCreatorBackgroundActivityStartMode` (opt-in becomes required by default only in A15).
- **⚠️ Note**: Gemini said "the creator must opt in" and "completely blocked" — not accurate; remember to distinguish creator from sender.

### Q18. `READ_MEDIA_VISUAL_USER_SELECTED` (A14)?
The user picks "Select photos and videos" → the app can read only the selected URIs via MediaStore; this permission is partial and can be revoked. (Rare on STBs.)

---

## 13.7 `android:protectionLevel` in detail (`signature|system`, `signatureOrSystem`, `privileged`...)

> A14 sources: `frameworks/base/core/res/res/values/attrs_manifest.xml:194-314` (flag definitions), `core/java/android/content/pm/PermissionInfo.java` (constants, `fixProtectionLevel`), `services/core/java/com/android/server/pm/permission/PermissionManagerServiceImpl.java:3420-3530` (grant logic).

### Q19. Are `signature|system`, `signatureOrSystem`, `signature|privileged` different?
**Today they are the same.** In `attrs_manifest.xml`:
```xml
<flag name="signature"          value="2" />
<flag name="signatureOrSystem"  value="3" />   <!-- deprecated -->
<flag name="privileged"         value="0x10" />
<flag name="system"             value="0x10" />   <!-- old name of privileged, SAME value -->
```
- `system` and `privileged` share the value `0x10`, so `signature|system` == `signature|privileged`.
- `signatureOrSystem` (= 3) is normalized by PMS into `signature|privileged` when parsed:
```java
// PermissionInfo.fixProtectionLevel()
if (level == PROTECTION_SIGNATURE_OR_SYSTEM) {
    level = PROTECTION_SIGNATURE | PROTECTION_FLAG_PRIVILEGED;
}
```
- History: before Android 4.4, "system" meant any app on `/system`. 4.4 introduced `/system/priv-app`, and only apps in `priv-app` are "privileged", so the flag was renamed `privileged` to match. **In an interview, say `signature|privileged`.**

### Q20. How does PMS decide to grant a `signature|privileged` permission?
The permission is granted if **either** condition holds:
1. **Signature** (`shouldGrantPermissionBySignature`): the app shares a certificate with the package that **defines** the permission, **or** the app is signed with the `android` package's certificate (platform key). In other words: a platform-signed app receives **every** `signature` permission, even those defined by other apps.
2. **Privileged** (`shouldGrantPermissionByProtectionFlags`): the app is a system app in `priv-app` (`pkgSetting.isPrivileged()`), **and** the permission is in the `privapp-permissions-*.xml` allowlist (missing → bootloop in enforce mode, see Q7).

Side points often asked:
- A **priv-app on `/vendor`** (or `/odm`) only gets a privileged permission if the permission also has the **`vendorPrivileged`** flag. Otherwise it logs: `Permission X cannot be granted to privileged vendor apk Y because it isn't a 'vendorPrivileged' permission`. → That's why OEM apps needing system permissions belong in `system_ext`/`product`, not `vendor`.
- **Updated system apps** (an update installed into `/data`): keep a privileged permission only if the original on the system image already requested it.
- `fixProtectionLevel` also drops the `vendorPrivileged` flag if `privileged` is missing.

### Q21. Other `protectionLevel` flags (A14)?
| Flag | Who gets it |
|------|-------------|
| `normal` (0) | every app, granted at install |
| `dangerous` (1) | at runtime, when the user allows it |
| `signature` (2) | same signature as the defining app, or platform-signed |
| `internal` (4) | only granted via additional flags (e.g. `internal\|role`) |
| `privileged` / `system` (0x10) | priv-app + allowlist |
| `development` (0x20) | granted with `pm grant` (e.g. `WRITE_SECURE_SETTINGS` for shell/adb) |
| `appop` (0x40) | permission tied to AppOps, toggled by the user in Settings (`SYSTEM_ALERT_WINDOW`, `PACKAGE_USAGE_STATS`, `REQUEST_INSTALL_PACKAGES`) |
| `pre23` (0x80) | auto-granted to apps with `targetSdk < 23` |
| `installer`, `verifier`, `setup`, `configurator`, `textClassifier`, `recents`, `companion`, `retailDemo`... | granted to the package holding that role (`KnownPackages`) |
| `preinstalled` (0x400) | every system app (doesn't need to be a priv-app) |
| `oem` (0x4000) | system apps the OEM allows via `<oem-permissions>` in sysconfig |
| `vendorPrivileged` (0x8000) | lets priv-apps on vendor/odm receive privileged permissions |
| `role` (0x4000000) | the app holding the matching role (`RoleManager`, e.g. the HOME, DIALER roles) |
| `knownSigner` (0x8000000, A12+) | apps signed with one of the certificates listed in `android:knownCerts` |
| `module` | APKs inside an APEX (mainline) |

### Q22. Defining and checking a custom permission for an OEM service (real example)
Suppose `DavidExtension` exposes a binder API letting other apps control the LED:
```xml
<!-- AndroidManifest.xml of the defining app -->
<permission android:name="com.davidmedia.permission.CONTROL_LED"
    android:protectionLevel="signature|privileged" />
```
Check on the service side; **never** trust client-side checks:
```java
@Override public void setLedColor(int color) {
    mContext.enforceCallingOrSelfPermission(
            "com.davidmedia.permission.CONTROL_LED", "setLedColor");
    final long token = Binder.clearCallingIdentity();   // call the HAL with our own identity
    try { mHal.setLedColor(color); }
    finally { Binder.restoreCallingIdentity(token); }
}
```
Other ways to protect:
- On components: `<service android:permission="...">`, `<receiver android:permission="...">`, `<provider android:readPermission/writePermission>`.
- Send a broadcast only to apps holding the permission: `sendBroadcast(intent, "com.davidmedia.permission.CONTROL_LED")`.
- A14: the `@EnforcePermission("...")` annotation in the `.aidl` file (e.g. `core/java/android/app/INotificationManager.aidl:239`). The AIDL compiler generates the check code, and lint verifies the service calls the matching `..._enforcePermission()`.

Trick questions:
- *Defining the same permission name in 2 apps with different signatures?* → the later install is rejected with `INSTALL_FAILED_DUPLICATE_PERMISSION` (prevents permission hijacking).
- *`checkCallingPermission` or `checkCallingOrSelfPermission`?* → `checkCallingPermission` returns DENIED when not inside a binder call (avoids accidentally granting yourself the permission). Use `...OrSelf` when the method may be called internally.
- *Can you just check the UID (`Binder.getCallingUid() == SYSTEM_UID`)?* → fine for internal APIs, but a permission is more flexible (granted to platform-signed apps without needing `sharedUserId`).

### Q23. Checking permission state on the device?
```bash
adb shell dumpsys package com.davidmedia.app.launcher | grep -A50 "install permissions"
adb shell dumpsys package permission com.davidmedia.permission.CONTROL_LED   # who defines it, protectionLevel
adb shell pm list permissions -f -g
adb shell pm grant <pkg> android.permission.WRITE_SECURE_SETTINGS           # only works for development permissions
adb shell cmd appops get <pkg> SYSTEM_ALERT_WINDOW
```
