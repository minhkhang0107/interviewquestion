# Chương 15 — Framework: Power/Standby, HDMI-CEC, Audio/Media, TV Input Framework, Alarm/Job, Multi-user

> Đã kiểm chứng với source A14. Bản gốc Gemini: `gemini_raw/03_gemini_framework_part2.md` (chủ đề 6–10).

## 15.1 PowerManagerService & standby STB

### Q1. Wakefulness của PowerManagerService?
`WAKEFULNESS_AWAKE` / `DREAMING` (screensaver/Ambient) / `DOZING` / `ASLEEP`; mọi thay đổi đi qua `updatePowerStateLocked()` (chạy nhiều pha: cập nhật wakefulness → display power → suspend blocker).
- Trên STB "màn hình tắt" thực chất là **tắt HDMI output** + đổi LED (project: `PowerStateManager` / `PowerLedManager` trong `DavidExtension` gọi HAL `DavidDeviceControl` đổi LED đỏ/trắng).

### Q2. Wakelock xuống kernel thế nào?
- `PowerManagerService` → native `SuspendBlocker` → **`system_suspend`** daemon (AIDL `android.system.suspend`, source `system/hardware/interfaces/suspend/`).
- `system_suspend` tự **autosuspend**: khi không còn wakelock, đọc `/sys/power/wakeup_count`, ghi lại, rồi ghi `mem` vào `/sys/power/state`; native wakelock vẫn có thể dùng `/sys/power/wake_lock`.
- **⚠️ Đã sửa**: Gemini nhắc tới "`/dev/wakeup_source`", không có node này.
- Debug: `dumpsys power` (wake locks, suspend blockers), `cat /sys/kernel/debug/wakeup_sources`, `dumpsys suspend_control_internal`.

### Q3. HDMI-CEC trên STB?
- `HdmiControlService` (system_server) → `HdmiCecController` → **HAL `android.hardware.tv.hdmi.cec` (AIDL)**. STB là **playback device** (`HdmiCecLocalDevicePlayback`, logical address 4/8/11).
- Bật STB (One Touch Play): gửi `<Image View On>` (0x04) / `<Text View On>` (0x0D) → `<Active Source>` (0x82, kèm physical address) để TV chuyển input.
- TV gửi `<Standby>` (0x36) → STB đi ngủ nếu cấu hình CEC cho phép (`HdmiCecConfig`, `power_state_change_on_active_source_lost`...).
- Debug: `dumpsys hdmi_control`, `cmd hdmi_control cec_setting get ...`.

### Q4. "Fake standby" vs suspend to RAM?
| | Suspend-to-RAM | Low-power "fake" standby |
|--|----------------|--------------------------|
| Điện năng | rất thấp | cao hơn (SoC + mạng vẫn chạy) |
| Thời gian bật | lâu hơn (resume + HDMI handshake) | gần như tức thì |
| Mạng / Cast / OTA ngầm | mất (trừ WoL/WoWLAN) | còn |
Thiết kế thường gặp: standby nông vài giờ → vào suspend sâu; wake source = IR/BT remote, CEC, WoL. Từ Android 13 có API **Low Power Standby** (A14 mở rộng thêm policy/allowlist theo lý do) (`PowerManager.isLowPowerStandbyEnabled`, `LowPowerStandbyController`) cho TV.
- (Các con số W/giây Gemini đưa ra là ước lượng, không lấy từ source.)

---

## 15.2 Audio & Media

### Q5. Đường đi của âm thanh?
`AudioTrack` (Java) → `AudioTrack.cpp` (libaudioclient) → `AudioFlinger::createTrack` (đã hỏi `AudioPolicyService::getOutputForAttr` để chọn output theo `AudioAttributes` + `audio_policy_configuration.xml`) → shared memory ring buffer → playback thread:
- `MixerThread` (PCM, trộn/resample), `DirectOutputThread` (passthrough/bitstream, không trộn), `OffloadThread` (DSP giải mã), `MmapThread` (AAudio low latency)
→ Audio HAL (`IStreamOut::write`) → tinyalsa → HDMI/SPDIF.
Debug: `dumpsys media.audio_flinger`, `dumpsys media.audio_policy`, `dumpsys audio`.

### Q6. Codec2 & MediaCodec?
- Codec2 thay OMX (A10+): `MediaCodec` → `CCodec` (`frameworks/av/media/codec2/sfplugin/`) → `Codec2Client` → service HAL `android.hardware.media.c2` (Amlogic: `vendor/amlogic/common/codec2`) → component `c2.amlogic.*`.
- Decode ra `Surface`: buffer output là `GraphicBuffer` (DMA-BUF) đưa thẳng vào BufferQueue của Surface → zero-copy; với nội dung DRM dùng **secure buffer** (`c2.*.secure`).
- Debug: `dumpsys media.codec`, `adb shell cmd media.resource_manager`, `logcat -s CCodec`.

### Q7. Tunneled playback?
- Decoder + audio đồng bộ **trên phần cứng** (HW AV sync), frame video đi thẳng ra display pipeline (sideband stream), không qua SurfaceFlinger composite từng frame.
```kotlin
val sessionId = audioManager.generateAudioSessionId()
format.setFeatureEnabled(MediaCodecInfo.CodecCapabilities.FEATURE_TunneledPlayback, true)
format.setInteger(MediaFormat.KEY_AUDIO_SESSION_ID, sessionId)        // ✔ API đúng
codec.configure(format, surface, null, 0)
// AudioTrack phải tạo với cùng sessionId + AudioAttributes.FLAG_HW_AV_SYNC
```
- **⚠️ Đã sửa**: Gemini dùng `MediaFormat.KEY_AUDIO_HW_SYNC` (không phải public API). Vòng 2 Gemini đã đồng ý.
- Lý do STB 4K cần: tiết kiệm băng thông bộ nhớ, lip-sync chuẩn, HDR/Dolby Vision metadata đi thẳng.

### Q8. Media key trên remote đi đâu?
`PhoneWindowManager.interceptKeyBeforeQueueing`: nếu có session "global priority" thì gửi luôn cho `MediaSessionService`; bình thường phím được giao cho **app đang foreground trước**; nếu app không xử lý → `PhoneFallbackEventHandler` (`core/java/com/android/internal/policy/`) gọi `MediaSessionManager.dispatchMediaKeyEventAsSystemService` → session ưu tiên (đang phát/gần nhất) → `MediaSession.Callback.onPlay/onPause`.
- **⚠️ Chú thích**: Gemini mô tả media key "đi thẳng tới session", hơi đơn giản hoá.

### Q9. Passthrough Dolby/DTS qua HDMI/ARC?
App dùng `AudioFormat.ENCODING_E_AC3`/`E_AC3_JOC`/`DTS`... → `AudioPolicyManager` kiểm tra EDID/encodings của sink (`AudioManager.getSurroundFormats`, `isDirectPlaybackSupported`) → `DirectOutputThread` → HAL gói IEC 61937 → TV/soundbar tự giải mã. Volume không áp lên bitstream.

---

## 15.3 TV Input Framework (TIF)

### Q10. Kiến trúc?
```
Live TV app: TvView ──(TvInputManager.Session)──► TvInputManagerService (system_server)
                                                   │ bind
                                                   ▼
                       TvInputService (app/vendor) → Session: onTune, onSetSurface, onCreateOverlayView
TvProvider (com.android.providers.tv): TvContract.Channels / Programs / PreviewPrograms / WatchNextPrograms
Hardware: TvInputHardwareManager ↔ TV Input HAL (hardware/interfaces/tv/input), HDMI-CEC, TvTuner HAL
```
- Surface: `TvView` (SurfaceView) → `Session.setSurface(surface)` qua binder → `TvInputService` đưa vào decoder (tuner/MediaCodec); overlay UI bằng `TYPE_APPLICATION_MEDIA_OVERLAY` (xem chương 11 Q46).
- Project liên quan: DTVKit/broadcast stack của Amlogic (`device.mk`: `vendor/amlogic/KSTB2500/broadcaststack/...dtvstack.mk`), launcher có quyền `ACCESS_ALL_EPG_DATA`.

### Q11. Đồng bộ EPG?
Ghi hàng loạt bằng `ContentResolver.applyBatch(TvContract.AUTHORITY, ops)`, xoá chương trình đã qua, sync bằng JobScheduler; không insert từng dòng.

---

## 15.4 AlarmManager & JobScheduler

### Q12. RTC vs ELAPSED_REALTIME?
- `RTC(_WAKEUP)`: theo giờ thực (đổi múi giờ/NTP ảnh hưởng). `ELAPSED_REALTIME(_WAKEUP)`: tính từ lúc boot, tính cả lúc ngủ (`CLOCK_BOOTTIME`). `_WAKEUP` dùng clock `*_ALARM` của kernel để đánh thức thiết bị.
- `AlarmManagerService` (A12+ ở `apex/jobscheduler/service/.../alarm/`) dùng `timerfd`; lưu alarm trong `AlarmStore` (lazy/batching).
- A14: `SCHEDULE_EXACT_ALARM` **bị từ chối mặc định** với app mới target 33+ (trừ app đồng hồ/lịch xin `USE_EXACT_ALARM`); `setExact*` không có quyền → `SecurityException`. App privileged/system có thể được grant sẵn.

### Q13. JobScheduler?
`JobSchedulerService` + các `StateController` (Connectivity, Battery, Idle, Storage, Time, Quota, ...) quyết định job sẵn sàng → bind `JobService`. App standby bucket (`ACTIVE`/`WORKING_SET`/`FREQUENT`/`RARE`/`RESTRICTED`) giới hạn quota. STB luôn "đang sạc". Debug: `dumpsys jobscheduler`, `cmd jobscheduler run -f <pkg> <id>`, `am set-standby-bucket`.

---

## 15.5 Multi-user, CE/DE, Direct Boot

### Q14. UID theo user?
`uid = userId * 100000 + appId` (`UserHandle.getUid`). User 0 = system user. `INTERACT_ACROSS_USERS(_FULL)` để gọi chéo user; service phải dùng `ActivityManager.handleIncomingUser(...)` (`am.handleIncomingUser`) để kiểm tra `userId` client gửi lên.

### Q15. CE vs DE storage, Direct Boot?
- DE (`/data/user_de/<user>/<pkg>`): dùng được ngay sau boot. CE (`/data/user/<user>/<pkg>`): mở khoá khi user unlock (key gắn với credential, qua Keystore/TEE, `vold` + `FsCrypt.cpp`).
- **Thực tế STB**: thường **không có lock screen** → CE được unlock tự động ngay khi boot (secret mặc định) → `LOCKED_BOOT_COMPLETED` tới rất nhanh rồi tới `BOOT_COMPLETED`. Direct Boot chỉ quan trọng khi có PIN/khóa. **⚠️ Chú thích**: kịch bản của Gemini kiểu "người dùng phải nhập mật khẩu sau khi mất điện" hiếm gặp trên STB.
- Thành phần cần chạy sớm: `android:directBootAware="true"`, dùng `createDeviceProtectedStorageContext()`, nhận `LOCKED_BOOT_COMPLETED`.
- Project: `fstab` dùng FBE `fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized` + metadata encryption (`keydirectory=/metadata/vold/metadata_encryption`).

### Q16. Notification trên Android TV?
Không có shade như điện thoại; TV dùng heads-up ngắn, thẻ "Now Playing" (MediaSession), và Home channels (`PreviewPrograms`). Google TV (A12+) có thêm panel thông báo ở system UI. **⚠️ Chú thích**: Gemini nói "TV không có notification" là quá tuyệt đối.
