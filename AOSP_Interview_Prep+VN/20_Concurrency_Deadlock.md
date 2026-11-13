# Chương 20 — Đồng thời & Deadlock (app, system_server, binder, native)

> Dựa trên AOSP 14 (`frameworks/base/services/core/java/com/android/server/Watchdog.java`, ART monitor). Hằng số/timeout nên kiểm tra lại trong tree của project trước khi trích dẫn.

## 20.1 Khái niệm cần nói được trong 30 giây

- **Deadlock**: các thread chờ nhau vòng tròn, không ai tiến. Cần đủ 4 điều kiện Coffman: *mutual exclusion*, *hold and wait*, *no preemption*, *circular wait* → phá **một** điều kiện là hết (thực tế: phá *circular wait* bằng **thứ tự lock**).
- **Livelock**: thread vẫn chạy nhưng không tiến (cùng retry, cùng nhường).
- **Starvation**: thread không bao giờ lấy được tài nguyên (ví dụ binder thread pool bị chiếm hết).
- **Priority inversion**: thread ưu tiên cao chờ lock do thread ưu tiên thấp giữ, thread trung bình chiếm CPU. Binder driver có **priority inheritance** cho transaction; `synchronized` của Java **không** có.

---

### Q1. Viết một ví dụ deadlock kinh điển và cách sửa?

```kotlin
val a = Any(); val b = Any()
thread { synchronized(a) { Thread.sleep(10); synchronized(b) { } } }   // A → B
thread { synchronized(b) { Thread.sleep(10); synchronized(a) { } } }   // B → A  → deadlock
```
Cách sửa:
1. **Thứ tự lock toàn cục**: mọi chỗ đều lấy `a` trước `b`.
2. **Thu nhỏ vùng lock**: copy dữ liệu ra khỏi lock rồi xử lý.
3. `ReentrantLock.tryLock(timeout)` → phát hiện và lùi lại (cẩn thận livelock).
4. **Thread confinement**: dồn mọi thay đổi state vào một `Handler`/single-thread executor → không cần lock.

---

### Q2. Deadlock trong system_server thường xảy ra thế nào?

system_server có vài lock "lớn":
- `ActivityManagerService` (`synchronized (this)` — "AM lock"), `mProcLock` (✅ kiểu `ActivityManagerGlobalLock`, A12+ tách bớt khỏi AM lock, bật bằng `ENABLE_PROC_LOCK`).
- `WindowManagerGlobalLock mGlobalLock` — **dùng chung bởi WMS và ATMS**.
- `PackageManagerService.mLock`, `mInstallLock`; `PowerManagerService.mLock`; lock riêng của từng service OEM.

Các mẫu deadlock điển hình:
1. **Ngược thứ tự lock giữa service**: service X giữ `mLock` gọi vào AMS (cần AM lock), trong khi AMS giữ AM lock gọi callback vào X (cần `mLock`).
2. **Gọi ra ngoài khi đang giữ lock**: gọi binder sang app/HAL, gọi listener, gửi broadcast **đồng bộ** từ trong `synchronized`.
3. **Đợi main/handler thread trong khi handler thread đó đang chờ lock của mình** (`Handler.runWithScissors`, `CountDownLatch.await`).

Quy ước của AOSP để tránh:
- Hàm có hậu tố **`...Locked()`** = caller phải giữ lock; hàm không hậu tố tự lấy lock.
- Chú thích `@GuardedBy("mLock")` (có checker Error Prone khi build).
- **Không gọi ra ngoài (binder, callback, service khác) khi đang giữ lock của mình** — gom dữ liệu trong lock, gọi sau khi thả lock.
- Callback tới app qua `RemoteCallbackList` + interface **`oneway`** để app chậm không chặn được system_server.

---

### Q3. Binder có gây deadlock không? Re-entrancy hoạt động thế nào?

- Gọi binder **đồng bộ** = thread gọi block tới khi server trả lời.
- **Re-entrancy**: A (thread T1) gọi B, B trong lúc xử lý lại gọi ngược A → driver binder giao transaction ngược này cho **chính T1** đang chờ (vì cùng chuỗi transaction), không phải thread khác của A. Nhờ vậy `synchronized` (reentrant) của T1 không tự khoá.
- Nhưng deadlock vẫn xảy ra khi:
  - Callback ngược được A xử lý ở **thread khác** (ví dụ post sang Handler rồi chờ kết quả), trong khi T1 đang giữ lock.
  - **Thread pool exhaustion**: tất cả binder thread của server (✅ `DEFAULT_MAX_BINDER_THREADS 15` trong `ProcessState.cpp`, `sMaxBinderThreads = 31` trong `SystemServer.java`) đều đang chờ một lock do thread khác giữ, thread này lại đang chờ một binder call vào chính server đó → không còn thread nào nhận → treo. Dấu hiệu log: `binder thread pool (N threads) starved for X ms` (✅ `IPCThreadState.cpp` chỉ log khi bị đói > 100ms).
  - Hai process gọi đồng bộ vào nhau từ hai chuỗi khác nhau và mỗi bên giữ lock.
- Giải pháp: `oneway` cho callback/notify, không giữ lock khi gọi binder, tách xử lý nặng sang Handler thread, giới hạn thời gian chờ.

---

### Q4. Watchdog của system_server phát hiện deadlock thế nào?

- `Watchdog` là thread riêng, mỗi chu kỳ gửi một "ping" tới các `HandlerChecker`. ✅ Source: `DEFAULT_TIMEOUT = DB ? 10s : 60s`, chu kỳ kiểm tra = timeout / 2 (30s), có thể đổi qua `Settings.Global.WATCHDOG_TIMEOUT_MILLIS`; monitor chạy trên thread `watchdog.monitor`.
  - Kiểm tra các **Looper quan trọng** (foreground thread, main thread, ui, io, display, animation...) — post message và xem có được xử lý không.
  - Gọi `monitor()` của các `Watchdog.Monitor` đã đăng ký: hàm này chỉ đơn giản `synchronized (mLock) {}` → nếu lock bị giữ mãi, monitor bị kẹt.
- Quá nửa thời gian → dump stack (`/data/anr/`); quá hạn → log `*** WATCHDOG KILLING SYSTEM PROCESS: Blocked in monitor com.android.server.am.ActivityManagerService on monitor thread (watchdog.monitor)` hoặc `Blocked in handler on foreground thread`, rồi **kill system_server** → zygote khởi động lại toàn bộ framework (**soft reboot**).
- Service OEM nên đăng ký:
```java
Watchdog.getInstance().addMonitor(this);           // implements Watchdog.Monitor
@Override public void monitor() { synchronized (mLock) { } }
Watchdog.getInstance().addThread(mHandler);          // nếu có Handler thread riêng quan trọng
```

---

### Q5. Đọc trace deadlock trong `/data/anr/` như thế nào?

```
"Binder:1234_5" prio=5 tid=88 Blocked
  | group="main" sCount=1 ucsCount=0 flags=1 obj=0x... self=0x...
  at com.android.server.am.ActivityManagerService.broadcastIntentWithFeature(ActivityManagerService.java:...)
  - waiting to lock <0x0a1b2c3d> (a com.android.server.am.ActivityManagerService) held by thread 120
  ...
"android.fg" prio=5 tid=120 Blocked
  at com.vendor.server.LedService.onStateChanged(LedService.java:88)
  - waiting to lock <0x0f0e0d0c> (a java.lang.Object) held by thread 88
```
Quy trình:
1. Tìm thread có trạng thái `Blocked` và dòng `waiting to lock <X> held by thread N`.
2. Đi theo chuỗi `held by` → nếu quay về thread ban đầu ⇒ **vòng deadlock**.
3. Nếu chuỗi kết thúc ở thread trạng thái `Native` trong `BinderProxy.transactNative` / `IPCThreadState::talkWithDriver` → đang chờ **binder call ra process khác** → xem process đích (trace của nó cũng nằm trong cùng file ANR/watchdog nếu có, hoặc dùng `debuggerd -b <pid>`).
4. Thread `Waiting`/`TimedWaiting` trên `Object.wait`, `CountDownLatch` → đang chờ một sự kiện, kiểm tra ai phải `notify`.

Công cụ:
```bash
adb shell kill -3 <pid>                         # dump Java stack của app/system_server
adb shell debuggerd -b <pid>                     # stack native (cả Java nếu là app)
adb shell cat /dev/binderfs/binder_logs/transactions   # hoặc /sys/kernel/debug/binder/transactions (máy cũ)
adb shell dumpsys activity processes | grep -i frozen   # process bị freeze cũng làm binder chờ
adb logcat | grep -i "Long monitor contention"   # ART log khi tranh chấp lock lâu
```
Perfetto: bật category `dalvik` → thấy slice `monitor contention with owner <thread> (<tid>) at <method>(<file>:<line>)` (✅ chuỗi trong `art/runtime/monitor.cc`), và slice `binder transaction`/`binder reply` nối giữa các process.

---

### Q6. Deadlock ở main thread của app (dẫn tới ANR)

Ví dụ:
```kotlin
// UI thread
val latch = CountDownLatch(1)
handler.post { /* chạy trên CHÍNH main looper */ latch.countDown() }
latch.await()          // main chờ một message mà chỉ main mới xử lý được → treo vĩnh viễn → ANR
```
Các biến thể:
- `runBlocking { withContext(Dispatchers.Main) { ... } }` gọi từ main thread.
- `Future.get()` trên main chờ task cần main.
- `synchronized` giữa main và worker, trong đó worker gọi `runOnUiThread` rồi chờ kết quả.
- ContentProvider/`SharedPreferences.commit()` chậm không phải deadlock nhưng cũng gây ANR (Chương 08).

---

### Q7. Deadlock trong native/HAL (C++)

Mẫu kinh điển trong HAL có callback:
```cpp
// SAI: gọi callback khi đang giữ mutex
void Hal::onHwEvent(int v) {
    std::lock_guard<std::mutex> lk(mMutex);
    for (auto& cb : mCallbacks) cb->onEvent(v);     // client có thể gọi ngược setX() → cần mMutex → deadlock
}
// ĐÚNG: copy danh sách trong lock, gọi ngoài lock; callback interface khai báo oneway
void Hal::onHwEvent(int v) {
    std::vector<std::shared_ptr<ICallback>> cbs;
    { std::lock_guard<std::mutex> lk(mMutex); cbs = mCallbacks; }
    for (auto& cb : cbs) cb->onEvent(v);
}
```
- `std::mutex` **không reentrant** → cùng thread lock 2 lần là tự deadlock; `std::recursive_mutex` chỉ nên là giải pháp cuối.
- Dùng `std::scoped_lock(m1, m2)` để lấy nhiều mutex không lo thứ tự.
- Clang Thread Safety Analysis (`GUARDED_BY`, `REQUIRES`) — AOSP bật `-Wthread-safety` ở nhiều module.
- Debug: `debuggerd -b <pid>` thấy nhiều thread ở `__futex_wait_ex`/`pthread_mutex_lock`.

---

### Q8. Coroutines có deadlock không?

- `Mutex` của kotlinx.coroutines **không reentrant**: `mutex.withLock { mutex.withLock { } }` → treo.
- `runBlocking` trên thread mà coroutine bên trong cần (main, hoặc dispatcher 1 thread) → treo.
- `Dispatchers.IO` bị chiếm hết 64 thread bởi việc blocking chờ nhau → starvation.
- Khuyến nghị: dùng `Mutex`/`Channel`/actor hoặc `limitedParallelism(1)` để tuần tự hoá thay vì `synchronized`; không gọi suspend function bên trong `synchronized` (không được phép biên dịch với lambda suspend, và giữ lock qua điểm suspend là sai thiết kế).

---

### Q9. Tình huống: "Sau khi thêm LedService vào system_server, thỉnh thoảng box soft-reboot khi bấm nút nguồn." Bạn xử lý thế nào?

1. Lấy bugreport → tìm `WATCHDOG KILLING SYSTEM PROCESS` và file trace tương ứng trong `FS/data/anr/`.
2. Xác định thread bị kẹt (ví dụ `PowerManagerService` handler) → đi theo chuỗi `held by`.
3. Giả sử kết quả: `PowerManagerService` (giữ `mLock` của Power) gọi listener `LedService.onWakefulnessChanged()` **đồng bộ**; `LedService` giữ `mLock` của mình và gọi HAL; HAL bị chậm 70s do sysfs block, hoặc LedService gọi `PowerManager.isInteractive()` → cần `mLock` của Power ⇒ vòng.
4. Sửa:
   - Listener của Power chỉ **post** sang Handler của LedService, không xử lý trực tiếp.
   - Gọi HAL ngoài lock, HAL interface dạng `oneway` hoặc có timeout; HAL không block vô hạn trên sysfs.
   - `addMonitor` cho LedService để nếu treo thì stack chỉ rõ thủ phạm.
5. Thêm test: stress bấm nguồn liên tục + `am hang`/monkey; theo dõi `Long monitor contention`.

---

## 20.2 Rapid-fire

1. **`synchronized` có reentrant không?** → Có (Java monitor); `std::mutex` và coroutine `Mutex` thì không.
2. **`volatile` có thay được lock?** → Chỉ đảm bảo visibility/ordering, không atomic cho `count++`; dùng `AtomicInteger`.
3. **`ConcurrentHashMap` có chống deadlock?** → Không liên quan; chỉ giảm tranh chấp. Tính toán phức tạp trong `computeIfAbsent` gọi lại map có thể gây lỗi/treo.
4. **Watchdog kill thì app thấy gì?** → Toàn bộ framework khởi động lại (màn hình boot animation), mọi app bị kill; không phải kernel reboot.
5. **"Binder thread pool starved" nghĩa là?** → Mọi binder thread đang bận → request mới phải chờ; thường do giữ lock/IO lâu trên binder thread.
6. **Vì sao callback nên `oneway`?** → Server không phải chờ client, client chậm/treo không kéo server theo; đổi lại không có giá trị trả về và không có exception.
7. **`Handler.runWithScissors` có nguy hiểm?** → Có: chặn thread gọi cho tới khi handler xử lý xong; nếu handler thread đang chờ thread gọi → deadlock. Chỉ dùng trong khởi tạo framework có kiểm soát.
