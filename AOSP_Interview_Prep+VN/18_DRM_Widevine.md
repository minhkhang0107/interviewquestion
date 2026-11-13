# Chương 18 — DRM & Widevine (trọng tâm STB/TV)

> Viết dựa trên AOSP 14 và tài liệu công khai của Android/Widevine. Các điểm có dấu ✅ **đã đối chiếu với source/`out/` của project** (Widevine CDM/OEMCrypto là code đóng nên chỉ kiểm tra được cấu hình build và file đầu ra).

## 18.0 ✅ DRM trong project KSTB2500 (dùng khi kể kinh nghiệm)

| Thành phần | Thực tế trong project |
|---|---|
| Widevine HAL | đóng gói dạng **vendor APEX** `com.google.android.widevine.nonupdatable` (prebuilt, `TARGET_BUILD_WIDEVINE := nonupdatable` trong `device/amlogic/common/core_amlogic.mk`), vì `BOARD_WIDEVINE_CDM_AIDL` không bật → không có binary `drm-service.widevine` rời trong `/vendor/bin/hw` |
| Mức bảo mật | `BOARD_WIDEVINE_OEMCRYPTO_LEVEL ?= 1` (`media.mk`) → **L1**, cài `liboemcrypto.so` vào `/vendor/lib*` |
| TEE | **OP-TEE** (`TARGET_USE_OPTEEOS := true`, daemon `tee-supplicant`); Widevine TA: `/vendor/lib/teetz/e043cde0-61d0-11e5-9c26-0002a5d5c51b.ta` |
| Secure video path | `BOARD_OMX_WITH_OPTEE_TVP := true` (TVP chạy trên OP-TEE) |
| Công cụ nhà máy | `tee_provision`, `tee_key_inject` (nạp key/keybox), `tee_hdcp`, `tee_attest` trong `/vendor/bin` |
| HAL DRM khác đã build | `android.hardware.drm-service.clearkey`, `android.hardware.drm-service.playready` (`BOARD_PLAYREADY_LEVEL=1` = PlayReady + OP-TEE + TVP), `android.hardware.cas-service.example` |
| Có trong cấu hình nhưng tuỳ cờ | Verimatrix, FairPlay, YouTube DRM (`media.mk`); `BUILD_WITH_WIDEVINECAS := false` |
| Decoder secure | `c2.amlogic.{avc,hevc,vp9,av1,mpeg2}.decoder.secure` (trong `media_codecs*.xml` của `out/`) |

## 18.1 Bức tranh tổng thể

```
App (ExoPlayer/Media3, Netflix, YouTube...)
  │  MediaDrm (session, license)          MediaCodec + MediaCrypto (giải mã + decode)
  ▼                                        ▼
frameworks/av: libmediadrm (DrmHal / CryptoHal)  ── binder ──►  mediadrmserver / mediaswcodec / media.codec
  ▼  (AIDL HAL android.hardware.drm từ A13, trước đó HIDL drm@1.x)
Vendor: android.hardware.drm-service.widevine   (IDrmFactory → IDrmPlugin / ICryptoPlugin)
  ▼  Widevine CDM (libwvhidl / libwvaidl...)
liboemcrypto.so  (OEM/SoC hiện thực OEMCrypto API)
  ▼  gọi vào TEE (OP-TEE / secure OS của SoC)
Trusted App Widevine trong TEE: giữ keybox/cert, giải mã nội dung vào secure buffer
  ▼
Secure video decoder (đọc secure buffer) → secure video path → HDMI (HDCP)
```

- **MediaDrm**: quản lý session, provisioning, license (key request/response).
- **MediaCrypto**: gắn session DRM vào `MediaCodec` để codec giải mã sample đã mã hoá.
- **DRM HAL plugin**: mỗi scheme (Widevine, ClearKey, PlayReady...) là một plugin, nhận dạng bằng **UUID** (Widevine `edef8ba9-79d6-4ace-a3c8-27dcd51d21ed`).
- **CAS** (Conditional Access, `MediaCas` + HAL `android.hardware.cas`) là thứ khác: dùng cho **broadcast** (DVB/ATSC) với Descrambler, thường đi kèm Tuner HAL. Đây là điểm hay bị hỏi trên STB.

---

### Q1. Widevine L1, L2, L3 khác nhau thế nào?

| Level | Giải mã (crypto) | Decode video | Output | Nội dung được phép (thường gặp) |
|---|---|---|---|---|
| **L1** | trong TEE | trong vùng bảo mật (secure decoder, secure buffer) | secure path + HDCP | HD/4K/HDR của Netflix, Prime, Disney+ |
| **L2** | trong TEE | ngoài TEE (normal world) | | hiếm gặp |
| **L3** | phần mềm (không TEE) | normal world | | thường giới hạn SD (480p/540p) |

Kiểm tra trên máy:
```kotlin
val drm = MediaDrm(WIDEVINE_UUID)
drm.getPropertyString("securityLevel")        // "L1" / "L3"
drm.getPropertyString("systemId")
drm.getPropertyString("hdcpLevel"); drm.getPropertyString("maxHdcpLevel")
drm.connectedHdcpLevel; drm.maxHdcpLevel       // API 28+
```
Pitfall: box ra L3 thay vì L1 thường do **keybox chưa nạp / hỏng**, TEE TA không chạy, hoặc `liboemcrypto.so` không load được (SELinux, sai đường dẫn, sai ABI). App như Netflix sẽ tự hạ chất lượng xuống SD mà không báo lỗi rõ ràng.

---

### Q2. Luồng phát một nội dung Widevine (license flow) từ góc độ app?

```kotlin
val drm = MediaDrm(WIDEVINE_UUID)
val sessionId = drm.openSession()
// initData = PSSH box lấy từ manifest DASH (<ContentProtection>) hoặc từ moov/pssh của MP4
val req = drm.getKeyRequest(sessionId, psshInitData, "video/mp4",
                            MediaDrm.KEY_TYPE_STREAMING, hashMapOf())
val resp = httpPost(licenseServerUrl, req.data)      // việc của app: gửi kèm token/auth
drm.provideKeyResponse(sessionId, resp)

val crypto = MediaCrypto(WIDEVINE_UUID, sessionId)
codec.configure(format, surface, crypto, 0)
// mỗi sample mã hoá:
codec.queueSecureInputBuffer(index, 0, cryptoInfo /* keyId, iv, mode, subsamples */, ptsUs, 0)
```
- `CryptoInfo.mode`: `CRYPTO_MODE_AES_CTR` (**cenc**) hoặc `CRYPTO_MODE_AES_CBC` + pattern (**cbcs**, dùng cho HLS/FairPlay-compatible). Subsample: phần header NAL để clear, chỉ payload bị mã hoá.
- Ngoại lệ hay gặp: `NotProvisionedException` (phải provision trước), `DeniedByServerException`, `MediaCodec.CryptoException` với `ERROR_NO_KEY`, `ERROR_KEY_EXPIRED`, `ERROR_INSUFFICIENT_OUTPUT_PROTECTION` (HDCP không đủ), `ERROR_RESOURCE_BUSY`/`ResourceBusyException` (hết session).
- Lắng nghe: `setOnEventListener` (`EVENT_KEY_REQUIRED` để renew license), `setOnKeyStatusChangeListener`, `setOnExpirationUpdateListener`.
- Offline (download): `KEY_TYPE_OFFLINE` → lưu `keySetId`, sau dùng `restoreKeys`; giải phóng bằng `KEY_TYPE_RELEASE`.
- Thực tế dùng Media3: `DefaultDrmSessionManager` + `HttpMediaDrmCallback` + `FrameworkMediaDrm`, app chỉ cấu hình `MediaItem.DrmConfiguration(C.WIDEVINE_UUID).setLicenseUri(...)`.

---

### Q3. Provisioning là gì? Keybox, Provisioning 3.0/4.0?

Thiết bị cần một **danh tính gốc** để license server tin tưởng:
- **Keybox (Provisioning 2.0)**: 128 byte gồm device ID + device key + magic, cấp cho từng thiết bị, **nạp ở nhà máy** vào secure storage (TEE). Lần đầu chạy, CDM dùng keybox để xin **Device Certificate (DRM cert)** từ Google provisioning server (`getProvisionRequest()` → POST → `provideProvisionResponse()`).
- **Provisioning 3.0 (OEM certificate)**: thay keybox bằng OEM cert trong TEE.
- **Provisioning 4.0**: dựa trên **Boot Certificate Chain (BCC)** / chuỗi khoá từ bootloader (gắn với Remote Key Provisioning của KeyMint) — hướng mới cho thiết bị đời mới; không còn nạp keybox từng máy.
- Cert sau khi provision lưu ở `/data/vendor/mediadrm/...` (theo cách hiện thực) → **factory reset là phải provision lại**; mất mạng lúc provision thì app báo lỗi.

Câu hỏi bẫy: *"Keybox có được log ra khi debug không?"* → Không bao giờ; keybox lộ thì phải **revoke** cả lô thiết bị (system ID). Tool nạp keybox ở nhà máy phải chạy qua TA, không ghi keybox thô ra filesystem normal world.

---

### Q4. Secure video path hoạt động thế nào? Vì sao chụp màn hình video DRM ra màu đen?

1. Sample mã hoá được copy vào **secure buffer** (vùng nhớ chỉ TEE/HW decoder truy cập — DMA-BUF heap bảo mật / carveout, trên Amlogic thường gọi là **TVP — Trusted Video Path**, quản lý qua `codec_mm`).
2. TEE giải mã **tại chỗ** trong secure memory; CPU normal world không đọc được plaintext.
3. Decoder `*.secure` (✅ trong project: `c2.amlogic.avc.decoder.secure`, `...hevc...`, `...vp9...`, `...av1...`) đọc secure buffer, output ra **protected buffer** (`GRALLOC_USAGE_PROTECTED`).
4. HWC/display composition đưa thẳng ra plane video (overlay), HDMI bật **HDCP**.

Hệ quả:
- Phải render vào **`SurfaceView`** (hoặc `setSecure` surface); `TextureView` / đọc pixel qua GPU **không được phép** → màn hình đen.
- `screencap`, screen recording, `MediaProjection` thấy đen ở vùng video — **đúng thiết kế**, không phải bug.
- App đặt `FLAG_SECURE` cho window để chặn screenshot UI.
- Tunneled playback (Chương 15 Q7) thường đi kèm với secure decoder trên STB.

Chọn decoder secure: `MediaCodecInfo.CodecCapabilities.FEATURE_SecurePlayback`, hoặc `MediaCrypto.requiresSecureDecoderComponent(mime)`.

---

### Q5. HDCP liên quan gì tới DRM?

- License có thể đặt **output protection** (ví dụ 4K yêu cầu HDCP 2.2, HD yêu cầu HDCP 1.x).
- CDM hỏi trạng thái HDCP hiện tại (qua OEMCrypto → driver HDMI/HDCP của SoC). Không đủ → `ERROR_INSUFFICIENT_OUTPUT_PROTECTION` hoặc server cấp key độ phân giải thấp hơn.
- Tình huống STB: cắm qua AV receiver/splitter cũ chỉ hỗ trợ HDCP 1.4 → 4K không phát, HD vẫn phát. Debug: `getConnectedHdcpLevel()`, log của driver HDCP, `dumpsys media.drm`.

---

### Q6. AIDL DRM HAL: interface chính và tiến trình nào chạy nó?

- Từ **Android 13**, DRM HAL là **AIDL**: package `android.hardware.drm`, interface `IDrmFactory` (`createDrmPlugin`, `createCryptoPlugin`, `isCryptoSchemeSupported`...), `IDrmPlugin` (session, key request/response, provision, property), `ICryptoPlugin` (`decrypt` vào shared memory/secure buffer).
- Service chạy riêng process vendor, ví dụ `android.hardware.drm-service.widevine`, `android.hardware.drm-service.clearkey`; khai báo trong VINTF manifest với instance `widevine`/`clearkey`. ✅ Trong project, Widevine HAL nằm **trong APEX** `com.google.android.widevine.nonupdatable` (mount ở `/apex/...`, manifest VINTF đi kèm APEX), còn ClearKey/PlayReady là binary trong `/vendor/bin/hw`.
- Phía framework: `libmediadrm` (`DrmHal`, `CryptoHal`) chọn HIDL hay AIDL tuỳ HAL có trên máy (`DrmHalHidl` / `DrmHalAidl`).
- Kiểm tra:
```bash
adb shell service list | grep -i drm          # AIDL
adb shell lshal | grep -i drm                  # HIDL (máy cũ)
adb shell dumpsys media.drm                    # plugin, session đang mở, metrics
adb shell ps -A | grep -i drm
adb logcat -s WVCdm MediaDrm DrmHal mediadrmserver
```
- SELinux: domain `hal_drm_widevine` / `hal_drm_default`, cần quyền truy cập device TEE (`/dev/tee*` hoặc node của SoC) và secure heap; denial ở đây → rơi về L3 hoặc DRM không chạy.

---

### Q7. Giới hạn số session và `ResourceManagerService`?

- Mỗi plugin giới hạn số session đồng thời (TEE có tài nguyên hạn chế).
- `ResourceManagerService` (trong `media.resource_manager`) theo dõi session DRM và codec theo **process priority**: khi app foreground cần session mà hết, nó **reclaim** session của app nền → app nền nhận `ResourceBusyException`/`MediaDrmStateException`.
- Bug hay gặp: app quên `drm.close()`/`release()` khi rời màn hình → leak session → app sau mở không được. Trên STB một app Launcher preview video DRM + app chính → tranh chấp.

---

### Q8. PlayReady, ClearKey, Verimatrix/Irdeto: vị trí của chúng?

- **ClearKey**: plugin chuẩn W3C trong AOSP, key không bảo vệ — dùng để **test** pipeline và CTS.
- **PlayReady** (Microsoft): plugin DRM HAL khác UUID (`9a04f079-9840-4286-ab92-e65be0885f95`), do SoC vendor/đối tác tích hợp; nhiều nhà mạng IPTV yêu cầu.
- **CAS (Verimatrix, Irdeto, Nagra...)** cho broadcast: qua `MediaCas` + `android.hardware.cas`, hoặc giải pháp riêng của nhà mạng tích hợp trong Tuner HAL/TIF.
- Một app có thể kiểm tra: `MediaDrm.isCryptoSchemeSupported(uuid)` / `MediaDrm.getSupportedCryptoSchemes()` (A13+).

---

### Q9. Chứng nhận Widevine L1 trên thiết bị Android TV/STB cần gì? (câu senior/lead)

- Hợp đồng với Google (Widevine), nhận CDM + OEMCrypto reference; SoC vendor thường cung cấp TA + `liboemcrypto.so`.
- Nạp keybox (hoặc cấu hình Provisioning 4.0) ở dây chuyền sản xuất, quy trình bảo mật nhà máy.
- Pass **CTS media/DRM**, Widevine **OEMCrypto unit tests** (`oemcrypto_test`) và `request_license_test`/CDM tests, **GTS** cho thiết bị GMS, cùng yêu cầu của từng đối tác nội dung (Netflix NRDP certification, YouTube...).
- Secure boot + AVB bật, bootloader khoá: thiết bị unlock bootloader → TEE/attestation báo trạng thái không tin cậy → thường **bị hạ về L3**.

---

### Q10. Tình huống: "Sau OTA, Netflix chỉ phát SD, trước đó phát 4K." Debug thế nào?

1. `MediaDrm.getPropertyString("securityLevel")` → còn L1 không?
2. Nếu thành L3: `logcat` tìm `WVCdm`, `OEMCrypto` lỗi init; kiểm tra (✅ theo project) `tee-supplicant` có chạy, TA `/vendor/lib/teetz/e043cde0-...51b.ta` còn đó, `liboemcrypto.so` có trong `/vendor/lib*/`, APEX Widevine đã mount (`ls /apex | grep widevine`); SELinux denial `hal_drm_widevine` (`dmesg | grep avc`).
3. Kiểm tra secure boot/AVB state (`ro.boot.verifiedbootstate` = `green`?), bootloader có bị unlock không.
4. Nếu vẫn L1: kiểm tra HDCP (`connectedHdcpLevel`), độ phân giải HDMI/EDID của TV (EDID mất 4K → app chọn thấp hơn), HDR capability (`Display.getHdrCapabilities`).
5. Provisioning: OTA có xoá `/data/vendor/mediadrm`? → provision lại có thành công không (mạng/clock — **sai giờ hệ thống** làm TLS/license fail là lỗi STB kinh điển khi chưa có NTP).
6. So sánh `dumpsys media.drm` trước/sau OTA; chạy ExoPlayer demo với stream Widevine test để tách lỗi app hay lỗi nền tảng.

---

## 18.2 Rapid-fire

1. **UUID dùng để làm gì?** → Định danh scheme DRM; `MediaDrm(uuid)` chọn plugin.
2. **PSSH là gì?** → Protection System Specific Header: dữ liệu khởi tạo (key ID, content ID) cho từng DRM, nằm trong MP4/manifest; là `initData` của `getKeyRequest`.
3. **cenc vs cbcs?** → AES-CTR toàn bộ vs AES-CBC theo pattern (1:9); cbcs cho phép dùng chung một bản mã cho Widevine/PlayReady/FairPlay.
4. **Ai gửi request lên license server?** → App, không phải framework; framework chỉ tạo/nhận blob.
5. **Tại sao phải `SurfaceView` với video L1?** → Output là protected buffer, chỉ hiển thị qua overlay/HWC, GPU không được đọc.
6. **mediadrmserver crash thì app thấy gì?** → `MediaDrm` nhận `MediaDrmResetException`/`DeadObjectException`; app phải tạo lại `MediaDrm`, session và `MediaCodec`.
7. **CAS khác DRM?** → CAS cho broadcast/descrambling luồng TS realtime; DRM cho nội dung OTT theo file/segment, license per content.
8. **Vì sao factory reset phải provision lại?** → DRM cert lưu ở `/data`, bị xoá cùng userdata; keybox nằm ở secure storage nên vẫn còn.
