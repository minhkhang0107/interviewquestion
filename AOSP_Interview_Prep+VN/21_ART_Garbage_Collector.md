# Chương 21 — ART & Garbage Collector

> Dựa trên ART trong AOSP 14 (`art/runtime/gc/`). ART là module Mainline (`com.android.art`) nên hành vi GC trên máy có thể khác bản AOSP gốc tuỳ phiên bản module đã cập nhật. Kiểm tra lại property/hằng số trước khi trích dẫn.

## 21.1 Lịch sử GC trên Android (nói được trong 1 phút)

| Thời kỳ | Runtime / GC | Đặc điểm |
|---|---|---|
| ≤ 4.4 | Dalvik, mark-sweep | pause dài, log `GC_FOR_ALLOC`, `GC_CONCURRENT` |
| 5.0–7.x | ART, **Concurrent Mark-Sweep (CMS)** | phần lớn chạy song song; nền thì compact khi app vào background |
| 8.0+ | ART, **Concurrent Copying (CC)** | dựa trên **region**, dùng **read barrier**, compact song song → không phân mảnh, pause rất ngắn |
| 10+ | CC **generational** (young/sticky + full) | thu gom đối tượng trẻ thường xuyên, rẻ hơn |
| 14+ (qua ART Mainline, rộng hơn ở 15) | **CMC — Concurrent Mark-Compact** dùng `userfaultfd` | bỏ read barrier (code nhanh hơn, nhỏ hơn), cần kernel hỗ trợ `userfaultfd`; bật/tắt theo `ro.dalvik.vm.enable_uffd_gc` (có thể bị ghi đè bởi `persist.device_config.runtime_native_boot.enable_uffd_gc`) |

> ✅ **Trong project**: `out/.../product/etc/build.prop` có `ro.dalvik.vm.enable_uffd_gc=false` → box KSTB2500 **chạy Concurrent Copying (generational)**, không dùng CMC. Khi được hỏi "máy bạn dùng GC gì", đây là câu trả lời có bằng chứng.

Ý chính: **ART GC là tracing GC** (đi từ GC roots), không phải reference counting. Ngược lại, **Binder** và native `sp<>/wp<>` (`RefBase`) dùng **đếm tham chiếu** — một điểm đối chiếu hay được hỏi.

---

### Q1. GC roots gồm những gì? Thế nào là "rò rỉ" khi đã có GC?

GC roots: biến cục bộ/tham số trên **stack của mọi thread**, **static field** của class đã load, **JNI global reference**, monitor đang bị giữ, class loader, đối tượng được runtime giữ (ví dụ thread đang chạy).

Leak trong Java/Kotlin = đối tượng **không còn dùng nhưng vẫn reachable** từ root:
- `static` giữ `Activity`/`View`/`Context`.
- Listener/callback đăng ký vào singleton/system service mà không gỡ (`registerReceiver`, `LocationManager`, `DisplayManager.registerDisplayListener`...).
- Inner class/lambda non-static giữ `this` của Activity, `Handler` có message delay (message → handler → activity).
- Coroutine trong `GlobalScope`, thread chạy mãi giữ tham chiếu.
- Cache không giới hạn (`HashMap` tĩnh).

---

### Q2. Heap của ART chia thành những vùng nào?

- **Image space**: `boot.art` (class/đối tượng framework đã preload, map từ file), dùng chung giữa mọi process.
- **Zygote space**: đối tượng zygote tạo khi preload; sau fork, app chia sẻ qua **copy-on-write** → GC không compact/không giải phóng vùng này để giữ trang chia sẻ.
- **Region space** (main space của CC/CMC): chia heap thành các region (✅ `kRegionSize = 256 * KB` trong `region_space.h`); thu gom bằng cách copy/compact đối tượng sống sang region khác rồi giải phóng region cũ.
- **Large Object Space (LOS)**: mảng primitive/String lớn (✅ ngưỡng `kMinLargeObjectThreshold = 3 * kPageSize` = 12KB với trang 4KB) cấp riêng bằng mmap, không di chuyển.
- **Non-moving space**: đối tượng không được di chuyển (một số class, đối tượng runtime).

---

### Q3. Concurrent Copying (CC) chạy thế nào? Read barrier là gì?

1. **Pause ngắn (flip)**: dừng thread trong thời gian rất ngắn để chọn vùng thu gom (from-space) và quét root trên stack (mỗi thread tự lật root qua checkpoint).
2. **Copy song song**: GC thread đánh dấu và **copy** đối tượng sống từ from-space sang to-space trong khi app vẫn chạy.
3. **Read barrier (Baker)**: mỗi lần app **đọc một field tham chiếu**, code biên dịch kèm barrier kiểm tra: nếu đối tượng đang ở from-space thì trả về bản đã copy (hoặc copy ngay). Đảm bảo app không bao giờ thấy con trỏ cũ.
4. **Reclaim**: giải phóng toàn bộ region from-space.

Ưu: pause thường < 1ms, không phân mảnh (giảm OOM do fragmentation). Nhược: read barrier tốn chi phí mỗi lần đọc field → CMC ra đời để bỏ barrier này (dùng `userfaultfd` bắt page fault khi app chạm trang chưa compact xong).

**Generational**: phần lớn đối tượng chết trẻ → thường chỉ thu gom vùng mới cấp phát (young/sticky), đôi khi mới full GC.

---

### Q4. Đọc log GC trong logcat

```
I/art: Background concurrent copying GC freed 123456(6MB) AllocSpace objects, 12(2MB) LOS objects,
       49% free, 9MB/18MB, paused 312us,58us total 98.123ms
I/art: Alloc young concurrent copying GC freed ... paused 1.2ms total 25ms
W/art: WaitForGcToComplete blocked Alloc on HeapTrim for 28.4ms
```
Nguyên nhân GC (GC cause) cần biết:
| Cause | Ý nghĩa | Ảnh hưởng |
|---|---|---|
| `Background` | GC nền khi heap chạm ngưỡng | ít ảnh hưởng |
| `Alloc` | cấp phát **thất bại**, thread đang cấp phát phải chờ GC | **gây jank** nếu xảy ra trên main/RenderThread |
| `Explicit` | gọi `System.gc()`/`Runtime.gc()` | tránh trong code app |
| `NativeAlloc` | native memory (vd. Bitmap) tăng nhanh qua `NativeAllocationRegistry` | thấy nhiều khi decode ảnh liên tục |
| `CollectorTransition` / `HomogeneousSpaceCompact` | app chuyển sang background → compact giảm bộ nhớ | |
| `Trim` / `HeapTrim` | trả bộ nhớ cho kernel | |

Chỉ số cần đọc: `paused` (thời gian dừng app — quan trọng cho jank), `total`, `% free`, `used/footprint`.

---

### Q5. Heap size của app được quyết định bởi gì? Tuning cho STB RAM thấp?

Property (đặt trong `device.mk`/`PRODUCT_PROPERTY_OVERRIDES`, thường lấy từ `frameworks/native/build/phone-xhdpi-2048-dalvik-heap.mk` hoặc bản tương ứng):
| Property | Ý nghĩa |
|---|---|
| `dalvik.vm.heapstartsize` | heap ban đầu |
| `dalvik.vm.heapgrowthlimit` | giới hạn heap Java app thường (`ActivityManager.getMemoryClass()`) |
| `dalvik.vm.heapsize` | giới hạn khi app khai báo `android:largeHeap="true"` (`getLargeMemoryClass()`) |
| `dalvik.vm.heaptargetutilization` | tỉ lệ sử dụng mục tiêu sau GC (0.75 → heap còn trống 25%) |
| `dalvik.vm.heapminfree` / `heapmaxfree` | khoảng trống tối thiểu/tối đa sau GC |

✅ **Giá trị thực tế của KSTB2500** (`out/.../vendor/build.prop`): nền là `frameworks/native/build/tablet-7in-hdpi-1024-dalvik-heap.mk` (gọi từ `device.mk`), sau đó `device/amlogic/common/product_property.mk` ghi đè:
| Property | Giá trị |
|---|---|
| `heapstartsize` | 8m |
| `heapgrowthlimit` | **256m** (mặc định 80m bị ghi đè; nếu `PRODUCT_SUPPORT_4K_UI=true` thì 384m) |
| `heapsize` | 384m |
| `heaptargetutilization` | 0.75 |
| `heapminfree` / `heapmaxfree` | 512k / 8m |

Kèm lmkd: `ro.lmk.kill_heaviest_task=true`, `ro.lmk.kill_timeout_ms=100`, `ro.lmk.use_new_strategy=true`. Đặt `heapgrowthlimit` quá cao → app tham bộ nhớ, **lmkd** kill nhiều hơn; quá thấp → OOM trong app 4K UI/Launcher. Thay đổi cần test với app thật (Netflix, YouTube, Launcher).

---

### Q6. `OutOfMemoryError` trong app khi máy vẫn còn RAM — vì sao?

```
java.lang.OutOfMemoryError: Failed to allocate a 33177612 byte allocation with 12582912 free bytes
and 24MB until OOM, target footprint 201326592, growth limit 201326592
```
- Giới hạn là **Java heap của app** (`growth limit`), không phải RAM máy.
- Thường do: decode Bitmap lớn (ảnh 4K poster: 3840×2160×4 ≈ 33MB), cache ảnh không giới hạn, leak Activity.
- Từ Android 8, **pixel Bitmap nằm ở native heap**, nhưng vẫn được "tính" cho GC qua `NativeAllocationRegistry` → GC `NativeAlloc` chạy thường xuyên; OOM kiểu mới thường là native/`lmkd` kill thay vì Java OOM.
- Khắc phục: `inSampleSize`/Glide/Coil với kích thước view, `HARDWARE` bitmap config, `LruCache` theo byte, gỡ leak.

---

### Q7. Soft/Weak/Phantom reference và finalizer?

- **Strong**: bình thường.
- **SoftReference**: bị thu gom khi bộ nhớ căng — Android **khuyên không dùng** làm cache (ART thu gom soft ref khá sớm); dùng `LruCache`.
- **WeakReference**: thu gom ở lần GC tiếp theo nếu chỉ còn weak ref; dùng để tránh leak (ví dụ `Handler` tĩnh + `WeakReference<Activity>`).
- **PhantomReference** + `ReferenceQueue`: biết khi đối tượng đã bị thu gom để dọn tài nguyên native; nền của `Cleaner`/`NativeAllocationRegistry`.
- **`finalize()`**: chạy trên `FinalizerDaemon`; `FinalizerWatchdogDaemon` giết process nếu một `finalize()` chạy quá **10s** (✅ `MAX_FINALIZE_NANOS` trong `libcore/libart/.../Daemons.java`) → crash `TimeoutException: ...finalize() timed out after 10 seconds`. Không nên dùng finalizer.

---

### Q8. Những "bộ nhớ" mà GC không quản lý (hay bị hỏi khi debug memory system)

- **JNI global ref**: tạo bằng `NewGlobalRef` mà không `DeleteGlobalRef` → rò rỉ, tới giới hạn (✅ `kGlobalsMax = 51200` trong `java_vm_ext.cc`) gây abort `JNI ERROR (app bug): global reference table overflow (max=51200)`.
- **BinderProxy**: mỗi binder object nhận từ process khác tạo một `BinderProxy`; system_server theo dõi số proxy theo uid và **kill app** gây ra quá nhiều (✅ `BINDER_PROXY_HIGH_WATERMARK = 6000`, `LOW = 5500` trong `ActivityManagerService.java`) để bảo vệ hệ thống.
- **Native heap** (malloc của thư viện C/C++), **graphics** (gralloc buffer, texture), **dmabuf** (video, codec) — xem bằng `dumpsys meminfo`, `dmabuf_dump`, Perfetto `heapprofd`.
- **Thread**: mỗi thread có stack ~1MB virtual; tạo thread không giới hạn → `pthread_create failed` OOM dù heap còn.

---

### Q9. Công cụ phân tích bộ nhớ

```bash
adb shell dumpsys meminfo <package|pid>         # Java Heap, Native Heap, Graphics, Code, Stack, TOTAL PSS/RSS
adb shell am dumpheap <pid> /data/local/tmp/app.hprof    # heap dump Java
adb pull /data/local/tmp/app.hprof && hprof-conv app.hprof app-conv.hprof   # để mở bằng MAT
adb shell cmd activity send-trim-memory <pkg> RUNNING_LOW  # mô phỏng áp lực bộ nhớ
adb shell dumpsys procstats --hours 3            # bộ nhớ trung bình theo process
```
- Android Studio Memory Profiler / LeakCanary (Chương 11 Q41).
- Perfetto: `java_hprof` (heap graph), `heapprofd` (native malloc theo callstack), track bộ nhớ RSS/swap, sự kiện `lmkd` kill.
- `showmap <pid>`, `procrank`/`librank` (userdebug) để xem PSS theo mapping.

---

### Q10. GC ảnh hưởng performance như thế nào và cách giảm?

- Jank: GC cause `Alloc` trên main/RenderThread, hoặc CPU bị GC thread chiếm trên SoC ít nhân (STB 4×A55).
- Giảm cấp phát trên đường nóng: không tạo object trong `onDraw`, `onBindViewHolder`, vòng lặp animation; tái sử dụng `Paint`, `Rect`, `StringBuilder`; tránh autoboxing (`List<Int>` → `IntArray`), tránh lambda capture tạo đối tượng mỗi frame.
- Kotlin: `inline` function tránh tạo lambda object; `Sequence` vs `List` (ít collection trung gian).
- Đo bằng Perfetto: slice GC (`concurrent copying GC`), `HeapTaskDaemon`, đồ thị heap size theo thời gian.

---

### Q11. Zygote, preload và GC liên quan gì đến tốc độ mở app và RAM của box?

- Zygote preload class (`preloaded-classes`) và resource, rồi chạy GC trước khi fork để heap gọn → app con chia sẻ trang qua COW.
- App ghi vào đối tượng trong zygote space → trang bị copy → mất phần chia sẻ (vì vậy GC không di chuyển zygote space).
- Boot image (`boot.art`, `boot.oat`) do `dexpreopt` tạo lúc build — giảm thời gian khởi động & first launch.
- Profile-guided compilation: `dex2oat` dùng profile (cloud profile / baseline profile) → compile đoạn code nóng; liên quan trực tiếp tới jank lần mở đầu.

---

## 21.2 Rapid-fire

1. **Android GC có phải stop-the-world?** → Chỉ có pause rất ngắn (flip/root scan); phần lớn chạy song song.
2. **`System.gc()` có đảm bảo GC chạy?** → Không, chỉ là gợi ý; ART thậm chí có thể bỏ qua.
3. **Read barrier là gì?** → Code chèn khi đọc field tham chiếu để trả về địa chỉ mới của đối tượng đã được di chuyển (CC).
4. **Vì sao moving GC giúp giảm OOM?** → Compact → không phân mảnh, luôn có vùng liên tục cho đối tượng lớn.
5. **Bitmap nằm ở đâu từ A8?** → Native heap, được đăng ký với GC qua `NativeAllocationRegistry`; `HARDWARE` bitmap nằm ở bộ nhớ đồ hoạ.
6. **`largeHeap` có nên bật?** → Chỉ khi thật cần (vd. Launcher 4K); nó làm app nặng hơn và dễ bị lmkd kill các app khác.
7. **`onTrimMemory` dùng làm gì?** → Hệ thống báo áp lực bộ nhớ/app vào background (`TRIM_MEMORY_UI_HIDDEN`, `TRIM_MEMORY_BACKGROUND`) → app giải phóng cache.
8. **Binder object được giải phóng khi nào?** → Đếm tham chiếu strong/weak trong driver; proxy phía client bị GC → giảm ref; hết ref thì đối tượng phía server mới được thu gom.
