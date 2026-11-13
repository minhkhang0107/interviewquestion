Dưới góc độ Senior AOSP Reviewer, tôi hoàn toàn tôn trọng bằng chứng từ mã nguồn thực tế (*source code is the single source of truth*). Dưới đây là phản hồi chi tiết cho từng điểm phản biện và 5 câu hỏi bổ sung cho bộ tài liệu phỏng vấn.

---

### Đối chiếu và phản hồi 14 điểm kỹ thuật trên AOSP Android 14

#### 1. Servicemanager A14 viết bằng Rust
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** Trong AOSP Android 14, daemon `servicemanager` chính thức vẫn nằm tại `frameworks/native/cmds/servicemanager/` và được triển khai hoàn toàn bằng **C++** (`ServiceManager.cpp`, `main.cpp`, `Access.cpp`). Thành phần core hệ thống được viết lại hoàn toàn bằng Rust từ Android 12 là **Keystore 2.0** (`system/security/keystore2`), không phải servicemanager.

#### 2. libbinder_ndk KHÔNG phải LL-NDK
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** `frameworks/native/libs/binder/ndk/Android.bp` có định nghĩa rõ ràng `llndk: { symbol_file: "libbinder_ndk.map.txt" }`. `libbinder_ndk` chính là **LL-NDK (Low-Level NDK)**, cho phép cả Vendor code (HALs) và System code liên lạc qua giao thức Binder ổn định theo ABI C thuần túy mà không bị ràng buộc bởi sự thay đổi ABI nội bộ của C++ `libbinder`.

#### 3. sharedUserId deprecated ở API 29 & platform app AOSP
- **ĐỒNG Ý**
- **Lý do kỹ thuật:**
  1. API 29 chính xác là **Android 10 (Q)**, không phải Android 11 (API 30).
  2. Mặc dù Google deprecated thuộc tính này đối với app bên thứ ba, các ứng dụng hệ thống cốt lõi trên AOSP A14 (`Settings`, `SystemUI`, `Telecom`) vẫn bắt buộc khai báo `android:sharedUserId="android.uid.system"` để chạy trong tiến trình hệ thống với UID 1000.

#### 4. Cơ chế restart Zygote khi system_server chết
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** Trong `com_android_internal_os_Zygote.cpp`, Zygote quản lý `gSystemServerPid`. Khi nhận tín hiệu `SIGCHLD` báo `system_server` đã chết, hàm xử lý tín hiệu ghi log: *"System server process %d has died. Restarting Zygote!"* và chủ động gọi `RuntimeAbort()`. Zygote **tự kết liễu**, sau đó `init` (qua cấu hình service trong `init.zygoteXX.rc`) mới phát hiện Zygote đã thoát và kích hoạt restart toàn bộ tiến trình liên quan.

#### 5. Bảng giá trị oom_adj trong ProcessList.java
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** Quy ước `oom_adj` của AOSP quy định số âm chỉ dành cho tiến trình cấp hệ thống / bất tử (`NATIVE_ADJ = -1000`, `SYSTEM_ADJ = -900`, `PERSISTENT_PROC_ADJ = -800`, `PERSISTENT_SERVICE_ADJ = -700`). App người dùng ở Foreground luôn bắt đầu từ `FOREGROUND_APP_ADJ = 0` và tăng dần lên `CACHED_APP_MAX_ADJ = 999`. Ghi `-700` cho foreground app là sai lệch hoàn toàn.

#### 6. Giá trị MAX_CACHED_PROCESSES mặc định
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** Trong `ActivityManagerConstants.java` trên Android 14, hằng số `DEFAULT_MAX_CACHED_PROCESSES = 1024`. Giới hạn cứng 32 process từ thời Android đời cũ đã được nới lỏng từ Android 13/14 để nhường quyền điều phối và thu hồi RAM cho kernel và **LMKD** dựa trên chỉ số áp lực bộ nhớ thực tế (**PSI** - Pressure Stall Information).

#### 7. Công thức tính trọng số khoảng cách trong FocusFinder
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** Tại hàm `getWeightedDistanceFor()` trong `FocusFinder.java`:
  $$\text{Distance} = 13 \times \text{major} \times \text{major} + \text{minor} \times \text{minor}$$
  Thuật toán bình phương cả hai trục và nhân hệ số 13 vào trục chính để phạt rất nặng độ lệch góc, đảm bảo điều hướng D-pad ưu tiên đi thẳng thay vì nhảy chéo phần tử.

#### 8. Timeout 6 giờ cho FGS dataSync
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** Trên Android 14, giới hạn thời gian chạy Foreground Service chỉ áp dụng cho loại `shortService` (~3 phút). Giới hạn thời gian 6 giờ cho `dataSync` và `mediaProcessing` (đi kèm callback `Service.onTimeout()`) là tính năng mới được đưa vào từ **Android 15 (API 35)**; trong `ActiveServices.java` của Android 14 chưa có cơ chế timeout này cho `dataSync`.

#### 9. Hành vi gọi startForeground() không truyền type trên A14
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** `Service.java` định nghĩa `startForeground(id, notification)` mặc định truyền `FOREGROUND_SERVICE_TYPE_MANIFEST`. `ActiveServices.java` sẽ đối chiếu với thuộc tính `android:foregroundServiceType` đã khai báo trong thẻ `<service>` tại `AndroidManifest.xml`. Ngoại lệ `MissingForegroundServiceTypeException` chỉ xảy ra khi target SDK 34 mà hoàn toàn không khai báo type nào trong manifest.

#### 10. Kích thước mặc định của CursorWindow
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** Trong `frameworks/base/core/res/res/values/config.xml` của Android 14, giá trị `config_cursorWindowSize` được cấu hình là `2048` (đơn vị KB, tương đương **2MB**). Con số 4MB là nhận định sai; 2MB vẫn là baseline chuẩn của AOSP trừ khi OEM override lại qua RRO.

#### 11. Danh sách miễn trừ Implicit Broadcast
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** Không có file `BroadcastConstants.java` chứa danh sách này. Danh sách implicit broadcast được phép đánh thức app chạy nền được cấu hình qua thẻ `<allow-implicit-broadcast>` trong các file XML quyền hạn tại `/etc/sysconfig/`, được `SystemConfig.java` nạp vào `mBackgroundLaunchBroadcasts` của AMS khi khởi động hệ thống.

#### 12. Xử lý phím cứng chuyên dụng (Global Key) trên Android TV
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** `PhoneWindowManager` ủy thác việc xử lý các phím đặc biệt cho `GlobalKeyManager`. Lớp này đọc cấu hình từ `res/xml/global_keys.xml` để dispatch broadcast `Intent.ACTION_GLOBAL_BUTTON` tới `ComponentName` đã đăng ký. Cơ chế chuẩn của AOSP/STB là dùng RRO overlay file XML này trỏ tới broadcast receiver của hãng để mở app, không hardcode `startActivity()` trong PWM.

#### 13. Khởi tạo Tunneled Video Playback (Tunnel mode)
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** API chuẩn ở tầng ứng dụng/framework để bật Tunnel mode là:
  1. Kiểm tra hỗ trợ qua `CodecCapabilities.FEATURE_TunneledPlayback`.
  2. Gắn Audio Session ID vào video format: `mediaFormat.setInteger(MediaFormat.KEY_AUDIO_SESSION_ID, audioSessionId)`.
  Hằng số `KEY_AUDIO_HW_SYNC` (`"audio-hw-sync"`) chỉ là định danh trao đổi nội bộ ở tầng native HAL (lấy qua `AudioManager.getAudioHwSyncForSession()`), không phải public API MediaFormat.

#### 14. Non-blocking behavior của Java ServiceManager.getService()
- **ĐỒNG Ý**
- **Lý do kỹ thuật:** Xác nhận đồng quan điểm. `ServiceManager.getService()` ở tầng Java gọi trực tiếp qua AIDL xuống servicemanager native và trả về ngay (trả về `null` nếu service chưa sẵn sàng; muốn chờ phải dùng `waitForService()`). Logic lặp chờ 5s kèm cảnh báo logcat *"Waiting for service..."* chỉ nằm ở hàm shim C++ `defaultServiceManager()->getService()`.

---

### Bổ sung 5 câu hỏi phỏng vấn tầng App / Framework (Android 14)

#### Câu 1: Cơ chế Broadcast Queue mới (Modern Queue) và app ở trạng thái Cached trên Android 14 hoạt động như thế nào?
- **Đáp án:** Android 14 viết lại hoàn toàn `BroadcastQueueImpl` thành `BroadcastQueueModernImpl`. Khi một tiến trình rơi vào trạng thái `CACHED` (bị đóng băng cgroup - frozen), hệ thống sẽ hoãn (defer) việc phân phối các broadcast không khẩn cấp (non-urgent/implicit). Các broadcast này được gom lại và chỉ xả (flush) khi tiến trình được unfreeze (ví dụ khi user mở lại app hoặc nhận được một broadcast khẩn cấp/explicit).

#### Câu 2: Android 14 siết chặt Background Activity Launch (BAL) đối với `PendingIntent` ra sao?
- **Đáp án:** Để ngăn chặn lỗ hổng leo thang đặc quyền khi truyền `PendingIntent` cho app khác kích hoạt, Android 14 yêu cầu bên tạo `PendingIntent` phải chủ động opt-in quyền khởi chạy ngầm bằng cách truyền `ActivityOptions.setPendingIntentBackgroundActivityStartMode(MODE_BACKGROUND_ACTIVITY_START_ALLOWED)`. Mặc định ở target SDK 34, việc mở Activity từ background qua `PendingIntent` bị chặn hoàn toàn.

#### Câu 3: Quyền truy cập ảnh/video một phần (`READ_MEDIA_VISUAL_USER_SELECTED`) hoạt động thế nào ở tầng Framework?
- **Đáp án:** Khi app target SDK 34 xin quyền `READ_MEDIA_IMAGES` hoặc `READ_MEDIA_VIDEO`, hệ thống hiển thị tùy chọn *"Select photos and videos"*. Nếu user chọn mục này, PackageManager chỉ cấp quyền tạm thời `READ_MEDIA_VISUAL_USER_SELECTED`. App chỉ truy vấn được đúng các Uri đã chọn thông qua `MediaStore`. Khi app bị hủy tiến trình hoặc chạy nền một thời gian, quyền tạm thời này có thể bị thu hồi tự động.

#### Câu 4: Cơ chế Predictive Back Gesture (Cử chỉ quay lại dự đoán) ở tầng WindowManager/Shell hoạt động ra sao?
- **Đáp án:** App cần khai báo `android:enableOnBackInvokedCallback="true"` và đăng ký `OnBackAnimationCallback` qua `OnBackInvokedDispatcher` thay cho `onBackPressed()`. Khi user vuốt mép màn hình, `BackAnimationController` trong `SystemUI/Shell` nhận event từ InputReader, trực tiếp điều khiển hoạt họa thu nhỏ/preview cửa sổ của Activity hiện tại để lộ màn hình phía sau trước khi user nhả tay để quyết định commit hay cancel thao tác back.

#### Câu 5: Về mặt kiến trúc hiển thị (SurfaceFlinger / WindowManager), tại sao video player trên STB/Android TV luôn ưu tiên dùng `SurfaceView` thay vì `TextureView`?
- **Đáp án:** `SurfaceView` tạo ra một `SurfaceControl` riêng, được `SurfaceFlinger` compositing độc lập qua hardware overlay plane, tách rời khỏi RenderThread của View hierarchy. Điều này giúp bypass hoàn toàn chu trình vẽ của ứng dụng, tiết kiệm băng thông bộ nhớ, hỗ trợ zero-copy, giải mã HDR, video DRM (Secure Surface) và Tunneled Playback. Ngược lại, `TextureView` biến buffer video thành OpenGL ES texture vẽ chung trong View tree, tiêu tốn gấp đôi băng thông GPU/RAM và không hỗ trợ Secure/Tunnel mode.
