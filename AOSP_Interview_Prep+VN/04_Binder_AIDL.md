# Chương 04 — Binder IPC & AIDL

## 4.1 Kiến trúc Binder

```
 Client process                                   Server process
┌───────────────────────┐                      ┌─────────────────────────┐
│ IFoo.Stub.Proxy (Java)│                      │ IFoo.Stub (Java)        │
│  / BpFoo (cpp,ndk)    │                      │  / BnFoo (cpp,ndk)      │
│        │ transact()   │                      │   onTransact() ▲        │
│   BinderProxy/BpBinder│                      │   BBinder      │        │
│   IPCThreadState      │                      │   IPCThreadState (pool) │
└────────┬──────────────┘                      └────────────────┼────────┘
         │ ioctl(BINDER_WRITE_READ)                              │
─────────▼───────────────────── kernel ──────────────────────────┴─────────
                      binder driver (drivers/android/binder.c)
     - copy dữ liệu 1 lần (mmap vùng buffer của server), quản lý ref/handle,
       chuyển fd, credential (uid/pid/SELinux sid), thread scheduling, death notice
```
- Driver nằm ở `drivers/android/` của kernel: `binder.c` (giao thức, node/ref, thread), `binder_alloc.c` (quản lý buffer mmap).
- Từ A11 với GKI, node được tạo động qua **binderfs** (`/dev/binderfs/`), còn `/dev/binder`, `/dev/hwbinder`, `/dev/vndbinder` là symlink trỏ vào đó.
- 3 device node: `/dev/binder` (framework ↔ app, và AIDL HAL), `/dev/hwbinder` (HIDL, legacy), `/dev/vndbinder` (IPC nội bộ vendor ↔ vendor, interface build với stability vendor).
- 3 service manager tương ứng: `servicemanager` (vẫn viết bằng **C++** ở A14: `frameworks/native/cmds/servicemanager/ServiceManager.cpp`), `hwservicemanager`, `vndservicemanager`.

---

### Q1. Vì sao Android dùng Binder thay vì socket/pipe/shared memory?

1. **Hiệu năng**: chỉ **1 lần copy** (sender → buffer kernel đã mmap vào receiver) so với 2 lần của socket/pipe.
2. **Bảo mật**: kernel đính kèm **UID/PID/SID** thật của caller (`Binder.getCallingUid()`), không giả mạo được → nền tảng cho permission check.
3. **Mô hình object**: tham chiếu object từ xa (handle), reference counting, **death notification**.
4. **Thread pool + priority inheritance**, truyền **file descriptor**, oneway async.
5. Đồng bộ (RPC) dễ dùng — giống gọi hàm local.

---

### Q2. Binder transaction diễn ra thế nào? (đi sâu)

1. Client gọi `proxy.foo(x)` → Proxy ghi interface token + tham số vào `Parcel` → `mRemote.transact(TRANSACTION_foo, data, reply, flags)`.
2. `IPCThreadState::transact` → `ioctl(fd, BINDER_WRITE_READ)` với lệnh `BC_TRANSACTION`.
3. Driver: tìm `binder_node` đích theo handle, cấp buffer trong vùng mmap của server, copy data **1 lần**, dịch handle/fd sang không gian của server, chọn thread rảnh trong pool server, đánh thức nó với `BR_TRANSACTION`.
4. Server thread: `BBinder::transact` → `onTransact` → Stub đọc Parcel, kiểm tra interface token (`enforceInterface`), gọi implementation.
5. Reply ghi vào Parcel → `BC_REPLY` → client nhận `BR_REPLY`, đọc kết quả. Client thread **block** trong suốt quá trình (trừ oneway).
- Giới hạn buffer: mỗi process mmap **~1MB** (`BINDER_VM_SIZE = 1MB − 2×PAGE_SIZE`, tức 1016KB với page 4KB). Đây là **tổng dung lượng dùng chung cho mọi transaction đang được xử lý đồng thời** tới process đó, không phải giới hạn của từng transaction. Transaction **oneway** chỉ được dùng tối đa **một nửa** vùng này (`free_async_space`) để chừa chỗ cho call đồng bộ. Vượt giới hạn → `TransactionTooLargeException` / `FAILED_TRANSACTION`. *(Gemini góp ý để làm rõ ý này.)*

---

### Q3. `oneway` là gì? Khi nào dùng? Pitfall?

- `oneway` method/interface: client **không chờ** reply, trả về ngay, không có giá trị trả về/out param.
- Các oneway call tới **cùng một binder object** được driver xếp hàng & **thực thi tuần tự** (async_todo).
- Dùng cho: callback/listener từ service về app (tránh service bị app treo), notify event.
- Pitfall:
  - Spam oneway → đầy async buffer (½ của 1MB) → `FAILED_TRANSACTION`, log `binder: ... oneway spamming`.
  - Exception phía server bị nuốt mất.
  - Gọi oneway tới binder **local (cùng process)** thì vẫn chạy đồng bộ.

```aidl
oneway interface IStatusCallback {
    void onLedChanged(int color);
}
```

---

### Q4. Death recipient — tại sao quan trọng? Viết ví dụ.

- Khi process server chết, driver gửi `BR_DEAD_BINDER` tới mọi client đã `linkToDeath` → client cleanup/reconnect.
- Ngược lại server dùng `RemoteCallbackList` (Java) để tự xóa callback của client đã chết.

Ví dụ thật trong project (`davidextensionsvc/.../NativeWrapperImpl.java`):
```java
mIDavidDeviceControl = IDavidDeviceControl.Stub.asInterface(
        ServiceManager.waitForDeclaredService(serviceAIDL));
mIDavidDeviceControl.asBinder().linkToDeath(this, 0);
...
@Override
public void binderDied() {
    Handler handler = new Handler(Looper.getMainLooper());
    handler.postDelayed(runnable /* gọi lại connectToHal(), retry tối đa 5 lần */, 1000);
}
```
**Nhận xét senior (rất hay để kể khi phỏng vấn):**
- `connectToHal()` dùng `waitForDeclaredService()` → **block** cho tới khi HAL lên lại; nó lại chạy trên **main looper** của persistent app → có thể gây **ANR** hoặc treo main thread nếu HAL không lên lại. Nên dùng `HandlerThread` riêng, hoặc `ServiceManager.registerForNotifications()` / `IServiceCallback`.
- Retry vô nghĩa vì `waitForDeclaredService` đã tự chờ; nên set `mIDavidDeviceControl = null` trong `binderDied()` để caller không gọi vào binder chết (`DeadObjectException`).
- Phía native NDK: `AIBinder_DeathRecipient_new(onDied)` + `AIBinder_linkToDeath(binder, recipient, cookie)`.

---

### Q5. ServiceManager hoạt động thế nào? Handle 0 là gì?

- `servicemanager` (`frameworks/native/cmds/servicemanager`) là **context manager**: đăng ký với driver bằng `BINDER_SET_CONTEXT_MGR`, mọi process truy cập nó qua **handle 0**.
- API: `addService(name, binder)`, `getService`, `checkService` (không chờ), `waitForService`, `isDeclared`, `registerForNotifications`.
- **Chú ý khác biệt Java vs C++** (Gemini phát hiện, đã kiểm chứng):
  - C++ `defaultServiceManager()->getService()` (`ServiceManagerShim`, `frameworks/native/libs/binder/IServiceManager.cpp:130`) **chờ tối đa ~5s** (vòng `sleep(1)`), log `Waiting for service ...` (deprecated, nên dùng `waitForService`/`checkService`).
  - Java `android.os.ServiceManager.getService()` **không chờ**: trả `null` ngay nếu chưa có (trừ binder đã được pre-seed trong `sCache`). Muốn chờ dùng `waitForService()`/`waitForDeclaredService()`.
- Mỗi `addService`/`find` được kiểm tra **SELinux** `service_manager { add find list }` dựa trên `service_contexts`.
- Với AIDL HAL vendor (`@VintfStability`), servicemanager còn kiểm tra **service có được khai báo trong VINTF manifest** không (`vintf_fragments`) — nếu không, addService bị từ chối: `Could not find ... in the VINTF manifest`.

---

### Q6. Các backend AIDL: Java, CPP, NDK, Rust — chọn cái nào?

| Backend | Thư viện | Dùng ở đâu | Ổn định ABI |
|---------|----------|-----------|-------------|
| Java | `framework.jar` binder | framework, app | — |
| CPP | `libbinder` | system native (không dùng cho vendor/HAL) | ✘ |
| NDK | `libbinder_ndk` | **vendor HAL**, APEX, mainline | ✔ (LL-NDK — `ndk/Android.bp` có stanza `llndk {}`; Gemini từng phản bác điểm này nhưng đã sai) |
| Rust | `libbinder_rs` | code Rust mới | ✔ |

Trong project, `aidl_interface` tắt CPP, bật Java (cho client `DavidExtension`) và NDK (mặc định, cho HAL service):
```bp
aidl_interface {
    name: "vendor.david.hardware.daviddevicecontrol",
    vendor_available: true,
    srcs: ["vendor/david/hardware/daviddevicecontrol/*.aidl"],
    stability: "vintf",
    backend: {
        cpp: { enabled: false },
        java: { enabled: true, sdk_version: "module_current" },
    },
}
```
→ sinh ra `vendor.david.hardware.daviddevicecontrol-V1-ndk` và `-V1-java`.

---

### Q7. Code generated từ AIDL gồm những gì?

Với `IDavidDeviceControl.aidl`:
- **NDK**: `aidl/vendor/david/hardware/daviddevicecontrol/IDavidDeviceControl.h` (interface), `BnDavidDeviceControl.h` (server stub), `BpDavidDeviceControl.h` (client proxy), enum `LedColor.h`.
- **Java**: `IDavidDeviceControl` với `Stub` (server, kế thừa `Binder`) và `Stub.Proxy` (client), `DESCRIPTOR`, `getInterfaceVersion()`, `getInterfaceHash()`.
- Xem tại `out/soong/.intermediates/vendor/david/interfaces/daviddevicecontrol/aidl/...-ndk-source/gen/`.

---

### Q8. Các directional tag `in`, `out`, `inout` và kiểu dữ liệu AIDL hỗ trợ?

- `in`: client → server (mặc định cho primitive). `out`: server ghi, client nhận. `inout`: cả hai (tốn 2 lần marshal).
- Primitive, `String`, `byte[]`, `List<T>`, `T[]`, `Map` (không dùng cho stable), `ParcelFileDescriptor`, `IBinder`, interface khác, `parcelable`, `enum` (`@Backing(type="int")`), `union`, `@nullable`, `const`.
- `Map` chỉ được hỗ trợ ở **backend Java**. Interface stable/VINTF cần sinh NDK/CPP/Rust nên thực tế **không dùng được `Map`**; thay bằng `List<parcelable Entry{key,value}>`. Parcelable Java tự viết (khai báo `parcelable Foo;` không có cấu trúc) cũng không được dùng; stable AIDL chỉ cho phép **structured parcelable**.

```aidl
@VintfStability
parcelable LedState {
    LedColor color = LedColor.OFF;
    boolean blinking;
    int periodMs;
}
```

---

### Q9. Binder thread pool: mặc định bao nhiêu thread? Lỗi "binder thread pool exhausted"?

- App/system_server: max **15** thread binder + main (system_server set 31). Native service tự set: `ABinderProcess_setThreadPoolMaxThreadCount(n)` + `ABinderProcess_startThreadPool()` / `joinThreadPool()`.
- Project HAL: `ABinderProcess_setThreadPoolMaxThreadCount(8); ... ABinderProcess_joinThreadPool();` — với HAL chỉ điều khiển LED qua sysfs, 8 thread là dư, có thể để 1-2; nhưng phải đảm bảo implementation **thread-safe** (nhiều thread có thể cùng ghi sysfs).
- Khi tất cả thread bận (deadlock, call chậm) → client block, log `binder thread pool (15 threads) starved`, dễ dẫn tới **ANR / watchdog**.

---

### Q10. `Binder.clearCallingIdentity()` dùng khi nào?

- Trong system service, khi đang xử lý call từ app (calling uid = app) nhưng cần gọi tiếp API khác **với quyền của system_server** (ví dụ gọi `ContentResolver`, `PackageManager` cần quyền system):
```java
final long token = Binder.clearCallingIdentity();
try {
    mContext.getContentResolver().notifyChange(...);
} finally {
    Binder.restoreCallingIdentity(token);
}
```
- Kiểm tra quyền **trước** khi clear identity, nếu không là lỗ hổng *confused deputy*.

---

### Q11. AIDL vs Messenger vs ContentProvider vs Broadcast?

- AIDL: RPC đa luồng, kiểu mạnh, nhiều method → dùng cho service phức tạp.
- Messenger: Handler-based, **tuần tự 1 thread**, đơn giản.
- ContentProvider: chia sẻ dữ liệu dạng bảng/URI, có permission theo URI.
- Broadcast: 1-n, lỏng lẻo, chậm, có hạn chế background (implicit broadcast).

---

### Q12. Exception/Status trong AIDL được truyền thế nào?

- Java: `RemoteException`, và các exception "đặc biệt" được parcel: `SecurityException`, `IllegalArgumentException`, `IllegalStateException`, `NullPointerException`, `UnsupportedOperationException`, `ServiceSpecificException`.
- NDK: `ndk::ScopedAStatus::ok()`, `fromExceptionCode(EX_ILLEGAL_ARGUMENT)`, `fromServiceSpecificError(code)`, `fromStatus(STATUS_...)`.
- HAL nên trả lỗi rõ ràng thay vì luôn `ok()`:
```cpp
::ndk::ScopedAStatus DavidDeviceControl::setLedColor(LedColor c) {
    if (!isValid(c)) return ndk::ScopedAStatus::fromExceptionCode(EX_ILLEGAL_ARGUMENT);
    if (!mDeviceControl->setLedColor(toNative(c)))
        return ndk::ScopedAStatus::fromServiceSpecificError(ERROR_SYSFS_WRITE);
    return ndk::ScopedAStatus::ok();
}
```
