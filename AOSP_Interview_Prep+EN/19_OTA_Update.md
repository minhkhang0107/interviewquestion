# Chapter 19 — OTA: A/B, Virtual A/B, update_engine

> Based on AOSP 14 (`system/update_engine`, `build/make/tools/releasetools`, `system/core/fs_mgr/libsnapshot`). Items marked ✅ **were cross-checked against the project source/`out/`**.

## 19.0 ✅ OTA in the KSTB2500 project

- **Virtual A/B with compression (VABC)**: `device/amlogic/KSTB2500/KSTB2500.mk` sets `AB_OTA_UPDATER := true`, inherits `virtual_ab_ota/compression.mk` + `virtual_ab_ota/launch_with_vendor_ramdisk.mk`, `PRODUCT_USE_DYNAMIC_PARTITIONS := true`. The generated `vendor/build.prop` has `ro.virtual_ab.enabled=true`, `ro.virtual_ab.compression.enabled=true`, `ro.virtual_ab.userspace.snapshots.enabled=true` (snapuserd), `ro.virtual_ab.io_uring.enabled=true`, `ro.virtual_ab.batch_writes=true`.
- **Partitions in the payload** (`ro.product.ab_ota_partitions`): `boot, bootloader, dtbo, init_boot, odm, odm_dlkm, odm_ext, oem, product, system, system_dlkm, system_ext, vbmeta, vbmeta_system, vendor, vendor_boot, vendor_dlkm` — notably **`bootloader` is included** (Amlogic updates the bootloader via A/B). No separate `recovery`: the recovery ramdisk lives in `vendor_boot` (GKI, `BOARD_INCLUDE_RECOVERY_RAMDISK_IN_VENDOR_BOOT`).
- **AVB anti-rollback**: `BOARD_AVB_BOOT_ROLLBACK_INDEX := $(PLATFORM_SECURITY_PATCH_TIMESTAMP)`, `ROLLBACK_INDEX_LOCATION := 3` → the boot rollback index is tied to the security patch level.
- **Release signing**: `signing_for_release.sh` → `vendor/david/scripts/release/build_release_package.sh`: `sign_target_files_apks -o -d <certs>` (with `--key_mapping` and many `--extra_apks` for APEX/Mainline APKs) → `ota_from_target_files --package_key <certs>/releasekey` → `img_from_target_files`. `make_vba_otapackage.sh` supports `--partial` and adding the `oem` image.
- **Updater app**: `vendor/amlogic/reference/apps/Updater/.../UpdateManager.java` uses `UpdateEngine.bind()`, `applyPayload()`, `resetStatus()`; `otacerts.zip` is present in `/system/etc/security/`; `update_verifier` is started by `bootable/recovery/update_verifier/update_verifier.rc`.

## 19.1 The three update types

| | Non-A/B (legacy) | A/B (seamless) | Virtual A/B (A11+, compressed since A12 — VABC) |
|---|---|---|---|
| Written where | overwrites the current partitions in **recovery** | the inactive slot (`_a`/`_b`) while the device keeps running | **COW snapshots** of dynamic partitions, stored in `/data` (and free space in `super`) |
| Downtime | several minutes on the recovery screen | a single reboot | a single reboot (+ background merge after boot) |
| Rollback on a bad boot | no (can brick) | yes (bootloader switches back to the old slot) | yes (drop the snapshot) |
| Space cost | lowest | double partitions | almost no doubling; needs free space on `/data` |
| Components | `recovery`, `uncrypt`, `updater` (edify) | `update_engine`, boot_control HAL | plus `libsnapshot`, `snapuserd`, `dm-snapshot`/`dm-user` |

---

### Q1. How is an OTA package produced from a build?

```bash
m dist                                     # → out/dist/<product>-target_files-*.zip
# Sign with release keys (NEVER ship testkeys)
sign_target_files_apks -o -d $KEYS target_files.zip signed-target_files.zip
# Full OTA
ota_from_target_files -k $KEYS/releasekey signed-target_files.zip full_ota.zip
# Incremental (delta) from an older build
ota_from_target_files -k $KEYS/releasekey -i old-signed-target_files.zip \
                      signed-target_files.zip incr_ota.zip
```
- An A/B package contains: `payload.bin` (per-partition operations: REPLACE, REPLACE_XZ, SOURCE_COPY, SOURCE_BSDIFF, PUFFDIFF, ZERO...; COW ops for VABC), `payload_properties.txt` (`FILE_HASH`, `FILE_SIZE`, `METADATA_HASH`, `METADATA_SIZE`), `care_map.pb`, `META-INF/com/android/metadata(.pb)` (pre-device, pre-build, post-timestamp...).
- `payload.bin` is **stored uncompressed** in the zip → it can be streamed directly by offset.
- Keys: `PRODUCT_DEFAULT_DEV_CERTIFICATE`, `otacerts.zip` on the device (`/system/etc/security/otacerts.zip`) contains the certs used to verify packages. Shipping testkeys = **anyone can sign an OTA for your box**.
- Useful flags: `--partial "boot vendor"` (only some partitions), `--downgrade` (requires a wipe), `--wipe_user_data`, `--skip_postinstall`, `--vabc_compression_param`.

---

### Q2. The A/B update flow on the device (deep dive)

```
OEM Updater app (system app)
  └─ android.os.UpdateEngine.applyPayload(url, offset, size, headerKeyValuePairs)   [@SystemApi]
        ── binder ──► update_engine (native daemon started by init, domain update_engine)
1. Download/stream the payload, verify the metadata signature with the certs in otacerts.zip
2. Check preconditions (source fingerprint for deltas, timestamp for anti-downgrade)
3. boot_control HAL: markSlotUnbootable(target_slot)
4. Apply each operation to the target slot (or build COW snapshots for Virtual A/B)
5. Verify the hash of each target partition
6. Postinstall: mount the new slot, run the postinstall script (e.g. otapreopt → dexopt apps for the new build)
7. boot_control HAL: setActiveBootSlot(target_slot)   → callback onPayloadApplicationComplete(SUCCESS)
The updater requests a reboot (usually waits for the user/standby)
Bootloader: boots the new slot with a retry count (typically 3–7 attempts)
init: update_verifier reads every block in care_map to trigger dm-verity
      → OK: markBootSuccessful()   |  verity errors/repeated crashes → retries exhausted → bootloader returns to the old slot
Virtual A/B: after sys.boot_completed, snapuserd + libsnapshot merge the snapshot into the base partitions
```

App callbacks:
```kotlin
updateEngine.bind(object : UpdateEngineCallback() {
    override fun onStatusUpdate(status: Int, percent: Float) { /* DOWNLOADING, VERIFYING, FINALIZING, UPDATED_NEED_REBOOT */ }
    override fun onPayloadApplicationComplete(errorCode: Int) { /* UpdateEngine.ErrorCodeConstants.SUCCESS ... */ }
})
updateEngine.applyPayload(url, payloadOffset, payloadSize, propsLinesFromPayloadPropertiesTxt)
```
- Other APIs: `suspend()`, `resume()`, `cancel()`, `resetStatus()`, `allocateSpace()` (Virtual A/B, checks for enough room on `/data` before downloading), `cleanupAppliedPayload()`.
- Access: update_engine's binder service is protected by **SELinux** (who may `find` it) — the updater must be a system app with the matching policy.

---

### Q3. Virtual A/B and merging: why is it needed and what are the risks?

- Dynamic partitions (`system`, `vendor`, `product`, `system_ext`...) live in `super`. Virtual A/B does not allocate two full copies; it creates **snapshots**: new image = old base + a COW holding the changed blocks.
- After booting the new slot, reads go through `dm-snapshot` (A11) or `dm-user` + the **`snapuserd`** daemon (VABC, A12+) to combine base + COW.
- Once boot succeeds → **merge**: write the COW into the base, then delete the snapshot. During the merge:
  - The next OTA cannot be applied (update_engine reports a merge in progress).
  - Power loss mid-merge is **safe** (the merge is checkpointed and resumes on the next boot).
  - **Rollback to the old slot is no longer possible** because the base has been overwritten.
- Risk: not enough space on `/data` for the COW → `allocateSpace` returns the number of bytes needed; the updater must ask the user to free storage. On an 8GB-eMMC STB this is a real problem.
- Debug: `snapshotctl dump`, `logcat -s update_engine snapuserd libsnapshot`, `update_engine_client --follow`.

---

### Q4. How do rollback and anti-downgrade work?

- **Automatic rollback (A/B)**: the new slot is never `markBootSuccessful`'d and its retry count hits 0 → the bootloader selects the old slot. This is the safety net for boot loops.
- **OTA anti-downgrade**: `ota_from_target_files` records `post-timestamp` (`ro.build.date.utc`); update_engine rejects payloads older than the running build (unless built with `--downgrade` + data wipe).
- **AVB rollback index**: each vbmeta carries a rollback index; the bootloader stores the minimum accepted value in tamper-evident storage (RPMB/eFuse). The index is only updated **after** the new slot is marked successful. Purpose: block flashing old builds with known vulnerabilities.
- **Security patch level** & KeyMint: lowering the SPL can make Keystore keys unusable (keys are bound to OS version/patch level) → a reason not to allow downgrades even when technically possible.

---

### Q5. How does a non-A/B (recovery) OTA work? (still seen on older STBs)

1. The app calls `RecoverySystem.installPackage(context, file)`.
2. If the package is on encrypted `/data`: `uncrypt` creates a **block map** (`/cache/recovery/block.map`) so recovery can read raw blocks without decrypting the filesystem.
3. A command is written to the **BCB** (bootloader control block in the `misc` partition) + `/cache/recovery/command`, then reboot into recovery.
4. Recovery verifies the signature and runs `updater` executing the **edify** script (`updater-script`) in `META-INF/com/google/android/`.
5. Power loss midway: recovery has a resume mechanism (stash), but bootloader/partition-table failures can **brick** the device.

---

### Q6. Testing an OTA locally without a server?

```bash
# Script shipped with AOSP: streams over adb
python3 system/update_engine/scripts/update_device.py --file incr_ota.zip
# Or manually:
adb push payload.bin /data/ota_package/
adb shell update_engine_client --update \
     --payload=file:///data/ota_package/payload.bin \
     --headers="$(cat payload_properties.txt)"
adb shell update_engine_client --follow             # follow progress
adb logcat -s update_engine update_verifier
adb shell bootctl get-current-slot ; adb shell bootctl get-suffix 0   # (if bootctl exists)
```
update_engine logs are also kept in `/data/misc/update_engine_log/`; progress prefs in `/data/misc/update_engine/prefs` (allowing **resume** after power loss/kill).

---

### Q7. Design an OTA system for an operator's 1 million STBs? (system design)

- **Server**: build catalog keyed by `device/model/fingerprint`; serve **deltas from the last N builds** + a full package as fallback; CDN; offline signing with an HSM.
- **Client (updater system app)**:
  - Periodic check via `JobScheduler` (network required), sending fingerprint + serial + rollout group.
  - **Staged rollout**: 1% → 10% → 100%, with a server-side kill switch.
  - Stream the payload directly (no need to download to `/data` first) → saves storage; with Virtual A/B call `allocateSpace` first.
  - Apply while the box is **in standby/idle**, throttle bandwidth; reboot at off-peak hours (e.g. 3 a.m.) or when the user turns the box off.
  - Report status to the server: the `onPayloadApplicationComplete` error code, the post-OTA boot result (was `markBootSuccessful` reached, was there a rollback).
- **Safety**: A/B + `update_verifier` + rollback; a post-boot health check (Launcher/TIF/DRM L1 still working?) before declaring success; anti-rollback index.
- **Separate channels**: APEX/Mainline updates via Play (GMS devices); OEM apps can be updated as standalone APKs without a full OTA.

---

### Q8. Common OTA failures & causes

| Symptom | Usual cause |
|---|---|
| `kPayloadHashMismatchError` / `kDownloadPayloadVerificationError` | corrupted download, wrong offset/size, signed with a key not in `otacerts.zip` |
| `kDownloadStateInitializationError` / source mismatch | delta built from a different source build (fingerprint differs), or source partitions modified (remount/root) |
| `kPayloadTimestampError` | downgrade |
| `kNotEnoughSpace` (Virtual A/B) | not enough room on `/data` for the COW |
| Boot loop after reboot, then back on the old build | new slot broken → A/B rollback worked as designed; check `last_kmsg`/`pstore`, `ro.boot.slot_suffix` |
| OTA succeeds but apps crash afterwards | data format changes without migration; postinstall dexopt skipped → slow first boot |

Tip: `adb remount` (disables verity) on a test device changes partition hashes → **later delta OTAs will fail**; always reflash cleanly before testing incrementals.

---

### Q9. The server pushes OTA commands over MQTT and the box downloads over Wi-Fi: how do you design the STB client?

> Scenario question: "The operator runs an MQTT broker for device management. When new firmware is released the server publishes an OTA command; the box, on Wi-Fi, must download and update itself. How do you build the Android side?"

- **What MQTT does: it is only the control plane. It does not carry the firmware.**
  - The message carries only metadata: `{version, fingerprint_from, url, size, sha256, payload_properties (FILE_HASH, FILE_SIZE, METADATA_HASH, METADATA_SIZE), offset, rollout_id, deadline}`.
  - The `payload.bin` itself is fetched over HTTPS/CDN (the data plane). A broker is not built to move multi-GB files.
- **Topics and QoS**:
  - The box subscribes to its own topic `stb/<serial>/ota/cmd` and to the group topic `stb/<model>/ota/broadcast`.
  - The box publishes its state to `stb/<serial>/ota/status`.
  - Commands use **QoS 1** with a persistent session (`cleanSession=false`), so a box that was offline when the server sent still receives the command on reconnect. Commands must be **idempotent** and carry a `command_id` so duplicates can be dropped.
  - Use **LWT (Last Will)** so the server learns when a box drops mid-update. Use a **retained message** for "the latest build for this model".
- **Security**:
  - Use TLS on 8883 with **mutual TLS** and a per-box client certificate. That certificate is provisioned at the factory, ideally with its key in Keystore/TEE.
  - Apply per-serial topic ACLs.
  - The MQTT command is **not the trust anchor**. Firmware integrity is enforced by `update_engine`, which checks the payload signature against `otacerts.zip` and checks the hashes. Someone who forges a command still cannot install a foreign package, and timestamp and rollback-index checks block downgrades.
- **Android side (system app / privileged service)**:
  - A **foreground service** (or persistent system app) holds the MQTT connection with a Paho/HiveMQ client, using a ~60s keepalive and reconnecting with exponential backoff.
  - Watch the network with `ConnectivityManager.registerDefaultNetworkCallback`. Reconnect when Wi-Fi drops, and start downloading only once the network is `VALIDATED`.
  - When a command arrives, check `fingerprint_from == Build.FINGERPRINT` (for delta packages) and check free space. On Virtual A/B, call `UpdateEngine.allocateSpace()` first.
  - Call `UpdateEngine.applyPayload(url, offset, size, headerKeyValuePairs)`. This is **streaming** mode: update_engine downloads over HTTP(S) and writes straight into slot B, with no copy in `/data`. Note: if you need resume or bandwidth control, you can download to `/data/ota_package` yourself and pass a `file://` URL instead.
  - Use `onStatusUpdate` (DOWNLOADING → VERIFYING → FINALIZING) and `onPayloadApplicationComplete(errorCode)` to publish progress and error codes on the status topic. Throttle these messages to one every 5–10%.
  - Once the state is `UPDATED_NEED_REBOOT`, wait for standby or an off-peak window before rebooting. After boot, wait for `markBootSuccessful`, then publish `{result: success, new_fingerprint}`. If the device rolled back, report that the old build is still running.
- **Wi-Fi failure cases to handle**:
  - Wi-Fi drops mid-download: update_engine retries HTTP on its own. If it still fails, call `cancel()` or `resetStatus()` and apply again. Virtual A/B keeps checkpoints, so the download does not restart from zero.
  - Standby or doze kills the TCP connection and with it the MQTT session: use a sensible keepalive and hold a `WifiLock` while downloading.
  - Captive portal or broken DNS: download only when the network has `NET_CAPABILITY_VALIDATED`.
  - Millions of boxes receiving one broadcast cause a **thundering herd** on the CDN: each box waits a **random jitter** before downloading, or the server rolls out in cohorts (1% → 10% → 100%).
- **One-line answer under pressure**: "MQTT to *push commands and collect reports* (push, lightweight, reaches boxes behind NAT); HTTPS/CDN to *download the payload*; `update_engine` to *apply and verify*; A/B + `markBootSuccessful` for *safe rollback*."

---

## 19.2 Rapid-fire

1. **What is `AB_OTA_PARTITIONS`?** → The list of partitions included in the A/B payload (boot, system, vendor, vbmeta, dtbo...).
2. **What does the boot_control HAL do?** → `getCurrentSlot`, `setActiveBootSlot`, `markBootSuccessful`, `setSlotAsUnbootable`... for update_engine and init.
3. **What is `update_verifier` for?** → It reads blocks listed in `care_map` so dm-verity checks the whole new slot before `markBootSuccessful`.
4. **What is postinstall for?** → Running a program from the new slot before switching (usually `otapreopt` to compile apps → fast first boot).
5. **Why is the payload stored uncompressed in the zip?** → So update_engine can stream it over HTTP by offset without unzipping.
6. **Does a factory reset undo an applied OTA?** → No; the active slot stays on the new build. A reset during a Virtual A/B merge requires the merge to finish before wiping (libsnapshot handles this).
7. **How does GKI affect OTA?** → The kernel (boot) and vendor modules (`vendor_boot`, `vendor_dlkm`) can be updated more independently.
