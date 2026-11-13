# Chương 14 — Framework: WindowManager, Graphics pipeline, Input pipeline, Looper native

> Đã kiểm chứng với source A14. Bản gốc Gemini: `gemini_raw/03_gemini_framework_part2.md` (chủ đề 1–5).

## 14.1 WindowManagerService

### Q1. Cây WindowContainer (A11+)?
```
RootWindowContainer
 └─ DisplayContent (display 0 = HDMI trên STB)
     └─ DisplayArea (theo feature: tầng dưới/ứng dụng/trên, IME...)
         └─ TaskDisplayArea
             └─ Task
                 └─ (TaskFragment)
                     └─ ActivityRecord  (extends WindowToken)
                         └─ WindowState  (1 window = 1 ViewRootImpl phía client)
```
- Mọi `WindowContainer` có `SurfaceControl` riêng → cây WM **khớp 1:1** với cây layer của SurfaceFlinger; thay đổi (vị trí, crop, alpha, z) được gom thành `SurfaceControl.Transaction` atomic (`BLASTSyncEngine`, Shell transitions) → không giật/nháy.

### Q2. `WindowManager.addView()` đi thế nào?
1. `WindowManagerImpl` → `WindowManagerGlobal.addView` → tạo `ViewRootImpl` → `setView()`.
2. `IWindowSession.addToDisplayAsUser` (binder tới `com.android.server.wm.Session`) → `WMS.addWindow`: kiểm tra type/token/permission (`mPolicy.checkAddPermission`, `DisplayPolicy.validateAddingWindowLw`), tạo `WindowState`, mở `InputChannel` (server giữ 1 đầu, trả đầu kia cho client).
3. Lần traversal đầu: `relayoutWindow` → WMS tạo `SurfaceControl` (`WindowStateAnimator.createSurfaceLocked`) → client tạo `BLASTBufferQueue` và vẽ.
- Lỗi thường gặp: `BadTokenException` (token sai/Activity đã chết), `WindowManager$InvalidDisplayException`, `permission denied for window type 2038` (thiếu overlay permission).

### Q3. Window type & z-order?
- Application `1–99` (`TYPE_BASE_APPLICATION`, `TYPE_APPLICATION`), sub-window `1000–1999` (`TYPE_APPLICATION_PANEL`, `TYPE_APPLICATION_MEDIA`, `TYPE_APPLICATION_MEDIA_OVERLAY`), system `2000+` (`TYPE_APPLICATION_OVERLAY`=2038, `TYPE_STATUS_BAR`, `TYPE_SYSTEM_DIALOG`...).
- Z-order: `DisplayAreaPolicy` + `WindowManagerPolicy.getWindowLayerFromTypeLw()` → `assignChildLayers()` → `Transaction.setLayer/setRelativeLayer`.
- `SurfaceView`: mặc định nằm **dưới** window (đục lỗ); `setZOrderMediaOverlay(true)` đặt trên SurfaceView khác nhưng dưới window; `setZOrderOnTop(true)` đè lên window.

### Q4. `SYSTEM_ALERT_WINDOW` trên TV?
- Là quyền có flag `appop`: WMS kiểm tra AppOps `OP_SYSTEM_ALERT_WINDOW`; app hệ thống thường được grant sẵn (privileged/default grant). Ví dụ: `DavidLauncher` có `SYSTEM_ALERT_WINDOW` trong privapp allowlist.
- Overlay chặn D-pad nếu có focus → dùng `FLAG_NOT_FOCUSABLE` cho banner.
- Nhiều overlay full-screen → HWC hết plane → SurfaceFlinger chuyển sang GPU composition → video 4K bị giật.

---

## 14.2 Graphics pipeline

### Q5. VSYNC-app, VSYNC-sf, Choreographer?
- HW VSYNC (từ HWC/DRM) → SurfaceFlinger scheduler (`frameworks/native/services/surfaceflinger/Scheduler/`: `VSyncPredictor`, `VSyncDispatchTimerQueue`, `VsyncSchedule`) dự đoán chu kỳ → phát **VSYNC-app** (cho `Choreographer` qua `DisplayEventReceiver`) và **VSYNC-sf** (cho SF composite) **lệch pha** nhau (phase offset / work duration) để app có thời gian vẽ trước khi SF lấy buffer.
- Đổi refresh rate 50/60Hz (rất hay gặp ở STB khi phát nội dung PAL) phải được HWC báo đúng, nếu không sẽ bị micro-stutter.

### Q6. BufferQueue → BLASTBufferQueue?
- Trước: app `queueBuffer` vào BufferQueue của SF; geometry đổi qua transaction khác → buffer và geometry lệch frame.
- BLAST (A11+, mặc định A12): **BufferQueue nằm trong process app**, khi có buffer thì gửi qua `SurfaceControl.Transaction.setBuffer()` → gộp được với transaction geometry → atomic. Code: `frameworks/native/libs/gui/BLASTBufferQueue.cpp`, `frameworks/base/graphics/java/android/graphics/BLASTBufferQueue.java`.

### Q7. Từ RenderThread tới HDMI?
RenderThread (Skia GL/Vulkan) vẽ vào `GraphicBuffer` (gralloc, DMA-BUF) → BLAST transaction + acquire fence → SurfaceFlinger (`commit` → `composite` trên A14) latch buffer → HWC (**AIDL composer3**) phân loại layer `DEVICE` (overlay plane phần cứng) / `CLIENT` (SF tự vẽ bằng RenderEngine GPU) → present → DRM/KMS → HDMI TX.
- **⚠️ Đã sửa nhẹ**: Gemini dùng tên luồng cũ `onMessageReceived → handleMessageInvalidate` (đã đổi trong SF mới).
- Debug: `dumpsys SurfaceFlinger` (layer, composition type), `dumpsys gfxinfo <pkg> framestats`, Perfetto.

### Q8. Triple buffering?
Android dùng triple buffering từ Project Butter (4.1) khi cần: GPU/SF không phải chờ → ít rớt khung, đổi lại thêm 1 frame latency và tốn RAM (1 buffer 1080p RGBA ≈ 8MB, 4K ≈ 33MB). Trên STB UI thường render 1080p rồi scale lên 4K bởi display engine để tiết kiệm.

---

## 14.3 Input pipeline

### Q9. Từ kernel tới View?
```
/dev/input/eventX ── EventHub (epoll + inotify) ── InputReader thread
   (InputDevice + InputMapper, KeyLayoutMap .kl, KeyCharacterMap .kcm)
   → InputDispatcher::notifyKey()  → policy.interceptKeyBeforeQueueing()  (PhoneWindowManager, trên reader thread)
   → inbound queue → InputDispatcher thread → tìm focused window (từ WMS qua InputWindowHandle)
   → policy.interceptKeyBeforeDispatching()  (PhoneWindowManager)
   → InputChannel (socketpair) → app: InputEventReceiver (Looper fd) → ViewRootImpl InputStage
   → View.dispatchKeyEvent → finishInputEvent() → InputDispatcher nhận "finished"
```
Source: `frameworks/native/services/inputflinger/{reader,dispatcher}/`, `frameworks/native/libs/input/InputTransport.cpp`.

### Q10. Input ANR 5s?
- InputDispatcher theo dõi event đã gửi nhưng chưa có "finished" (`waitQueue` của connection) + `AnrTracker`; quá dispatching timeout (mặc định **5s**, `DEFAULT_DISPATCHING_TIMEOUT`) → `onAnrLocked` → `InputManagerService.notifyWindowUnresponsive/notifyNoFocusedWindowAnr` → AMS ANR.
- 2 kiểu: window không phản hồi; **không có focused window** (app đang mở nhưng chưa có window focus — hay gặp khi Activity khởi tạo chậm).
- **⚠️ Đã sửa**: Gemini nhầm `KEY_WAITING_FOR_EVENTS_TIMEOUT` (500ms, dùng cho việc chờ các event trước được xử lý) với timeout ANR 5s.

### Q11. `interceptKeyBeforeQueueing` vs `interceptKeyBeforeDispatching`?
| | BeforeQueueing | BeforeDispatching |
|--|----------------|-------------------|
| Thread | InputReader (gọi từ `notifyKey`) | InputDispatcher |
| Khi nào | trước khi vào hàng đợi, kể cả khi màn hình tắt | ngay trước khi gửi cho window focus |
| Dùng cho | POWER, wake keys, volume/mute, media khi không tương tác | HOME, SEARCH/ASSIST, global keys, shortcuts |
| Kết quả | bỏ cờ `ACTION_PASS_TO_USER` → không tới app | trả `-1` → nuốt phím, `>0` → thử lại sau |

### Q12. Global keys trên Android TV (ví dụ thật trong project)?
- `PhoneWindowManager` dùng `GlobalKeyManager` đọc `com.android.internal.R.xml.global_keys`; nếu phím có trong danh sách → **gửi broadcast `Intent.ACTION_GLOBAL_BUTTON`** (kèm `KeyEvent`) tới component khai báo, kể cả khi app khác đang foreground; `dispatchWhenNonInteractive` cho phép gửi khi đang standby.
- Project override bằng RRO `DavidFrameworkOverlay`:
```xml
<!-- vendor/david/apps/DavidFrameworkOverlay/res/xml/global_keys.xml -->
<global_keys version="1">
    <key keyCode="KEYCODE_F9"  component="com.davidmedia.settings.customizer/.receiver.GlobalKeyReceiver" dispatchWhenNonInteractive="true" />
    <key keyCode="KEYCODE_F10" component="com.davidmedia.settings.customizer/.receiver.GlobalKeyReceiver" dispatchWhenNonInteractive="true" />
    ...
</global_keys>
```
- **⚠️ Đã sửa**: Gemini viết PhoneWindowManager tự `startActivity` Netflix. Thực tế AOSP gửi broadcast tới receiver, receiver mới mở app (vòng 2 Gemini đã đồng ý).

### Q13. `.kl` / `.kcm`?
- Tìm file theo thứ tự: `Vendor_XXXX_Product_XXXX_Version_XXXX.kl` → `Vendor_XXXX_Product_XXXX.kl` → `<device name>.kl` → `Generic.kl`, trong `/odm`, `/vendor`, `/system` `/usr/keylayout/`. Project có thư mục `vendor/david/keylayout/`.
```
key 103   DPAD_UP
key 352   DPAD_CENTER
key 116   POWER        WAKE
key 67    F9                       # sau đó global_keys map F9 → GlobalKeyReceiver
```
- Flag `WAKE` (đánh thức thiết bị khi ngủ). Remote không đánh thức được: thiếu `WAKE` trong `.kl` **hoặc** IR/BT driver không cấu hình wakeup ở kernel.
- Debug: `getevent -lt`, `dumpsys input` (xem file `.kl` được dùng cho từng device).

**Ví dụ thật trong project (remote BT David ALVA M1/M4)**: `vendor/david/keylayout/Vendor_0217_Product_0001.kl` + `.idc`
```
key 116   POWER
key usage 0x0c0075 F9        # map theo HID usage (consumer page), không theo scancode
key 183   F10                # M1
key usage 0x0c0077 F10       # M4
```
```
# Vendor_0217_Product_0001.idc
device.internal = 1          # coi remote BT như thiết bị "internal"
keyboard.builtIn = 0
audio.mic = 1                # remote có micro (voice search)
```
David còn **sửa AOSP** ở `frameworks/native/services/inputflinger/reader/mapper/KeyboardInputMapper.cpp:166-167`:
```cpp
// DAVID: Set doNotWakeByDefault = true by default to prevent external keyboards from waking the device from standby mode
mParameters.doNotWakeByDefault = config.getBool("keyboard.doNotWakeByDefault").value_or(true);
```
(AOSP mặc định `false`, nghĩa là mọi phím của **bàn phím external** đều đánh thức máy, xem đoạn `if (down && isExternal() && !doNotWakeByDefault) policyFlags |= POLICY_FLAG_WAKE;`.) Bản `CursorInputMapper.cpp` cũng có patch tương tự. Câu hỏi phỏng vấn hay rút ra từ đây:
1. *Vì sao bàn phím/chuột USB đánh thức STB ngoài ý muốn và sửa thế nào?* → Có 2 cách: (a) sửa giá trị mặc định trong inputflinger như David đã làm (ảnh hưởng mọi thiết bị); (b) không sửa code, mỗi thiết bị khai `keyboard.doNotWakeByDefault = 1` trong `.idc` riêng. Cách (b) "sạch" hơn, dễ rebase khi lên Android mới.
2. *Vậy remote đánh thức máy bằng gì?* → Với thiết bị internal hoặc đã tắt wake mặc định, phải khai `WAKE` cho từng phím trong `.kl` (comment của AOSP ghi rõ "e.g. TV remotes"); riêng `POWER` thì `PhoneWindowManager.interceptKeyBeforeQueueing` xử lý wake/sleep.

---

## 14.4 Looper native

### Q14. `nativePollOnce` / `nativeWake`?
- `Looper` C++ (`system/core/libutils/Looper.cpp`): `epoll_create1` + `eventfd` (`mWakeEventFd`); `pollOnce` → `epoll_wait(timeout)`; `wake()` ghi `uint64_t 1` vào eventfd.
- Java `MessageQueue.enqueueMessage` tính `needWake` (message mới ở đầu queue, hoặc async message đầu tiên phía sau barrier khi thread đang ngủ) → `nativeWake`.
- `Looper.addFd()` cho phép theo dõi fd khác (InputChannel, DisplayEventReceiver) trên cùng thread.

### Q15. `nativePollOnce` chiếm nhiều thời gian trong trace — có phải lỗi?
Không: đó là thread **đang rảnh** (ngủ trong `epoll_wait`, trạng thái S). Lỗi thật là các slice **trước** nó dài (measure/layout/binder), hoặc thread ở trạng thái `D` (I/O) / `R` lâu (thiếu CPU).

---

## 14.5 Resources & RRO

### Q16. `AssetManager2` resolve ID `0xPPTTEEEE`?
PP = package (0x01 framework, 0x7f app), TT = type, EEEE = entry. `AssetManager2::GetResource` → chọn package group → tìm config phù hợp nhất với `ResTable_config` hiện tại (locale, `uiMode=television`, density...) → nếu có overlay thì dùng idmap để thay giá trị (`frameworks/base/libs/androidfw/AssetManager2.cpp`).

### Q17. RRO, fabricated overlay?
- Overlay APK khai `<overlay android:targetPackage=... android:isStatic=... android:priority=.../>`; `OverlayManagerService` + `idmap2` tạo idmap trong `/data/resource-cache/`.
- Static overlay (như `DavidFrameworkOverlay`) luôn bật, không tắt được bằng `cmd overlay`.
- Fabricated overlay (A12+): tạo overlay bằng code (`FabricatedOverlay`) — dùng để đổi theme theo nhà mạng.
- Một firmware cho nhiều nhà mạng: trong project, cấu hình theo nhà mạng được tách ra `device/amlogic/KSTB2500/oem/sku_*` và `launcher_configuration_<operator>.xml` → so sánh ưu/nhược giữa RRO, oem partition và build-time config.

### Q18. Qualifier cho TV?
`television` (uiMode), `notouch`, `dpad` (navigation). Density: guideline Android TV cho UI 1080p là `xhdpi` (320dpi → 960×540dp); **project này đặt `ro.sf.lcd_density=240`** (`device/amlogic/KSTB2500/KSTB2500.mk:38`, hdpi → 1280×720dp), nên layout app bên thứ ba thiết kế theo 960×540dp sẽ trông nhỏ hơn. Đây là câu hỏi hay: *"density do ai quyết định?"* → `ro.sf.lcd_density` (build) hoặc `wm density` (runtime, lưu `Settings.Secure.display_density_forced`). Đổi độ phân giải HDMI → `Configuration` mới → relaunch hoặc `onConfigurationChanged`.
