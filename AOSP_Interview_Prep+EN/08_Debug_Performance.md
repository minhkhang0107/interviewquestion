# Chapter 08 — Debug, Stability & Performance

### Q1. Basic debugging toolkit and when to use each tool?

| Tool | Used for |
|---------|---------|
| `logcat -b main,system,crash,events,radio,kernel` | logs; `-s TAG`, `--pid`, `-v threadtime` |
| `dmesg` / `logcat -b kernel` | kernel log, avc denials, drivers |
| `dumpsys <service>` / `dumpsys -l` | service state (activity, window, package, power, media.audio_flinger…) |
| `service list`, `service check <name>` | list binder services |
| `lshal`, `lshal debug <iface>` | list HALs (HIDL + AIDL), PIDs, clients |
| `getprop` / `setprop` / `watchprops` | properties |
| `ps -AZ -T`, `top -H`, `cat /proc/<pid>/status` | processes/threads, SELinux domain |
| `debuggerd -b <pid>` | dump native backtraces (without killing the process) |
| `kill -3 <pid>` | dump Java stacks to `/data/anr/` |
| `bugreport` / `bugreportz` | collect everything (dumpstate) |
| `perfetto` / `systrace` / `atrace` | performance tracing |
| `simpleperf` | native CPU profiling |
| `showmap`, `dumpsys meminfo`, `procrank`, `librank` | memory |
| `strace` (userdebug), `gdbclient.py`/`lldbclient.py` | native debugging |
| `cmd <service> ...` | a service's shell commands |

---

### Q2. What is an ANR? Types & timeouts? How do you read an ANR trace?

- ANR (Application Not Responding): the main thread did not respond in time:
  - Input dispatching: **5s**
  - BroadcastReceiver foreground **10s**, background **60s**
  - Service foreground **20s** / background **200s** (`ActivityManagerConstants.DEFAULT_SERVICE_TIMEOUT`, `×10` for background)
  - `startForeground()` after `startForegroundService()`: **30s** on A14 (`DEFAULT_SERVICE_START_FOREGROUND_TIMEOUT_MS = 30 * 1000`; older versions used 10s) → `ForegroundServiceDidNotStartInTimeException`
  - ContentProvider publish: 10s
- Traces live in `/data/anr/anr_*` (or `traces.txt`), and in bugreports.
- Reading: find the `"main"` thread → state (`Blocked`, `Waiting`, `Native`) → stack → if it sits in `BinderProxy.transactNative`, go to the target process and see which binder thread is handling it (use `/sys/kernel/debug/binder/` or the binder info in the bugreport: "Binder transactions").
- Common causes: I/O on the main thread, lock contention, synchronous binder calls into a slow service, **a hung HAL**.
- Example from the project: `NativeWrapperImpl`'s `binderDied()` posts a retry to the **main looper** and calls `waitForDeclaredService()` (blocking) → if the HAL doesn't come back, `DavidExtension`'s main thread is blocked → ANR (for a persistent app, the user may see a dialog, or the app gets killed/restarted repeatedly).

---

### Q3. Native crash: how do you read a tombstone?

- `debuggerd` (crash_dump) catches signals (SIGSEGV, SIGABRT, SIGBUS...) → writes `/data/tombstones/tombstone_XX` (+ `.pb`).
- Contents: `pid, tid, name`, signal & fault addr, `Abort message`, registers, **backtrace** (`#00 pc 000123 /vendor/bin/hw/xxx (func+20)`), memory map.
- Symbolize: the `stack` tool or `ndk-stack`, `addr2line -e out/target/product/KSTB2500/symbols/vendor/bin/hw/<binary> 0x123`.
- Real scenario: the HAL's `CHECK(status == STATUS_OK)` fails → `SIGABRT` with abort message `Failed to add DavidDeviceControl Factory, status=-1` → init restarts the service in a loop; if the service is `critical` → reboot into the bootloader.

---

### Q4. What happens on a Java crash in system_server?

- Uncaught exception in system_server → the process dies → zygote (the parent process) receives `SIGCHLD`, sees that the child is system_server and **kills itself** (`SigChldHandler`: logs "Exit zygote because system server (pid N) has terminated" then `kill(getpid(), SIGKILL)`, `com_android_internal_os_Zygote.cpp:465-468`) → init sees the `zygote` service died and restarts it, along with the `onrestart restart audioserver/cameraserver/media...` entries declared in `init.zygote64.rc` → **soft reboot** (boot animation again, the kernel does not restart).
- Log: `*** FATAL EXCEPTION IN SYSTEM PROCESS: <thread>`.
- Look in `logcat -b crash`, `dropbox` (`dumpsys dropbox --print system_server_crash`).

---

### Q5. Watchdog timeout — how do you analyze it?

- Log: `Watchdog: *** WATCHDOG KILLING SYSTEM PROCESS: Blocked in handler on foreground thread (android.fg), Blocked in monitor com.android.server.am.ActivityManagerService on ...`.
- Traces are taken twice (30s & 60s) in `/data/anr/`. Compare the 2 dumps → which thread is still stuck at the same place.
- Look for the thread holding the lock (`- locked <0x...>`) and the waiting threads (`- waiting to lock <0x...> held by thread 23`).
- If the thread is in a binder call to a HAL → `debuggerd -b <HAL pid>` to see where the HAL is stuck (e.g. a sysfs write blocked in the driver).

---

### Q6. How do you use Perfetto / systrace?

```bash
adb shell perfetto -o /data/misc/perfetto-traces/trace.pftrace -t 10s \
    sched freq idle am wm gfx view binder_driver hal dalvik input
adb pull /data/misc/perfetto-traces/trace.pftrace   # open in ui.perfetto.dev
```
- Adding traces in code: Java `Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "name")`, native `ATRACE_CALL()` / `ATRACE_NAME("...")`.
- Look at: jank (frame timeline), binder transactions (find slow replies), CPU scheduling, lock contention (`monitor contention`).

---

### Q7. Memory leak / OOM / LMK?

- `dumpsys meminfo <pid>` (PSS/RSS, Java heap, native heap, graphics), `am dumpheap <pid> /data/local/tmp/x.hprof`.
- Native: `malloc_debug` (`setprop libc.debug.malloc.options backtrace`, `libc.debug.malloc.program`), `heapprofd` (perfetto).
- **lmkd** (low memory killer daemon) uses PSI (pressure stall information) + the `oom_score_adj` set by AMS → kills background processes.
- STBs usually have little RAM (1–2GB) → tune `ro.lmk.*`, `ro.config.low_ram`, zram (project: `zramsize=33%`, `comp_algorithm amfc`).

---

### Q8. Binder debugging?

```bash
adb shell cat /sys/kernel/debug/binder/state          # or /dev/binderfs/binder_logs/state
adb shell cat /sys/kernel/debug/binder/failed_transaction_log
adb shell dumpsys binder_calls_stats                  # call statistics inside system_server
adb shell am trace-ipc start ; ... ; am trace-ipc stop --dump-file /data/local/tmp/ipc.txt
```
- `TransactionTooLargeException` → reduce the payload, use `ParcelFileDescriptor`/`SharedMemory`.
- `DeadObjectException` → the other process died → check tombstones/logs.

---

### Q9. What does a bugreport contain? How do you analyze it quickly?

- `adb bugreport out.zip`: `dumpstate` (logcat, dumpsys all, `/proc`, tombstones, anr, kernel log, properties), `dumpstate_board` (from the vendor IDumpstateDevice HAL).
- Tools: **Battery Historian**, **ChkBugReport**; quick keyword search: `FATAL`, `ANR in`, `Watchdog`, `tombstone`, `avc: denied`, `am_crash`, `am_anr`, `binder: ... failed`.

---

### Q10. Scenario: "After flashing the firmware, the LED doesn't change color in standby." How do you debug it?

1. `adb shell ps -AZ | grep daviddevicecontrol` → is the HAL running? right domain?
2. `adb shell service list | grep -i david` → is it registered?
3. `logcat -s daviddevicecontrol DeviceControl DavidExtService PowerLedManager NativeWrapperImpl` → does the client call down?
4. `dmesg | grep avc | grep -i led` → is the sysfs write blocked by SELinux?
5. Try directly: `adb root; echo 1 > /sys/class/leds/<red>/brightness` → does the driver work?
6. Check DAC permissions: `ls -l /sys/class/leds/*/` (does ueventd.rc set owner `system`?).
7. Check VINTF: `adb shell lshal | grep david`, `vintf` → is it declared in the manifest?
8. Check power events: `dumpsys power`, `dumpsys hdmi_control` (STB standby often goes through CEC), `PowerStateManager` logs.
9. Write up the reproduction steps + root cause + fix (sepolicy/ueventd/logic).
