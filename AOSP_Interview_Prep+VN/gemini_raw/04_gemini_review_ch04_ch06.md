Dưới đây là bản phản biện chi tiết từ góc nhìn **Senior AOSP Framework Reviewer** cho 2 chương tài liệu ôn phỏng vấn Android 14.

---

# PHẦN 1: PHẢN BIỆN CHI TIẾT (LỖI KỸ THUẬT & ĐIỂM THIẾU CHÍNH XÁC)

---

## I. Chương 04 — Binder IPC & AIDL

### 1. Vị trí mã nguồn Binder Driver trong Linux Kernel
* **Trích nguyên văn:**  
  > `binder driver (drivers/android/binder.c)`
* **Phân tích kỹ thuật:**  
  Từ Android 8/Kernel 4.14 trở đi, và đặc biệt trên **Android 14 (GKI Kernel 5.15 / 6.1)**, driver Binder không còn là một file đơn lẻ `binder.c`. Nó đã được module hóa và tách thành nhiều file trong thư mục `drivers/android/`:
  - `binder.c`: Quản lý giao thức IPC, xử lý lệnh `ioctl`, IPC thread, Binder node/ref.
  - `binder_alloc.c`: Đảm nhận toàn bộ việc quản lý bộ nhớ, `mmap`, cấp phát buffer giao dịch (`binder_buffer`), tracking vùng nhớ trống.
  - `binder_alloc_selftest.c`, `binder_internal.h`, `binder_trace.h`.  
  Nhầm lẫn này thể hiện ứng viên chưa từng đọc qua source kernel driver hiện đại của AOSP.
* **Sửa đúng:**  
  > `binder driver (drivers/android/: binder.c, binder_alloc.c, binder_internal.h)`

---

### 2. Quản lý Device Node và Binderfs trên Android 14
* **Trích nguyên văn:**  
  > `- 3 device node: /dev/binder (framework ↔ app, và AIDL HAL), /dev/hwbinder (HIDL), /dev/vndbinder (vendor ↔ vendor, AIDL non-stable).`  
  > `- 3 service manager tương ứng: servicemanager, hwservicemanager, vndservicemanager.`
* **Phân tích kỹ thuật:**
  1. **Thiếu Binderfs:** Từ Android 10+, Android chuyển sang dùng **`binderfs`** (mount tại `/dev/binderfs`). Các đường dẫn `/dev/binder`, `/dev/vndbinder`, `/dev/hwbinder` chỉ là symlink trỏ tới `/dev/binderfs/binder`, `/dev/binderfs/vndbinder`,...
  2. **Trạng thái của HIDL / `hwservicemanager` trên Android 14:** HIDL đã bị **deprecated từ Android 11**, đóng băng hoàn toàn ở Android 12/13. Trên Android 14, mọi HAL mới **bắt buộc** dùng AIDL HAL. `hwservicemanager` chỉ còn tồn tại để tương thích ngược cho vendor cũ, không được đưa vào như một kiến trúc tiêu chuẩn đang phát triển của Android 14.
  3. **Hiểu sai về `/dev/vndbinder`:** Nói `/dev/vndbinder` chỉ dùng cho "AIDL non-stable" là sai. `/dev/vndbinder` dành cho IPC nội bộ giữa các tiến trình Vendor (`vendor ↔ vendor`). Các interface này vẫn có thể dùng AIDL có cấu trúc hoặc `@VintfStability` nếu cần, không quy định phải là "non-stable".
* **Sửa đúng:**  
  > - Các node được cấp phát động qua **`binderfs`** (mount tại `/dev/binderfs/` và symlink ra `/dev/`):
  >   - `/dev/binderfs/binder`: Dùng cho Framework ↔ App, App ↔ App, và System ↔ AIDL HAL (Vendor).
  >   - `/dev/vndbinder`: Dùng cho Vendor ↔ Vendor IPC.
  >   - `/dev/hwbinder`: Dùng cho HIDL cũ (deprecated hoàn toàn trên Android 14, đang bị loại bỏ).
  > - Service Manager tương ứng: `servicemanager` (quản lý `/dev/binder`), `vndservicemanager` (quản lý `/dev/vndbinder`), và `hwservicemanager` (legacy).

---

### 3. Cơ chế "1 lần copy" (Zero-copy phía Receiver)
* **Trích nguyên văn:**  
  > `Hiệu năng: chỉ 1 lần copy (sender → buffer kernel đã mmap vào receiver) so với 2 lần của socket/pipe.`
* **Phân tích kỹ thuật:**  
  Câu viết gây hiểu nhầm rằng kernel tạo ra một buffer kernel độc lập rồi sau đó `mmap` vào receiver.  
  Thực tế diễn ra như sau:  
  Khi receiver khởi tạo (trong `ProcessState`), nó gọi `mmap()` lên device node. Driver Binder cấp phát một dải địa chỉ ảo (VMA) trong tiến trình receiver. Khi sender gửi dữ liệu (`BC_TRANSACTION`), kernel cấp phát các physical pages nằm trực tiếp trong vùng mmap của receiver thông qua hàm `binder_alloc_new_buf()` trong `binder_alloc.c`, sau đó gọi **`copy_from_user()`** chép trực tiếp từ userspace buffer của sender vào physical page đó. Receiver chỉ việc đọc dữ liệu ra mà không cần copy thêm lần nào nữa.
* **Sửa đúng:**  
  > `Hiệu năng: Chỉ 1 lần copy. Kernel driver sử dụng copy_from_user() chép dữ liệu trực tiếp từ Userspace của Sender vào trang nhớ vật lý đã được mmap sẵn trong không gian địa chỉ ảo của Receiver.`

---

### 4. Bản chất giới hạn Buffer ~1MB và cơ chế Oneway
* **Trích nguyên văn:**  
  > `Giới hạn buffer: ~1MB mỗi process (thực tế 1MB − 8KB, oneway chỉ một nửa) → TransactionTooLargeException / FAILED_TRANSACTION.`
* **Phân tích kỹ thuật:**  
  Đây là câu trả lời trượt phỏng vấn kinh điển vì hiểu sai phạm vi của con số 1MB:
  1. Con số `1MB - 8KB` (chính xác là `1024KB - 2 * 4KB = 1016KB` với page 4KB, hoặc `1024KB - 32KB` trên hệ thống page 16KB của Android 14) là **tổng kích thước buffer pool mà Driver cấp cho TOÀN BỘ tiến trình Receiver để nhận tất cả các transaction đồng thời**, KHÔNG PHẢI giới hạn cho một transaction đơn lẻ!
  2. Nếu có 10 transaction đồng thời đang gửi đến một process, chúng phải **chia sẻ chung** vùng nhớ 1016KB này.
  3. **Oneway limit:** Driver giới hạn tổng dung lượng các giao dịch bất đồng bộ (`oneway`) không được vượt quá **50% dung lượng trống còn lại** của buffer pool, nhằm ngăn chặn việc spam oneway làm cạn kiệt bộ nhớ của các giao dịch đồng bộ (sync).
* **Sửa đúng:**  
  > `Buffer pool được cấp phát khi mở Binder driver là ~1MB (cụ thể 1MB - 2*PAGE_SIZE, khoảng 1016KB với page 4KB). Đây là TỔNG dung lượng đệm dùng chung cho MỌI giao dịch đang xử lý đồng thời gửi tới tiến trình Receiver (không phải giới hạn cho từng giao dịch đơn lẻ). Riêng tổng các giao dịch oneway chỉ được chiếm tối đa 50% dung lượng trống của pool này để chống nghẽn sync calls.`

---

### 5. Ngôn ngữ của ServiceManager trên Android 14 và API `getService()`
* **Trích nguyên văn:**  
  > `servicemanager (frameworks/native/cmds/servicemanager) là context manager: đăng ký với driver bằng BINDER_SET_CONTEXT_MGR... API: getService (chờ tối đa ~5s, deprecated), checkService (không chờ), waitForService...`
* **Phân tích kỹ thuật:**
  1. **Ngôn ngữ triển khai:** Trên Android 14, `servicemanager` đã được Google **viết lại hoàn toàn bằng Rust** (`frameworks/native/cmds/servicemanager/src/main.rs`). Bản C++ chỉ còn là legacy hoặc shim wrapper.
  2. **Nhầm lẫn cơ chế `getService` giữa Java và C++:**
     - Trong Java (`android.os.ServiceManager`): `getService(name)` là non-blocking, nó trả về `null` ngay lập tức nếu service chưa được đăng ký!
     - Vòng lặp chờ 5 giây (thử lại 5 lần, mỗi lần sleep 1 giây) trước đây chỉ nằm ở tầng C++ Native client (`IServiceManager::getService`). Java không bao giờ có timeout 5s này.
* **Sửa đúng:**  
  > `servicemanager (từ Android 14 được viết hoàn toàn bằng Rust tại frameworks/native/cmds/servicemanager/src/main.rs) là Context Manager... Trong Java, ServiceManager.getService() không block (trả về null ngay nếu chưa có). Muốn chờ phải dùng waitForService() hoặc waitForDeclaredService().`

---

### 6. Thư viện NDK Backend không phải LL-NDK
* **Trích nguyên văn:**  
  > `NDK | libbinder_ndk | vendor HAL, APEX, mainline | ✔ (LL-NDK)`
* **Phân tích kỹ thuật:**  
  `libbinder_ndk` **không phải là LL-NDK**.  
  Trong quy chuẩn Treble và NDK của Google:
  - **LL-NDK (Low-Level NDK)** là các thư viện hệ thống chạy ở `/system` chia sẻ trực tiếp với `/vendor` mà không bị cô lập VNDK, ví dụ: `libc.so`, `libm.so`, `libdl.so`, `liblog.so`.
  - `libbinder_ndk` là một **Stable C API (NDK API)** có ABI cố định, có thể link ở cả System, APEX và Vendor, nhưng không thuộc nhóm LL-NDK.
* **Sửa đúng:**  
  > Đổi cột "Ổn định ABI" thành `✔ (Stable NDK API / ABI Stable)`.

---

### 7. Hỗ trợ kiểu `Map` trong Stable AIDL
* **Trích nguyên văn:**  
  > `Primitive, String, byte[], List<T>, T[], Map (không dùng cho stable)...` và `Stable AIDL không cho dùng Map không kiểu...`
* **Phân tích kỹ thuật:**  
  Câu chữ gây hiểu lầm rằng "nếu dùng Map có kiểu (typed Map) thì Stable AIDL cho phép".  
  **Thực tế:** Stable AIDL (`@VintfStability` hoặc `stability: "vintf"`) **cấm hoàn toàn 100% kiểu `Map`** (kể cả typed hay untyped), vì `Map` không có cấu trúc layout bộ nhớ đồng nhất và không thể map tương thích an toàn 1-1 giữa Java, C++ STL, NDK và Rust. Trình biên dịch `aidl` sẽ báo lỗi ngay lập tức nếu thấy từ khóa `Map` trong file AIDL VINTF.
* **Sửa đúng:**  
  > `Stable AIDL (@VintfStability) cấm hoàn toàn kiểu Map (dù có kiểu hay không). Chỉ cho phép dùng List, mảng (Array) và Structured Parcelable.`

---

## II. Chương 06 — System Service & Manager

### 8. LỖI CHÍ MẠNG: Gọi hàm Blocking (`waitForDeclaredService`) trong Boot Phase của SystemServer
* **Trích nguyên văn (Mục 6.1, Q1 - Bước 2):**
  ```java
  @Override
  public void onBootPhase(int phase) {
      if (phase == PHASE_SYSTEM_SERVICES_READY) {
          IBinder b = ServiceManager.waitForDeclaredService(
                  IDavidDeviceControl.DESCRIPTOR + "/default");
          if (b != null) mHal = IDavidDeviceControl.Stub.asInterface(b);
      }
  }
  ```
* **Phân tích kỹ thuật:**  
  Đây là **lỗi nghiêm trọng nhất trong tài liệu**.  
  Ở Chương 04 Q4, tác giả vừa chỉ trích việc gọi `waitForDeclaredService()` trên Main Thread vì gây ANR, nhưng sang Chương 06 Q1 lại viết chính hàm này vào `onBootPhase()` của `SystemServer`!
  - `SystemServer` chạy các lifecycle callback (`onStart`, `onBootPhase`) tuần tự trên **Main Thread của SystemServer**.
  - `waitForDeclaredService()` là hàm **block vô hạn** cho tới khi service xuất hiện.
  - Nếu Driver HAL bị crash loop, vendor init script lỗi, hoặc phần cứng phản hồi chậm lúc boot, `SystemServer` sẽ bị treo cứng tại `PHASE_SYSTEM_SERVICES_READY`. Sau 60 giây, **Watchdog của SystemServer sẽ phát hiện main thread bị block và lập tức PANIC/REBOOT toàn bộ hệ thống (Bootloop)**.
* **Sửa đúng:**  
  Không bao giờ block Main thread của SystemServer. Phải lấy non-blocking hoặc chuyển sang worker thread:
  ```java
  @Override
  public void onBootPhase(int phase) {
      if (phase == PHASE_SYSTEM_SERVICES_READY) {
          FgThread.getHandler().post(() -> {
              IBinder b = ServiceManager.waitForDeclaredService(
                      IDavidDeviceControl.DESCRIPTOR + "/default");
              synchronized (mLock) {
                  if (b != null) {
                      mHal = IDavidDeviceControl.Stub.asInterface(b);
                      linkToHalDeath(b);
                  }
              }
          });
      }
  }
  ```

---

### 9. LỖI KIẾN TRÚC: Gọi IPC ra bên ngoài khi đang giữ Lock nội bộ
* **Trích nguyên văn (Mục 6.1, Q1 - Bước 2):**
  ```java
  synchronized (mLock) {
      // AIDL enum @Backing(type="int") sinh ra Java là hằng int: LedColor.RED, ...
      mHal.setLedColor(color);
      mColor = color;
  }
  ```
* **Phân tích kỹ thuật:**  
  Chính tác giả ở câu Q7 đã viết: *"Nguyên nhân hay gặp: gọi HAL/binder đồng bộ trong lúc giữ lock quan trọng... Bài học: không gọi HAL chậm trong synchronized(mLock)"*. Nhưng code mẫu ở Q1 lại vi phạm chính nguyên tắc này.
  - `mHal.setLedColor(color)` là một **Synchronous Binder Call** sang tiến trình HAL khác.
  - Nếu daemon HAL bị treo, deadlocked hoặc đang bận ghi sysfs bị block bởi I/O kernel, thread trong `system_server` sẽ giữ `mLock` vô thời hạn.
  - Mọi lời gọi khác vào `getColor()` hoặc các method khác cần `mLock` sẽ bị nghẽn dây chuyền. Nếu một service quan trọng khác (như AMS/WMS) cần lock này, Watchdog sẽ nổ sau 60s.
* **Sửa đúng:**  
  Thực hiện IPC **ngoài khối synchronized**:
  ```java
  @Override
  public void setColor(int color) {
      getContext().enforceCallingOrSelfPermission(
              android.Manifest.permission.CONTROL_LED, "setColor");
      
      IDavidDeviceControl hal;
      synchronized (mLock) {
          hal = mHal;
      }
      if (hal == null) {
          Slog.e(TAG, "HAL service not ready");
          return;
      }

      final long token = Binder.clearCallingIdentity();
      try {
          // Gọi Binder ra ngoài khi KHÔNG giữ lock
          hal.setLedColor(color);
          synchronized (mLock) {
              mColor = color;
          }
      } catch (RemoteException e) {
          Slog.e(TAG, "Failed to call HAL", e);
      } finally {
          Binder.restoreCallingIdentity(token);
      }
  }
  ```

---

### 10. `ServiceManager.sCache` không tự động cache dịch vụ tùy biến
* **Trích nguyên văn (Mục 6.1, Q5):**  
  > `Lần đầu gọi ServiceManager.getService(name) (cache trong ServiceManager.sCache cho một số service quan trọng được AMS truyền sẵn lúc bindApplication).`
* **Phân tích kỹ thuật:**  
  Câu này diễn đạt sai bản chất hoạt động của `ServiceManager.sCache`:
  - `ServiceManager.sCache` là một map tĩnh nội bộ. Nó **chỉ** được điền trước (pre-seed) một lần duy nhất lúc khởi động tiến trình trong `ActivityThread.handleBindApplication()` thông qua dữ liệu `AppBindData.services` do AMS gửi sang (chứa các service cốt lõi: `activity`, `package`, `window`,...).
  - Khi app gọi `ServiceManager.getService("led")`, `ServiceManager` chỉ truy vấn Binder Driver sang `servicemanager`. Nó **KHÔNG HỀ ghi kết quả "led" vào `sCache`**!
  - Việc cache các Manager/Service cho app nằm ở **`SystemServiceRegistry`** thông qua `CachedServiceFetcher` (lưu trong mảng `mServiceCache` của từng `ContextImpl`), chứ không phải do `ServiceManager` tự cache.
* **Sửa đúng:**  
  > `ServiceManager.sCache chỉ chứa các binder cốt lõi do AMS gửi qua AppBindData lúc bindApplication. Lời gọi ServiceManager.getService() cho các dịch vụ thông thường không được ghi vào sCache; việc cache instance được thực hiện ở tầng framework bởi SystemServiceRegistry (thông qua ContextImpl.mServiceCache).`

---

### 11. Deprecated của `sharedUserId` trên Android 14
* **Trích nguyên văn (Mục 6.1, Q4):**  
  > `Persistent app sharedUserId=android.uid.system`
* **Phân tích kỹ thuật:**  
  Từ Android 11 (API 29), Google đã chính thức **Deprecate `sharedUserId`**. Trên Android 14, việc dùng `android:sharedUserId="android.uid.system"` bị cảnh báo nghiêm ngặt và metalava/lint sẽ chặn trên các target API mới, do nó phá vỡ mô hình phân quyền SELinux per-package và gây rủi ro bảo mật lớn.  
  Trong câu hỏi phỏng vấn Senior, nếu đưa ra giải pháp dùng `sharedUserId` mà không cảnh báo deprecation và không nêu giải pháp thay thế (chạy dưới dạng Apex standalone service hoặc cấp privilege qua `privapp-permissions` whitelist) thì sẽ bị đánh giá là kiến thức cũ (pre-Android 10).
* **Sửa đúng:**  
  > Cần bổ sung ghi chú: `sharedUserId="android.uid.system"` đã bị Google deprecate từ Android 11+. Trên Android 14, kiến trúc chuẩn khuyến nghị tách thành standalone service chạy dưới UID riêng, cấp quyền thông qua privapp allowlist XML và cấu hình sepolicy domain riêng biệt thay vì gom chung vào system_app.

---

# PHẦN 2: CÁC CÂU HỎI PHỎNG VẤN SENIOR CÒN THIẾU & ĐÁP ÁN CHUẨN

Dưới đây là 6 câu hỏi chuyên sâu cấp độ Senior/Staff Framework Engineer thường xuyên xuất hiện tại các vòng System Design & Framework Internals.

---

### Q13 (Binder). Priority Inversion là gì? Binder Driver giải quyết bài toán này như thế nào?

**Trả lời:**
* **Vấn đề (Priority Inversion):** Xảy ra khi một Client có độ ưu tiên cao (ví dụ: UI Thread đang chạy `SCHED_FIFO` hoặc high-nice CFS) gọi đồng bộ sang một Server Thread đang chạy ở độ ưu tiên thấp (`SCHED_OTHER`, low-nice). Client sẽ bị block chờ Server xử lý xong. Nếu có một tiến trình độ ưu tiên trung bình chiếm CPU, Server không được chạy, gián tiếp làm treo Client độ ưu tiên cao → Gây jank frame hoặc drop audio.
* **Cơ chế giải quyết của Binder Driver (Priority Inheritance):**
  1. Khi Client thực hiện cuộc gọi đồng bộ (`BC_TRANSACTION`), Binder Driver ghi nhận policy lập lịch (`sched_policy`) và giá trị nice/priority của Client thread.
  2. Driver tìm kiếm thread nhận trong process đích và **tạm thời nâng priority của Server thread lên bằng với priority của Caller thread** (áp dụng cho cả CFS nice value và Real-time SCHED_FIFO/SCHED_RR).
  3. Sau khi Server xử lý xong và gửi `BC_REPLY`, Driver sẽ **hạ priority của Server thread về lại mức ban đầu** trước khi trả kết quả về cho Client.
* **Lưu ý chuyên sâu:** Kỹ thuật này chỉ hoạt động với cuộc gọi **đồng bộ (synchronous)**. Với `oneway` (asynchronous), Driver **không** áp dụng Priority Inheritance vì Client không block chờ kết quả.

---

### Q14 (Binder). Truyền dữ liệu lớn qua Binder: Tại sao CursorWindow, Ashmem, SharedMemory, và ParcelFileDescriptor tránh được lỗi `TransactionTooLargeException`?

**Trả lời:**
* **Bản chất lỗi:** Binder Buffer pool chỉ có ~1MB dùng chung. Nếu truyền Payload > vài trăm KB (ảnh bitmap, danh sách database hàng nghìn dòng), giao dịch sẽ văng `TransactionTooLargeException`.
* **Giải pháp Zero-copy qua Shared Memory / File Descriptor:**
  Thay vì đẩy mảng byte qua Parcel:
  1. **Ashmem / ASharedMemory (Android Shared Memory):**
     - Tiến trình gửi tạo một vùng nhớ chia sẻ ẩn danh (anonymous shared memory) trong kernel bằng `ashmem_create_region()` hoặc `ASharedMemory_create()`.
     - Kernel trả về một `file descriptor` (fd). Dữ liệu được ghi trực tiếp vào vùng nhớ này.
     - Sender truyền `fd` qua Binder Parcel (`ParcelFileDescriptor`).
     - Khi Binder Driver gặp một object kiểu `BINDER_TYPE_FD`, nó thực hiện **dup fd** vào bảng file descriptor của tiến trình Receiver.
     - Receiver nhận fd và gọi `mmap()`, ánh xạ trực tiếp vào cùng một vùng nhớ vật lý đó. **Zero IPC copy** cho payload chính.
  2. **CursorWindow (Dùng trong SQLite / ContentProvider):**
     - Thực chất là một lớp bọc (wrapper) phía trên Ashmem/SharedMemory.
     - Dữ liệu truy vấn DB được fill vào CursorWindow (mặc định cấu hình từ 2MB đến hàng chục MB tùy vendor). ContentProvider chỉ trả về Binder token trỏ tới fd của CursorWindow.
  3. **ParcelFileDescriptor (PFD):**
     - Mở rộng cơ chế pipe hoặc file disk thông thường qua `ParcelFileDescriptor.createPipe()`, cho phép stream dữ liệu không giới hạn kích thước giữa 2 tiến trình.

---

### Q15 (Binder). Vòng đời đối tượng phân tán: Cơ chế dọn rác (GC) giữa Java Layer, Native C++ (`sp/wp`), và Kernel Binder Node diễn ra như thế nào?

**Trả lời:**
Kiến trúc quản lý bộ nhớ của Binder là mô hình **Cross-process Distributed Reference Counting**:
1. **Liên kết các tầng:**
   - Client Java giữ `BinderProxy` → giữ con trỏ native `BpBinder`.
   - `BpBinder` giữ `handle` nguyên thủy trỏ tới `binder_ref` trong Kernel.
   - Kernel `binder_ref` trỏ tới `binder_node` của Server.
   - `binder_node` trỏ tới `BBinder` (Native Server) → giữ tham chiếu JNI Global Ref tới `Binder` (Java Server implementation).
2. **Cơ chế Reference Count của Kernel:**
   - Kernel duy trì 2 bộ đếm cho mỗi node: `strong count` và `weak count`.
   - Khi `BinderProxy` ở Client được khởi tạo, nó tăng strong ref của `BpBinder`, driver gửi lệnh `BC_ACQUIRE` tăng strong count của node.
3. **Khi Client không còn dùng (GC):**
   - Khi không còn biến Java nào trỏ tới `BinderProxy`, Java GC thu gom nó.
   - Destructor của `BinderProxy` gọi JNI xóa `BpBinder`.
   - Destructor của `BpBinder` gửi lệnh **`BC_RELEASE`** về Binder Driver để giảm `strong count`.
   - Khi `strong count` của node về 0, Driver gửi lệnh **`BR_DECREFS` / `BR_RELEASE`** sang Server.
   - `BBinder` ở Server xóa JNI Global Reference tới đối tượng Java Service, cho phép Java GC của Server thu gom service object.
* **Pitfall (Circular Reference):** Nếu Server đăng ký một listener callback ngược lại Client mà không dùng `DeathRecipient` hoặc weak reference, hai tiến trình sẽ tạo thành một đồ thị tham chiếu vòng chéo tiến trình (Cross-process circular ref), khiến Java GC ở cả 2 bên đều không thể giải phóng bộ nhớ.

---

### Q16 (Security/Binder). Tại sao `Binder.getCallingPid()` không đáng tin cậy trong kiểm tra bảo mật? Cách Android 12+ giải quyết bằng `AttributionSource`?

**Trả lời:**
* **Lỗ hổng của `getCallingPid()` (PID Recycling Attack):**
  - PID do Linux Kernel quản lý có phạm vi giới hạn (mặc định tối đa 32768).
  - Khi Client gửi một request qua Binder rồi lập tức chết hoặc bị kill, Linux Kernel có thể tái sử dụng (recycle) PID đó cho một tiến trình khác gần như ngay lập tức.
  - Nếu Server lấy `Binder.getCallingPid()` để truy vấn package name hoặc quyền qua `PackageManager`, nó có thể truy vấn nhầm thông tin của tiến trình mới được sinh ra mang PID đó → **Lỗ hổng leo thang đặc quyền (Privilege Escalation)**.
  - Do đó, **chỉ có `Binder.getCallingUid()` là an toàn tuyệt đối** vì UID của app không bị thay đổi trong suốt vòng đời cài đặt.
* **Giải quyết chuỗi ủy quyền qua `AttributionSource` (Android 12+):**
  - Trong bài toán IPC đa chặng (Multi-hop): App A gọi Service B, Service B lại gọi tiếp sang LocationManagerService C.
  - Nếu C chỉ kiểm tra UID của B, C sẽ thấy B là `system_server` (đủ quyền), dẫn tới lỗ hổng *Confused Deputy*.
  - Android 12 giới thiệu **`AttributionSource`**: một token đại diện cho toàn bộ chuỗi gọi (Calling Chain: `App A -> Service B -> Service C`).
  - Mỗi chặng trong IPC phải đính kèm `AttributionSource`. Tại Service C, framework dùng `PermissionChecker.checkPermissionForDataDelivery()` để xác thực quyền của **tất cả** các node trong chuỗi, đảm bảo App A thực sự có quyền thì Service B mới được phép lấy dữ liệu thay cho A.

---

### Q17 (System Server). SystemServer Watchdog hoạt động như thế nào? Cơ chế phát hiện Deadlock và xử lý?

**Trả lời:**
* **Bản chất của `Watchdog.java`:** Chạy một thread độc lập (`Thread` riêng) trong `system_server` với chu kỳ kiểm tra mỗi 30 giây.
* **Cấu trúc giám sát gồm 2 cơ chế chính:**
  1. **`HandlerChecker`:** Theo dõi MessageQueue của các Thread cốt lõi trong `system_server` (Main thread, `fg` (Foreground), `ui`, `io`, `display`, `animation`). Mỗi vòng lặp 30s, Watchdog post một Runnable đo thời gian (`mCompleted = false`) vào MessageQueue của từng thread. Nếu sau chu kỳ tiếp theo mà Runnable chưa được thực thi, thread đó được coi là đang bị nghẽn (stuck/blocked).
  2. **`Monitor` interface (Lock checker):** Các service trọng yếu (như `ActivityManagerService`, `WindowManagerService`, `PowerManagerService`) implement interface `Watchdog.Monitor`. Trong hàm `monitor()`, service chỉ thực hiện duy nhất một việc: cố gắng lấy lock nội bộ của nó (ví dụ: `synchronized (this) {}`). Nếu lock bị chiếm giữ quá lâu bởi một thread khác đang bị treo, hàm `monitor()` sẽ bị block.
* **Quy trình xử lý khi phát hiện Stuck (Timeout 60s):**
  - **Mốc 30s (Half-timeout):** Ghi log warning, dump nhẹ trạng thái.
  - **Mốc 60s (Full-timeout):**
    1. Gửi tín hiệu kernel **`SIGQUIT` (Signal 3)** tới chính `system_server` và các native daemon liên quan (SurfaceFlinger, AudioFlinger...) để ghi toàn bộ Thread Stack Traces vào file `/data/anr/traces.txt`.
    2. Đợi vài giây để hoàn tất việc ghi log/Dropbox.
    3. Gọi **`Process.killProcess(Process.myPid())`** để tự sát (Crash System Server).
    4. Tiến trình cha là `init` nhận thấy `system_server` thoát (`onrestart`), nó sẽ kill toàn bộ `zygote` và mọi app đang chạy, sau đó khởi động lại Zygote và SystemServer từ đầu (hiện tượng **Soft Reboot / Framework Crash**).

---

### Q18 (System Service Architecture). Phân biệt `SystemService` (AOSP class) và `IBinder/Stub`? Tại sao không cho `SystemService` kế thừa trực tiếp từ `Stub`?

**Trả lời:**
* **`SystemService` (`com.android.server.SystemService`):**
  - Là class trừu tượng đại diện cho **Vòng đời hệ thống nội bộ** (System Lifecycle).
  - Nó chỉ sống bên trong `system_server`, do `SystemServiceManager` khởi tạo qua reflection và quản lý các trạng thái: `onStart()`, `onBootPhase(int)`, `onUserStarting()`, `onUserSwitching()`.
  - Không thể được gọi từ bên ngoài thông qua IPC.
* **`Stub` (ví dụ `ILedManager.Stub` extends `android.os.Binder`):**
  - Là class đại diện cho **Giao diện IPC từ xa** (Remote Interface / RPC Endpoint).
  - Được đăng ký công khai với `ServiceManager` để các process khác (App, Vendor) có thể lấy handle và gọi hàm thông qua Binder.
* **Tại sao không thiết kế 1 class duy nhất vừa `extends SystemService` vừa `implements ILedManager` (hoặc extends `Stub`)?**
  1. **Đơn nhiệm (Single Responsibility Principle):** Tách bạch rõ ràng giữa *Quản lý vòng đời hệ thống* và *Xử lý giao tiếp IPC Client-Server*.
  2. **Ngăn chặn lộ Private State (Security & Encapsulation):** Nếu gộp chung, các method nội bộ của `SystemService` có nguy cơ bị expose nhầm ra Binder RPC nếu cấu hình AIDL không chuẩn, hoặc các method public của AIDL sẽ bị trộn lẫn với các hàm quản lý tài nguyên của hệ thống.
  3. **Mô hình Internal Service (LocalServices):** Service thường cung cấp thêm một interface nội bộ không-binder (ví dụ: `LedManagerInternal`) để các service khác cùng nằm trong `system_server` gọi trực tiếp với tốc độ cao mà không phải chịu overhead của Binder và không cần check permission. Tách class giúp kiến trúc phân lớp sạch sẽ:
     ```
     LedManagerService (extends SystemService)
         ├── BinderService (extends ILedManager.Stub) -> Cho app/bên ngoài gọi
         └── LocalService (extends LedManagerInternal) -> Cho AMS/PMS trong system_server gọi
     ```
