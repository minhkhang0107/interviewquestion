# Chapter 21 — ART & the Garbage Collector

> Based on ART in AOSP 14 (`art/runtime/gc/`). ART is a Mainline module (`com.android.art`), so GC behavior on a device may differ from plain AOSP depending on the module version installed. Verify properties/constants before quoting them.

## 21.1 GC history on Android (say it in 1 minute)

| Era | Runtime / GC | Characteristics |
|---|---|---|
| ≤ 4.4 | Dalvik, mark-sweep | long pauses, logs `GC_FOR_ALLOC`, `GC_CONCURRENT` |
| 5.0–7.x | ART, **Concurrent Mark-Sweep (CMS)** | mostly concurrent; compacts when the app goes to background |
| 8.0+ | ART, **Concurrent Copying (CC)** | **region**-based, uses a **read barrier**, concurrent compaction → no fragmentation, very short pauses |
| 10+ | **generational** CC (young/sticky + full) | frequent cheap collections of young objects |
| 14+ (via ART Mainline, broader in 15) | **CMC — Concurrent Mark-Compact** using `userfaultfd` | drops the read barrier (faster, smaller code), needs kernel `userfaultfd` support; toggled by `ro.dalvik.vm.enable_uffd_gc` (can be overridden by `persist.device_config.runtime_native_boot.enable_uffd_gc`) |

> ✅ **In the project**: `out/.../product/etc/build.prop` has `ro.dalvik.vm.enable_uffd_gc=false` → the KSTB2500 box **runs (generational) Concurrent Copying**, not CMC. If asked "which GC does your device use", this is the evidence-backed answer.

Key point: **ART's GC is a tracing GC** (from GC roots), not reference counting. By contrast, **Binder** and native `sp<>/wp<>` (`RefBase`) use **reference counting** — a contrast interviewers like.

---

### Q1. What are the GC roots? What does a "leak" mean when there is a GC?

GC roots: locals/arguments on **every thread's stack**, **static fields** of loaded classes, **JNI global references**, held monitors, class loaders, objects held by the runtime (e.g. running threads).

A leak in Java/Kotlin = an object that is **no longer used but still reachable** from a root:
- A `static` holding an `Activity`/`View`/`Context`.
- Listeners/callbacks registered with a singleton/system service and never removed (`registerReceiver`, `LocationManager`, `DisplayManager.registerDisplayListener`...).
- Non-static inner classes/lambdas capturing the Activity's `this`, a `Handler` with delayed messages (message → handler → activity).
- Coroutines in `GlobalScope`, long-running threads holding references.
- Unbounded caches (a static `HashMap`).

---

### Q2. Which spaces make up the ART heap?

- **Image space**: `boot.art` (preloaded framework classes/objects, mapped from a file), shared by all processes.
- **Zygote space**: objects created by zygote during preload; after fork, apps share them via **copy-on-write** → the GC does not compact/free this space to keep pages shared.
- **Region space** (CC/CMC main space): the heap is split into regions (✅ `kRegionSize = 256 * KB` in `region_space.h`); collection copies/compacts live objects into other regions and frees the old ones.
- **Large Object Space (LOS)**: large primitive arrays/Strings (✅ threshold `kMinLargeObjectThreshold = 3 * kPageSize` = 12KB with 4KB pages) allocated separately via mmap, never moved.
- **Non-moving space**: objects that must not move (some classes, runtime objects).

---

### Q3. How does Concurrent Copying (CC) work? What is a read barrier?

1. **Short pause (flip)**: threads are paused very briefly to pick the collection set (from-space) and scan stack roots (each thread flips its roots at a checkpoint).
2. **Concurrent copy**: GC threads mark and **copy** live objects from from-space to to-space while the app keeps running.
3. **Read barrier (Baker)**: every time the app **reads a reference field**, compiled code runs a barrier: if the object is in from-space, return the copied version (or copy it now). The app never sees a stale pointer.
4. **Reclaim**: free all from-space regions.

Pros: pauses usually < 1ms, no fragmentation (fewer fragmentation OOMs). Cons: the read barrier costs something on every field read → CMC was introduced to remove it (using `userfaultfd` to trap page faults when the app touches a page not yet compacted).

**Generational**: most objects die young → usually only the recently allocated area (young/sticky) is collected, with occasional full GCs.

---

### Q4. Reading GC logs in logcat

```
I/art: Background concurrent copying GC freed 123456(6MB) AllocSpace objects, 12(2MB) LOS objects,
       49% free, 9MB/18MB, paused 312us,58us total 98.123ms
I/art: Alloc young concurrent copying GC freed ... paused 1.2ms total 25ms
W/art: WaitForGcToComplete blocked Alloc on HeapTrim for 28.4ms
```
GC causes you should know:
| Cause | Meaning | Impact |
|---|---|---|
| `Background` | background GC when the heap reaches its threshold | low |
| `Alloc` | an allocation **failed**; the allocating thread must wait for GC | **causes jank** on main/RenderThread |
| `Explicit` | `System.gc()`/`Runtime.gc()` was called | avoid in app code |
| `NativeAlloc` | native memory (e.g. Bitmaps) grows quickly via `NativeAllocationRegistry` | common when decoding images continuously |
| `CollectorTransition` / `HomogeneousSpaceCompact` | app moved to background → compact to save memory | |
| `Trim` / `HeapTrim` | return memory to the kernel | |

Numbers to read: `paused` (time the app is stopped — what matters for jank), `total`, `% free`, `used/footprint`.

---

### Q5. What determines an app's heap size? Tuning for a low-RAM STB?

Properties (set in `device.mk`/`PRODUCT_PROPERTY_OVERRIDES`, usually from `frameworks/native/build/phone-xhdpi-2048-dalvik-heap.mk` or an equivalent):
| Property | Meaning |
|---|---|
| `dalvik.vm.heapstartsize` | initial heap |
| `dalvik.vm.heapgrowthlimit` | Java heap limit for normal apps (`ActivityManager.getMemoryClass()`) |
| `dalvik.vm.heapsize` | limit for apps declaring `android:largeHeap="true"` (`getLargeMemoryClass()`) |
| `dalvik.vm.heaptargetutilization` | target utilization after GC (0.75 → 25% free) |
| `dalvik.vm.heapminfree` / `heapmaxfree` | min/max free space after GC |

✅ **Actual KSTB2500 values** (`out/.../vendor/build.prop`): the base is `frameworks/native/build/tablet-7in-hdpi-1024-dalvik-heap.mk` (pulled in by `device.mk`), then `device/amlogic/common/product_property.mk` overrides it:
| Property | Value |
|---|---|
| `heapstartsize` | 8m |
| `heapgrowthlimit` | **256m** (the 80m default is overridden; 384m if `PRODUCT_SUPPORT_4K_UI=true`) |
| `heapsize` | 384m |
| `heaptargetutilization` | 0.75 |
| `heapminfree` / `heapmaxfree` | 512k / 8m |

Together with lmkd: `ro.lmk.kill_heaviest_task=true`, `ro.lmk.kill_timeout_ms=100`, `ro.lmk.use_new_strategy=true`. A `heapgrowthlimit` too high → greedy apps, **lmkd** kills more; too low → OOMs in 4K UIs/Launchers. Any change must be tested with real apps (Netflix, YouTube, Launcher).

---

### Q6. Why does an app hit `OutOfMemoryError` while the device still has free RAM?

```
java.lang.OutOfMemoryError: Failed to allocate a 33177612 byte allocation with 12582912 free bytes
and 24MB until OOM, target footprint 201326592, growth limit 201326592
```
- The limit is the **app's Java heap** (`growth limit`), not device RAM.
- Usual causes: decoding large Bitmaps (a 4K poster: 3840×2160×4 ≈ 33MB), unbounded image caches, leaked Activities.
- Since Android 8 **Bitmap pixels live in the native heap**, but are still "accounted" to the GC via `NativeAllocationRegistry` → frequent `NativeAlloc` GCs; modern OOMs are more often native/`lmkd` kills than Java OOMs.
- Fixes: `inSampleSize`/Glide/Coil sized to the view, `HARDWARE` bitmap config, byte-bounded `LruCache`, fix leaks.

---

### Q7. Soft/Weak/Phantom references and finalizers?

- **Strong**: normal.
- **SoftReference**: collected under memory pressure — Android **advises against** using them for caches (ART clears soft refs fairly eagerly); use `LruCache`.
- **WeakReference**: collected at the next GC if only weak refs remain; used to avoid leaks (e.g. a static `Handler` + `WeakReference<Activity>`).
- **PhantomReference** + `ReferenceQueue`: tells you an object has been collected so you can free native resources; the basis of `Cleaner`/`NativeAllocationRegistry`.
- **`finalize()`**: runs on the `FinalizerDaemon`; the `FinalizerWatchdogDaemon` kills the process if a `finalize()` takes longer than **10s** (✅ `MAX_FINALIZE_NANOS` in `libcore/libart/.../Daemons.java`) → crash `TimeoutException: ...finalize() timed out after 10 seconds`. Avoid finalizers.

---

### Q8. Memory the GC does not manage (common in system memory debugging)

- **JNI global refs**: created with `NewGlobalRef` and never `DeleteGlobalRef`'d → leak; at the limit (✅ `kGlobalsMax = 51200` in `java_vm_ext.cc`) you get the abort `JNI ERROR (app bug): global reference table overflow (max=51200)`.
- **BinderProxy**: each binder object received from another process creates a `BinderProxy`; system_server tracks proxy counts per uid and **kills an app** that creates too many (✅ `BINDER_PROXY_HIGH_WATERMARK = 6000`, `LOW = 5500` in `ActivityManagerService.java`) to protect the system.
- **Native heap** (malloc from C/C++ libraries), **graphics** (gralloc buffers, textures), **dmabuf** (video, codecs) — inspect with `dumpsys meminfo`, `dmabuf_dump`, Perfetto `heapprofd`.
- **Threads**: each thread has ~1MB of virtual stack; unbounded thread creation → `pthread_create failed` OOM even with free heap.

---

### Q9. Memory analysis tools

```bash
adb shell dumpsys meminfo <package|pid>         # Java Heap, Native Heap, Graphics, Code, Stack, TOTAL PSS/RSS
adb shell am dumpheap <pid> /data/local/tmp/app.hprof    # Java heap dump
adb pull /data/local/tmp/app.hprof && hprof-conv app.hprof app-conv.hprof   # to open in MAT
adb shell cmd activity send-trim-memory <pkg> RUNNING_LOW  # simulate memory pressure
adb shell dumpsys procstats --hours 3            # average memory per process
```
- Android Studio Memory Profiler / LeakCanary (Chapter 11 Q41).
- Perfetto: `java_hprof` (heap graph), `heapprofd` (native malloc by callstack), RSS/swap tracks, `lmkd` kill events.
- `showmap <pid>`, `procrank`/`librank` (userdebug) for PSS per mapping.

---

### Q10. How does GC affect performance, and how do you reduce it?

- Jank: `Alloc` GCs on main/RenderThread, or GC threads hogging CPU on few-core SoCs (STB 4×A55).
- Reduce allocation on hot paths: no object creation in `onDraw`, `onBindViewHolder`, animation loops; reuse `Paint`, `Rect`, `StringBuilder`; avoid autoboxing (`List<Int>` → `IntArray`), avoid capturing lambdas allocated per frame.
- Kotlin: `inline` functions avoid lambda objects; `Sequence` vs `List` (fewer intermediate collections).
- Measure with Perfetto: GC slices (`concurrent copying GC`), `HeapTaskDaemon`, heap size over time.

---

### Q11. How do Zygote, preload and GC relate to app launch speed and box RAM?

- Zygote preloads classes (`preloaded-classes`) and resources, then runs a GC before forking so the heap is compact → children share pages via COW.
- An app writing to objects in zygote space forces a page copy → loses sharing (hence the GC never moves zygote space).
- The boot image (`boot.art`, `boot.oat`) is produced by `dexpreopt` at build time — shortens boot & first launch.
- Profile-guided compilation: `dex2oat` uses profiles (cloud/baseline profiles) to compile hot code; directly related to first-launch jank.

---

## 21.2 Rapid-fire

1. **Is Android's GC stop-the-world?** → Only very short pauses (flip/root scan); most work is concurrent.
2. **Does `System.gc()` guarantee a GC?** → No, it is only a hint; ART may even ignore it.
3. **What is a read barrier?** → Code inserted on reference field reads that returns the new address of a moved object (CC).
4. **Why does a moving GC reduce OOMs?** → Compaction → no fragmentation, contiguous space is always available for large objects.
5. **Where do Bitmaps live since A8?** → Native heap, registered with the GC via `NativeAllocationRegistry`; `HARDWARE` bitmaps live in graphics memory.
6. **Should you enable `largeHeap`?** → Only when truly needed (e.g. a 4K Launcher); it makes the app heavier and gets other apps killed by lmkd.
7. **What is `onTrimMemory` for?** → The system signals memory pressure/app moving to background (`TRIM_MEMORY_UI_HIDDEN`, `TRIM_MEMORY_BACKGROUND`) → the app frees caches.
8. **When is a Binder object freed?** → Strong/weak reference counts in the driver; when the client-side proxy is GC'd the ref drops; only when no refs remain can the server-side object be collected.
