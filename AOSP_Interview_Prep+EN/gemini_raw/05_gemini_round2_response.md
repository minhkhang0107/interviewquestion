From the perspective of a Senior AOSP Reviewer, I fully respect evidence from the actual source code (*source code is the single source of truth*). Below is a detailed response to each rebuttal point, plus 5 additional questions for the interview materials.

---

### Cross-check and response to the 14 technical points on AOSP Android 14

#### 1. Servicemanager in A14 is written in Rust
- **AGREE**
- **Technical reason:** In AOSP Android 14, the official `servicemanager` daemon still lives at `frameworks/native/cmds/servicemanager/` and is implemented entirely in **C++** (`ServiceManager.cpp`, `main.cpp`, `Access.cpp`). The core system component that was fully rewritten in Rust starting from Android 12 is **Keystore 2.0** (`system/security/keystore2`), not servicemanager.

#### 2. libbinder_ndk is NOT LL-NDK
- **AGREE**
- **Technical reason:** `frameworks/native/libs/binder/ndk/Android.bp` explicitly defines `llndk: { symbol_file: "libbinder_ndk.map.txt" }`. `libbinder_ndk` is in fact **LL-NDK (Low-Level NDK)**, which allows both Vendor code (HALs) and System code to communicate over a stable Binder protocol through a pure C ABI, without being tied to internal ABI changes of the C++ `libbinder`.

#### 3. sharedUserId deprecated in API 29 & AOSP platform apps
- **AGREE**
- **Technical reason:**
  1. API 29 is precisely **Android 10 (Q)**, not Android 11 (API 30).
  2. Although Google deprecated this attribute for third-party apps, the core system apps in AOSP A14 (`Settings`, `SystemUI`, `Telecom`) still must declare `android:sharedUserId="android.uid.system"` to run in the system process with UID 1000.

#### 4. Zygote restart mechanism when system_server dies
- **AGREE**
- **Technical reason:** In `com_android_internal_os_Zygote.cpp`, Zygote tracks `gSystemServerPid`. When it receives a `SIGCHLD` signal reporting that `system_server` has died, the signal handler logs: *"System server process %d has died. Restarting Zygote!"* and proactively calls `RuntimeAbort()`. Zygote **kills itself**; only then does `init` (via the service configuration in `init.zygoteXX.rc`) detect that Zygote has exited and trigger a restart of all related processes.

#### 5. oom_adj value table in ProcessList.java
- **AGREE**
- **Technical reason:** The AOSP `oom_adj` convention reserves negative values only for system-level / immortal processes (`NATIVE_ADJ = -1000`, `SYSTEM_ADJ = -900`, `PERSISTENT_PROC_ADJ = -800`, `PERSISTENT_SERVICE_ADJ = -700`). Foreground user apps always start at `FOREGROUND_APP_ADJ = 0` and increase up to `CACHED_APP_MAX_ADJ = 999`. Writing `-700` for a foreground app is completely wrong.

#### 6. Default MAX_CACHED_PROCESSES value
- **AGREE**
- **Technical reason:** In `ActivityManagerConstants.java` on Android 14, the constant is `DEFAULT_MAX_CACHED_PROCESSES = 1024`. The hard limit of 32 processes from older Android versions has been relaxed since Android 13/14, handing control of scheduling and RAM reclamation to the kernel and **LMKD** based on actual memory pressure metrics (**PSI** - Pressure Stall Information).

#### 7. Weighted distance formula in FocusFinder
- **AGREE**
- **Technical reason:** In the `getWeightedDistanceFor()` function in `FocusFinder.java`:
  $$\text{Distance} = 13 \times \text{major} \times \text{major} + \text{minor} \times \text{minor}$$
  The algorithm squares both axes and multiplies the major axis by a factor of 13 to heavily penalize angular deviation, ensuring D-pad navigation prefers moving straight rather than jumping diagonally between elements.

#### 8. 6-hour timeout for FGS dataSync
- **AGREE**
- **Technical reason:** On Android 14, the Foreground Service runtime limit only applies to the `shortService` type (~3 minutes). The 6-hour limit for `dataSync` and `mediaProcessing` (with the `Service.onTimeout()` callback) is a new feature introduced in **Android 15 (API 35)**; Android 14's `ActiveServices.java` does not yet have this timeout mechanism for `dataSync`.

#### 9. Behavior of calling startForeground() without passing a type on A14
- **AGREE**
- **Technical reason:** `Service.java` defines `startForeground(id, notification)` to pass `FOREGROUND_SERVICE_TYPE_MANIFEST` by default. `ActiveServices.java` then checks it against the `android:foregroundServiceType` attribute declared in the `<service>` tag in `AndroidManifest.xml`. The `MissingForegroundServiceTypeException` only occurs when targeting SDK 34 without declaring any type at all in the manifest.

#### 10. Default CursorWindow size
- **AGREE**
- **Technical reason:** In Android 14's `frameworks/base/core/res/res/values/config.xml`, the value `config_cursorWindowSize` is configured as `2048` (in KB, equivalent to **2MB**). The 4MB figure is incorrect; 2MB remains the standard AOSP baseline unless the OEM overrides it via RRO.

#### 11. Implicit Broadcast exemption list
- **AGREE**
- **Technical reason:** There is no `BroadcastConstants.java` file containing this list. The list of implicit broadcasts allowed to wake background apps is configured via the `<allow-implicit-broadcast>` tag in the permission XML files under `/etc/sysconfig/`, which `SystemConfig.java` loads into AMS's `mBackgroundLaunchBroadcasts` at system startup.

#### 12. Handling dedicated hardware keys (Global Key) on Android TV
- **AGREE**
- **Technical reason:** `PhoneWindowManager` delegates handling of special keys to `GlobalKeyManager`. This class reads its configuration from `res/xml/global_keys.xml` to dispatch the `Intent.ACTION_GLOBAL_BUTTON` broadcast to the registered `ComponentName`. The standard AOSP/STB mechanism is to RRO-overlay this XML file to point to the vendor's broadcast receiver to open the app, rather than hardcoding `startActivity()` in PWM.

#### 13. Initializing Tunneled Video Playback (Tunnel mode)
- **AGREE**
- **Technical reason:** The standard app/framework-level API to enable Tunnel mode is:
  1. Check support via `CodecCapabilities.FEATURE_TunneledPlayback`.
  2. Attach the Audio Session ID to the video format: `mediaFormat.setInteger(MediaFormat.KEY_AUDIO_SESSION_ID, audioSessionId)`.
  The constant `KEY_AUDIO_HW_SYNC` (`"audio-hw-sync"`) is only an internal identifier exchanged at the native HAL layer (obtained via `AudioManager.getAudioHwSyncForSession()`), not a public MediaFormat API.

#### 14. Non-blocking behavior of Java ServiceManager.getService()
- **AGREE**
- **Technical reason:** Confirmed, same view. `ServiceManager.getService()` at the Java layer calls directly through AIDL down to the native servicemanager and returns immediately (returning `null` if the service isn't ready yet; to wait, you must use `waitForService()`). The 5s wait-loop logic with the logcat warning *"Waiting for service..."* only lives in the C++ shim function `defaultServiceManager()->getService()`.

---

### 5 additional App / Framework layer interview questions (Android 14)

#### Question 1: How does the new Broadcast Queue mechanism (Modern Queue) work with apps in the Cached state on Android 14?
- **Answer:** Android 14 completely rewrote `BroadcastQueueImpl` as `BroadcastQueueModernImpl`. When a process enters the `CACHED` state (cgroup-frozen), the system defers delivery of non-urgent (non-urgent/implicit) broadcasts. These broadcasts are batched and only flushed when the process is unfrozen (e.g., when the user reopens the app or an urgent/explicit broadcast arrives).

#### Question 2: How does Android 14 tighten Background Activity Launch (BAL) for `PendingIntent`?
- **Answer:** To prevent privilege escalation vulnerabilities when a `PendingIntent` is handed to another app to trigger, Android 14 requires the `PendingIntent` creator to explicitly opt in to background launch rights by passing `ActivityOptions.setPendingIntentBackgroundActivityStartMode(MODE_BACKGROUND_ACTIVITY_START_ALLOWED)`. By default at target SDK 34, launching an Activity from the background via a `PendingIntent` is fully blocked.

#### Question 3: How does partial photo/video access (`READ_MEDIA_VISUAL_USER_SELECTED`) work at the Framework layer?
- **Answer:** When an app targeting SDK 34 requests `READ_MEDIA_IMAGES` or `READ_MEDIA_VIDEO`, the system shows a *"Select photos and videos"* option. If the user picks it, PackageManager only grants the temporary permission `READ_MEDIA_VISUAL_USER_SELECTED`. The app can only query exactly the selected Uris through `MediaStore`. When the app's process is killed or it stays in the background for a while, this temporary permission may be revoked automatically.

#### Question 4: How does the Predictive Back Gesture mechanism work at the WindowManager/Shell layer?
- **Answer:** The app needs to declare `android:enableOnBackInvokedCallback="true"` and register an `OnBackAnimationCallback` via `OnBackInvokedDispatcher` instead of `onBackPressed()`. When the user swipes from the screen edge, `BackAnimationController` in `SystemUI/Shell` receives the event from InputReader and directly drives the shrink/preview animation of the current Activity's window to reveal the screen behind it, before the user releases to decide whether to commit or cancel the back action.

#### Question 5: Architecturally (SurfaceFlinger / WindowManager), why do video players on STB/Android TV always prefer `SurfaceView` over `TextureView`?
- **Answer:** `SurfaceView` creates its own `SurfaceControl`, which `SurfaceFlinger` composites independently via a hardware overlay plane, decoupled from the View hierarchy's RenderThread. This completely bypasses the app's drawing cycle, saves memory bandwidth, and supports zero-copy, HDR decoding, DRM video (Secure Surface), and Tunneled Playback. In contrast, `TextureView` turns the video buffer into an OpenGL ES texture drawn together within the View tree, consuming twice the GPU/RAM bandwidth and not supporting Secure/Tunnel mode.
