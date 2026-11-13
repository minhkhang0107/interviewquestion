# Chương 19 — OTA: A/B, Virtual A/B, update_engine

> Viết dựa trên AOSP 14 (`system/update_engine`, `build/make/tools/releasetools`, `system/core/fs_mgr/libsnapshot`). Các điểm có dấu ✅ **đã đối chiếu với source/`out/` của project**.

## 19.0 ✅ OTA trong project KSTB2500

- **Virtual A/B có nén (VABC)**: `device/amlogic/KSTB2500/KSTB2500.mk` đặt `AB_OTA_UPDATER := true`, kế thừa `virtual_ab_ota/compression.mk` + `virtual_ab_ota/launch_with_vendor_ramdisk.mk`, `PRODUCT_USE_DYNAMIC_PARTITIONS := true`. `vendor/build.prop` sinh ra có `ro.virtual_ab.enabled=true`, `ro.virtual_ab.compression.enabled=true`, `ro.virtual_ab.userspace.snapshots.enabled=true` (snapuserd), `ro.virtual_ab.io_uring.enabled=true`, `ro.virtual_ab.batch_writes=true`.
- **Partition trong payload** (`ro.product.ab_ota_partitions`): `boot, bootloader, dtbo, init_boot, odm, odm_dlkm, odm_ext, oem, product, system, system_dlkm, system_ext, vbmeta, vbmeta_system, vendor, vendor_boot, vendor_dlkm` — đáng chú ý là **có cả `bootloader`** (Amlogic cập nhật bootloader qua A/B). Không có `recovery` riêng: recovery ramdisk nằm trong `vendor_boot` (GKI, `BOARD_INCLUDE_RECOVERY_RAMDISK_IN_VENDOR_BOOT`).
- **Anti-rollback AVB**: `BOARD_AVB_BOOT_ROLLBACK_INDEX := $(PLATFORM_SECURITY_PATCH_TIMESTAMP)`, `ROLLBACK_INDEX_LOCATION := 3` → rollback index của boot gắn với security patch level.
- **Ký release**: `signing_for_release.sh` → `vendor/david/scripts/release/build_release_package.sh`: `sign_target_files_apks -o -d <certs>` (kèm `--key_mapping`, nhiều `--extra_apks` cho APK của APEX/Mainline) → `ota_from_target_files --package_key <certs>/releasekey` → `img_from_target_files`. `make_vba_otapackage.sh` hỗ trợ `--partial` và thêm ảnh `oem`.
- **Updater app**: `vendor/amlogic/reference/apps/Updater/.../UpdateManager.java` dùng `UpdateEngine.bind()`, `applyPayload()`, `resetStatus()`; `otacerts.zip` có ở `/system/etc/security/`; `update_verifier` khởi động bằng `bootable/recovery/update_verifier/update_verifier.rc`.

## 19.1 Ba kiểu cập nhật

| | Non-A/B (legacy) | A/B (seamless) | Virtual A/B (A11+, nén từ A12 — VABC) |
|---|---|---|---|
| Ghi vào đâu | ghi đè partition hiện tại trong **recovery** | slot không hoạt động (`_a`/`_b`) khi máy vẫn chạy | **snapshot COW** của partition dynamic, lưu trong `/data` (và vùng trống của `super`) |
| Downtime | vài phút ở màn hình recovery | chỉ 1 lần reboot | chỉ 1 lần reboot (+ merge nền sau boot) |
| Rollback khi boot lỗi | không (có thể brick) | có (bootloader chuyển về slot cũ) | có (bỏ snapshot) |
| Tốn dung lượng | ít nhất | gấp đôi partition | gần như không gấp đôi; cần chỗ trống trên `/data` |
| Thành phần | `recovery`, `uncrypt`, `updater` (edify) | `update_engine`, boot_control HAL | thêm `libsnapshot`, `snapuserd`, `dm-snapshot`/`dm-user` |

---

### Q1. Từ build đến gói OTA được tạo thế nào?

```bash
m dist                                     # → out/dist/<product>-target_files-*.zip
# Ký bằng release key (KHÔNG dùng testkey cho sản phẩm thật)
sign_target_files_apks -o -d $KEYS target_files.zip signed-target_files.zip
# Full OTA
ota_from_target_files -k $KEYS/releasekey signed-target_files.zip full_ota.zip
# Incremental (delta) từ bản cũ
ota_from_target_files -k $KEYS/releasekey -i old-signed-target_files.zip \
                      signed-target_files.zip incr_ota.zip
```
- Gói A/B chứa: `payload.bin` (các operation cho từng partition: REPLACE, REPLACE_XZ, SOURCE_COPY, SOURCE_BSDIFF, PUFFDIFF, ZERO...; với VABC thêm COW ops), `payload_properties.txt` (`FILE_HASH`, `FILE_SIZE`, `METADATA_HASH`, `METADATA_SIZE`), `care_map.pb`, `META-INF/com/android/metadata(.pb)` (pre-device, pre-build, post-timestamp...).
- `payload.bin` được lưu **không nén** (stored) trong zip → có thể stream trực tiếp bằng offset.
- Key: `PRODUCT_DEFAULT_DEV_CERTIFICATE`, `otacerts.zip` trên thiết bị (`/system/etc/security/otacerts.zip`) chứa cert dùng để verify gói. Dùng nhầm testkey = **bất kỳ ai cũng ký được OTA cho box**.
- Các cờ hay dùng: `--partial "boot vendor"` (chỉ vài partition), `--downgrade` (bắt buộc kèm wipe), `--wipe_user_data`, `--skip_postinstall`, `--vabc_compression_param`.

---

### Q2. Luồng A/B update trên thiết bị (đi sâu)

```
OEM Updater app (system app)
  └─ android.os.UpdateEngine.applyPayload(url, offset, size, headerKeyValuePairs)   [@SystemApi]
        ── binder ──► update_engine (daemon native, chạy từ init, domain update_engine)
1. Tải/stream payload, verify chữ ký metadata bằng cert trong otacerts.zip
2. Kiểm tra pre-condition (fingerprint nguồn với delta, timestamp chống downgrade)
3. boot_control HAL: markSlotUnbootable(target_slot)
4. Áp từng operation vào slot đích (hoặc tạo snapshot COW nếu Virtual A/B)
5. Verify hash từng partition đích
6. Postinstall: mount slot mới, chạy script postinstall (vd. otapreopt → dexopt app cho bản mới)
7. boot_control HAL: setActiveBootSlot(target_slot)   → callback onPayloadApplicationComplete(SUCCESS)
Updater yêu cầu reboot (thường đợi người dùng/standby)
Bootloader: boot slot mới với retry count (thường 3–7 lần)
init: update_verifier đọc toàn bộ block trong care_map để kích hoạt dm-verity
      → OK: markBootSuccessful()   |  lỗi verity/crash nhiều lần → hết retry → bootloader về slot cũ
Virtual A/B: sau sys.boot_completed, snapuserd + libsnapshot merge snapshot vào base partition
```

Callback cho app:
```kotlin
updateEngine.bind(object : UpdateEngineCallback() {
    override fun onStatusUpdate(status: Int, percent: Float) { /* DOWNLOADING, VERIFYING, FINALIZING, UPDATED_NEED_REBOOT */ }
    override fun onPayloadApplicationComplete(errorCode: Int) { /* UpdateEngine.ErrorCodeConstants.SUCCESS ... */ }
})
updateEngine.applyPayload(url, payloadOffset, payloadSize, propsLinesFromPayloadPropertiesTxt)
```
- Các API khác: `suspend()`, `resume()`, `cancel()`, `resetStatus()`, `allocateSpace()` (Virtual A/B, kiểm tra đủ chỗ trên `/data` trước khi tải), `cleanupAppliedPayload()`.
- Quyền: binder service của update_engine được bảo vệ bởi **SELinux** (ai được `find` service) — updater phải là system app có policy tương ứng.

---

### Q3. Virtual A/B và merge: tại sao cần, có rủi ro gì?

- Partition dynamic (`system`, `vendor`, `product`, `system_ext`...) nằm trong `super`. Virtual A/B không cấp đủ hai bản, mà tạo **snapshot**: bản mới = base cũ + COW chứa các block thay đổi.
- Sau khi boot slot mới, đọc partition đi qua `dm-snapshot` (A11) hoặc `dm-user` + daemon **`snapuserd`** (VABC, A12+) để ghép base + COW.
- Khi boot thành công → **merge**: ghi COW vào base, sau đó xoá snapshot. Trong lúc merge:
  - Không áp được OTA tiếp theo (update_engine báo đang merge).
  - Mất điện giữa merge **an toàn** (merge có checkpoint, tiếp tục ở lần boot sau).
  - **Không được rollback** về slot cũ nữa vì base đã bị ghi.
- Rủi ro: `/data` không đủ chỗ cho COW → `allocateSpace` trả về số byte cần; updater phải báo người dùng dọn bộ nhớ. Trên STB 8GB eMMC đây là vấn đề thật.
- Debug: `snapshotctl dump`, `logcat -s update_engine snapuserd libsnapshot`, `dumpsys` không có — dùng `update_engine_client --follow`.

---

### Q4. Rollback và chống downgrade hoạt động thế nào?

- **Rollback tự động (A/B)**: slot mới chưa `markBootSuccessful` mà retry count về 0 → bootloader chọn slot cũ. Đây là lưới an toàn khi boot loop.
- **Chống downgrade OTA**: `ota_from_target_files` ghi `post-timestamp` (`ro.build.date.utc`); update_engine từ chối payload có timestamp cũ hơn bản đang chạy (trừ khi tạo với `--downgrade` + wipe data).
- **AVB rollback index**: mỗi vbmeta có rollback index; bootloader lưu giá trị nhỏ nhất được chấp nhận trong bộ nhớ chống giả mạo (RPMB/eFuse). Chỉ cập nhật index **sau khi** slot mới được đánh dấu boot thành công. Mục đích: chặn flash bản cũ có lỗ hổng.
- **Security patch level** & KeyMint: hạ SPL có thể làm khoá trong Keystore không dùng được (key gắn với OS version/patch level) → lý do không nên cho downgrade dù kỹ thuật làm được.

---

### Q5. Non-A/B (recovery) OTA chạy thế nào? (còn gặp trên STB đời cũ)

1. App gọi `RecoverySystem.installPackage(context, file)`.
2. Nếu gói nằm trên `/data` mã hoá: `uncrypt` tạo **block map** (`/cache/recovery/block.map`) để recovery đọc được block thô mà không cần giải mã FS.
3. Ghi lệnh vào **BCB** (bootloader control block trong partition `misc`) + `/cache/recovery/command`, reboot vào recovery.
4. Recovery verify chữ ký, chạy `updater` thực thi script **edify** (`updater-script`) trong `META-INF/com/google/android/`.
5. Mất điện giữa chừng: recovery có cơ chế resume (stash), nhưng lỗi bootloader/partition table có thể làm **brick**.

---

### Q6. Thử nghiệm OTA cục bộ không cần server?

```bash
# Script có sẵn trong AOSP: stream qua adb
python3 system/update_engine/scripts/update_device.py --file incr_ota.zip
# Hoặc thủ công:
adb push payload.bin /data/ota_package/
adb shell update_engine_client --update \
     --payload=file:///data/ota_package/payload.bin \
     --headers="$(cat payload_properties.txt)"
adb shell update_engine_client --follow             # theo dõi tiến độ
adb logcat -s update_engine update_verifier
adb shell bootctl get-current-slot ; adb shell bootctl get-suffix 0   # (nếu có bootctl)
```
Log update_engine còn lưu ở `/data/misc/update_engine_log/`; prefs tiến độ ở `/data/misc/update_engine/prefs` (cho phép **resume** sau khi mất điện/kill).

---

### Q7. Thiết kế hệ thống OTA cho 1 triệu STB của nhà mạng? (system design)

- **Server**: catalog build theo `device/model/fingerprint`; cung cấp **delta từ N bản gần nhất** + full làm dự phòng; CDN; ký offline bằng HSM.
- **Client (updater system app)**:
  - Check định kỳ bằng `JobScheduler` (yêu cầu mạng), gửi fingerprint + serial + nhóm rollout.
  - **Staged rollout**: 1% → 10% → 100%, có kill switch phía server.
  - Stream thẳng payload (không cần tải về `/data` trước) → tiết kiệm bộ nhớ; với Virtual A/B gọi `allocateSpace` trước.
  - Áp payload khi box **standby/nhàn rỗi**, giới hạn băng thông; reboot vào giờ thấp điểm (ví dụ 3h sáng) hoặc khi người dùng tắt box.
  - Báo cáo trạng thái về server: mã lỗi `onPayloadApplicationComplete`, kết quả boot sau OTA (đã `markBootSuccessful` chưa, có rollback không).
- **An toàn**: A/B + `update_verifier` + rollback; healthcheck sau boot (Launcher/TIF/DRM L1 vẫn chạy?) trước khi coi là thành công; anti-rollback index.
- **Đường riêng**: APEX/Mainline cập nhật qua Play (thiết bị GMS); app OEM có thể cập nhật bằng APK riêng, không cần full OTA.

---

### Q8. Lỗi OTA hay gặp & nguyên nhân

| Triệu chứng | Nguyên nhân thường gặp |
|---|---|
| `kPayloadHashMismatchError` / `kDownloadPayloadVerificationError` | gói bị hỏng khi tải, sai offset/size, ký bằng key khác `otacerts.zip` |
| `kDownloadStateInitializationError` / mismatch source | delta tạo từ bản nguồn khác bản đang chạy (fingerprint khác), hoặc partition nguồn bị sửa (remount/root) |
| `kPayloadTimestampError` | downgrade |
| `kNotEnoughSpace` (Virtual A/B) | `/data` thiếu chỗ cho COW |
| Boot loop sau reboot rồi tự quay về bản cũ | slot mới lỗi → rollback A/B hoạt động đúng; xem `last_kmsg`/`pstore`, `ro.boot.slot_suffix` |
| OTA ok nhưng app crash sau update | thay đổi data format, thiếu migration; dexopt postinstall bị skip → first boot chậm |

Mẹo: `adb remount` (tắt verity) trên máy test làm partition khác hash → **delta OTA sau đó sẽ fail**; luôn flash lại sạch trước khi test incremental.

---

### Q9. Server đẩy lệnh OTA qua MQTT, box tải gói qua Wi-Fi: thiết kế client trên STB thế nào?

> Câu hỏi tình huống: "Nhà mạng có MQTT broker để quản lý thiết bị. Khi có firmware mới, server publish lệnh OTA; box đang dùng Wi-Fi phải tự tải và cập nhật. Bạn làm phía Android thế nào?"

- **Vai trò của MQTT: chỉ là kênh điều khiển (control plane), không dùng để tải firmware.**
  - Message chỉ chứa metadata: `{version, fingerprint_from, url, size, sha256, payload_properties (FILE_HASH, FILE_SIZE, METADATA_HASH, METADATA_SIZE), offset, rollout_id, deadline}`.
  - Gói `payload.bin` thì tải qua HTTPS/CDN (data plane). Broker không được thiết kế để chuyển file hàng GB.
- **Topic và QoS**:
  - Box subscribe topic riêng `stb/<serial>/ota/cmd` và topic nhóm `stb/<model>/ota/broadcast`.
  - Box publish trạng thái lên `stb/<serial>/ota/status`.
  - Lệnh dùng **QoS 1** kèm `persistent session` (`cleanSession=false`), để box offline lúc server gửi vẫn nhận được khi kết nối lại. Lệnh phải **idempotent**, có `command_id` để bỏ qua bản trùng.
  - Dùng **LWT (Last Will)** để server biết box mất kết nối giữa chừng. Dùng **retained message** cho "bản mới nhất của model".
- **Bảo mật**:
  - Dùng TLS 8883 + **mutual TLS** với client cert riêng từng box. Cert này provision ở nhà máy; tốt nhất key nằm trong Keystore/TEE.
  - Phân quyền topic (ACL) theo serial.
  - Lệnh MQTT **không phải là nguồn tin cậy**. Tính toàn vẹn của firmware do `update_engine` đảm bảo: nó kiểm chữ ký payload bằng `otacerts.zip` và kiểm hash. Kẻ giả lệnh cũng không cài được gói lạ, và downgrade bị chặn bởi timestamp và rollback index.
- **Phía Android (system app / privileged service)**:
  - Một **foreground service** (hoặc persistent system app) giữ kết nối MQTT bằng Paho/HiveMQ client, với keepalive khoảng 60s và reconnect theo exponential backoff.
  - Theo dõi mạng bằng `ConnectivityManager.registerDefaultNetworkCallback`. Khi Wi-Fi rớt thì reconnect; khi mạng `VALIDATED` mới bắt đầu tải.
  - Nhận lệnh xong thì kiểm `fingerprint_from == Build.FINGERPRINT` (với gói delta) và kiểm dung lượng. Với Virtual A/B, gọi `UpdateEngine.allocateSpace()` trước.
  - Gọi `UpdateEngine.applyPayload(url, offset, size, headerKeyValuePairs)`. Đây là chế độ **streaming**: update_engine tự tải HTTP(S) và ghi thẳng vào slot B, không cần lưu file vào `/data`. Ghi chú: nếu cần resume hay kiểm soát băng thông, có thể tự tải về `/data/ota_package` rồi truyền `file://`.
  - Từ `onStatusUpdate` (DOWNLOADING → VERIFYING → FINALIZING) và `onPayloadApplicationComplete(errorCode)`, publish tiến độ và mã lỗi lên topic status. Không publish quá dày: chỉ gửi mỗi 5–10%.
  - Hoàn tất (`UPDATED_NEED_REBOOT`) thì chờ box standby hoặc giờ thấp điểm mới reboot. Sau khi boot, đợi `markBootSuccessful` rồi publish `{result: success, new_fingerprint}`. Nếu bị rollback thì báo bản đang chạy vẫn là bản cũ.
- **Các lỗi mạng Wi-Fi cần xử lý**:
  - Wi-Fi chập chờn giữa lúc tải: update_engine tự retry HTTP. Nếu thất bại thì gọi `cancel()` hoặc `resetStatus()` rồi apply lại. Với Virtual A/B có checkpoint nên không phải tải lại từ đầu.
  - Box ngủ (doze/standby) làm rớt TCP, gây mất MQTT: dùng keepalive hợp lý và giữ `WifiLock` khi đang tải.
  - Captive portal hoặc DNS lỗi: chỉ tải khi network có `NET_CAPABILITY_VALIDATED`.
  - Nhiều triệu box cùng nhận broadcast sẽ gây **thundering herd** lên CDN: trước khi tải, box chờ một khoảng **random jitter**, hoặc server chia rollout theo nhóm (1% → 10% → 100%).
- **Câu trả lời ngắn khi bị hỏi dồn**: "MQTT để *đẩy lệnh và nhận báo cáo* (push, nhẹ, nhận được cả khi box sau NAT); HTTPS/CDN để *tải payload*; `update_engine` để *áp và xác thực*; A/B + `markBootSuccessful` để *rollback an toàn*."

---

## 19.2 Rapid-fire

1. **`AB_OTA_PARTITIONS` là gì?** → Danh sách partition có trong payload A/B (boot, system, vendor, vbmeta, dtbo...).
2. **boot_control HAL làm gì?** → `getCurrentSlot`, `setActiveBootSlot`, `markBootSuccessful`, `setSlotAsUnbootable`... cho update_engine và init.
3. **`update_verifier` để làm gì?** → Đọc block theo `care_map` để dm-verity kiểm tra toàn bộ slot mới trước khi `markBootSuccessful`.
4. **Postinstall dùng làm gì?** → Chạy chương trình trên slot mới trước khi switch (thường là `otapreopt` để compile app → first boot nhanh).
5. **Vì sao payload stored (không nén) trong zip?** → Để update_engine stream qua HTTP bằng offset, không cần giải nén.
6. **Factory reset có xoá OTA đã áp?** → Không; slot đã active vẫn là slot mới. Nhưng reset giữa lúc merge Virtual A/B thì merge phải hoàn tất trước khi wipe (libsnapshot xử lý).
7. **GKI ảnh hưởng OTA?** → Kernel (boot) và module vendor (`vendor_boot`, `vendor_dlkm`) có thể cập nhật tách biệt hơn.
