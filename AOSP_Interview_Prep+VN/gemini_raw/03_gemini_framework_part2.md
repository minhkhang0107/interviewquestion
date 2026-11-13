Chào bạn. Dưới đây là bộ câu hỏi và đáp án phỏng vấn chuyên sâu dành cho vị trí **Senior Android Platform/Framework Engineer** (tập trung vào Android 14 trên nền tảng Android TV Set-Top Box - STB). 

Tất cả các nội dung đều đi sâu vào kiến trúc bên trong (internals), đối chiếu trực tiếp mã nguồn trong AOSP (`frameworks/base`, `frameworks/native`, `system/core`), các cấu trúc dữ liệu, cơ chế IPC/Kernel, mã minh họa và các pitfall thực tế khi phát triển sản phẩm thương mại.

---

# PHẦN 2: CHUYÊN ĐỀ TẦNG FRAMEWORK (AOSP FRAMEWORKS/BASE)

---

## CHỦ ĐỀ 1: WindowManagerService (WMS)

### Q1. Hãy phân tích cấu trúc phân cấp (Hierarchy) trong WMS từ Android 11 đến Android 14: Phân biệt `WindowContainer`, `DisplayContent`, `Task`, `ActivityRecord`, `WindowToken`, và `WindowState`. Tại sao Google chuyển dịch cấu trúc cây của WMS đồng bộ 1:1 sang SurfaceControl Hierarchy?

**Trả lời:**
* **Cơ chế bên trong & Hierarchy:**
  Trước Android 11, WMS duy trì cây logic riêng (`DisplayContent` -> `TaskStack` -> `Task` -> `AppWindowToken` -> `WindowState`) và đồng bộ thủ công với cây layer của SurfaceFlinger. Từ Android 11 và hoàn thiện trên Android 14, toàn bộ cây WMS kế thừa từ lớp trừu tượng `WindowContainer<E extends WindowContainer>` (`frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java`).
  Cấu trúc cây từ trên xuống dưới:
  1. `RootWindowContainer`: Nút gốc toàn hệ thống.
  2. `DisplayContent`: Đại diện cho một màn hình vật lý/ảo (trên STB là HDMI display, Default Display id = 0).
  3. `DisplayArea`: Phân vùng display (chứa các layer dành cho Features, Ime, Windows).
  4. `TaskDisplayArea`: Nơi chứa các `Task` ứng dụng.
  5. `Task` / `TaskOrganizer`: Quản lý các task, hỗ trợ multi-window/PiP.
  6. `ActivityRecord`: Kế thừa từ `WindowToken`, đại diện cho một Activity.
  7. `WindowToken`: Gom nhóm các cửa sổ có cùng một mục đích hoặc thuộc cùng một Client token (Binder IBinder).
  8. `WindowState`: Nút lá trong cây logic, đại diện cho một cửa sổ thực tế (một instance của `ViewRootImpl` bên phía client).
* **Đồng bộ 1:1 sang SurfaceControl Hierarchy:**
  Mỗi `WindowContainer` đều sở hữu một `SurfaceControl mSurfaceControl`. Bất kỳ thay đổi nào về vị trí, kích thước, visibility, crop, transform matrix trên cây `WindowContainer` đều được ánh xạ trực tiếp thành các thao tác trên `SurfaceControl.Transaction`.
  * *Lý do:* Giảm thiểu IPC và triệt tiêu bất đồng bộ trạng thái (desynchronization) giữa WMS và SurfaceFlinger. Giúp các thao tác Animation (như transition giữa các Activity, PiP animation) được tính toán hoàn toàn phía Shell/WMS và commit nguyên một Transaction nguyên tử (`BLASTSyncEngine`) xuống SurfaceFlinger, loại bỏ hiện tượng xé hình hoặc nhấp nháy đen (flickering).
* **Tên file/class AOSP:**
  * `frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java`
  * `frameworks/base/services/core/java/com/android/server/wm/WindowState.java`
  * `frameworks/base/services/core/java/com/android/server/wm/WindowToken.java`
  * `frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java`
* **Pitfall trên Android TV STB:**
  Khi bật tính năng Picture-in-Picture (PiP) cho luồng phát video Live TV hoặc xử lý HDMI input overlay, nếu developer tự ý gọi `SurfaceView.setZOrderOnTop(true)` hoặc `setZOrderMediaOverlay(true)` mà không kiểm soát thứ tự gán parent trong cây `WindowContainer`, `SurfaceView` có thể bypass qua z-order của cây Activity, đè lên cả System Dialog, Subtitle hoặc UI của launcher Leanback.

---

### Q2. Trình bày chi tiết luồng thực thi của `WindowManager.addView()` từ Client Application qua `ViewRootImpl`, IPC sang WMS, và quá trình cấp phát `SurfaceControl` cho Client.

**Trả lời:**
* **Luồng thực thi chi tiết:**
  1. **Client Side (`WindowManagerImpl` -> `WindowManagerGlobal`):**
     * Ứng dụng gọi `windowManager.addView(view, params)`.
     * `WindowManagerGlobal.addView()` khởi tạo một instance `ViewRootImpl` mới và liên kết với `View` (DecorView) qua `ViewRootImpl.setView(view, params, panelParentView)`.
  2. **Yêu cầu WMS cấp phát Window (`ViewRootImpl.setView`):**
     * `ViewRootImpl` gọi `mWindowSession.addToDisplayAsUser(...)` qua Binder IPC.
     * `mWindowSession` là Binder proxy trỏ tới `Session.java` bên trong tiến trình `system_server`.
  3. **Xử lý tại `WindowManagerService` (`WindowManagerService.java`):**
     * WMS kiểm tra quyền, policy của display qua `PhoneWindowManager.prepareAddWindowLw()`.
     * Tìm kiếm hoặc tạo mới `WindowToken`. Tạo đối tượng `WindowState` tương ứng với Client Window.
     * `WindowState` được gắn vào cây phân cấp thông qua `DisplayContent.addWindowToList()`.
     * WMS gán một `InputChannel` (server side) cho Window này để tiếp nhận input event.
     * Server trả về `InputChannel` (client side) và mã trạng thái (`ADD_OKAY`, v.v.) cho `ViewRootImpl`.
  4. **Cấp phát Surface (`ViewRootImpl.relayoutWindow`):**
     * Trong chu kỳ dựng hình đầu tiên (`performTraversals()`), `ViewRootImpl` gọi `mWindowSession.relayout(...)`.
     * `WindowManagerService.relayoutWindow()` uỷ quyền cho `WindowStateAnimator` / `WindowState.createSurfaceControl()`.
     * WMS tạo một `SurfaceControl` phía native và ghi đè metadata vào đối tượng `Surface` trống của client thông qua Binder transaction (`Surface.copyFrom(outSurfaceControl)`).
     * Phía client, `ViewRootImpl` lấy được `SurfaceControl` hợp lệ trỏ tới layer tương ứng trong SurfaceFlinger và bắt đầu quá trình render qua `BLASTBufferQueue`.
* **Tên file/class AOSP:**
  * `frameworks/base/core/java/android/view/WindowManagerGlobal.java`
  * `frameworks/base/core/java/android/view/ViewRootImpl.java`
  * `frameworks/base/services/core/java/com/android/server/wm/Session.java`
  * `frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java`
* **Code ví dụ thêm view cấp hệ thống từ Service:**
```kotlin
val wm = context.getSystemService(Context.WINDOW_SERVICE) as WindowManager
val params = WindowManager.LayoutParams(
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN,
    PixelFormat.TRANSLUCENT
).apply {
    gravity = Gravity.TOP or Gravity.START
}
val customOverlayView = LayoutInflater.from(context).inflate(R.layout.tv_diagnostics_overlay, null)
wm.addView(customOverlayView, params)
```
* **Pitfall:**
  Gọi `wm.addView()` ngay trong `Service.onCreate()` khi WindowToken của display chưa sẵn sàng (hoặc `DisplayContent` đang reconfigure do HDMI resolution switch) sẽ gây ra crash `BadTokenException` hoặc `InvalidDisplayException`. Luôn cần đảm bảo sử dụng `WindowContext` (Android 12+) thay vì `ApplicationContext`.

---

### Q3. Cơ chế tính toán Z-Order, Window Type (`TYPE_APPLICATION`, `TYPE_APPLICATION_OVERLAY`, `TYPE_SYSTEM_ALERT`) và cách WMS phân chia tầng lớp thông qua SurfaceControl Transaction trên Android 14 như thế nào?

**Trả lời:**
* **Phân loại Window Type:**
  Trong `WindowManager.LayoutParams`:
  * *Application Windows (1 - 99):* `TYPE_BASE_APPLICATION`, `TYPE_APPLICATION`.
  * *Sub-windows (1000 - 1999):* `TYPE_APPLICATION_PANEL`, `TYPE_APPLICATION_MEDIA` (cho SurfaceView). Z-order của sub-window luôn gắn chặt và phụ thuộc vào Window chủ (Parent Window).
  * *System Windows (2000+):* `TYPE_APPLICATION_OVERLAY` (thay thế cho `TYPE_SYSTEM_ALERT`, `TYPE_SYSTEM_OVERLAY` từ Android 8+), `TYPE_STATUS_BAR`, `TYPE_NAVIGATION_BAR`.
* **Cơ chế tính toán Z-Order:**
  * WMS sử dụng `DisplayAreaPolicy` và các `DisplayArea` con (WindowContainers) để phân bổ các tầng:
    * `APPLICATION_LAYER`: Nơi chứa Activity.
    * `ABOVE_APPLICATION_LAYER`: Chứa các System Dialog, Overlay.
    * `BELOW_APPLICATION_LAYER`: Media layer, Wallpaper.
  * Mỗi `WindowContainer` con được gán một layer số học relative hoặc absolute thông qua hàm `assignChildLayers(Transaction t)`.
  * WMS gán Z-order thực tế thông qua gọi lệnh `SurfaceControl.Transaction.setLayer(SurfaceControl sc, int z)` hoặc `setRelativeLayer(SurfaceControl sc, SurfaceControl relativeTo, int z)`.
  * Đối với Sub-window như `SurfaceView`, khi gọi `setZOrderMediaOverlay(true)`, WMS đặt Surface này có relative layer cao hơn layer của Surface Activity chính nhưng vẫn nằm dưới `Window` của Subtitle hay Control controls.
* **Tên file/class AOSP:**
  * `frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java`
  * `frameworks/base/services/core/java/com/android/server/wm/DisplayArea.java`
  * `frameworks/base/services/core/java/com/android/server/wm/WindowLayersController.java`
* **Pitfall:**
  Trên Android TV, nhiều ứng dụng launcher của nhà mạng (Operator Tier STB) cố gắng tạo một cửa sổ `TYPE_APPLICATION_OVERLAY` để hiển thị banner khẩn cấp (Emergency Broadcast) nhưng quên không set flag `FLAG_NOT_TOUCH_MODAL` hoặc `FLAG_NOT_FOCUSABLE`. Hậu quả là cửa sổ này cướp toàn bộ Focus của DPAD, khiến người dùng không thể bấm remote để điều khiển các ứng dụng video bên dưới.

---

### Q4. Quyền `SYSTEM_ALERT_WINDOW` và `Settings.canDrawOverlays()` hoạt động như thế nào ở mức Framework? Trên Android TV STB, có điểm gì đặc thù đối với Overlay Permission so với Mobile?

**Trả lời:**
* **Cơ chế kiểm tra quyền ở Framework:**
  Khi Client gọi `mWindowSession.addToDisplayAsUser()` với window type là `TYPE_APPLICATION_OVERLAY`:
  1. WMS chuyển quyền kiểm tra sang `DisplayPolicy.checkAddPermission()`.
  2. Framework truy vấn `AppOpsManager.noteOpNoThrow(AppOpsManager.OP_SYSTEM_ALERT_WINDOW, uid, packageName)`.
  3. Quyền này không chỉ là permission manifest thông thường (`android.permission.SYSTEM_ALERT_WINDOW`) mà phụ thuộc vào trạng thái của AppOp `MODE_ALLOWED`.
  4. Nếu AppOp trả về `MODE_ERRORED` hoặc `MODE_IGNORED`, WMS ném ra ngoại lệ `BadTokenException: Unable to add window -- token null is not valid; is your activity running?`.
* **Đặc thù trên Android TV STB:**
  1. *Không có Intent Cài Đặt chuẩn:* Trên TV, Activity `Settings.ACTION_MANAGE_OVERLAY_PERMISSION` thường không có giao diện chuẩn (AOSP Leanback Settings không cài sẵn màn hình toggle cho từng app vì lý do bảo mật và UX remote). Nếu app cố tình gọi Intent này, hệ thống sẽ gặp `ActivityNotFoundException`.
  2. *Quyền cho System/Privileged App:* Các ứng dụng do nhà mạng (Operator STB) cài trong `/system/priv-app/` hoặc `/vendor/priv-app/` phải được whitelist trong file xml: `/etc/permissions/privapp-permissions-xxx.xml` với quyền `SYSTEM_ALERT_WINDOW`, hoặc được OEM cấp cứng `default-grant` trong `DefaultPermissionGrantPolicy.java`.
  3. *Hạn chế hiệu năng phần cứng STB:* Phần cứng STB cấp thấp (budget SoC như Amlogic S905Y4, Realtek RTD1319) có băng thông bộ nhớ (DRAM bandwidth) rất hạn chế. Việc vẽ nhiều tầng `TYPE_APPLICATION_OVERLAY` làm SurfaceFlinger phải fallback từ Hardware Overlay plane (HWC) sang GPU Composition (RenderEngine/GLES), gây tụt FPS trầm trọng cho luồng video 4K bên dưới.

---

## CHỦ ĐỀ 2: Graphics Pipeline

### Q1. Trình bày chi tiết mối quan hệ giữa VSYNC-app, VSYNC-sf, `Choreographer`, `DispSync`/`VsyncSchedule`. Tại sao lại có pha trễ (phase offset) giữa VSYNC-app và VSYNC-sf?

**Trả lời:**
* **Cơ chế kiến trúc:**
  * Màn hình phát ra tín hiệu phần cứng định kỳ `HW_VSYNC` (thông qua HDMI Driver/DRM-KMS driver gửi đến Hardware Composer - HWC).
  * Trong SurfaceFlinger, `VsyncSchedule` (trước đây là `DispSync`) tiếp nhận `HW_VSYNC` để huấn luyện một mô hình phần mềm (Phase-Locked Loop - PLL) nhằm dự đoán chu kỳ VSYNC tiếp theo, giúp giảm tải ngắt phần cứng (hardware interrupts).
  * Từ bộ dự đoán này, hệ thống phân tách thành hai xung nhịp logic lệch pha nhau:
    1. `VSYNC-app`: Phát cho ứng dụng để thực hiện việc đo đạc, bố trí và vẽ (`Choreographer.doFrame()`).
    2. `VSYNC-sf`: Phát cho `SurfaceFlinger` để gom các buffer từ các client, chuẩn bị cho quá trình composite.
* **Tại sao cần Phase Offset (Pha trễ)?**
  * Trong một chu kỳ quét (ví dụ 60Hz = 16.6ms hoặc 120Hz = 8.3ms):
    * Khi `VSYNC-app` kích hoạt, ứng dụng bắt đầu: Input -> Animation -> Traversal (Measure/Layout/Draw) -> RenderThread gửi lệnh sang GPU/Vulkan. Quá trình này cần thời gian $T_{app}$.
    * Sau khi GPU vẽ xong vào Buffer, buffer được đưa vào hàng đợi (`queueBuffer`).
    * `SurfaceFlinger` cần thức dậy tại `VSYNC-sf` (trễ hơn `VSYNC-app` một khoảng offset định trước $Offset_{sf}$) để nhận buffer vừa vẽ, gọi HWC chuẩn bị hình ảnh trước khi tín hiệu quét màn hình phần cứng tiếp theo diễn ra.
  * *Nếu offset = 0:* `SurfaceFlinger` thức dậy cùng lúc với app. Lúc đó buffer mới của app chưa vẽ xong, SurfaceFlinger buộc phải dùng lại buffer cũ của khung hình trước -> Gây ra độ trễ hiển thị (latency) tăng thêm trọn vẹn 1 frame (16.6ms).
* **Tên file/class AOSP:**
  * `frameworks/base/core/java/android/view/Choreographer.java`
  * `frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp`
  * `frameworks/native/services/surfaceflinger/Scheduler/VSYNCDispatchTimerQueue.cpp`
* **Pitfall:**
  Trên STB kết nối các TV 4K có tần số quét biến thiên hoặc khi người dùng đổi chế độ ngõ ra HDMI giữa 50Hz (chuẩn PAL châu Âu) và 59.94Hz/60Hz (chuẩn NTSC), nếu driver HDMI không bắn lại hotplug event chính xác đến HWC, `VsyncSchedule` sẽ tính sai chu kỳ, dẫn đến hiện tượng Micro-stuttering (khung hình bị giật lặp đi lặp lại mỗi vài giây một lần dù app không drop frame nào).

---

### Q2. Phân tích sự chuyển dịch từ `BufferQueue` truyền thống sang `BLASTBufferQueue` (Buffer-as-a-Transaction) từ Android 11 đến Android 14. Tại sao Google thực hiện kiến trúc này?

**Trả lời:**
* **Kiến trúc BufferQueue truyền thống:**
  * Client (Producer) gọi `dequeueBuffer()` -> vẽ -> `queueBuffer()` trực tiếp vào `BufferQueueCore`.
  * `BufferQueue` bắn callback `onFrameAvailable()` tới `SurfaceFlinger` (Consumer).
  * Đồng thời, khi kích thước hoặc vị trí cửa sổ thay đổi, WMS phải gửi một IPC riêng biệt (`SurfaceControl.Transaction.setPosition()`, `setSize()`) xuống SurfaceFlinger.
  * *Hạn chế:* Việc đẩy Buffer và đẩy Geometry State đi qua 2 đường ống độc lập. Dẫn tới hiện tượng: Buffer mới (kích thước mới) đã xuống SurfaceFlinger nhưng Transaction đổi frame/bounds từ WMS chưa tới kịp (hoặc ngược lại) -> Gây ra hiện tượng đen góc màn hình, lệch scale hình ảnh trong vài ms khi xoay màn hình hoặc resize window.
* **Kiến trúc `BLASTBufferQueue` (Android 11-14):**
  * BLAST = **B**uffer **L**ike **A** **S**urfaceControl **T**ransaction.
  * Thay vì gửi buffer trực tiếp vào `BufferQueue` của SurfaceFlinger, `ViewRootImpl` đẩy buffer vào `BLASTBufferQueue` chạy cục bộ ngay trong process của client hoặc qua wrapper `SurfaceComposerClient`.
  * Khi buffer sẵn sàng, `BLASTBufferQueue` đóng gói Buffer đó thành một thuộc tính bên trong chính đối tượng `SurfaceControl.Transaction`:
    ```cpp
    transaction.setBuffer(surfaceControl, graphicBuffer);
    transaction.setAcquireFence(fence);
    transaction.apply();
    ```
  * Transaction này có thể gộp chung (merged) với Transaction thay đổi vị trí, kích thước, matrix từ WMS thành **một Transaction nguyên tử duy nhất**.
* **Ưu điểm vượt trội:**
  1. *Nguyên tử hóa tuyệt đối:* Buffer và Window properties hiển thị đồng thời tại cùng một frame trong SurfaceFlinger.
  2. *Hỗ trợ Shell Transitions & PiP mượt mà:* Khi thu phóng video sang cửa sổ PiP trên Android TV, kích thước khung hình video và vị trí window co dãn hoàn hảo 1:1, triệt tiêu hoàn toàn giật bóng hình.
* **Tên file/class AOSP:**
  * `frameworks/native/libs/gui/BLASTBufferQueue.cpp`
  * `frameworks/base/core/java/android/view/ViewRootImpl.java` (biến `mBlastBufferQueue`)
  * `frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp`

---

### Q3. Đi qua luồng dữ liệu đồ họa hoàn chỉnh từ Client render (Skia/Vulkan/OpenGL ES) -> GraphicBuffer -> SurfaceFlinger -> Hardware Composer (HWC3 AIDL) -> DRM/KMS.

**Trả lời:**
* **Các bước của Graphics Pipeline:**
  1. **Client Draw (`RenderThread`):**
     * `ViewRootImpl` yêu cầu render qua `HardwareRenderer`.
     * `RenderThread` giao tiếp với thư viện đồ họa (Skia dùng Vulkan pipeline hoặc GLES).
     * `BLASTBufferQueue` cung cấp một `GraphicBuffer` (cấp phát bởi `GraphicBufferAllocator` từ Linux ION/DMA-BUF heap thông qua Gralloc HAL).
     * GPU thực thi các lệnh vẽ vào buffer này. Khi hoàn thành, một sync fence (`acquire_fence`) được sinh ra.
  2. **Gửi sang SurfaceFlinger:**
     * `BLASTBufferQueue` chuyển buffer kèm `acquire_fence` qua Transaction Binder IPC tới SurfaceFlinger.
     * SurfaceFlinger lưu trữ buffer vào trạng thái tạm thời của layer (`mCurrentState`).
  3. **SurfaceFlinger Message Refresh:**
     * Khi nhận tín hiệu `VSYNC-sf`, SurfaceFlinger gọi `onMessageReceived()` -> `handleMessageInvalidate()`.
     * `latchBuffers()`: Kiểm tra các sync fence; nếu GPU client vẽ xong, buffer chính thức chuyển thành `mDrawingState`.
  4. **Hardware Composer Composition:**
     * SurfaceFlinger gửi danh sách các Layer kèm Buffer Handle xuống HWC3 HAL qua giao tiếp AIDL (`IComposerClient`).
     * `HWC` phân loại các Layer:
       * `DEVICE`: HWC/Display Processor phần cứng của chip STB tự gánh việc scale, convert color space (YUV sang RGB) và tổng hợp trực tiếp lên màn hình mà không cần tốn tài nguyên GPU.
       * `CLIENT`: HWC không xử lý nổi (quá nhiều layer, blend phức tạp, thiếu phần cứng overlay), yêu cầu SurfaceFlinger phải tự vẽ các layer này bằng GPU (RenderEngine) vào một Framebuffer trung gian.
  5. **Xuất hình (DRM/KMS):**
     * HWC cấu hình các hardware plane trong Linux Kernel thông qua Direct Rendering Manager / Kernel Mode Setting (`DRM/KMS` ioctl `DRM_IOCTL_MODE_ATOMIC_COMMIT`).
     * Tín hiệu quét ra cổng HDMI Tx để đưa lên màn hình TV thông qua TMDS/FRL clock.
* **Tên file/class AOSP:**
  * `frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp`
  * `frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp`
  * `hardware/interfaces/graphics/composer/`

---

### Q4. So sánh Double Buffering và Triple Buffering. Trong trường hợp nào Android TV STB 4K HDR rơi vào Triple Buffering và cái giá phải trả là gì?

**Trả lời:**
* **Bản chất kỹ thuật:**
  * **Double Buffering:** Gồm 2 buffer (Front Buffer - màn hình đang đọc; Back Buffer - GPU đang vẽ).
    * Nếu GPU vẽ chậm, không hoàn thành trước VSYNC tiếp theo: Back Buffer bị khóa (locked), GPU không có buffer mới để vẽ khung hình kế tiếp và buộc phải nhàn rỗi (idle) cho tới kỳ VSYNC sau. Hệ thống lập tức bị tụt một nửa số khung hình (từ 60fps rớt xuống 30fps).
  * **Triple Buffering:** Bổ sung thêm buffer thứ ba.
    * Khi GPU vẽ lố qua VSYNC, Front Buffer vẫn đang scan-out, Back Buffer vẫn đang hoàn thiện vẽ, GPU không phải dừng lại mà lấy ngay buffer thứ 3 ra để bắt đầu chuẩn bị khung hình tiếp theo. Điều này triệt tiêu hiện tượng drop fps sâu, duy trì animation đều đặn.
* **Trên Android TV STB 4K HDR:**
  * Khi xuất hình ở độ phân giải 4K (3840x2160), mỗi `GraphicBuffer` (chuẩn RGBA_8888 hoặc HDR10 YU10) tiêu tốn:
    $$3840 \times 2160 \times 4 \text{ bytes} \approx 33.17 \text{ MB / buffer}$$
    Triple buffer cho một Layer giao diện tiêu tốn $\approx 100\text{MB}$. Nếu có nhiều layer (UI, PiP Video, Subtitle), con số này nhân lên nhanh chóng.
  * **Khi nào rơi vào Triple Buffering?**
    * Khi phát video 4K HDR đồng thời hiển thị UI của launcher phong phú (nhiều Blur, Lottie animation hoặc Recyclerview tải poster tốc độ cao), GPU Mali/PowerVR trên STB bị quá tải, thời gian vẽ vượt 16.6ms -> `BufferQueueProducer` sẽ cấp phát buffer thứ 3 (nếu `maxBufferCount` cho phép).
* **Cái giá phải trả (Pitfalls/Trade-offs):**
  1. *Input Latency (Độ trễ remote):* Khi Triple Buffering hoạt động hết công suất, khung hình người dùng nhìn thấy trên TV đã được render trước đó 2 đến 3 chu kỳ VSYNC ($\sim 33 - 50\text{ms}$). Người dùng bấm phím điều hướng DPAD sẽ cảm nhận rõ độ trễ (input lag), cảm giác giao diện bị "trượt" hoặc không ăn phím.
  2. *Áp lực VRAM/DRAM:* STB phân khúc phổ thông chỉ có 2GB RAM dùng chung (Unified Memory giữa CPU và GPU). Việc ngốn thêm hàng trăm MB cho Graphic Buffers dễ dẫn đến Kernel Low Memory Killer (`lmkd`) kích hoạt, kill các background services ngầm của hệ thống.

---

## CHỦ ĐỀ 3: Input Pipeline

### Q1. Trình bày kiến trúc Input Pipeline trong Android 14: Luồng sự kiện từ Kernel `evdev` -> `EventHub` -> `InputReader` -> `InputDispatcher` -> `InputChannel` (socket pair).

**Trả lời:**
* **Kiến trúc các thành phần:**
  Toàn bộ nằm trong tiến trình native của `system_server` dưới tên `InputManagerService` (gọi sang C++ `InputManager.cpp`).
  1. **`EventHub` (`EventHub.cpp`):**
     * Sử dụng Linux `INotify` để theo dõi các thiết bị input mới trong `/dev/input/` (Bluetooth Remote, USB Keyboard, IR Receiver).
     * Dùng `epoll` để lắng nghe dữ liệu ngắt từ các file descriptor `evdev`.
     * Khi người dùng bấm phím, đọc cấu trúc raw `input_event` của Linux kernel qua lệnh gọi `read()`.
  2. **`InputReader` (`InputReader.cpp`):**
     * Chạy trên thread riêng biệt `InputReaderThread`.
     * Nhận raw event từ `EventHub`. Dựa vào device type, gọi các `InputMapper` (ví dụ `KeyboardInputMapper`) để xử lý.
     * Dùng file Key Layout (`.kl`) chuyển đổi Linux Scan Code sang Android Key Code (ví dụ `KEY_UP` -> `KEYCODE_DPAD_UP`).
     * Tạo ra các `NotifyKeyArgs` hoặc `NotifyMotionArgs` rồi đẩy vào hàng đợi của `InputDispatcher`.
  3. **`InputDispatcher` (`InputDispatcher.cpp`):**
     * Chạy trên thread riêng `InputDispatcherThread`.
     * Quản lý trạng thái focus: nắm giữ danh sách các cửa sổ nhận input (`mWindowHandles`) do WMS đẩy sang thông qua `InputMonitor`.
     * Xác định mục tiêu (Target Window): Tìm ra `WindowState` nào đang có focus hoặc nhận touch event.
     * Gửi sự kiện vào `InputChannel` tương ứng của Window đó.
  4. **`InputChannel` (`InputTransport.cpp`):**
     * Là một lớp bọc quanh cặp Unix Domain Socket (`socketpair`) full-duplex non-blocking.
     * Phía Server (`InputDispatcher`) ghi `InputMessage` vào socket.
     * Phía Client (`ViewRootImpl`), socket descriptor được đưa vào `Looper` của main thread qua `nativePollOnce`. Khi có event, `InputConsumer` đọc từ socket, chuyển thành đối tượng Java `KeyEvent`/`MotionEvent` và gửi lên cây View (`View.dispatchKeyEvent`). Sau khi xử lý xong, Client gửi ngược lại tín hiệu `finishedSignal` về cho Server.
* **Tên file/class AOSP:**
  * `frameworks/native/services/inputflinger/reader/EventHub.cpp`
  * `frameworks/native/services/inputflinger/reader/InputReader.cpp`
  * `frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp`
  * `frameworks/native/libs/input/InputTransport.cpp`

---

### Q2. Phân tích nguyên nhân gốc rễ và cơ chế phát hiện Input ANR (5 giây) trong `InputDispatcher`.

**Trả lời:**
* **Cơ chế phát hiện Input ANR:**
  * Trong `InputDispatcher.cpp`, mỗi sự kiện gửi đi được theo dõi qua một danh sách hàng đợi outbound gọi là `WaitQueue` (chứa các lệnh đã gửi qua socket nhưng chưa nhận được `finishedSignal` từ client).
  * Khi sự kiện mới đến:
    1. `InputDispatcher` kiểm tra xem Window mục tiêu có đang bận không (hàng đợi `WaitQueue` của window đó có sự kiện cũ nào chưa được xử lý không).
    2. Nếu có sự kiện cũ tồn đọng và thời gian vượt quá `KEY_WAITING_FOR_EVENTS_TIMEOUT` (mặc định 5 giây đối với input thông thường):
    3. `InputDispatcher` gọi hàm `processWaitQueue()` và phát hiện timeout. Nó kích hoạt hàm `onANRLocked()`.
    4. Gửi thông báo sang `InputManagerCallback` (Java side) -> `InputManagerService.java` -> `AnrController` -> `ActivityManagerService.appNotResponding()`.
* **Nguyên nhân gốc rễ (Root Causes):**
  1. *Main thread của App bị nghẽn (Blocked):* Đang chạy tính toán nặng, query Room Database đồng bộ, hoặc gọi SharedPreferences trên UI thread khiến `Looper` không quay lại để `InputConsumer` đọc socket.
  2. *Binder IPC Lock Contention:* Main thread gọi một hàm IPC đồng bộ (synchronous binder call) sang một service khác (như `PackageManager`, `MediaSessionManager`) nhưng service đó đang bị treo hoặc dính deadlock.
  3. *Deadlock trong `View.dispatchKeyEvent()`:* Các thư viện UI tùy biến can thiệp vào sự kiện điều hướng D-PAD và rơi vào vòng lặp vô hạn hoặc chờ synchronized lock của một worker thread khác.
* **Pitfall trên Android TV:**
  Lập trình viên TV thường có thói quen handle sự kiện bấm phím D-PAD trong `onKeyDown()` và xử lý load preview hình ảnh/video ngay lập tức. Khi người dùng đè chặt phím D-PAD (long press để cuộn nhanh qua danh sách hàng trăm phim), hàng trăm `KeyEvent` liên tục được bắn vào socket. Nếu không drop các sự kiện lặp (`event.getRepeatCount() > 0`), hàng đợi socket đầy phình to, dẫn đến Client xử lý chậm và dính Input ANR tức thì.

---

### Q3. Phân tích luồng Intercept Key trong `PhoneWindowManager`: Phân biệt `interceptKeyBeforeQueueing` và `interceptKeyBeforeDispatching`. Cách xử lý các phím chuyên dụng trên Remote TV (Netflix, YouTube, Prime Video, TV Power).

**Trả lời:**
* **`PhoneWindowManager` Key Interception Lifecycle:**
  Khi một phím được bấm trên Remote TV:
  1. **`interceptKeyBeforeQueueing(KeyEvent event, int policyFlags)`:**
     * Chạy ngay trên luồng `InputReaderThread` của `InputManagerService`.
     * Mục đích: Đón đầu các phím chức năng hệ thống cấp thấp trước cả khi chúng được xếp vào hàng đợi của `InputDispatcher`.
     * Các phím xử lý tại đây: Phím Power (`KEYCODE_POWER`), Phím CEC Standby, Mute âm lượng (`KEYCODE_VOLUME_MUTE`), Home button (`KEYCODE_HOME`).
     * *Đặc điểm:* Nếu hàm này trả về giá trị tiêu thụ phím (hoặc không gắn cờ dispatch), phím đó **bị triệt tiêu hoàn toàn**, không bao giờ đi tới bất kỳ ứng dụng nào hay cửa sổ nào.
  2. **`interceptKeyBeforeDispatching(IBinder focusedToken, KeyEvent event, int policyFlags)`:**
     * Chạy trên luồng `InputDispatcherThread` ngay trước khi sự kiện được ghi vào socket để gửi cho App có focus.
     * Mục đích: Cho phép hệ thống thực thi các lối tắt toàn cục (Global Key Shortcuts) nhưng vẫn cho phép app có cơ hội ghi đè nếu hệ thống cho phép.
     * Xử lý: Phím `KEYCODE_SEARCH` (kích hoạt Google Assistant), Phím Menu, Phím App Switcher.
* **Xử lý các phím cứng OTT (Netflix, YouTube) trên Android TV Remote:**
  * Các phím OTT có mã phím riêng như `KEYCODE_BUTTON_1`, `KEYCODE_BUTTON_2` hoặc được chuẩn hóa từ Android 10+ thành: `KEYCODE_PAIRING`, `KEYCODE_GUIDE`, hoặc mã dành riêng do OEM định nghĩa.
  * Trong `PhoneWindowManager.java`, hàm `interceptKeyBeforeDispatching()` sẽ đọc file cấu hình hệ thống: `res/xml/global_keys.xml`.
  * Khi bấm phím "Netflix", hệ thống chặn tại đây, đóng gói một explicit Intent:
    ```java
    Intent intent = new Intent(Intent.ACTION_VIEW);
    intent.setPackage("com.netflix.ninja");
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    mContext.startActivity(intent);
    ```
  * Sự kiện không được dispatch xuống app đang chạy phía trước (Foreground App), đảm bảo nút OTT luôn có tác dụng tức thì trong bất kỳ trạng thái nào.
* **Tên file/class AOSP:**
  * `frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java`
  * `frameworks/base/core/res/res/xml/global_keys.xml`

---

### Q4. Cấu hình Key Layout (`.kl`) và Key Character Map (`.kcm`) cho Remote Bluetooth/IR trên Android TV. Quá trình ánh xạ từ Linux Scan Code sang Android Keycode diễn ra ở đâu?

**Trả lời:**
* **Quá trình ánh xạ (Mapping Process):**
  1. Remote bấm phím -> IR sensor hoặc Bluetooth HID module trên STB nhận tín hiệu -> Linux kernel driver (ví dụ `hid-input`) decode thành mã **Linux Scancode** (ví dụ `103` cho phím Lên, `108` cho phím Xuống).
  2. `EventHub.cpp` mở thiết bị `/dev/input/eventX`, đọc thông tin Vendor ID (VID), Product ID (PID) và Device Name.
  3. `EventHub` tìm kiếm file cấu hình theo thứ tự ưu tiên trong `/vendor/usr/keylayout/`, `/system/usr/keylayout/`:
     * `Vendor_XXXX_Product_XXXX_Version_XXXX.kl`
     * `Vendor_XXXX_Product_XXXX.kl`
     * `DEVICE_NAME.kl`
     * `Generic.kl` (Fallback mặc định của AOSP).
  4. `InputReader` sử dụng bảng này để chuyển Scan Code thành **Android Keycode** tương ứng.
* **Cấu trúc file `.kl` (Key Layout):**
  Định dạng: `key <Linux_Scan_Code> <Android_Keycode_Name> <Flags>`
  ```text
  # File: /vendor/usr/keylayout/Vendor_1234_Product_5678.kl
  key 103   DPAD_UP
  key 108   DPAD_DOWN
  key 105   DPAD_LEFT
  key 106   DPAD_RIGHT
  key 352   DPAD_CENTER
  key 172   HOME
  key 217   ASSIST            WAKE
  key 580   TV_INPUT          WAKE
  key 590   BUTTON_1          # Nút Custom OTT Netflix
  ```
  * *Flags:*
    * `WAKE`: Cho phép phím này đánh thức SoC của STB khi đang ở chế độ Deep Sleep/Standby.
* **File `.kcm` (Key Character Map):**
  * Định nghĩa ký tự Unicode thực tế sinh ra khi kết hợp với các phím bổ trợ (Shift, Alt, Fn) hoặc khai báo loại bàn phím (`type SPECIAL_FUNCTION` đối với Remote TV).
* **Pitfall:**
  Một lỗi kinh điển khi mang driver remote Bluetooth lên Android TV: Remote không thể wake-up được STB khi rơi vào sleep. Lý do là trong file `.kl`, kỹ sư quên gắn cờ `WAKE` sau mã phím `HOME` hoặc `POWER`, hoặc cờ `KEY_WAKE` chưa được bật ở tầng kernel driver trong struct `input_dev->keybit`.

---

## CHỦ ĐỀ 4: Handler/Looper Native

### Q1. Bản chất bên dưới của `MessageQueue.nativePollOnce(ptr, timeout)` là gì? Đi sâu vào C++ `Looper.cpp`, Linux `epoll`, và `eventfd`.

**Trả lời:**
* **Bản chất kỹ thuật:**
  `nativePollOnce` không phải là một hàm sleep tiêu tốn CPU. Đây là một cơ chế đồng bộ hóa I/O multiplexing dựa trên Linux Kernel system call: `epoll_wait()`.
* **Cơ chế hoạt động bên trong C++ (`Looper.cpp`):**
  1. Khi đối tượng Java `MessageQueue` được khởi tạo, nó gọi xuống hàm `nativeInit()` của JNI.
  2. JNI tạo một đối tượng C++ `NativeMessageQueue`, và đối tượng này tạo một C++ `android::Looper`.
  3. Trong `Looper::Looper()`, hệ thống khởi tạo một epoll instance qua `epoll_create1(EPOLL_CLOEXEC)`.
  4. Hệ thống tạo một file descriptor để báo thức thông qua `eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC)` (trước Linux 2.6.27 dùng `pipe()`). Biến này lưu là `mWakeEventFd`.
  5. Gọi `epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, &eventItem)` để giám sát sự kiện đọc trên `mWakeEventFd`.
  6. Khi Java `MessageQueue.next()` không có tin nhắn cần xử lý ngay, nó gọi `nativePollOnce(ptr, nextPollTimeoutMillis)`.
  7. C++ `Looper::pollOnce()` gọi `Looper::pollInner()` -> Cuối cùng gọi:
     ```cpp
     int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
     ```
  8. Tại thời điểm này, Linux Kernel đưa thread của ứng dụng vào trạng thái **WAITING / SLEEP** (bỏ ra khỏi hàng đợi scheduler của CPU). CPU hoàn toàn không tốn clock cycle nào cho thread này.
* **Tên file/class AOSP:**
  * `frameworks/base/core/jni/android_os_MessageQueue.cpp`
  * `system/core/libutils/Looper.cpp`

---

### Q2. Cơ chế đánh thức Looper (`Looper.wake()`) diễn ra như thế nào khi có một tin nhắn mới được gửi vào MessageQueue từ một Worker Thread?

**Trả lời:**
* **Luồng đánh thức từ Java sang C++:**
  1. Một Worker Thread gọi `handler.sendMessage(msg)` hoặc `handler.post(runnable)`.
  2. Trong `MessageQueue.enqueueMessage(Message msg, long when)`:
     * Đồng bộ hóa bằng `synchronized (this)`.
     * Chèn `Message` mới vào danh sách liên kết đơn dựa trên biến thời gian `when`.
  3. **Xác định điều kiện cần Wake:**
     * Biến boolean `needWake` được tính toán:
       * Nếu tin nhắn mới nằm ở đầu hàng đợi (`msg == mMessages`).
       * Hoặc nếu Main Thread đang bị block bởi một Synchronization Barrier (tin nhắn có target null) và tin nhắn mới chèn vào là tin nhắn bất đồng bộ (Asynchronous message) đầu tiên.
  4. **Kích hoạt Wakeup:**
     * Nếu `needWake == true` và `mBlocked == true` (thread đang ngủ trong `nativePollOnce`), Java gọi `nativeWake(mPtr)`.
     * `android_os_MessageQueue_nativeWake()` gọi sang C++ `Looper::wake()`.
  5. **C++ `Looper::wake()`:**
     * Hàm này ghi một giá trị số nguyên 64-bit (giá trị = 1) vào `mWakeEventFd` bằng hàm `write()`:
       ```cpp
       uint64_t inc = 1;
       ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
       ```
     * Việc ghi dữ liệu này làm cho `mWakeEventFd` trong Linux kernel có trạng thái `EPOLLIN` (có dữ liệu để đọc).
  6. `epoll_wait()` trong Main Thread lập tức kết thúc block, trả về quyền thực thi cho `Looper.cpp`.
  7. `Looper::pollInner()` đọc bỏ dữ liệu khỏi `mWakeEventFd` bằng `read()`, thoát khỏi JNI và quay trở lại Java `MessageQueue.next()` để lấy `Message` vừa được thêm vào và dispatch.

---

### Q3. Phân tích Thread Safety giữa Java và Native Layer của `MessageQueue`. Cơ chế hoạt động và ứng dụng thực tiễn của `IdleHandler` trên Android TV là gì?

**Trả lời:**
* **Thread Safety giữa Java và Native:**
  * Tại Java layer, mọi thao tác chèn (`enqueueMessage`), lấy (`next`), và xóa (`removeMessages`) đều được bảo vệ nghiêm ngặt bằng monitor lock nội tại của chính instance `MessageQueue`: `synchronized(this)`.
  * Tại Native layer, C++ `Looper` bảo vệ các biến nội tại (như danh sách các fd đăng ký thêm qua `addFd()`, `mRequests`) bằng một `std::mutex mLock`.
  * Cầu nối giữa hai bên: Khi Java gọi `nativePollOnce()`, nó **nhả hoàn toàn** lock `synchronized(this)` của Java trước khi vào C++ sleep. Nhờ đó, bất kỳ worker thread nào khác vẫn có thể lấy lock của `MessageQueue` để gọi `enqueueMessage()` mà không bị deadlock.
* **Cơ chế hoạt động của `IdleHandler`:**
  * `IdleHandler` là một interface định nghĩa phương thức `boolean queueIdle()`.
  * Khi `MessageQueue.next()` chạy:
    1. Không tìm thấy message nào trong queue, hoặc message tiếp theo có thời gian thực thi `when > now`.
    2. Trước khi quyết định đi vào giấc ngủ trong `nativePollOnce()`, queue kiểm tra danh sách `mIdleHandlers`.
    3. Nếu có, nó duyệt qua mảng và gọi `queueIdle()` tuần tự.
    4. Nếu trả về `false`, handler đó sẽ bị tự động xóa khỏi danh sách; nếu `true`, sẽ tiếp tục được gọi trong các lần idle sau.
* **Ứng dụng thực tiễn trên Android TV STB:**
  * *Tối ưu hóa khởi động Launcher Leanback:* Khi mở màn hình Home, việc tải poster, khởi tạo các player thumbnail ẩn tiêu tốn rất nhiều tài nguyên. Thay vì chạy ngay trong `onCreate()` hay `onResume()` gây rớt khung hình (jank), ta đăng ký `IdleHandler`:
    ```kotlin
    Looper.myQueue().addIdleHandler {
        // Chỉ chạy khi Main Thread hoàn toàn rảnh rỗi và đã vẽ xong khung hình đầu tiên
        prefetchEPGMetadataInBackground()
        initializeVoiceAssistantSDK()
        false // Tự hủy sau khi chạy xong
    }
    ```

---

### Q4. Pitfall: Tại sao `nativePollOnce` thường chiếm thời gian rất lớn trong Systrace/Perfetto? Làm thế nào để phân biệt giữa Thread nhàn rỗi (Idle) và bị nghẽn (Blocked/Contention)?

**Trả lời:**
* **Tại sao `nativePollOnce` hiển thị thời gian lớn?**
  * Đây là lỗi phổ biến nhất của các kỹ sư mới làm quen với công cụ profiling: Nhìn vào trace thấy `nativePollOnce` chiếm 90% thời gian của Main Thread và kết luận ứng dụng bị nghẽn tại đây.
  * *Thực tế:* Main thread của Android là một vòng lặp sự kiện (Event-driven). Khi ứng dụng không có việc gì làm (người dùng không bấm remote, không có animation, không có network callback), thread sẽ ngủ bên trong `nativePollOnce` thông qua `epoll_wait` để chờ sự kiện tiếp theo. Do đó, thời gian `nativePollOnce` kéo dài chứng tỏ **hệ thống đang rảnh rỗi**, CPU đang được tiết kiệm điện.
* **Cách phân biệt giữa Nhàn rỗi (Idle) và Bị nghẽn (Blocked):**
  1. **Quan sát Trạng thái Thread (Thread State) trong Perfetto:**
     * *Idle thật sự:* Thread State là `Sleeping` (hoặc `Interruptible Sleep`). CPU core không cấp phát time-slice cho thread này. Không có Frame Request nào bị bỏ lỡ.
     * *Bị nghẽn (Contention/Bug):*
       * Ngay trước lát cắt `nativePollOnce`, có một lát cắt `obtainView()` hoặc `onMeasure()` chạy quá 16.6ms.
       * Thread State hiển thị màu xanh lá đậm (`Running`) nhưng CPU xung nhịp cao, hoặc màu đỏ (`Uninterruptible Sleep - D State`) do đang bị nghẽn I/O đọc ổ cứng flash eMMC/UFS.
  2. **Kiểm tra Synchronization Barrier Leak (Pitfall nguy hiểm):**
     * Nếu ai đó gọi `MessageQueue.postSyncBarrier()` (thường do `ViewRootImpl` gọi trước khi vẽ) nhưng vì một lỗi exception nào đó mà hàm `removeSyncBarrier()` tương ứng **không bao giờ được gọi**.
     * Toàn bộ các tin nhắn đồng bộ thông thường (UI events, input clicks) sẽ bị kẹt mãi mãi trong queue. `nativePollOnce` có thể ngủ vô tận hoặc chỉ xử lý các asynchronous message, dẫn đến ứng dụng bị đơ toàn diện mà không sinh ra crash log.

---

## CHỦ ĐỀ 5: Resources/AssetManager, RRO & TV Qualifiers

### Q1. Trình bày kiến trúc `AssetManager2` trong C++ (`frameworks/base/libs/androidfw/AssetManager2.cpp`). Cơ chế phân giải Resource ID từ file `resources.arsc` diễn ra như thế nào?

**Trả lời:**
* **Kiến trúc `AssetManager2`:**
  * Kể từ Android 9 và duy trì ổn định đến Android 14, toàn bộ logic tải và phân giải tài nguyên (resources) được viết lại trong thư viện C++ native `libandroidfw` dưới tên gọi `AssetManager2`.
  * `AssetManager2` quản lý một mảng các đối tượng `ApkAssets` (đại diện cho APK của framework-res, APK ứng dụng, và các APK của Runtime Resource Overlay - RRO).
  * Sử dụng cấu trúc dữ liệu tối ưu bộ nhớ không lock (lock-free / immutable cache) giúp tăng tốc độ truy vấn đa luồng từ các thread của ứng dụng.
* **Cơ chế phân giải Resource ID (ví dụ: `0x7f040012`):**
  Một mã định danh tài nguyên (ResID) là một số 32-bit gồm 3 phần: `0xPPTTEEEE`:
  * `PP` (Package ID - 8 bit): `0x01` là tài nguyên của Framework (`android.R`), `0x7f` là tài nguyên của ứng dụng chính. RRO có thể có package ID động từ `0x7f` đến `0xff`.
  * `TT` (Type ID - 8 bit): Xác định kiểu tài nguyên (`drawable`, `layout`, `string`, `id`...). Ánh xạ vào danh sách các kiểu trong `resources.arsc`.
  * `EEEE` (Entry ID - 16 bit): Index của mục tài nguyên đó trong bảng `ResTable_type`.
* **Luồng tìm kiếm (Resolution Flow):**
  1. `AssetManager2.GetResource(resId, ...)` nhận ID.
  2. Dựa vào `PP`, chọn đúng `PackageGroup`.
  3. Duyệt qua cấu trúc `ResTable_type` trong bộ nhớ đã được `mmap` trực tiếp từ file `resources.arsc` (không tốn dung lượng giải nén).
  4. Lọc cấu hình tối ưu nhất: Duyệt qua danh sách các cấu hình (`ResTable_config`) hiện tại của thiết bị (Locale, Screen Size, Density, UI Mode...).
  5. Nếu có RRO (Overlay), `AssetManager2` kiểm tra bảng `Idmap` (do `idmap2` sinh ra). Nếu ID này bị ghi đè bởi overlay, giá trị từ APK của overlay sẽ lập tức thay thế giá trị gốc.
* **Tên file/class AOSP:**
  * `frameworks/base/libs/androidfw/AssetManager2.cpp`
  * `frameworks/base/libs/androidfw/ResourceTypes.cpp`
  * `frameworks/base/core/java/android/content/res/AssetManager.java`

---

### Q2. Runtime Resource Overlay (RRO) và Fabricated Overlays hoạt động như thế nào? Cách các nhà mạng (OEM STB) sử dụng RRO để tùy biến giao diện mà không cần build lại hệ điều hành?

**Trả lời:**
* **Bản chất của RRO:**
  * RRO là cơ chế cho phép thay thế tài nguyên (strings, colors, drawables, layouts, dimensions) của ứng dụng đích (target app) hoặc của chính System UI bằng tài nguyên từ một APK overlay độc lập tại thời điểm runtime mà không cần sửa đổi hoặc decompile app gốc.
* **Cơ chế hoạt động bên trong (Idmap2):**
  1. Một APK overlay chứa một file `AndroidManifest.xml` chỉ rõ target package:
     ```xml
     <overlay android:targetPackage="com.android.tv.settings"
              android:targetName="TvSettingsCustomization"
              android:priority="10"
              android:isStatic="true"/>
     ```
  2. Service hệ thống `OverlayManagerService` (OMS) trong `system_server` nhận diện overlay.
  3. OMS gọi công cụ native `idmap2` (`frameworks/base/cmds/idmap2`) để parse cả APK gốc và APK overlay.
  4. `idmap2` tạo ra một file nhị phân header nhỏ gọi là file `idmap` lưu trong `/data/resource-cache/`. File này ánh xạ trực tiếp:
     $$\text{Target Resource ID } (0x7f010001) \longrightarrow \text{Overlay Resource ID } (0x7f020005)$$
  5. Khi app đích khởi chạy, `AssetManager2` tự động nạp các file `idmap` này. Quá trình phân giải tài nguyên diễn ra trong C++ với chi phí tiệm cận 0 (chỉ là một phép lookup bảng mảng).
* **Fabricated Overlays (Android 11+ đến 14):**
  * Không cần tạo APK! Cho phép ứng dụng hệ thống tạo ra các overlay động trực tiếp từ code bằng `FabricatedOverlay` API thông qua `OverlayManager`. Rất hữu ích cho STB đổi theme dựa trên Brand của nhà mạng (White-label STB).
* **Ứng dụng thực tế của OEM/Operator STB:**
  * Một bản ROM Android TV dùng chung cho 5 nhà mạng viễn thông khác nhau. Thay vì duy trì 5 source code tree hoặc 5 image firmware khác nhau, OEM chỉ cần tạo 5 file APK overlay chứa logo nhà mạng, màu chủ đạo, chuỗi string thông tin CSKH. Khi nạp thiết bị cho nhà mạng nào, factory script chỉ việc enable overlay tương ứng qua lệnh:
    ```bash
    cmd overlay enable com.operator.customization.overlay
    ```

---

### Q3. Trình bày các Resource Qualifiers đặc thù cho Android TV STB: `television`, `leanback`, `dpad`, density (`tvdpi`, `xhdpi`). Khi HDMI EDID thay đổi độ phân giải màn hình, cơ chế xử lý configuration ra sao?

**Trả lời:**
* **Các TV Qualifiers quan trọng:**
  * `ui_mode`: `television` (được cấu hình qua `UiModeManager.setCurrentModeType(Configuration.UI_MODE_TYPE_TELEVISION)`). Đây là qualifier cốt lõi để phân biệt layout TV với Phone/Tablet.
  * `navigation`: `dpad` (Directional Pad - điều khiển không có cảm ứng, chỉ có 4 hướng và nút Center).
  * `touchscreen`: `notouch` (thông báo rằng màn hình hoàn toàn không có cảm ứng, hệ thống sẽ tự động bật navigation focus engine).
  * `density`: `tvdpi` ($\approx 213\text{ dpi}$ - chuẩn trung gian lịch sử cho màn hình 720p/1080p ở khoảng cách phòng khách). Tuy nhiên, trên STB Android 14 hiện đại xuất ra màn hình 4K, hệ thống thường set density thực tế là `xhdpi` (320dpi) hoặc `xxhdpi` (480dpi) để kích thước text và icon vừa vặn với khoảng cách nhìn 2-3 mét (10-foot experience).
* **Cơ chế xử lý khi HDMI EDID thay đổi (Hotplug Event):**
  1. Người dùng cắm STB từ TV Full HD sang TV 4K, hoặc thay đổi Resolution trong Settings (ví dụ: từ 1080p@60Hz sang 4K@60Hz).
  2. Driver HDMI nhận diện EDID mới, gửi `UEVENT` lên Linux Kernel.
  3. `SurfaceFlinger` bắt được sự kiện hiển thị mới, cập nhật kích thước Display vật lý và thông báo cho `WindowManagerService`.
  4. WMS cập nhật lại `DisplayInfo` và gọi `ActivityTaskManagerService.updateConfigurationLocked()`.
  5. Hệ thống sinh ra một đối tượng `Configuration` mới với các trường `screenWidthDp`, `screenHeightDp`, `densityDpi` mới.
  6. Các ứng dụng nhận callback `onConfigurationChanged(Configuration newConfig)`.
* **Pitfall:**
  Nếu nhà phát triển hardcode kích thước view theo pixel (`px`) thay vì dùng `dp`, hoặc cố tình bắt layout theo aspect ratio cố định mà không lắng nghe sự kiện thay đổi độ phân giải từ HDMI, khi chuyển độ phân giải từ 1080p lên 4K, giao diện ứng dụng sẽ co cụm lại thành một ô nhỏ ở góc trên bên trái màn hình.

---

### Q4. Quá trình xử lý `onConfigurationChanged` vs Recreate Activity trên Android TV. Pitfall liên quan đến rò rỉ bộ nhớ (Memory Leak) Context/Resources khi xảy ra HDMI Hotplug là gì?

**Trả lời:**
* **Cơ chế tái tạo Activity vs Callback:**
  * Mặc định trong Android, khi có bất kỳ thay đổi nào trong `Configuration` (kể cả thay đổi ngôn ngữ, mật độ màn hình, hoặc kích thước hiển thị khi cắm HDMI), hệ thống sẽ tiêu hủy Activity hiện tại (`onDestroy()`) và tạo mới lại hoàn toàn (`onCreate()`) để nạp lại toàn bộ resources tương ứng từ qualifiers mới.
  * Nếu developer khai báo trong `AndroidManifest.xml`:
    ```xml
    android:configChanges="screenSize|smallestScreenSize|screenLayout|orientation|density"
    ```
    Hệ thống sẽ **không** recreate Activity mà gọi trực tiếp phương thức `Activity.onConfigurationChanged(newConfig)`. Developer phải tự tay cập nhật lại các View bị ảnh hưởng.
* **Pitfall rò rỉ bộ nhớ nghiêm trọng trên STB (HDMI Hotplug Memory Leak):**
  * Trên STB, việc rút/cắm dây HDMI hoặc TV tắt/bật (khiến đường truyền tín hiệu TMDS ngắt kết nối tạm thời) kích hoạt chuỗi sự kiện `Display Listener -> onDisplayChanged -> Configuration Changed`.
  * *Nguyên nhân rò rỉ:*
    1. Nhiều SDK phân tích hoặc SDK media player đăng ký lắng nghe sự kiện Display: `DisplayManager.registerDisplayListener(listener, handler)`. Nếu đăng ký bằng `Activity Context` thay vì `ApplicationContext` và quên gọi `unregister` trong `onDestroy()`.
    2. Khi người dùng tắt/bật TV, hệ thống recreate Activity liên tục. Toàn bộ `Activity` cũ cùng cây View khổng lồ (chứa nhiều Bitmap posters của app streaming) bị giữ chặt lại trong heap thông qua tham chiếu mạnh từ `DisplayManagerGlobal.mDisplayListeners`.
    3. Hậu quả: Sau vài ngày cắm nguồn liên tục và bật/tắt TV hàng ngày, STB bị tràn RAM (OutOfMemory) và crash launcher của nhà mạng.

---

## CHỦ ĐỀ 6: PowerManagerService, WakeLock & HDMI-CEC

### Q1. Trình bày State Machine trong `PowerManagerService` (PMS). Sự khác nhau giữa các trạng thái `AWAKE`, `DREAMING`, `DOZE`, và `SLEEP` trên Android TV STB so với điện thoại thông minh?

**Trả lời:**
* **Cấu trúc State Machine của PMS:**
  `PowerManagerService.java` quản lý trạng thái nguồn thông qua các biến nội tại và phương thức `updatePowerStateLocked()`.
  Các trạng thái cơ bản:
  1. `WAKEFULNESS_AWAKE`: Thiết bị hoạt động đầy đủ. Màn hình bật (HDMI out active), tương tác bình thường.
  2. `WAKEFULNESS_DREAMING`: Trạng thái hiển thị Screensaver (Ambient Mode / Daydream trên Android TV). Vẫn có ngõ ra HDMI, CPU vẫn chạy nhưng giảm tải, chuẩn bị chuyển sang trạng thái nghỉ.
  3. `WAKEFULNESS_DOZE`: Trên điện thoại, đây là màn hình Always-on Display (AOD). Trên STB, nó thường là trạng thái trung gian khi tắt tín hiệu hình ảnh HDMI nhưng hệ thống vẫn duy trì xử lý audio nền (ví dụ: nghe nhạc qua Spotify khi tắt màn hình TV).
  4. `WAKEFULNESS_ASLEEP`: Trạng thái ngủ. Màn hình tắt. Tín hiệu HDMI TMDS clock bị ngắt hoàn toàn.
* **Sự khác biệt cốt lõi giữa Android TV STB và Smartphone:**
  * *Điện thoại:* Trọng tâm là tiết kiệm từng milliampere pin. Khi không dùng, thiết bị rơi vào Deep Sleep, CPU vào trạng thái suspend (suspend-to-RAM), tắt sóng nếu cần, định kỳ thức dậy bằng `AlarmManager` trong chế độ Doze Mode.
  * *Android TV STB:*
    * Cắm nguồn trực tiếp (AC Power), không lo cạn pin.
    * Khi người dùng bấm nút Power trên remote, STB **không bao giờ shutdown thực sự**. Thay vào đó, nó chuyển sang trạng thái gọi là **Fake Standby / Low Power State**.
    * Trạng thái `ASLEEP` trên STB phải đảm bảo: Tắt ngõ ra HDMI (để TV tự động tắt theo hoặc nhận biết No Signal), tắt đèn LED mặt trước, nhưng **chipset mạng (Ethernet/Wi-Fi) và Bluetooth HID module phải luôn luôn giữ điện**. Điều này bắt buộc để người dùng có thể "Cast" video từ điện thoại (Chromecast built-in) đánh thức STB dậy, hoặc bấm phím trên Remote Bluetooth để bật máy trong tích tắc (< 1 giây).

---

### Q2. Phân loại WakeLocks (`PARTIAL_WAKE_LOCK`, `PROXIMITY_SCREEN_OFF_WAKE_LOCK`...). Cơ chế giao tiếp giữa PMS và Kernel Autosleep/Wakeup subsystem diễn ra qua file descriptor nào?

**Trả lời:**
* **Phân loại WakeLocks:**
  * `PARTIAL_WAKE_LOCK`: Đảm bảo CPU tiếp tục chạy ngay cả khi màn hình tắt hoàn toàn. Đây là loại WakeLock phổ biến nhất trên STB (dùng khi đang ghi hình chương trình TV - DVR, tải firmware OTA ngầm, hoặc stream nhạc nền).
  * `SCREEN_DIM_WAKE_LOCK` / `SCREEN_BRIGHT_WAKE_LOCK`: Giữ màn hình sáng (đã bị deprecated từ lâu, thay bằng `FLAG_KEEP_SCREEN_ON` trong WindowManager).
* **Cơ chế giao tiếp từ PMS xuống Linux Kernel:**
  * Ngày xưa (Android cũ), framework giao tiếp qua sysfs legacy: `/sys/power/wake_lock` và `/sys/power/wake_unlock`.
  * Từ Android 10 đến Android 14: Hệ thống sử dụng giao diện chuẩn hóa của System HAL: **`ISystemSuspend` AIDL/HIDL** (`hardware/interfaces/suspend/`).
  * Luồng thực thi:
    1. Khi có `PARTIAL_WAKE_LOCK` được giữ trong Java, `PowerManagerService` gọi xuống native `com_android_server_power_PowerManagerService.cpp`.
    2. Native code gọi IPC sang daemon `system_suspend` (`system/hardware/interfaces/suspend/`).
    3. Daemon `system_suspend` quản lý các WakeLock thông qua một file descriptor trỏ vào Kernel Wakeup Source: `/dev/wakeup_source` hoặc sysfs autosleep `/sys/power/autosleep`.
    4. Chừng nào biến đếm reference count của các WakeLock lớn hơn 0, `system_suspend` sẽ ngăn cản Linux kernel kích hoạt cơ chế `pm_suspend()` (ngừng ghi chuỗi `"mem"` vào `/sys/power/state`).
* **Pitfall:**
  Một Background Service của ứng dụng STB vô tình giữ một `PARTIAL_WAKE_LOCK` vô thời hạn mà không có cơ chế timeout (`wakeLock.acquire(10000)`). Mặc dù STB cắm điện không sợ hết pin, nhưng CPU luôn chạy ở xung nhịp cao, gây hiện tượng thiết bị bị quá nhiệt (Thermal Throttling). Khi người dùng bật TV lên sử dụng lại, STB bị giật lag nghiêm trọng do CPU bị ép hạ xung để bảo vệ phần cứng.

---

### Q3. Trình bày kiến trúc HDMI-CEC (`HdmiControlService`). Quá trình xử lý khi người dùng bấm nút Standby trên Remote TV truyền sang STB và ngược lại diễn ra như thế nào qua các CEC Opcode?

**Trả lời:**
* **Kiến trúc HDMI-CEC Framework:**
  * `HdmiControlService` (`frameworks/base/services/core/java/com/android/server/hdmi/HdmiControlService.java`): Service trung tâm trong `system_server`.
  * `HdmiCecController`: Tầng wrapper C++/JNI giao tiếp với `HdmiCec HAL` (AIDL `android.hardware.tv.hdmi.cec`).
  * `HdmiCecLocalDevice`: Đại diện cho vai trò của thiết bị trên đường truyền CEC.
    * Trên Android TV STB, thiết bị đóng vai trò là **Playback Device** (Logical Address = 4: `HdmiCecLocalDevicePlayback.java`).
    * Nếu là tivi tích hợp sẵn Android, nó đóng vai trò là **TV** (Logical Address = 0: `HdmiCecLocalDeviceTv.java`).
* **Kịch bản 1: Người dùng bấm phím Power trên Remote STB để BẬT máy:**
  1. STB chuyển từ `ASLEEP` sang `AWAKE`.
  2. `HdmiCecLocalDevicePlayback` kích hoạt tính năng **One Touch Play**:
  3. STB gửi broadcast opcode `<Image View On>` (`0x04`) hoặc `<Text View On>` (`0x0D`) tới Logical Address 0 (TV) để yêu cầu TV bật màn hình.
  4. Tiếp tục gửi opcode `<Active Source>` (`0x82`) kèm theo physical address của cổng HDMI (ví dụ `0x1000` - Cổng HDMI 1) để yêu cầu TV tự động chuyển ngõ vào (Input Source) sang cổng của STB.
* **Kịch bản 2: Người dùng bấm phím Tắt trên Remote của TV (TV Standby):**
  1. TV gửi broadcast opcode CEC: `<Standby>` (`0x36`).
  2. Dây cáp HDMI dẫn tín hiệu vào chip điều khiển CEC của STB -> kích hoạt ngắt phần cứng -> `HdmiCec HAL` nhận byte code.
  3. `HdmiControlService` bắt được opcode `<Standby>`.
  4. Kiểm tra cấu hình cài đặt hệ thống: `Settings.Global.HDMI_CONTROL_AUTO_DEVICE_OFF_ENABLED`.
  5. Nếu enabled, `HdmiControlService` gọi `PowerManager.goToSleep()` để đưa STB vào chế độ Low Power Standby đồng bộ với TV.
* **Tên file/class AOSP:**
  * `frameworks/base/services/core/java/com/android/server/hdmi/HdmiControlService.java`
  * `frameworks/base/services/core/java/com/android/server/hdmi/HdmiCecLocalDevicePlayback.java`
  * `frameworks/base/services/core/java/com/android/server/hdmi/HdmiCecMessageBuilder.java`

---

### Q4. Xử lý Standby trên STB: Phân tích sự đánh đổi giữa Deep Sleep (Suspend to RAM) vs Fake Standby (Tắt HDMI/Audio, giữ mạng). Giải pháp kiến trúc nào tối ưu cho Android 14 STB?

**Trả lời:**
* **So sánh hai phương thức Standby:**
| Tiêu chí | Deep Sleep (STR - Suspend to RAM) | Fake Standby / Low Power State |
| :--- | :--- | :--- |
| **Tiêu thụ điện năng** | Rất thấp ($< 0.5\text{W}$), đáp ứng chuẩn Energy Star của EU/Mỹ. | Cao hơn ($3\text{W} - 7\text{W}$) do SoC vẫn chạy mạng và RAM. |
| **Thời gian khởi động lại** | Chậm ($3 - 5\text{s}$) để khôi phục trạng thái CPU và handshake lại HDMI. | Tức thì ($< 500\text{ms}$), chỉ cần bật lại tín hiệu video output. |
| **Kết nối mạng & Casting**| Mất kết nối Wi-Fi/Ethernet. Không thể Cast video từ điện thoại. | Kết nối liên tục. Hỗ trợ đầy đủ Google Cast / Spotify Connect. |
| **Bảo trì hệ thống (Maintenance)**| Không thể cập nhật EPG, không ghi hình DVR, không cài OTA ngầm. | Chạy background tasks, sync EPG, tải bản cập nhật lúc nửa đêm. |

* **Giải pháp kiến trúc tối ưu trên Android 14 (Hybrid Architecture):**
  Các STB chuẩn Tier-1 của nhà mạng hiện nay áp dụng mô hình lai:
  1. Khi người dùng bấm nút Power trên remote: Hệ thống đi vào **Fake Standby**.
     * Tắt ngõ ra HDMI Physical layer (PHY power down).
     * Ngắt các Audio DAC/DSP output.
     * Hạ xung nhịp CPU xuống mức tối thiểu (Minimum Frequency Governor).
     * Bật các bộ lọc Wake-on-LAN (WoL) và Wake-on-WLAN (WoWLAN).
  2. Duy trì trạng thái này trong khoảng thời gian cấu hình được (ví dụ 4 tiếng sau khi tắt). Trong thời gian này, trải nghiệm người dùng là hoàn hảo (bật lên tức thì, cast được nhạc).
  3. Nếu không có tương tác nào trong 4 tiếng (hoặc vào khung giờ 2h - 5h sáng sau khi đã chạy xong các Job sync dữ liệu): Hệ thống tự động chuyển sang **Deep Sleep (STR)** để thỏa mãn các quy định bảo vệ môi trường và tiết kiệm năng lượng. Khi ở Deep Sleep, chỉ có tín hiệu ngắt từ phím cứng Power trên remote (IR/BLE) mới có thể kích hoạt PMIC bật lại máy.

---

## CHỦ ĐỀ 7: Audio/Media Framework

### Q1. Trình bày chi tiết đường đi của luồng âm thanh: `AudioTrack` (Java) -> `AudioTrack` (C++) -> `AudioFlinger` (`MixerThread` vs `DirectOutputThread`) -> `AudioPolicyService` -> Audio HAL.

**Trả lời:**
* **Chi tiết luồng Audio Data:**
  1. **Application Layer (`AudioTrack.java`):**
     * App khởi tạo `AudioTrack` với cấu hình AudioAttributes (Usage, ContentType), Sample Rate, Channel Mask, PCM format.
     * Gọi `write()` để đẩy dữ liệu byte audio vào Native layer qua JNI.
  2. **Native Client (`AudioTrack.cpp`):**
     * Giao tiếp với `AudioPolicyService` qua Binder để xin một routing hợp lệ thông qua hàm `AudioPolicyService::getOutputForAttr()`.
     * Gọi sang `AudioFlinger` để tạo một đối tượng kết nối `IAudioTrack` tương ứng với một vùng nhớ chia sẻ (`Shared Memory` - `IMemory`). Dữ liệu âm thanh từ client ghi vào buffer này qua cơ chế RingBuffer (FIFO).
  3. **Audio Server (`AudioFlinger.cpp`):**
     * `AudioFlinger` là trung tâm xử lý âm thanh trong tiến trình `audioserver`.
     * Quản lý các Playback Threads:
       * **`MixerThread`:** Dùng cho âm thanh hệ thống, âm thanh app thông thường (UI clicks, game, YouTube PCM stereo). Luồng này lấy dữ liệu từ nhiều Track khác nhau, chạy thuật toán trộn âm phần mềm (`AudioMixer`), convert sample rate (resampling) về một tần số chuẩn (thường là 48kHz) rồi đẩy xuống HAL.
       * **`DirectOutputThread` / `OffloadThread`:** Dùng khi phát các định dạng nén chất lượng cao (Dolby Digital Plus / E-AC-3, Dolby TrueHD, DTS). Âm thanh **không được qua bộ trộn** để tránh làm biến dạng gói bitstream nén, mà được gửi thẳng trực tiếp xuống Hardware.
  4. **Quyết định định tuyến (`AudioPolicyService`):**
     * Lắng nghe các sự kiện thay đổi thiết bị (cắm tai nghe 3.5mm, kết nối HDMI ARC/eARC, xuất quang SPDIF).
     * Đọc file cấu hình phần cứng `audio_policy_configuration.xml` để quyết định xem luồng âm thanh nào sẽ đi vào Output Stream nào của phần cứng.
  5. **Audio HAL (`hardware/interfaces/audio/`):**
     * Nhận PCM data hoặc Bitstream thô từ AudioFlinger.
     * Giao tiếp với driver ALSA của kernel thông qua `tinyalsa` (lệnh gọi `pcm_write` hoặc `compress_write`), xuất dữ liệu qua chip DSP âm thanh ra ngõ HDMI Tx.
* **Tên file/class AOSP:**
  * `frameworks/av/media/libaudioclient/AudioTrack.cpp`
  * `frameworks/av/services/audioflinger/AudioFlinger.cpp`
  * `frameworks/av/services/audioflinger/Threads.cpp`
  * `frameworks/av/services/audiopolicy/AudioPolicyService.cpp`

---

### Q2. Phân tích kiến trúc Codec2 (`C2ComponentStore`, `C2Buffer`, `Codec2Client`) thay thế cho OpenMAX (OMX) trong MediaCodec. Quá trình giải mã phần cứng (Hardware Decoding) đẩy khung hình ra Surface diễn ra như thế nào?

**Trả lời:**
* **Codec2 Architecture (Android 10+ và chuẩn hóa Android 14):**
  * Thay thế hoàn toàn kiến trúc OpenMAX (OMX) cũ kỹ vốn phụ thuộc vào các thư viện C cứng nhắc và không an toàn về bộ nhớ.
  * Codec2 sử dụng giao tiếp IPC IPC-safe (Treble compliant) dựa trên HIDL/AIDL.
  * Các thành phần chính:
    * `C2ComponentStore`: Quản lý danh sách các codec hỗ trợ trên thiết bị (ví dụ `c2.android.avc.decoder` là phần mềm, `c2.amlogic.hevc.decoder` là phần cứng).
    * `C2Component`: Đại diện cho một phiên làm việc của decoder/encoder cụ thể.
    * `C2Buffer`: Cấu trúc dữ liệu chứa buffer hình ảnh, hỗ trợ zero-copy bằng cách bọc quanh `GraphicBuffer` / `DMA-BUF`.
    * `Codec2Client`: Wrapper phía C++ của `MediaCodec` để giao tiếp với tiến trình `media.codec` độc lập.
* **Quá trình giải mã phần cứng ra Surface:**
  1. `MediaCodec.configure()` nhận một đối tượng `Surface` (tạo từ `SurfaceView` của app).
  2. `MediaCodec` trích xuất `SurfaceControl` và `IGraphicBufferProducer` từ Surface này gửi xuống Native Codec2.
  3. App đọc file media (qua `MediaExtractor`), đưa các gói nén (H.264/H.265 NAL units) vào Input Buffer thông qua `queueInputBuffer()`.
  4. Codec2 gửi buffer này sang tiến trình phần cứng của chip vendor (ví dụ SoC Video Processing Unit - VPU).
  5. VPU giải mã bitstream thành các khung hình raw (YUV420 Planar / Semi-planar).
  6. **Zero-Copy Rendering:** VPU ghi trực tiếp dữ liệu thô vào các `GraphicBuffer` do `SurfaceView` cấp phát sẵn trong bộ nhớ DMA-BUF.
  7. Driver của VPU gắn kèm một `Sync Fence` rồi gọi `queueBuffer()` đẩy ngược lại SurfaceFlinger mà không cần copy dữ liệu qua lại giữa CPU và GPU.
* **Tên file/class AOSP:**
  * `frameworks/av/media/codec2/sfplugin/CCodec.cpp`
  * `frameworks/av/media/codec2/sfplugin/Codec2Client.cpp`
  * `frameworks/av/media/libstagefright/MediaCodec.cpp`

---

### Q3. Tunneled Video & Audio Playback trên Android TV là gì? Tại sao các dòng STB 4K HDR 60fps bắt buộc phải sử dụng Tunnel Mode?

**Trả lời:**
* **Bản chất của Tunneled Playback (Tunnel Mode):**
  * Trong chế độ phát thông thường (Non-tunneled):
    * Video: Decoder -> Giải mã ra Buffer -> Gửi sang SurfaceFlinger -> Chờ VSYNC -> SurfaceFlinger gửi HWC -> Xuất ra màn hình.
    * Audio: AudioTrack -> AudioFlinger -> Audio Mixer -> Audio HAL -> Xuất ra loa.
    * Sự đồng bộ âm thanh/hình ảnh (A/V Sync) hoàn toàn phụ thuộc vào phần mềm (Application hoặc MediaPlayer tính toán clock dựa trên System Time).
  * Trong chế độ **Tunneled Playback** (`FEATURE_TunneledPlayback`):
    * Cả Video Decoder và Audio HAL được kết nối trực tiếp với nhau ở tầng phần cứng thấp nhất (thường thông qua một Hardware Clock chung của SoC gọi là **STC - System Time Clock** hoặc **PCR - Program Clock Reference**).
    * Video frames sau khi giải mã xong được VPU **đẩy thẳng ra Display Controller/HDMI pipeline** mà hoàn toàn bỏ qua (bypass) vòng lặp compositing của SurfaceFlinger.
    * Audio data được gửi thẳng xuống Audio DSP phần cứng kèm theo timestamp phần cứng.
* **Tại sao bắt buộc cho 4K HDR 60fps trên STB:**
  1. *Triệt tiêu áp lực Băng thông RAM:* Giải mã 4K 60fps 10-bit HDR tạo ra lưu lượng dữ liệu khổng lồ:
     $$3840 \times 2160 \times 1.25 \text{ bytes (YUV420 10-bit)} \times 60 \text{ frames/s} \approx 622 \text{ MB/s}$$
     Nếu đi qua SurfaceFlinger, dữ liệu này bị đọc/ghi nhiều lần qua lại bus RAM. Chip SoC của STB không đủ băng thông, dẫn đến nghẽn bus bộ nhớ làm UI bị đơ và rớt khung hình video.
  2. *A/V Sync hoàn hảo:* Loại bỏ hoàn toàn hiện tượng lệch tiếng (Lip-sync error) do độ trễ xử lý phần mềm của OS gây ra.
  3. *Hỗ trợ HDR Metadata động:* Cho phép truyền siêu dữ liệu HDR10+ và Dolby Vision trực tiếp tới ngõ ra HDMI mà không bị biến đổi màu sắc bởi RenderEngine của Android.
* **Cách kích hoạt trong Code:**
```kotlin
val format = MediaFormat.createVideoFormat(MediaFormat.MIMETYPE_VIDEO_HEVC, 3840, 2160)
// Kích hoạt tính năng Tunneling
format.setFeatureEnabled(CodecCapabilities.FEATURE_TunneledPlayback, true)
// Gắn Audio Session ID để Hardware đồng bộ Clock âm thanh và hình ảnh
format.setInteger(MediaFormat.KEY_AUDIO_HW_SYNC, audioSessionId)
mediaCodec.configure(format, surfaceView.holder.surface, null, 0)
```

---

### Q4. Kiến trúc MediaSession và Dispatching Media Keys trên Android TV. Cơ chế Passthrough Audio (Dolby Atmos, DTS:X) qua cổng HDMI ARC/eARC hoạt động như thế nào?

**Trả lời:**
* **Kiến trúc `MediaSession` & Phân phối phím Media:**
  * `MediaSessionRecord`: Đại diện cho một phiên phát phương tiện trong `MediaSessionService` (`system_server`).
  * Khi người dùng bấm các phím đa phương tiện trên remote (Play, Pause, FastForward):
    1. Phím đi vào `PhoneWindowManager.interceptKeyBeforeQueueing()`.
    2. Nếu không có xử lý đặc biệt, nó được chuyển đến `MediaSessionService.dispatchMediaKeyEvent()`.
    3. Hệ thống tìm kiếm "Active MediaSession" ưu tiên cao nhất (phiên đang phát âm thanh, hoặc phiên gần nhất được kích hoạt).
    4. Sự kiện được chuyển thẳng vào callback `MediaSession.Callback.onPlay()`, `onPause()` của ứng dụng đó mà ứng dụng không cần phải đang ở màn hình Foreground.
* **Cơ chế Audio Passthrough (Bitstream) qua HDMI ARC/eARC:**
  * **ARC (Audio Return Channel) / eARC (Enhanced Audio Return Channel):** Cho phép STB truyền tín hiệu âm thanh đa kênh chất lượng cao lên TV, hoặc TV truyền ngược xuống AV Receiver/Soundbar.
  * **Cơ chế hoạt động:**
    1. Khi khởi tạo `AudioTrack`, app chỉ định định dạng nén thay vì PCM:
       ```kotlin
       val audioFormat = AudioFormat.Builder()
           .setEncoding(AudioFormat.ENCODING_E_AC3_JOC) // Dolby Atmos over DD+
           .setSampleRate(48000)
           .setChannelMask(AudioFormat.CHANNEL_OUT_5POINT1)
           .build()
       ```
    2. `AudioPolicyManager` truy vấn cấu hình thiết bị ngõ ra HDMI thông qua EDID handshake.
    3. Nếu TV hỗ trợ codec này: AudioFlinger cấp phát một `DirectOutputThread`.
    4. Không có bất kỳ thuật toán giải mã hay can thiệp âm lượng (Volume attenuation) nào được áp dụng trên STB. Dữ liệu nén (Bitstream thô) được đóng gói vào các khung IEC 61937 và truyền trực tiếp qua giao diện vật lý HDMI.
    5. Thiết bị cuối (Soundbar hoặc dàn loa Hi-Fi rời) tự chịu trách nhiệm giải mã bitstream ra các kênh âm thanh vật lý.

---

## CHỦ ĐỀ 8: TV Input Framework (TIF)

### Q1. Trình bày kiến trúc tổng thể của TV Input Framework (TIF): Phân tích vai trò và mối quan hệ giữa `TvInputManagerService`, `TvInputService`, `TvView`, và `TvInputService.Session`.

**Trả lời:**
* **Kiến trúc các thành phần:**
  TIF là framework chuẩn hóa của Google nhằm biến Android TV thành một chiếc tivi hoàn chỉnh, quản lý các nguồn phát truyền hình (DVB-T2/C/S2 Tuner, Cổng HDMI In, IPTV/OTT).
  1. **`TvInputManagerService` (TIMS):**
     * System service trung tâm (`frameworks/base/services/core/java/com/android/server/tv/TvInputManagerService.java`).
     * Quản lý trạng thái của tất cả các phần cứng TV Input, các luồng Session, bảo mật, và phân quyền truy cập giữa các app xem TV (Client) và các app cung cấp nguồn kênh (Service Provider).
  2. **`TvView` (`android.media.tv.TvView`):**
     * Một UI Component đặc biệt phía client (thường nằm trong ứng dụng Live TV hoặc Launcher).
     * Bản chất bên trong `TvView` chứa một `SurfaceView` được tối ưu hóa để hiển thị video từ TV Input.
  3. **`TvInputService` (`android.media.tv.TvInputService`):**
     * Một Service chạy ngầm do nhà phát triển nguồn kênh (hoặc OEM tích hợp tuner phần cứng) hiện thực.
     * Đóng vai trò là Producer: kết nối với nguồn phát (demux luồng TS - Transport Stream từ bộ dò sóng tuner hoặc luồng IPTV multicast).
  4. **`TvInputService.Session`:**
     * Đại diện cho một phiên làm việc cụ thể giữa một `TvView` và một `TvInputService`.
     * Quản lý vòng đời phát của một kênh: `onTune(Uri channelUri)`, `onSetSurface(Surface surface)`, `onSetStreamVolume(float volume)`.
* **Mối quan hệ hoạt động:**
  * Client nhúng `TvView` -> gọi `tvView.tune(inputId, channelUri)`.
  * `TvView` gọi IPC sang `TvInputManagerService.createSession()`.
  * TIMS bind tới `TvInputService` tương ứng và yêu cầu tạo một `Session`.
  * `TvInputManagerService` đóng vai trò là một cầu nối Binder trung gian (Proxy) điều phối toàn bộ luồng lệnh và sự kiện giữa Client và Session.

---

### Q2. Cơ chế luân chuyển Surface và Video Rendering trong TIF diễn ra như thế nào? Làm thế nào `TvView` có thể nhúng một Surface từ process này sang process của `TvInputService`?

**Trả lời:**
* **Cơ chế nhúng Surface xuyên Process (Cross-Process Surface Sharing):**
  1. Phía Client (Ứng dụng Live TV):
     * `TvView` tạo ra một `SurfaceView` con.
     * Khi `SurfaceView` sẵn sàng, nó nhận được callback `SurfaceHolder.Callback.surfaceCreated(SurfaceHolder holder)`.
     * `TvView` lấy đối tượng `Surface` (thực chất là một Parcelable wrapper bao quanh native C++ `IGraphicBufferProducer`).
  2. Chuyển giao qua Binder IPC:
     * `TvView` gọi `mSession.setSurface(surface)`.
     * `ITvSession.aidl` ghi `Surface` vào Parcel thông qua phương thức `Surface.writeToParcel()`.
     * Dưới native, Binder serialize file descriptor và token kết nối của SurfaceControl xuống driver `/dev/binder`.
  3. Phía Service Provider (`TvInputService.Session`):
     * Tiến trình service unmarshal parcel và tái tạo đối tượng `Surface` hợp lệ trỏ trực tiếp vào cùng một BufferQueue của `SurfaceView` phía Client.
     * `TvInputService.Session.onSetSurface(surface)` được kích hoạt.
  4. Tiến hành Render:
     * `TvInputService` chuyển `Surface` này trực tiếp vào `MediaPlayer`, `ExoPlayer`, hoặc `MediaCodec` phần cứng.
     * Khi player giải mã video, các khung hình được vẽ thẳng vào Surface đó.
     * SurfaceFlinger composite khung hình của `TvInputService` lên chính tọa độ mà `TvView` đang chiếm giữ trên màn hình của Client mà không cần tốn chi phí copy bộ nhớ (Zero IPC Frame Copy).

---

### Q3. Trình bày kiến trúc `TvProvider` và `TvContract`. Cơ chế đồng bộ dữ liệu lịch phát sóng (EPG - Electronic Program Guide) hoạt động ra sao?

**Trả lời:**
* **`TvContract` & `TvProvider` Architecture:**
  * `TvProvider` là một ContentProvider cấp hệ thống (`com.android.providers.tv`), bảo vệ bởi cơ sở dữ liệu SQLite chuyên dụng (`tv.db`).
  * `TvContract` là contract class cung cấp URI và các định nghĩa bảng chuẩn hóa:
    * `TvContract.Channels`: Chứa danh sách các kênh truyền hình (Channel Number, Call Sign, Tuning URI, Logo, Service Type).
    * `TvContract.Programs`: Chứa lịch phát sóng EPG của từng kênh (Title, Description, Start Time, End Time, Content Rating, Poster Art).
    * `TvContract.WatchNextPrograms`: Hàng hiển thị nội dung tiếp tục xem trên màn hình chính của Android TV.
* **Cơ chế đồng bộ dữ liệu EPG:**
  1. `TvInputService` sử dụng một `JobScheduler` định kỳ thức dậy vào ban đêm.
  2. Service tải dữ liệu XMLTV hoặc đọc bảng DVB-SI (EIT - Event Information Table) từ luồng sóng phần cứng.
  3. Sử dụng `ContentResolver.applyBatch()` với danh sách các thao tác `ContentProviderOperation` (Insert/Update/Delete) để ghi dữ liệu lớn vào `TvContract.Programs.CONTENT_URI` trong một transaction SQLite duy nhất:
```kotlin
val operations = ArrayList<ContentProviderOperation>()
programs.forEach { program ->
    operations.add(
        ContentProviderOperation.newInsert(TvContract.Programs.CONTENT_URI)
            .withValue(TvContract.Programs.COLUMN_CHANNEL_ID, channelId)
            .withValue(TvContract.Programs.COLUMN_TITLE, program.title)
            .withValue(TvContract.Programs.COLUMN_START_TIME_UTC_MILLIS, program.startTime)
            .withValue(TvContract.Programs.COLUMN_END_TIME_UTC_MILLIS, program.endTime)
            .build()
    )
}
context.contentResolver.applyBatch(TvContract.AUTHORITY, operations)
```
* **Pitfall:**
  Dữ liệu EPG có thể lên tới hàng chục nghìn records cho hàng trăm kênh trong 7 ngày. Nếu query hoặc insert từng dòng đơn lẻ mà không dùng `applyBatch` và không xóa các chương trình đã phát trong quá khứ (`COLUMN_END_TIME_UTC_MILLIS < System.currentTimeMillis()`), dung lượng database `tv.db` sẽ phình to hàng trăm megabytes, làm chậm toàn bộ các truy vấn của launcher TV và gây giật khung hình khi người dùng mở thanh EPG.

---

### Q4. Quản lý Hardware TV Inputs (HDMI In, DVB Tuner phần cứng). Vai trò của `TvInputHardwareManager` và TV HAL là gì?

**Trả lời:**
* **Kiến trúc phần cứng TIF:**
  * Các cổng kết nối vật lý trên TV/STB (HDMI Input, Cổng cáp RF DVB-T2) không thể khởi chạy như một phần mềm media thông thường. Chúng được quản lý riêng biệt bởi `TvInputHardwareManager.java` trong `system_server`.
* **Vai trò của TV HAL (`hardware/interfaces/tv/input/`):**
  * Giao tiếp trực tiếp với kernel driver của chip xử lý video chuyên dụng (HDMI Rx Receiver, DVB Demux chip).
  * Khai báo các thiết bị phần cứng có sẵn (`tv_input_device_info_t`) và trạng thái kết nối cáp (Cable Connected / Disconnected).
* **Luồng xử lý khi mở kênh từ Hardware Input:**
  1. Khi một `TvInputService` phần cứng được khởi động, nó yêu cầu quyền truy cập phần cứng thông qua `TvInputManager.acquireTvInputHardware()`.
  2. `TvInputHardwareManager` kiểm tra quyền `android.permission.TV_INPUT_HARDWARE` (chỉ cấp cho System signature).
  3. Kết nối luồng dữ liệu phần cứng:
     * Tùy thuộc vào khả năng của SoC, TV HAL có thể định tuyến video thô trực tiếp từ cổng HDMI In vào một Audio/Video processor riêng biệt, rồi đưa trực tiếp ra Display Plane mà không qua Android Framework.
     * `TvInputHardwareManager` cấu hình luồng âm thanh thông qua `AudioSystem.setDeviceConnectionState()` để dẫn trực tiếp âm thanh từ HDMI In sang Audio HAL, đảm bảo không có độ trễ (Zero-latency monitoring).

---

## CHỦ ĐỀ 9: Notification, JobScheduler, AlarmManager

### Q1. Đi sâu vào kiến trúc của `AlarmManagerService`. Sự khác nhau giữa RTC và ELAPSED_REALTIME dưới tầng Linux Kernel (`timerfd` / RTC interrupt) là gì?

**Trả lời:**
* **Kiến trúc `AlarmManagerService`:**
  * Chạy trong `system_server`, duy trì các danh sách ưu tiên (`mAlarmBatches`) được sắp xếp theo thời gian kích hoạt.
  * Gom cụm (Batching): Hệ thống tự động gộp các alarm có thời gian gần nhau để đánh thức CPU một lần duy nhất, giảm thiểu số lần CPU phải thoát khỏi trạng thái tiết kiệm điện.
* **Bản chất dưới tầng Linux Kernel:**
  * C++ native backend của AlarmManager (`AlarmManagerService.cpp` hoặc `com_android_server_alarm_AlarmManagerService.cpp`) sử dụng Linux **`timerfd` API** (`timerfd_create`, `timerfd_settime`).
  * Trước Linux 3.x, Android dùng driver character device `/dev/alarm`. Từ Android 5+ đến 14, toàn bộ chuyển sang các file descriptor `timerfd` chuẩn POSIX.
* **Sự khác biệt giữa RTC và ELAPSED_REALTIME:**
  1. **`RTC` / `RTC_WAKEUP` (Real Time Clock):**
     * Dựa trên giờ thực tế của thế giới (Wall-clock time: UTC epoch).
     * Dưới kernel: Sử dụng clock `CLOCK_REALTIME`.
     * Khi hệ thống sleep, kernel lập lịch cho chip RTC phần cứng (Real-Time Clock chip có pin CMOS/thạch anh riêng) để bắn ngắt phần cứng (Hardware Interrupt) đánh thức CPU dậy khi đến giờ.
     * *Nhược điểm:* Bị ảnh hưởng khi người dùng đổi giờ hệ thống, đổi múi giờ, hoặc khi mạng NTP cập nhật lại giờ.
  2. **`ELAPSED_REALTIME` / `ELAPSED_REALTIME_WAKEUP`:**
     * Dựa trên khoảng thời gian trôi qua kể từ khi thiết bị khởi động (Monotonic clock bao gồm cả thời gian thiết bị ngủ).
     * Dưới kernel: Sử dụng clock `CLOCK_BOOTTIME`.
     * Hoàn toàn miễn nhiễm với việc thay đổi múi giờ hoặc sai lệch đồng hồ NTP. Đây là lựa chọn bắt buộc cho các tác vụ định kỳ nội bộ của hệ thống.

---

### Q2. Cơ chế hoạt động của `JobSchedulerService`: Đánh giá các Constraints (Network, Charging, Idle) và Standby Buckets ảnh hưởng như thế nào đến Background Sync trên STB?

**Trả lời:**
* **Cơ chế đánh giá ràng buộc (Constraint Evaluation):**
  * `JobSchedulerService` (JSS) quản lý một tập hợp các bộ theo dõi gọi là `StateController`:
    * `BatteryController`: Theo dõi trạng thái sạc (trên STB trường này luôn báo là `CHARGING = true` và `BATTERY_NOT_LOW = true`).
    * `ConnectivityController`: Lắng nghe qua `ConnectivityManager` để đảm bảo mạng có kết nối, không bị metered hoặc unmetered theo yêu cầu của Job.
    * `IdleController`: Xác định thiết bị có đang rảnh rỗi không (không có tương tác remote, màn hình đang tắt hoặc screensaver).
  * Khi tất cả các ràng buộc của một `JobInfo` thỏa mãn, JSS đẩy Job vào hàng đợi thực thi `JobPackageTracker` và bind tới `JobService` của ứng dụng qua `IJobService.aidl`.
* **App Standby Buckets trên Android TV STB:**
  Ngay cả khi cắm nguồn liên tục, Android 14 vẫn áp dụng cơ chế phân nhóm ứng dụng để hạn chế lạm dụng CPU:
  * `ACTIVE`: Ứng dụng đang hiển thị hoặc đang phát video. Không bị giới hạn.
  * `WORKING_SET`: Ứng dụng thường xuyên được dùng. Bị hạn chế nhẹ về tần suất kích hoạt Job.
  * `FREQUENT` / `RARE`: Ứng dụng hiếm khi mở. Job có thể bị hoãn lại nhiều giờ hoặc cả ngày.
  * `RESTRICTED`: Bị cấm hoàn toàn chạy background jobs trừ khi người dùng chủ động mở app.
* **Ảnh hưởng đến STB Background Sync:**
  * Nếu ứng dụng Live TV hoặc OTT của nhà mạng bị rơi vào nhóm `RARE` (do người dùng chỉ dùng ứng dụng khác như Netflix), các Job tải trước EPG hoặc kiểm tra bản quyền thuê bao ngầm sẽ bị hệ thống trì hoãn vô thời hạn cho tới khi thiết bị được cắm sạc và ở trạng thái Idle sâu.

---

### Q3. Notification Framework trên Android TV khác biệt thế nào so với Mobile? Trình bày cơ chế hiển thị Now Playing bar và Recommendations/Channel Programs.

**Trả lời:**
* **Sự khác biệt cốt lõi:**
  * *Mobile:* Có thanh trạng thái (Status Bar) và ngăn kéo thông báo (Notification Shade) để người dùng vuốt xuống xem bất cứ lúc nào.
  * *Android TV:* **Không có Notification Shade!** Người dùng điều khiển bằng Remote ở khoảng cách xa nên việc hiển thị thông báo liên tục gây gián đoạn nghiêm trọng trải nghiệm xem phim (Leanback experience).
* **Các hình thức hiển thị thay thế trên TV:**
  1. *Heads-up Notification (Toasts cấp cao):* Chỉ những thông báo có mức độ ưu tiên tối cao (`IMPORTANCE_HIGH` trở lên) mới được phép trượt ra từ góc trên bên phải màn hình trong vài giây rồi biến mất (ví dụ: Thông báo pin remote yếu, có cuộc gọi video đến).
  2. *Now Playing Card / Media Notification:*
     * Khi một app phát nhạc/video nền, nó kích hoạt một `MediaStyle` notification kèm theo `MediaSession`.
     * Thay vì hiện trong notification shade, System UI của Android TV hiển thị một thẻ "Now Playing" chuyên dụng ngay trên hàng đầu tiên của màn hình Home Launcher, cho phép người dùng bấm vào để quay lại màn hình phát hoặc pause/resume nhanh chóng.
  3. *HomeScreen Channels & Recommendations (TIF & TvContract):*
     * Thay vì gửi notification khuyến mãi, ứng dụng TV tương tác với Android TV Home qua API `TvContractCompat.PreviewPrograms`.
     * Các gợi ý phim mới được hiển thị trực quan dưới dạng các hàng kênh (Channels) trên màn hình chính, người dùng có thể duyệt qua lại bằng DPAD.

---

### Q4. Quyền `SCHEDULE_EXACT_ALARM` trên Android 14 và các hạn chế thực thi nền (Background Execution Limits). Cách thiết kế tác vụ Background Sync ổn định cho một Operator App trên STB là gì?

**Trả lời:**
* **Hạn chế trên Android 14 đối với `SCHEDULE_EXACT_ALARM`:**
  * Bắt đầu từ Android 14, quyền `SCHEDULE_EXACT_ALARM` **mặc định bị tắt** đối với hầu hết các ứng dụng được cài đặt mới nhắm tới Android 14 (targetSDK 34), trừ các ứng dụng lịch hoặc báo thức chuyên biệt.
  * Nếu app cố gọi `setExactAndAllowWhileIdle()` mà không có quyền này, hệ thống sẽ ném ra ngoại lệ `SecurityException`.
* **Giải pháp thiết kế Background Sync cho Operator App:**
  Đối với ứng dụng nhà mạng vận hành hệ thống truyền hình trả tiền (Operator Tier STB), cần một cơ chế đồng bộ ngầm ổn định tuyệt đối để nạp EPG, refresh DRM license, và cập nhật firmware:
  1. **Tận dụng đặc quyền System/Privileged:**
     * Đặt app trong phân vùng `/system/priv-app/` hoặc `/vendor/priv-app/`.
     * Xin quyền `USE_EXACT_ALARM` (quyển này tự động được cấp mà không cần người dùng duyệt thủ công trong Settings).
  2. **Kiến trúc đồng bộ chuẩn (Best Practice):**
     * Kết hợp linh hoạt giữa **Push Notification (FCM / WebSocket)** và **JobScheduler**:
       * *Không dùng polling:* Tuyệt đối không dùng AlarmManager để đánh thức app mỗi 5 phút một lần để check data.
       * Duy trì một kết nối WebSocket nhẹ thông qua một Foreground Service đặc biệt của hệ thống, hoặc dùng Firebase Cloud Messaging (FCM) với data message ưu tiên cao.
       * Khi server có thay đổi EPG, gửi một push silent ping tới STB.
       * App nhận ping và lập tức lên lịch một `JobScheduler` với điều kiện `NetworkType.CONNECTED` để tiến hành tải dữ liệu lớn trong background mà không bị OS kill giữa chừng.

---

## CHỦ ĐỀ 10: Multi-User, Storage Encryption & Direct Boot

### Q1. Trình bày kiến trúc Multi-User trong `UserManagerService` (UMS). Phân biệt `UserHandle`, `UserInfo`, các cờ `FLAG_SYSTEM`, `FLAG_FULL`, Profile vs Secondary User.

**Trả lời:**
* **Kiến trúc Multi-User:**
  * Quản lý bởi `UserManagerService` (`frameworks/base/services/core/java/com/android/server/pm/UserManagerService.java`).
  * Mỗi người dùng trong hệ thống được gán một số nguyên duy nhất gọi là `userId`:
    * `User 0`: System User (luôn tồn tại, khởi chạy đầu tiên khi boot).
    * `User 10, 11...`: Các Secondary User hoặc Work Profile được tạo thêm.
  * Trong Linux Kernel, Android biến đổi UID của ứng dụng dựa trên `userId`:
    $$\text{Kernel UID} = \text{userId} \times 100000 + (\text{appId} \% 100000)$$
    Nhờ cơ chế này, cùng một ứng dụng (cùng `appId`) khi chạy dưới hai user khác nhau sẽ có hai Linux UID hoàn toàn độc lập, đảm bảo cô lập sandbox ở mức kernel.
* **Phân biệt các khái niệm:**
  * **`UserInfo`:** Chứa metadata của user (tên, icon, flags, thời gian tạo).
  * **`UserHandle`:** Đối tượng đại diện gọn nhẹ (chứa `mHandle` là số int `userId`) dùng trong các API giao tiếp IPC.
  * **`FLAG_SYSTEM` (User 0):** User nắm quyền điều khiển phần cứng, chạy toàn bộ các system service cốt lõi.
  * **`FLAG_FULL`:** Một người dùng độc lập hoàn chỉnh, có không gian lưu trữ riêng, giao diện riêng, danh sách app và cài đặt độc lập.
  * **Profile (Managed Profile / Clone):** Một user phụ thuộc gắn chặt vào tài khoản của một Full User cha. Profile này dùng chung một số thiết lập và có thể chia sẻ một phần dữ liệu với cha dưới sự kiểm soát chặt chẽ.
* **Đặc thù trên Android TV STB:**
  * Kể từ Android 11/12+, Android TV hỗ trợ tính năng **Google TV Profiles** (cho phép nhiều thành viên trong gia đình chuyển đổi qua lại để nhận đề xuất phim cá nhân hóa).
  * Khác với Phone, trên TV quá trình switch user không được làm gián đoạn luồng phát video toàn màn hình (Background playback) và không được ngắt kết nối các thiết bị ngoại vi Bluetooth Remote.

---

### Q2. Phân tích cơ chế Direct Boot: Sự khác biệt bản chất giữa CE (Credential Encrypted) Storage và DE (Device Encrypted) Storage. Khóa mã hóa được dẫn xuất và giải mã ở giai đoạn nào?

**Trả lời:**
* **Bản chất của Direct Boot:**
  * Cho phép thiết bị sau khi khởi động lại (reboot) có thể chạy một số ứng dụng và dịch vụ ngầm cần thiết **trước khi người dùng mở khóa màn hình lần đầu tiên** (nhập PIN/Password).
* **Phân biệt CE và DE Storage:**
  1. **Device Encrypted (DE) Storage:**
     * Vùng lưu trữ mã hóa gắn liền với phần cứng thiết bị.
     * Đường dẫn: `/data/user_de/<userId>/<packageName>/`.
     * *Khóa giải mã (DE Key):* Được dẫn xuất trực tiếp từ một khóa bí mật phần cứng trong chip bảo mật (KeyMint / TEE - Trusted Execution Environment). Khóa này sẵn sàng ngay khi Linux Kernel và daemon `vold` (Volume Daemon) khởi động xong.
     * Do đó, dữ liệu trong DE Storage có thể đọc/ghi ngay lập tức ở chế độ Direct Boot mà không cần sự tương tác nào từ con người.
  2. **Credential Encrypted (CE) Storage:**
     * Vùng lưu trữ mã hóa gắn liền với thông tin xác thực của người dùng.
     * Đường dẫn mặc định: `/data/user/<userId>/<packageName>/`.
     * *Khóa giải mã (CE Key):* Là sự kết hợp mã hóa giữa Hardware Key (từ TEE) và **Mật khẩu/Mã PIN/Pattern** do người dùng nhập vào.
     * Khi chưa mở khóa, CE Key hoàn toàn không tồn tại trong RAM. Dữ liệu trên flash eMMC/UFS tại phân vùng này là các khối byte mã hóa vô nghĩa. Bất kỳ nỗ lực truy cập nào vào CE Storage trước khi unlock đều dẫn tới lỗi I/O Crash.
* **Tên file/class AOSP:**
  * `frameworks/base/services/core/java/com/android/server/StorageManagerService.java`
  * `system/vold/FsCrypt.cpp` (Xử lý mã hóa ext4/f2fs encryption trong C++).

---

### Q3. Thiết kế ứng dụng STB hỗ trợ Direct Boot: Xử lý bài toán tự động phát luồng Live TV hoặc chạy Service nền ngay sau khi có điện trở lại (Power Outage Recovery) mà không cần người dùng thao tác Remote.

**Trả lời:**
* **Bối cảnh thực tế trên STB:**
  Ở nhiều khu vực, tình trạng cúp điện đột ngột diễn ra thường xuyên. Khi có điện trở lại, STB tự khởi động lại. Người dùng kỳ vọng tivi bật lên là phải có ngay hình ảnh kênh truyền hình cáp/vệ tinh (Live TV) mà không bắt họ phải cầm remote nhập mật khẩu hay chờ đợi unlock.
* **Giải pháp thiết kế kiến trúc Direct Boot:**
  1. **Khai báo trong `AndroidManifest.xml`:**
     Gắn cờ `android:directBootAware="true"` cho Application hoặc các thành phần cốt lõi:
     ```xml
     <service
         android:name=".service.LiveTvPlaybackService"
         android:directBootAware="true"
         android:exported="false">
     </service>
     <receiver
         android:name=".receiver.BootCompletedReceiver"
         android:directBootAware="true"
         android:exported="true">
         <intent-filter>
             <action android:name="android.intent.action.LOCKED_BOOT_COMPLETED" />
         </intent-filter>
     </receiver>
     ```
  2. **Xử lý Context và Storage:**
     Trong code, các thông tin cấu hình cần thiết để phát kênh (URL streaming, thông số tần số Tuner, token xác thực đã lưu) bắt buộc phải lưu trong **Device Protected Storage Context**:
     ```kotlin
     class BootCompletedReceiver : BroadcastReceiver() {
         override fun onReceive(context: Context, intent: Intent) {
             if (intent.action == Intent.ACTION_LOCKED_BOOT_COMPLETED) {
                 // Tạo context trỏ tới vùng nhớ DE
                 val deContext = context.createDeviceProtectedStorageContext()
                 val prefs = deContext.getSharedPreferences("tv_boot_prefs", Context.MODE_PRIVATE)
                 val lastChannelUri = prefs.getString("last_played_channel", DEFAULT_CHANNEL_URI)
                 
                 // Kích hoạt Service phát video ngầm ngay trước khi Unlock
                 val serviceIntent = Intent(deContext, LiveTvPlaybackService::class.java).apply {
                     data = Uri.parse(lastChannelUri)
                 }
                 deContext.startForegroundService(serviceIntent)
             }
         }
     }
     ```
  3. **Lắng nghe sự kiện Unlock sau đó:**
     Đăng ký thêm receiver lắng nghe `Intent.ACTION_BOOT_COMPLETED` để khi người dùng tương tác mở khóa, ứng dụng mới chuyển sang truy cập database trong CE storage (như thông tin tài khoản ngân hàng, lịch sử thanh toán gói cước).

---

### Q4. Phân tích an toàn bảo mật khi thực hiện Cross-User IPC trong Android Framework: Cách kiểm tra quyền bằng `Binder.getCallingUserId()`, `ActivityManager.handleIncomingUser()`, và rủi ro rò rỉ dữ liệu `ContentProvider`.

**Trả lời:**
* **Cơ chế kiểm soát Cross-User trong Framework:**
  Khi một Client từ User 10 gọi một phương thức IPC sang một Service hệ thống chạy ở User 0 (System User):
  1. **Nhận diện Caller:**
     Service kiểm tra UID của bên gọi thông qua `Binder.getCallingUid()`. Từ UID suy ra userId bằng công thức:
     ```java
     int callingUserId = UserHandle.getUserId(Binder.getCallingUid());
     ```
  2. **Xác thực quyền hạn xuyên User:**
     Service không được tin tưởng tham số do client gửi lên mà phải gọi hàm thẩm tra của hệ thống:
     ```java
     int targetUserId = ActivityManagerService.handleIncomingUser(
         Binder.getCallingPid(),
         Binder.getCallingUid(),
         requestedUserId,
         false, // allowAll: cho phép truy cập USER_ALL không
         true,  // requireFull: yêu cầu quyền đầy đủ
         "callCustomApi",
         callingPackage
     );
     ```
  3. Hệ thống sẽ kiểm tra xem client có quyền `android.permission.INTERACT_ACROSS_USERS` hoặc `android.permission.INTERACT_ACROSS_USERS_FULL` hay không. Nếu không có, lệnh gọi bị ném lỗi `SecurityException`.
* **Rủi ro rò rỉ dữ liệu qua `ContentProvider` (Vulnerabilities):**
  * `ContentProvider` mặc định là **User-aware**. Mỗi user có một instance database riêng biệt trong phân vùng dữ liệu của mình.
  * *Lỗ hổng (Pitfall):* Một lập trình viên tạo ra một `ContentProvider` lưu cache danh sách kênh truyền hình hoặc thông tin thanh toán, và đặt thuộc tính:
    ```xml
    android:singleUser="true"
    android:exported="true"
    ```
    *Cờ `singleUser="true"`* ép `ContentProvider` này chỉ chạy duy nhất một instance bên trong User 0.
    Nếu lập trình viên không override phương thức xác thực `enforceReadPermissionInner()` và không kiểm tra `handleIncomingUser()`, bất kỳ ứng dụng độc hại nào chạy ở User phụ (ví dụ một Guest User hoặc Restricted Profile tạo cho trẻ em) đều có thể truy vấn `ContentResolver.query()` và đánh cắp toàn bộ thông tin nhạy cảm của User 0.
