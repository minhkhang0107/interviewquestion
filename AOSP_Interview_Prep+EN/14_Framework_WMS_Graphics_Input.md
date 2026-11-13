# Chapter 14 — Framework: WindowManager, Graphics pipeline, Input pipeline, native Looper

> Verified against the A14 source. Gemini's original: `gemini_raw/03_gemini_framework_part2.md` (topics 1–5).

## 14.1 WindowManagerService

### Q1. The WindowContainer tree (A11+)?
```
RootWindowContainer
 └─ DisplayContent (display 0 = HDMI on an STB)
     └─ DisplayArea (by feature: below/app/above layers, IME...)
         └─ TaskDisplayArea
             └─ Task
                 └─ (TaskFragment)
                     └─ ActivityRecord  (extends WindowToken)
                         └─ WindowState  (1 window = 1 ViewRootImpl on the client side)
```
- Every `WindowContainer` has its own `SurfaceControl` → the WM tree **matches SurfaceFlinger's layer tree 1:1**; changes (position, crop, alpha, z) are batched into atomic `SurfaceControl.Transaction`s (`BLASTSyncEngine`, Shell transitions) → no jank/flicker.

### Q2. How does `WindowManager.addView()` flow?
1. `WindowManagerImpl` → `WindowManagerGlobal.addView` → creates a `ViewRootImpl` → `setView()`.
2. `IWindowSession.addToDisplayAsUser` (binder to `com.android.server.wm.Session`) → `WMS.addWindow`: checks type/token/permission (`mPolicy.checkAddPermission`, `DisplayPolicy.validateAddingWindowLw`), creates the `WindowState`, opens an `InputChannel` (the server keeps one end and returns the other to the client).
3. First traversal: `relayoutWindow` → WMS creates the `SurfaceControl` (`WindowStateAnimator.createSurfaceLocked`) → the client creates a `BLASTBufferQueue` and draws.
- Common errors: `BadTokenException` (wrong token/dead Activity), `WindowManager$InvalidDisplayException`, `permission denied for window type 2038` (missing overlay permission).

### Q3. Window types & z-order?
- Application `1–99` (`TYPE_BASE_APPLICATION`, `TYPE_APPLICATION`), sub-windows `1000–1999` (`TYPE_APPLICATION_PANEL`, `TYPE_APPLICATION_MEDIA`, `TYPE_APPLICATION_MEDIA_OVERLAY`), system `2000+` (`TYPE_APPLICATION_OVERLAY`=2038, `TYPE_STATUS_BAR`, `TYPE_SYSTEM_DIALOG`...).
- Z-order: `DisplayAreaPolicy` + `WindowManagerPolicy.getWindowLayerFromTypeLw()` → `assignChildLayers()` → `Transaction.setLayer/setRelativeLayer`.
- `SurfaceView`: by default sits **below** the window (punches a hole); `setZOrderMediaOverlay(true)` puts it above other SurfaceViews but below the window; `setZOrderOnTop(true)` puts it above the window.

### Q4. `SYSTEM_ALERT_WINDOW` on TV?
- It's a permission with the `appop` flag: WMS checks the AppOps `OP_SYSTEM_ALERT_WINDOW`; system apps are usually pre-granted (privileged/default grant). Example: `DavidLauncher` has `SYSTEM_ALERT_WINDOW` in its privapp allowlist.
- An overlay that takes focus blocks the D-pad → use `FLAG_NOT_FOCUSABLE` for banners.
- Many fullscreen overlays → HWC runs out of planes → SurfaceFlinger falls back to GPU composition → 4K video stutters.

---

## 14.2 Graphics pipeline

### Q5. VSYNC-app, VSYNC-sf, Choreographer?
- HW VSYNC (from HWC/DRM) → the SurfaceFlinger scheduler (`frameworks/native/services/surfaceflinger/Scheduler/`: `VSyncPredictor`, `VSyncDispatchTimerQueue`, `VsyncSchedule`) predicts the period → emits **VSYNC-app** (for `Choreographer` via `DisplayEventReceiver`) and **VSYNC-sf** (for SF composition), **phase-shifted** from each other (phase offset / work duration) so the app has time to draw before SF picks up the buffer.
- Switching refresh rate 50/60Hz (very common on STBs when playing PAL content) must be reported correctly by the HWC, otherwise you get micro-stutter.

### Q6. BufferQueue → BLASTBufferQueue?
- Before: the app `queueBuffer`ed into SF's BufferQueue; geometry changed through a separate transaction → buffer and geometry could be off by a frame.
- BLAST (A11+, default in A12): **the BufferQueue lives in the app process**; when a buffer is ready it's sent via `SurfaceControl.Transaction.setBuffer()` → can be merged with the geometry transaction → atomic. Code: `frameworks/native/libs/gui/BLASTBufferQueue.cpp`, `frameworks/base/graphics/java/android/graphics/BLASTBufferQueue.java`.

### Q7. From the RenderThread to HDMI?
The RenderThread (Skia GL/Vulkan) draws into a `GraphicBuffer` (gralloc, DMA-BUF) → BLAST transaction + acquire fence → SurfaceFlinger (`commit` → `composite` on A14) latches the buffer → HWC (**AIDL composer3**) classifies layers as `DEVICE` (hardware overlay plane) / `CLIENT` (SF draws it itself with the RenderEngine GPU) → present → DRM/KMS → HDMI TX.
- **⚠️ Minor correction**: Gemini used the old thread function names `onMessageReceived → handleMessageInvalidate` (renamed in newer SF).
- Debug: `dumpsys SurfaceFlinger` (layers, composition type), `dumpsys gfxinfo <pkg> framestats`, Perfetto.

### Q8. Triple buffering?
Android has used triple buffering since Project Butter (4.1) when needed: the GPU/SF don't have to wait → fewer dropped frames, at the cost of 1 extra frame of latency and more RAM (one 1080p RGBA buffer ≈ 8MB, 4K ≈ 33MB). On STBs the UI is usually rendered at 1080p and scaled to 4K by the display engine to save resources.

---

## 14.3 Input pipeline

### Q9. From the kernel to a View?
```
/dev/input/eventX ── EventHub (epoll + inotify) ── InputReader thread
   (InputDevice + InputMapper, KeyLayoutMap .kl, KeyCharacterMap .kcm)
   → InputDispatcher::notifyKey()  → policy.interceptKeyBeforeQueueing()  (PhoneWindowManager, on the reader thread)
   → inbound queue → InputDispatcher thread → find the focused window (from WMS via InputWindowHandle)
   → policy.interceptKeyBeforeDispatching()  (PhoneWindowManager)
   → InputChannel (socketpair) → app: InputEventReceiver (Looper fd) → ViewRootImpl InputStage
   → View.dispatchKeyEvent → finishInputEvent() → InputDispatcher receives "finished"
```
Source: `frameworks/native/services/inputflinger/{reader,dispatcher}/`, `frameworks/native/libs/input/InputTransport.cpp`.

### Q10. The 5s input ANR?
- InputDispatcher tracks events that were sent but haven't received "finished" (the connection's `waitQueue`) + `AnrTracker`; past the dispatching timeout (default **5s**, `DEFAULT_DISPATCHING_TIMEOUT`) → `onAnrLocked` → `InputManagerService.notifyWindowUnresponsive/notifyNoFocusedWindowAnr` → AMS ANR.
- 2 kinds: the window doesn't respond; **no focused window** (the app is open but has no focused window yet — common when an Activity initializes slowly).
- **⚠️ Corrected**: Gemini confused `KEY_WAITING_FOR_EVENTS_TIMEOUT` (500ms, used to wait for previous events to be processed) with the 5s ANR timeout.

### Q11. `interceptKeyBeforeQueueing` vs `interceptKeyBeforeDispatching`?
| | BeforeQueueing | BeforeDispatching |
|--|----------------|-------------------|
| Thread | InputReader (called from `notifyKey`) | InputDispatcher |
| When | before entering the queue, even with the screen off | right before delivery to the focused window |
| Used for | POWER, wake keys, volume/mute, media keys while non-interactive | HOME, SEARCH/ASSIST, global keys, shortcuts |
| Result | clear the `ACTION_PASS_TO_USER` flag → doesn't reach the app | return `-1` → consume the key, `>0` → retry later |

### Q12. Global keys on Android TV (real example in the project)?
- `PhoneWindowManager` uses `GlobalKeyManager`, which reads `com.android.internal.R.xml.global_keys`; if the key is in the list → it **sends the `Intent.ACTION_GLOBAL_BUTTON` broadcast** (with the `KeyEvent`) to the declared component, even if another app is in the foreground; `dispatchWhenNonInteractive` allows sending it while in standby.
- The project overrides it with the `DavidFrameworkOverlay` RRO:
```xml
<!-- vendor/david/apps/DavidFrameworkOverlay/res/xml/global_keys.xml -->
<global_keys version="1">
    <key keyCode="KEYCODE_F9"  component="com.davidmedia.settings.customizer/.receiver.GlobalKeyReceiver" dispatchWhenNonInteractive="true" />
    <key keyCode="KEYCODE_F10" component="com.davidmedia.settings.customizer/.receiver.GlobalKeyReceiver" dispatchWhenNonInteractive="true" />
    ...
</global_keys>
```
- **⚠️ Corrected**: Gemini wrote that PhoneWindowManager `startActivity`s Netflix itself. In reality AOSP sends a broadcast to the receiver, and the receiver opens the app (Gemini agreed in round 2).

### Q13. `.kl` / `.kcm`?
- Files are looked up in this order: `Vendor_XXXX_Product_XXXX_Version_XXXX.kl` → `Vendor_XXXX_Product_XXXX.kl` → `<device name>.kl` → `Generic.kl`, under `/odm`, `/vendor`, `/system` `/usr/keylayout/`. The project has `vendor/david/keylayout/`.
```
key 103   DPAD_UP
key 352   DPAD_CENTER
key 116   POWER        WAKE
key 67    F9                       # then global_keys maps F9 → GlobalKeyReceiver
```
- The `WAKE` flag (wakes the device from sleep). Remote can't wake the device: `WAKE` missing in the `.kl` **or** the IR/BT driver isn't configured as a wakeup source in the kernel.
- Debug: `getevent -lt`, `dumpsys input` (shows which `.kl` file each device uses).

**Real example in the project (David ALVA M1/M4 BT remote)**: `vendor/david/keylayout/Vendor_0217_Product_0001.kl` + `.idc`
```
key 116   POWER
key usage 0x0c0075 F9        # mapped by HID usage (consumer page), not by scancode
key 183   F10                # M1
key usage 0x0c0077 F10       # M4
```
```
# Vendor_0217_Product_0001.idc
device.internal = 1          # treat the BT remote as an "internal" device
keyboard.builtIn = 0
audio.mic = 1                # the remote has a microphone (voice search)
```
David also **modified AOSP** in `frameworks/native/services/inputflinger/reader/mapper/KeyboardInputMapper.cpp:166-167`:
```cpp
// DAVID: Set doNotWakeByDefault = true by default to prevent external keyboards from waking the device from standby mode
mParameters.doNotWakeByDefault = config.getBool("keyboard.doNotWakeByDefault").value_or(true);
```
(AOSP defaults to `false`, meaning every key of an **external keyboard** wakes the device; see `if (down && isExternal() && !doNotWakeByDefault) policyFlags |= POLICY_FLAG_WAKE;`.) `CursorInputMapper.cpp` has a similar patch. Good interview questions from this:
1. *Why do USB keyboards/mice wake the STB unexpectedly and how do you fix it?* → 2 ways: (a) change the default in inputflinger as David did (affects every device); (b) no code change, each device declares `keyboard.doNotWakeByDefault = 1` in its own `.idc`. Option (b) is "cleaner" and easier to rebase when moving to a new Android.
2. *Then how does the remote wake the device?* → For internal devices, or when default wake is disabled, declare `WAKE` per key in the `.kl` (the AOSP comment explicitly says "e.g. TV remotes"); `POWER` is handled by `PhoneWindowManager.interceptKeyBeforeQueueing` for wake/sleep.

---

## 14.4 Native Looper

### Q14. `nativePollOnce` / `nativeWake`?
- C++ `Looper` (`system/core/libutils/Looper.cpp`): `epoll_create1` + `eventfd` (`mWakeEventFd`); `pollOnce` → `epoll_wait(timeout)`; `wake()` writes `uint64_t 1` to the eventfd.
- Java `MessageQueue.enqueueMessage` computes `needWake` (the new message is at the head of the queue, or it's the first async message behind a barrier while the thread sleeps) → `nativeWake`.
- `Looper.addFd()` lets the same thread watch other fds (InputChannel, DisplayEventReceiver).

### Q15. `nativePollOnce` takes a lot of time in a trace — is that a bug?
No: that's the thread **being idle** (sleeping in `epoll_wait`, state S). The real problems are long slices **before** it (measure/layout/binder), or the thread spending a long time in state `D` (I/O) / `R` (CPU starvation).

---

## 14.5 Resources & RRO

### Q16. How does `AssetManager2` resolve an ID `0xPPTTEEEE`?
PP = package (0x01 framework, 0x7f app), TT = type, EEEE = entry. `AssetManager2::GetResource` → picks the package group → finds the config that best matches the current `ResTable_config` (locale, `uiMode=television`, density...) → if there's an overlay, uses the idmap to substitute the value (`frameworks/base/libs/androidfw/AssetManager2.cpp`).

### Q17. RRO, fabricated overlays?
- An overlay APK declares `<overlay android:targetPackage=... android:isStatic=... android:priority=.../>`; `OverlayManagerService` + `idmap2` create the idmap in `/data/resource-cache/`.
- Static overlays (like `DavidFrameworkOverlay`) are always on and can't be disabled with `cmd overlay`.
- Fabricated overlays (A12+): overlays created in code (`FabricatedOverlay`) — used for per-operator theming.
- One firmware for many operators: in the project, per-operator configuration is split into `device/amlogic/KSTB2500/oem/sku_*` and `launcher_configuration_<operator>.xml` → compare the trade-offs between RRO, an oem partition and build-time config.

### Q18. Qualifiers for TV?
`television` (uiMode), `notouch`, `dpad` (navigation). Density: the Android TV guideline for a 1080p UI is `xhdpi` (320dpi → 960×540dp); **this project sets `ro.sf.lcd_density=240`** (`device/amlogic/KSTB2500/KSTB2500.mk:38`, hdpi → 1280×720dp), so third-party app layouts designed for 960×540dp look smaller. Good question: *"who decides the density?"* → `ro.sf.lcd_density` (build) or `wm density` (runtime, stored in `Settings.Secure.display_density_forced`). Changing the HDMI resolution → a new `Configuration` → relaunch or `onConfigurationChanged`.
