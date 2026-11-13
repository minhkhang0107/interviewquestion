# Chapter 15 — Framework: Power/Standby, HDMI-CEC, Audio/Media, TV Input Framework, Alarm/Job, Multi-user

> Verified against the A14 source. Gemini's original: `gemini_raw/03_gemini_framework_part2.md` (topics 6–10).

## 15.1 PowerManagerService & STB standby

### Q1. PowerManagerService wakefulness?
`WAKEFULNESS_AWAKE` / `DREAMING` (screensaver/Ambient) / `DOZING` / `ASLEEP`; every change goes through `updatePowerStateLocked()` (which runs in phases: update wakefulness → display power → suspend blockers).
- On an STB, "screen off" really means **turning off the HDMI output** + changing the LED (project: `PowerStateManager` / `PowerLedManager` in `DavidExtension` call the `DavidDeviceControl` HAL to switch the LED red/white).

### Q2. How do wakelocks reach the kernel?
- `PowerManagerService` → native `SuspendBlocker` → the **`system_suspend`** daemon (AIDL `android.system.suspend`, source `system/hardware/interfaces/suspend/`).
- `system_suspend` **autosuspends** by itself: when no wakelocks remain, it reads `/sys/power/wakeup_count`, writes it back, then writes `mem` to `/sys/power/state`; native wakelocks can still use `/sys/power/wake_lock`.
- **⚠️ Corrected**: Gemini mentioned "`/dev/wakeup_source`"; that node doesn't exist.
- Debug: `dumpsys power` (wake locks, suspend blockers), `cat /sys/kernel/debug/wakeup_sources`, `dumpsys suspend_control_internal`.

### Q3. HDMI-CEC on an STB?
- `HdmiControlService` (system_server) → `HdmiCecController` → **HAL `android.hardware.tv.hdmi.cec` (AIDL)**. The STB is a **playback device** (`HdmiCecLocalDevicePlayback`, logical address 4/8/11).
- Turning the STB on (One Touch Play): send `<Image View On>` (0x04) / `<Text View On>` (0x0D) → `<Active Source>` (0x82, with the physical address) so the TV switches input.
- The TV sends `<Standby>` (0x36) → the STB goes to sleep if the CEC config allows it (`HdmiCecConfig`, `power_state_change_on_active_source_lost`...).
- Debug: `dumpsys hdmi_control`, `cmd hdmi_control cec_setting get ...`.

### Q4. "Fake standby" vs suspend to RAM?
| | Suspend-to-RAM | Low-power "fake" standby |
|--|----------------|--------------------------|
| Power draw | very low | higher (SoC + network keep running) |
| Wake-up time | longer (resume + HDMI handshake) | almost instant |
| Network / Cast / background OTA | lost (except WoL/WoWLAN) | kept |
Common design: shallow standby for a few hours → then deep suspend; wake sources = IR/BT remote, CEC, WoL. Since Android 13 there is the **Low Power Standby** API (A14 extended it with policies/allowlists by reason) (`PowerManager.isLowPowerStandbyEnabled`, `LowPowerStandbyController`) for TVs.
- (The W/second figures Gemini gave are estimates, not from the source.)

---

## 15.2 Audio & Media

### Q5. The audio path?
`AudioTrack` (Java) → `AudioTrack.cpp` (libaudioclient) → `AudioFlinger::createTrack` (after asking `AudioPolicyService::getOutputForAttr` to pick the output from `AudioAttributes` + `audio_policy_configuration.xml`) → shared-memory ring buffer → playback thread:
- `MixerThread` (PCM, mixing/resampling), `DirectOutputThread` (passthrough/bitstream, no mixing), `OffloadThread` (DSP decoding), `MmapThread` (AAudio low latency)
→ Audio HAL (`IStreamOut::write`) → tinyalsa → HDMI/SPDIF.
Debug: `dumpsys media.audio_flinger`, `dumpsys media.audio_policy`, `dumpsys audio`.

### Q6. Codec2 & MediaCodec?
- Codec2 replaces OMX (A10+): `MediaCodec` → `CCodec` (`frameworks/av/media/codec2/sfplugin/`) → `Codec2Client` → the `android.hardware.media.c2` HAL service (Amlogic: `vendor/amlogic/common/codec2`) → `c2.amlogic.*` components.
- Decoding to a `Surface`: output buffers are `GraphicBuffer`s (DMA-BUF) pushed straight into the Surface's BufferQueue → zero-copy; DRM content uses **secure buffers** (`c2.*.secure`).
- Debug: `dumpsys media.codec`, `adb shell cmd media.resource_manager`, `logcat -s CCodec`.

### Q7. Tunneled playback?
- Decoder + audio are synchronized **in hardware** (HW AV sync); video frames go straight into the display pipeline (sideband stream), not composited frame by frame by SurfaceFlinger.
```kotlin
val sessionId = audioManager.generateAudioSessionId()
format.setFeatureEnabled(MediaCodecInfo.CodecCapabilities.FEATURE_TunneledPlayback, true)
format.setInteger(MediaFormat.KEY_AUDIO_SESSION_ID, sessionId)        // ✔ correct API
codec.configure(format, surface, null, 0)
// The AudioTrack must be created with the same sessionId + AudioAttributes.FLAG_HW_AV_SYNC
```
- **⚠️ Corrected**: Gemini used `MediaFormat.KEY_AUDIO_HW_SYNC` (not a public API). Gemini agreed in round 2.
- Why a 4K STB needs it: saves memory bandwidth, accurate lip-sync, HDR/Dolby Vision metadata passes straight through.

### Q8. Where do remote media keys go?
`PhoneWindowManager.interceptKeyBeforeQueueing`: if there's a "global priority" session, the key goes straight to `MediaSessionService`; normally the key is given to the **foreground app first**; if the app doesn't handle it → `PhoneFallbackEventHandler` (`core/java/com/android/internal/policy/`) calls `MediaSessionManager.dispatchMediaKeyEventAsSystemService` → the preferred session (playing/most recent) → `MediaSession.Callback.onPlay/onPause`.
- **⚠️ Note**: Gemini described media keys as "going straight to the session", a bit oversimplified.

### Q9. Dolby/DTS passthrough over HDMI/ARC?
The app uses `AudioFormat.ENCODING_E_AC3`/`E_AC3_JOC`/`DTS`... → `AudioPolicyManager` checks the sink's EDID/encodings (`AudioManager.getSurroundFormats`, `isDirectPlaybackSupported`) → `DirectOutputThread` → the HAL packs IEC 61937 → the TV/soundbar decodes it. Volume is not applied to the bitstream.

---

## 15.3 TV Input Framework (TIF)

### Q10. Architecture?
```
Live TV app: TvView ──(TvInputManager.Session)──► TvInputManagerService (system_server)
                                                   │ bind
                                                   ▼
                       TvInputService (app/vendor) → Session: onTune, onSetSurface, onCreateOverlayView
TvProvider (com.android.providers.tv): TvContract.Channels / Programs / PreviewPrograms / WatchNextPrograms
Hardware: TvInputHardwareManager ↔ TV Input HAL (hardware/interfaces/tv/input), HDMI-CEC, TvTuner HAL
```
- Surface: `TvView` (SurfaceView) → `Session.setSurface(surface)` over binder → `TvInputService` feeds it to the decoder (tuner/MediaCodec); overlay UI uses `TYPE_APPLICATION_MEDIA_OVERLAY` (see chapter 11 Q46).
- Related in the project: Amlogic's DTVKit/broadcast stack (`device.mk`: `vendor/amlogic/KSTB2500/broadcaststack/...dtvstack.mk`); the launcher has the `ACCESS_ALL_EPG_DATA` permission.

### Q11. EPG synchronization?
Write in bulk with `ContentResolver.applyBatch(TvContract.AUTHORITY, ops)`, delete past programs, sync with JobScheduler; don't insert row by row.

---

## 15.4 AlarmManager & JobScheduler

### Q12. RTC vs ELAPSED_REALTIME?
- `RTC(_WAKEUP)`: wall-clock time (affected by time zone/NTP changes). `ELAPSED_REALTIME(_WAKEUP)`: time since boot, including sleep (`CLOCK_BOOTTIME`). `_WAKEUP` uses the kernel's `*_ALARM` clocks to wake the device.
- `AlarmManagerService` (A12+ in `apex/jobscheduler/service/.../alarm/`) uses `timerfd`; alarms are stored in an `AlarmStore` (lazy/batching).
- A14: `SCHEDULE_EXACT_ALARM` is **denied by default** for newly installed apps targeting 33+ (except clock/calendar apps that request `USE_EXACT_ALARM`); `setExact*` without the permission → `SecurityException`. Privileged/system apps can be pre-granted.

### Q13. JobScheduler?
`JobSchedulerService` + `StateController`s (Connectivity, Battery, Idle, Storage, Time, Quota, ...) decide when a job is ready → bind the `JobService`. App standby buckets (`ACTIVE`/`WORKING_SET`/`FREQUENT`/`RARE`/`RESTRICTED`) limit quota. An STB is always "charging". Debug: `dumpsys jobscheduler`, `cmd jobscheduler run -f <pkg> <id>`, `am set-standby-bucket`.

---

## 15.5 Multi-user, CE/DE, Direct Boot

### Q14. UIDs per user?
`uid = userId * 100000 + appId` (`UserHandle.getUid`). User 0 = system user. `INTERACT_ACROSS_USERS(_FULL)` for cross-user calls; services must use `ActivityManager.handleIncomingUser(...)` (`am.handleIncomingUser`) to validate the `userId` a client sends.

### Q15. CE vs DE storage, Direct Boot?
- DE (`/data/user_de/<user>/<pkg>`): usable right after boot. CE (`/data/user/<user>/<pkg>`): unlocked when the user unlocks (key bound to the credential, via Keystore/TEE, `vold` + `FsCrypt.cpp`).
- **STB reality**: usually **no lock screen** → CE is unlocked automatically at boot (default secret) → `LOCKED_BOOT_COMPLETED` arrives very quickly, followed by `BOOT_COMPLETED`. Direct Boot only matters with a PIN/lock. **⚠️ Note**: Gemini's scenario of "the user must enter a password after a power loss" is rare on STBs.
- Components that must run early: `android:directBootAware="true"`, use `createDeviceProtectedStorageContext()`, receive `LOCKED_BOOT_COMPLETED`.
- Project: `fstab` uses FBE `fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized` + metadata encryption (`keydirectory=/metadata/vold/metadata_encryption`).

### Q16. Notifications on Android TV?
There's no shade like on phones; TV uses short heads-up notifications, a "Now Playing" card (MediaSession), and Home channels (`PreviewPrograms`). Google TV (A12+) also has a notification panel in the system UI. **⚠️ Note**: Gemini's "TV has no notifications" is too absolute.
