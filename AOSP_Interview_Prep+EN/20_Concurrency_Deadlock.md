# Chapter 20 — Concurrency & Deadlock (app, system_server, binder, native)

> Based on AOSP 14 (`frameworks/base/services/core/java/com/android/server/Watchdog.java`, ART monitors). Verify constants/timeouts in the project tree before quoting them.

## 20.1 Concepts you should explain in 30 seconds

- **Deadlock**: threads wait on each other in a cycle and none progresses. Requires all 4 Coffman conditions: *mutual exclusion*, *hold and wait*, *no preemption*, *circular wait* → break **one** and it is gone (in practice: break *circular wait* with a **lock order**).
- **Livelock**: threads keep running but make no progress (retrying/yielding in lockstep).
- **Starvation**: a thread never gets the resource (e.g. the binder thread pool is fully occupied).
- **Priority inversion**: a high-priority thread waits for a lock held by a low-priority thread while a medium-priority thread hogs the CPU. The binder driver has **priority inheritance** for transactions; Java `synchronized` does **not**.

---

### Q1. Write a classic deadlock and fix it

```kotlin
val a = Any(); val b = Any()
thread { synchronized(a) { Thread.sleep(10); synchronized(b) { } } }   // A → B
thread { synchronized(b) { Thread.sleep(10); synchronized(a) { } } }   // B → A  → deadlock
```
Fixes:
1. **Global lock order**: everyone takes `a` before `b`.
2. **Shrink the critical section**: copy data out of the lock, process afterwards.
3. `ReentrantLock.tryLock(timeout)` → detect and back off (beware of livelock).
4. **Thread confinement**: funnel all state changes onto one `Handler`/single-thread executor → no lock needed.

---

### Q2. How do deadlocks typically happen in system_server?

system_server has a few "big" locks:
- `ActivityManagerService` (`synchronized (this)` — the "AM lock"), `mProcLock` (✅ of type `ActivityManagerGlobalLock`, split out of the AM lock since A12, enabled by `ENABLE_PROC_LOCK`).
- `WindowManagerGlobalLock mGlobalLock` — **shared by WMS and ATMS**.
- `PackageManagerService.mLock`, `mInstallLock`; `PowerManagerService.mLock`; each OEM service's own lock.

Typical patterns:
1. **Inverted lock order between services**: service X holds `mLock` and calls into AMS (needs the AM lock), while AMS holds the AM lock and calls back into X (needs `mLock`).
2. **Calling out while holding a lock**: binder calls to apps/HALs, invoking listeners, sending **synchronous** broadcasts from inside `synchronized`.
3. **Waiting for a main/handler thread that is itself waiting for your lock** (`Handler.runWithScissors`, `CountDownLatch.await`).

AOSP conventions to avoid them:
- Methods suffixed **`...Locked()`** = the caller must hold the lock; un-suffixed methods take the lock themselves.
- `@GuardedBy("mLock")` annotations (checked by Error Prone at build time).
- **Never call out (binder, callbacks, other services) while holding your own lock** — collect data under the lock, call after releasing it.
- Callbacks to apps via `RemoteCallbackList` + **`oneway`** interfaces so a slow app cannot block system_server.

---

### Q3. Can binder cause deadlocks? How does re-entrancy work?

- A **synchronous** binder call blocks the calling thread until the server replies.
- **Re-entrancy**: A (thread T1) calls B; while handling it, B calls back into A → the binder driver delivers this nested transaction to **T1 itself**, which is waiting (same transaction chain), not to another thread of A. That is why T1's `synchronized` (reentrant) does not self-deadlock.
- Deadlocks still happen when:
  - A handles the callback on a **different thread** (e.g. posts to a Handler and waits for the result) while T1 holds a lock.
  - **Thread pool exhaustion**: every binder thread of the server (✅ `DEFAULT_MAX_BINDER_THREADS 15` in `ProcessState.cpp`, `sMaxBinderThreads = 31` in `SystemServer.java`) is waiting on a lock held by another thread, which is itself waiting on a binder call into that same server → nobody can accept it → hang. Log hint: `binder thread pool (N threads) starved for X ms` (✅ `IPCThreadState.cpp` only logs starvation longer than 100ms).
  - Two processes call each other synchronously from two different chains while each holds a lock.
- Fixes: `oneway` for callbacks/notifications, never hold a lock across a binder call, move heavy work to a Handler thread, bound waiting with timeouts.

---

### Q4. How does the system_server Watchdog detect deadlocks?

- `Watchdog` is a dedicated thread; every cycle it "pings" its `HandlerChecker`s. ✅ Source: `DEFAULT_TIMEOUT = DB ? 10s : 60s`, check interval = timeout / 2 (30s), overridable via `Settings.Global.WATCHDOG_TIMEOUT_MILLIS`; monitors run on the `watchdog.monitor` thread.
  - Checks the **important Loopers** (foreground thread, main thread, ui, io, display, animation...) — posts a message and checks it gets processed.
  - Calls `monitor()` on registered `Watchdog.Monitor`s: the method is just `synchronized (mLock) {}` → if the lock is held forever, the monitor gets stuck.
- Past half the timeout → stack dump (`/data/anr/`); past the timeout → log `*** WATCHDOG KILLING SYSTEM PROCESS: Blocked in monitor com.android.server.am.ActivityManagerService on monitor thread (watchdog.monitor)` or `Blocked in handler on foreground thread`, then **kill system_server** → zygote restarts the whole framework (**soft reboot**).
- OEM services should register:
```java
Watchdog.getInstance().addMonitor(this);           // implements Watchdog.Monitor
@Override public void monitor() { synchronized (mLock) { } }
Watchdog.getInstance().addThread(mHandler);          // if it owns an important Handler thread
```

---

### Q5. How do you read a deadlock trace in `/data/anr/`?

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
Procedure:
1. Find threads in state `Blocked` with `waiting to lock <X> held by thread N`.
2. Follow the `held by` chain → if it returns to the starting thread ⇒ **a deadlock cycle**.
3. If the chain ends in a `Native` thread inside `BinderProxy.transactNative` / `IPCThreadState::talkWithDriver` → it is waiting on a **binder call to another process** → inspect the target process (its trace may be in the same ANR/watchdog file, or use `debuggerd -b <pid>`).
4. `Waiting`/`TimedWaiting` threads on `Object.wait`, `CountDownLatch` → waiting for an event; check who should `notify`.

Tools:
```bash
adb shell kill -3 <pid>                         # Java stack dump of an app/system_server
adb shell debuggerd -b <pid>                     # native stacks (plus Java for apps)
adb shell cat /dev/binderfs/binder_logs/transactions   # or /sys/kernel/debug/binder/transactions (older devices)
adb shell dumpsys activity processes | grep -i frozen   # frozen processes also make binder calls wait
adb logcat | grep -i "Long monitor contention"   # ART logs long lock contention
```
Perfetto: enable the `dalvik` category → slices like `monitor contention with owner <thread> (<tid>) at <method>(<file>:<line>)` (✅ string from `art/runtime/monitor.cc`), plus `binder transaction`/`binder reply` slices linked across processes.

---

### Q6. Deadlocks on an app's main thread (leading to ANR)

Example:
```kotlin
// UI thread
val latch = CountDownLatch(1)
handler.post { /* runs on the SAME main looper */ latch.countDown() }
latch.await()          // main waits for a message only main can process → hangs forever → ANR
```
Variants:
- `runBlocking { withContext(Dispatchers.Main) { ... } }` called from the main thread.
- `Future.get()` on main waiting for a task that needs main.
- `synchronized` shared between main and a worker, where the worker calls `runOnUiThread` and waits for the result.
- Slow ContentProvider/`SharedPreferences.commit()` is not a deadlock but also causes ANRs (Chapter 08).

---

### Q7. Deadlocks in native/HAL code (C++)

The classic pattern in a HAL with callbacks:
```cpp
// WRONG: invoking callbacks while holding the mutex
void Hal::onHwEvent(int v) {
    std::lock_guard<std::mutex> lk(mMutex);
    for (auto& cb : mCallbacks) cb->onEvent(v);     // the client may call back setX() → needs mMutex → deadlock
}
// RIGHT: copy the list under the lock, invoke outside; declare the callback interface oneway
void Hal::onHwEvent(int v) {
    std::vector<std::shared_ptr<ICallback>> cbs;
    { std::lock_guard<std::mutex> lk(mMutex); cbs = mCallbacks; }
    for (auto& cb : cbs) cb->onEvent(v);
}
```
- `std::mutex` is **not reentrant** → locking twice on the same thread self-deadlocks; `std::recursive_mutex` should be a last resort.
- Use `std::scoped_lock(m1, m2)` to take several mutexes without worrying about order.
- Clang Thread Safety Analysis (`GUARDED_BY`, `REQUIRES`) — AOSP enables `-Wthread-safety` in many modules.
- Debug: `debuggerd -b <pid>` shows many threads in `__futex_wait_ex`/`pthread_mutex_lock`.

---

### Q8. Can coroutines deadlock?

- kotlinx.coroutines `Mutex` is **not reentrant**: `mutex.withLock { mutex.withLock { } }` → hangs.
- `runBlocking` on a thread that the inner coroutine needs (main, or a single-threaded dispatcher) → hangs.
- `Dispatchers.IO` with all 64 threads blocked waiting on each other → starvation.
- Recommendation: use `Mutex`/`Channel`/actors or `limitedParallelism(1)` to serialize instead of `synchronized`; never call suspend functions inside `synchronized` (it won't compile with a suspend lambda, and holding a lock across a suspension point is a design error).

---

### Q9. Scenario: "After adding LedService to system_server, the box occasionally soft-reboots when the power key is pressed." How do you handle it?

1. Grab a bugreport → find `WATCHDOG KILLING SYSTEM PROCESS` and the matching trace in `FS/data/anr/`.
2. Identify the stuck thread (e.g. the `PowerManagerService` handler) → follow the `held by` chain.
3. Suppose the result is: `PowerManagerService` (holding Power's `mLock`) calls the listener `LedService.onWakefulnessChanged()` **synchronously**; `LedService` holds its own `mLock` and calls the HAL; the HAL stalls 70s on a blocking sysfs write, or LedService calls `PowerManager.isInteractive()` → needs Power's `mLock` ⇒ cycle.
4. Fix:
   - Power's listener only **posts** to LedService's Handler; no direct processing.
   - Call the HAL outside the lock; make the HAL interface `oneway` or give it a timeout; the HAL must not block forever on sysfs.
   - `addMonitor` for LedService so that a hang points straight at the culprit.
5. Add a test: stress the power key + `am hang`/monkey; watch for `Long monitor contention`.

---

## 20.2 Rapid-fire

1. **Is `synchronized` reentrant?** → Yes (Java monitors); `std::mutex` and coroutine `Mutex` are not.
2. **Can `volatile` replace a lock?** → It only guarantees visibility/ordering, not atomicity for `count++`; use `AtomicInteger`.
3. **Does `ConcurrentHashMap` prevent deadlocks?** → Unrelated; it only reduces contention. Complex work inside `computeIfAbsent` that touches the same map can fail/hang.
4. **What do apps see when the Watchdog kills system_server?** → The whole framework restarts (boot animation), all apps are killed; it is not a kernel reboot.
5. **What does "binder thread pool starved" mean?** → All binder threads are busy → new requests wait; usually caused by holding locks/doing long I/O on binder threads.
6. **Why should callbacks be `oneway`?** → The server doesn't wait for the client, so a slow/hung client can't drag the server down; the trade-off is no return value and no exceptions.
7. **Is `Handler.runWithScissors` dangerous?** → Yes: it blocks the caller until the handler finishes; if the handler thread is waiting on the caller → deadlock. Only for controlled framework initialization.
