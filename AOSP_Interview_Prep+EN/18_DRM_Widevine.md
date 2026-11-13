# Chapter 18 — DRM & Widevine (STB/TV focus)

> Based on AOSP 14 and public Android/Widevine documentation. Items marked ✅ **were cross-checked against the project source/`out/`** (the Widevine CDM/OEMCrypto are closed source, so only build configuration and outputs could be checked).

## 18.0 ✅ DRM in the KSTB2500 project (use when telling your experience)

| Component | What the project actually has |
|---|---|
| Widevine HAL | shipped as the **vendor APEX** `com.google.android.widevine.nonupdatable` (prebuilt, `TARGET_BUILD_WIDEVINE := nonupdatable` in `device/amlogic/common/core_amlogic.mk`), because `BOARD_WIDEVINE_CDM_AIDL` is not set → no standalone `drm-service.widevine` binary in `/vendor/bin/hw` |
| Security level | `BOARD_WIDEVINE_OEMCRYPTO_LEVEL ?= 1` (`media.mk`) → **L1**, installs `liboemcrypto.so` into `/vendor/lib*` |
| TEE | **OP-TEE** (`TARGET_USE_OPTEEOS := true`, `tee-supplicant` daemon); Widevine TA: `/vendor/lib/teetz/e043cde0-61d0-11e5-9c26-0002a5d5c51b.ta` |
| Secure video path | `BOARD_OMX_WITH_OPTEE_TVP := true` (TVP on OP-TEE) |
| Factory tools | `tee_provision`, `tee_key_inject` (key/keybox injection), `tee_hdcp`, `tee_attest` in `/vendor/bin` |
| Other DRM HALs built | `android.hardware.drm-service.clearkey`, `android.hardware.drm-service.playready` (`BOARD_PLAYREADY_LEVEL=1` = PlayReady + OP-TEE + TVP), `android.hardware.cas-service.example` |
| Configured but flag-dependent | Verimatrix, FairPlay, YouTube DRM (`media.mk`); `BUILD_WITH_WIDEVINECAS := false` |
| Secure decoders | `c2.amlogic.{avc,hevc,vp9,av1,mpeg2}.decoder.secure` (in `media_codecs*.xml` under `out/`) |

## 18.1 The big picture

```
App (ExoPlayer/Media3, Netflix, YouTube...)
  │  MediaDrm (session, license)          MediaCodec + MediaCrypto (decrypt + decode)
  ▼                                        ▼
frameworks/av: libmediadrm (DrmHal / CryptoHal)  ── binder ──►  mediadrmserver / mediaswcodec / media.codec
  ▼  (AIDL HAL android.hardware.drm since A13, HIDL drm@1.x before)
Vendor: android.hardware.drm-service.widevine   (IDrmFactory → IDrmPlugin / ICryptoPlugin)
  ▼  Widevine CDM
liboemcrypto.so  (OEM/SoC implementation of the OEMCrypto API)
  ▼  calls into the TEE (OP-TEE / SoC secure OS)
Widevine Trusted App in the TEE: holds keybox/cert, decrypts content into secure buffers
  ▼
Secure video decoder (reads secure buffers) → secure video path → HDMI (HDCP)
```

- **MediaDrm**: manages sessions, provisioning, licenses (key request/response).
- **MediaCrypto**: attaches a DRM session to a `MediaCodec` so the codec can decrypt encrypted samples.
- **DRM HAL plugin**: each scheme (Widevine, ClearKey, PlayReady...) is a plugin identified by a **UUID** (Widevine `edef8ba9-79d6-4ace-a3c8-27dcd51d21ed`).
- **CAS** (Conditional Access, `MediaCas` + `android.hardware.cas` HAL) is something else: used for **broadcast** (DVB/ATSC) with a Descrambler, usually together with the Tuner HAL. A frequent STB question.

---

### Q1. What is the difference between Widevine L1, L2 and L3?

| Level | Decryption (crypto) | Video decode | Output | Typically allowed content |
|---|---|---|---|---|
| **L1** | in the TEE | in a secure area (secure decoder, secure buffers) | secure path + HDCP | HD/4K/HDR on Netflix, Prime, Disney+ |
| **L2** | in the TEE | outside the TEE (normal world) | | rare |
| **L3** | software (no TEE) | normal world | | usually capped at SD (480p/540p) |

Check on the device:
```kotlin
val drm = MediaDrm(WIDEVINE_UUID)
drm.getPropertyString("securityLevel")        // "L1" / "L3"
drm.getPropertyString("systemId")
drm.getPropertyString("hdcpLevel"); drm.getPropertyString("maxHdcpLevel")
drm.connectedHdcpLevel; drm.maxHdcpLevel       // API 28+
```
Pitfall: a box reporting L3 instead of L1 is usually caused by a **missing/corrupt keybox**, the TEE TA not running, or `liboemcrypto.so` failing to load (SELinux, wrong path, wrong ABI). Apps like Netflix silently drop to SD without a clear error.

---

### Q2. What does the license flow for Widevine content look like from the app side?

```kotlin
val drm = MediaDrm(WIDEVINE_UUID)
val sessionId = drm.openSession()
// initData = PSSH box from the DASH manifest (<ContentProtection>) or the MP4 moov/pssh
val req = drm.getKeyRequest(sessionId, psshInitData, "video/mp4",
                            MediaDrm.KEY_TYPE_STREAMING, hashMapOf())
val resp = httpPost(licenseServerUrl, req.data)      // the app's job: add token/auth
drm.provideKeyResponse(sessionId, resp)

val crypto = MediaCrypto(WIDEVINE_UUID, sessionId)
codec.configure(format, surface, crypto, 0)
// for each encrypted sample:
codec.queueSecureInputBuffer(index, 0, cryptoInfo /* keyId, iv, mode, subsamples */, ptsUs, 0)
```
- `CryptoInfo.mode`: `CRYPTO_MODE_AES_CTR` (**cenc**) or `CRYPTO_MODE_AES_CBC` + pattern (**cbcs**, used for HLS/FairPlay-compatible content). Subsamples: NAL headers stay clear, only the payload is encrypted.
- Common exceptions: `NotProvisionedException` (provision first), `DeniedByServerException`, `MediaCodec.CryptoException` with `ERROR_NO_KEY`, `ERROR_KEY_EXPIRED`, `ERROR_INSUFFICIENT_OUTPUT_PROTECTION` (insufficient HDCP), `ERROR_RESOURCE_BUSY`/`ResourceBusyException` (out of sessions).
- Listeners: `setOnEventListener` (`EVENT_KEY_REQUIRED` for license renewal), `setOnKeyStatusChangeListener`, `setOnExpirationUpdateListener`.
- Offline (downloads): `KEY_TYPE_OFFLINE` → store the `keySetId`, later `restoreKeys`; release with `KEY_TYPE_RELEASE`.
- In practice with Media3: `DefaultDrmSessionManager` + `HttpMediaDrmCallback` + `FrameworkMediaDrm`; the app only configures `MediaItem.DrmConfiguration(C.WIDEVINE_UUID).setLicenseUri(...)`.

---

### Q3. What is provisioning? Keybox, Provisioning 3.0/4.0?

The device needs a **root identity** that the license server trusts:
- **Keybox (Provisioning 2.0)**: 128 bytes containing device ID + device key + magic, unique per device, **injected at the factory** into secure storage (TEE). On first use, the CDM uses the keybox to request a **Device Certificate (DRM cert)** from Google's provisioning server (`getProvisionRequest()` → POST → `provideProvisionResponse()`).
- **Provisioning 3.0 (OEM certificate)**: an OEM cert in the TEE replaces the keybox.
- **Provisioning 4.0**: based on the **Boot Certificate Chain (BCC)** / key chain from the bootloader (tied to KeyMint Remote Key Provisioning) — the direction for newer devices; no per-device keybox injection.
- The provisioned cert lives under `/data/vendor/mediadrm/...` (implementation-dependent) → **a factory reset requires provisioning again**; no network during provisioning → the app reports an error.

Trick question: *"Can the keybox be logged while debugging?"* → Never; a leaked keybox means **revoking** the whole batch (system ID). The factory injection tool must go through the TA and never write the raw keybox to the normal-world filesystem.

---

### Q4. How does the secure video path work? Why are screenshots of DRM video black?

1. Encrypted samples are copied into a **secure buffer** (memory only the TEE/HW decoder can access — a secure DMA-BUF heap / carveout; on Amlogic this is usually called **TVP — Trusted Video Path**, managed via `codec_mm`).
2. The TEE decrypts **in place** in secure memory; the normal-world CPU never sees the plaintext.
3. The `*.secure` decoder (✅ in the project: `c2.amlogic.avc.decoder.secure`, `...hevc...`, `...vp9...`, `...av1...`) reads the secure buffer and outputs a **protected buffer** (`GRALLOC_USAGE_PROTECTED`).
4. HWC/display composition sends it straight to a video plane (overlay); HDMI enables **HDCP**.

Consequences:
- You must render into a **`SurfaceView`** (or a secure surface); `TextureView` / reading pixels via the GPU is **not allowed** → black screen.
- `screencap`, screen recording and `MediaProjection` show black in the video area — **by design**, not a bug.
- Apps set `FLAG_SECURE` on their window to block UI screenshots.
- Tunneled playback (Chapter 15 Q7) usually goes together with secure decoders on STBs.

Choosing a secure decoder: `MediaCodecInfo.CodecCapabilities.FEATURE_SecurePlayback`, or `MediaCrypto.requiresSecureDecoderComponent(mime)`.

---

### Q5. How does HDCP relate to DRM?

- A license can require **output protection** (e.g. 4K requires HDCP 2.2, HD requires HDCP 1.x).
- The CDM queries the current HDCP state (via OEMCrypto → the SoC HDMI/HDCP driver). Insufficient → `ERROR_INSUFFICIENT_OUTPUT_PROTECTION`, or the server issues keys for a lower resolution.
- STB scenario: connected through an old AV receiver/splitter that only supports HDCP 1.4 → 4K fails, HD plays. Debug: `getConnectedHdcpLevel()`, HDCP driver logs, `dumpsys media.drm`.

---

### Q6. AIDL DRM HAL: main interfaces and which process runs it?

- Since **Android 13** the DRM HAL is **AIDL**: package `android.hardware.drm`, interfaces `IDrmFactory` (`createDrmPlugin`, `createCryptoPlugin`, `isCryptoSchemeSupported`...), `IDrmPlugin` (sessions, key request/response, provisioning, properties), `ICryptoPlugin` (`decrypt` into shared memory/secure buffers).
- The service runs in its own vendor process, e.g. `android.hardware.drm-service.widevine`, `android.hardware.drm-service.clearkey`; declared in the VINTF manifest with instances `widevine`/`clearkey`. ✅ In the project, the Widevine HAL lives **inside the APEX** `com.google.android.widevine.nonupdatable` (mounted under `/apex/...`, with its own VINTF fragment), while ClearKey/PlayReady are binaries in `/vendor/bin/hw`.
- Framework side: `libmediadrm` (`DrmHal`, `CryptoHal`) picks HIDL or AIDL depending on which HAL is present (`DrmHalHidl` / `DrmHalAidl`).
- Checks:
```bash
adb shell service list | grep -i drm          # AIDL
adb shell lshal | grep -i drm                  # HIDL (older devices)
adb shell dumpsys media.drm                    # plugins, open sessions, metrics
adb shell ps -A | grep -i drm
adb logcat -s WVCdm MediaDrm DrmHal mediadrmserver
```
- SELinux: domain `hal_drm_widevine` / `hal_drm_default`, needs access to the TEE device (`/dev/tee*` or a SoC-specific node) and the secure heap; denials here → fall back to L3 or DRM does not work.

---

### Q7. Session limits and `ResourceManagerService`?

- Each plugin limits the number of concurrent sessions (TEE resources are limited).
- `ResourceManagerService` (in `media.resource_manager`) tracks DRM sessions and codecs by **process priority**: when a foreground app needs a session and none are left, it **reclaims** a background app's session → the background app gets `ResourceBusyException`/`MediaDrmStateException`.
- Common bug: the app forgets `drm.close()`/`release()` when leaving the screen → session leak → the next app cannot open one. On an STB, a Launcher previewing DRM video + the main app → contention.

---

### Q8. PlayReady, ClearKey, Verimatrix/Irdeto: where do they fit?

- **ClearKey**: the W3C standard plugin in AOSP, keys are not protected — used to **test** the pipeline and in CTS.
- **PlayReady** (Microsoft): a DRM HAL plugin with a different UUID (`9a04f079-9840-4286-ab92-e65be0885f95`), integrated by the SoC vendor/partners; required by many IPTV operators.
- **CAS (Verimatrix, Irdeto, Nagra...)** for broadcast: via `MediaCas` + `android.hardware.cas`, or an operator-specific solution integrated into the Tuner HAL/TIF.
- Apps can check: `MediaDrm.isCryptoSchemeSupported(uuid)` / `MediaDrm.getSupportedCryptoSchemes()` (A13+).

---

### Q9. What does Widevine L1 certification on an Android TV/STB involve? (senior/lead question)

- An agreement with Google (Widevine) to receive the CDM + OEMCrypto reference; the SoC vendor usually provides the TA + `liboemcrypto.so`.
- Keybox injection (or Provisioning 4.0 setup) on the production line, with a secure factory process.
- Pass **CTS media/DRM**, Widevine **OEMCrypto unit tests** (`oemcrypto_test`) and `request_license_test`/CDM tests, **GTS** for GMS devices, plus each content partner's requirements (Netflix NRDP certification, YouTube...).
- Secure boot + AVB enabled, bootloader locked: an unlocked bootloader → TEE/attestation reports an untrusted state → typically **downgraded to L3**.

---

### Q10. Scenario: "After an OTA, Netflix only plays SD; it used to play 4K." How do you debug?

1. `MediaDrm.getPropertyString("securityLevel")` → still L1?
2. If it is L3: search `logcat` for `WVCdm`/`OEMCrypto` init errors; check (✅ per the project) that `tee-supplicant` is running, the TA `/vendor/lib/teetz/e043cde0-...51b.ta` is present, `liboemcrypto.so` exists in `/vendor/lib*/`, and the Widevine APEX is mounted (`ls /apex | grep widevine`); SELinux denials for `hal_drm_widevine` (`dmesg | grep avc`).
3. Check secure boot/AVB state (`ro.boot.verifiedbootstate` = `green`?) and whether the bootloader is unlocked.
4. If still L1: check HDCP (`connectedHdcpLevel`), the HDMI resolution/EDID of the TV (EDID lost 4K → the app picks a lower one), HDR capability (`Display.getHdrCapabilities`).
5. Provisioning: did the OTA wipe `/data/vendor/mediadrm`? → does re-provisioning succeed (network/clock — **a wrong system clock** breaking TLS/license requests before NTP syncs is a classic STB bug).
6. Compare `dumpsys media.drm` before/after the OTA; run the ExoPlayer demo with a Widevine test stream to separate app issues from platform issues.

---

## 18.2 Rapid-fire

1. **What is the UUID for?** → It identifies the DRM scheme; `MediaDrm(uuid)` selects the plugin.
2. **What is PSSH?** → Protection System Specific Header: per-DRM init data (key IDs, content ID) inside the MP4/manifest; it is the `initData` of `getKeyRequest`.
3. **cenc vs cbcs?** → AES-CTR over the whole payload vs pattern AES-CBC (1:9); cbcs lets a single encrypted copy serve Widevine/PlayReady/FairPlay.
4. **Who sends the request to the license server?** → The app, not the framework; the framework only produces/consumes blobs.
5. **Why must L1 video use a `SurfaceView`?** → The output is a protected buffer, displayable only via overlay/HWC; the GPU may not read it.
6. **What does the app see if mediadrmserver crashes?** → `MediaDrm` gets `MediaDrmResetException`/`DeadObjectException`; the app must recreate `MediaDrm`, the session and the `MediaCodec`.
7. **CAS vs DRM?** → CAS is for broadcast/real-time TS descrambling; DRM is for OTT content per file/segment with per-content licenses.
8. **Why does a factory reset require re-provisioning?** → The DRM cert lives in `/data` and is wiped with userdata; the keybox lives in secure storage and survives.
