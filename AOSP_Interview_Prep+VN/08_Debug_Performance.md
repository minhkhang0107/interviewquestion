# Chương 08 — Debug, Stability & Performance

### Q1. Bộ công cụ debug cơ bản và khi nào dùng?

| Công cụ | Dùng để |
|---------|---------|
| `logcat -b main,system,crash,events,radio,kernel` | log; `-s TAG`, `--pid`, `-v threadtime` |
| `dmesg` / `logcat -b kernel` | log kernel, avc denial, driver |
| `dumpsys <service>` / `dumpsys -l` | state service (activity, window, package, power, media.audio_flinger…) |
| `service list`, `service check <name>` | liệt kê binder service |
| `lshal`, `lshal debug <iface>` | liệt kê HAL (HIDL + AIDL), PID, client |
| `getprop` / `setprop` / `watchprops` | property |
| `ps -AZ -T`, `top -H`, `cat /proc/<pid>/status` | process/thread, domain SELinux |
| `debuggerd -b <pid>` | dump backtrace native (không giết process) |
| `kill -3 <pid>` | dump stack Java vào `/data/anr/` |
| `bugreport` / `bugreportz` | gói toàn bộ (dumpstate) |
| `perfetto` / `systrace` / `atrace` | trace hiệu năng |
| `simpleperf` | CPU profiling native |
| `showmap`, `dumpsys meminfo`, `procrank`, `librank` | bộ nhớ |
| `strace` (userdebug), `gdbclient.py`/`lldbclient.py` | debug native |
| `cmd <service> ...` | shell command của service |

---

### Q2. ANR là gì? Các loại & timeout? Đọc ANR trace thế nào?

- ANR (Application Not Responding): main thread không xử lý kịp:
  - Input dispatching: **5s**
  - BroadcastReceiver foreground **10s**, background **60s**
  - Service foreground **20s** / background **200s** (`ActivityManagerConstants.DEFAULT_SERVICE_TIMEOUT`, `×10` cho background)
  - `startForeground()` sau `startForegroundService()`: **30s** trên A14 (`DEFAULT_SERVICE_START_FOREGROUND_TIMEOUT_MS = 30 * 1000`; các bản cũ là 10s) → `ForegroundServiceDidNotStartInTimeException`
  - ContentProvider publish: 10s
- Trace ở `/data/anr/anr_*` (hoặc `traces.txt`), trong bugreport.
- Đọc: tìm thread `"main"` → state (`Blocked`, `Waiting`, `Native`) → stack → nếu đang ở `BinderProxy.transactNative` thì sang process đích xem binder thread nào xử lý (dùng `/sys/kernel/debug/binder/` hoặc `dumpsys` binder info trong bugreport: "Binder transactions").
- Nguyên nhân phổ biến: I/O trên main thread, lock contention, binder call đồng bộ tới service chậm, **HAL treo**.
- Ví dụ từ project: `binderDied()` của `NativeWrapperImpl` post retry lên **main looper** và gọi `waitForDeclaredService()` (blocking) → nếu HAL không lên lại, main thread của `DavidExtension` bị block → ANR (với persistent app, user có thể thấy dialog hoặc app bị kill/restart liên tục).

---

### Q3. Native crash: tombstone đọc thế nào?

- `debuggerd` (crash_dump) bắt signal (SIGSEGV, SIGABRT, SIGBUS...) → ghi `/data/tombstones/tombstone_XX` (+ `.pb`).
- Nội dung: `pid, tid, name`, signal & fault addr, `Abort message`, registers, **backtrace** (`#00 pc 000123 /vendor/bin/hw/xxx (func+20)`), memory map.
- Symbolize: `stack` tool hoặc `ndk-stack`, `addr2line -e out/target/product/KSTB2500/symbols/vendor/bin/hw/<binary> 0x123`.
- Tình huống thật: HAL `CHECK(status == STATUS_OK)` fail → `SIGABRT` với abort message `Failed to add DavidDeviceControl Factory, status=-1` → init restart service liên tục; nếu service là `critical` → reboot vào bootloader.

---

### Q4. Java crash trong system_server thì sao?

- Uncaught exception trong system_server → process chết → zygote (process cha) nhận `SIGCHLD`, thấy con là system_server thì **tự kill chính nó** (`SigChldHandler`: log "Exit zygote because system server (pid N) has terminated" rồi `kill(getpid(), SIGKILL)`, `com_android_internal_os_Zygote.cpp:465-468`) → init thấy service `zygote` chết thì restart nó, kéo theo các `onrestart restart audioserver/cameraserver/media...` khai trong `init.zygote64.rc` → **soft reboot** (boot animation lại, kernel không khởi động lại).
- Log: `*** FATAL EXCEPTION IN SYSTEM PROCESS: <thread>`.
- Tìm trong `logcat -b crash`, `dropbox` (`dumpsys dropbox --print system_server_crash`).

---

### Q5. Watchdog timeout — phân tích như thế nào?

- Log: `Watchdog: *** WATCHDOG KILLING SYSTEM PROCESS: Blocked in handler on foreground thread (android.fg), Blocked in monitor com.android.server.am.ActivityManagerService on ...`.
- Trace 2 lần (30s & 60s) trong `/data/anr/`. So sánh 2 lần dump → thread nào vẫn đứng yên chỗ cũ.
- Xem thread đang giữ lock (`- locked <0x...>`) và thread chờ (`- waiting to lock <0x...> held by thread 23`).
- Nếu thread đang ở binder call sang HAL → `debuggerd -b <pid HAL>` để xem HAL kẹt ở đâu (ví dụ ghi sysfs bị block trong driver).

---

### Q6. Perfetto / systrace dùng thế nào?

```bash
adb shell perfetto -o /data/misc/perfetto-traces/trace.pftrace -t 10s \
    sched freq idle am wm gfx view binder_driver hal dalvik input
adb pull /data/misc/perfetto-traces/trace.pftrace   # mở ui.perfetto.dev
```
- Thêm trace trong code: Java `Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "name")`, native `ATRACE_CALL()` / `ATRACE_NAME("...")`.
- Xem: jank (frame timeline), binder transaction (tìm reply chậm), CPU scheduling, lock contention (`monitor contention`).

---

### Q7. Memory leak / OOM / LMK?

- `dumpsys meminfo <pid>` (PSS/RSS, Java heap, native heap, graphics), `am dumpheap <pid> /data/local/tmp/x.hprof`.
- Native: `malloc_debug` (`setprop libc.debug.malloc.options backtrace`, `libc.debug.malloc.program`), `heapprofd` (perfetto).
- **lmkd** (low memory killer daemon) dùng PSI (pressure stall information) + `oom_score_adj` do AMS set → kill process nền.
- STB thường RAM thấp (1–2GB) → cấu hình `ro.lmk.*`, `ro.config.low_ram`, zram (project: `zramsize=33%`, `comp_algorithm amfc`).

---

### Q8. Binder debug?

```bash
adb shell cat /sys/kernel/debug/binder/state          # hoặc /dev/binderfs/binder_logs/state
adb shell cat /sys/kernel/debug/binder/failed_transaction_log
adb shell dumpsys binder_calls_stats                  # thống kê call trong system_server
adb shell am trace-ipc start ; ... ; am trace-ipc stop --dump-file /data/local/tmp/ipc.txt
```
- `TransactionTooLargeException` → giảm payload, dùng `ParcelFileDescriptor`/`SharedMemory`.
- `DeadObjectException` → process kia chết → xem tombstone/log.

---

### Q9. Bugreport gồm gì? Phân tích nhanh thế nào?

- `adb bugreport out.zip`: `dumpstate` (logcat, dumpsys all, `/proc`, tombstones, anr, kernel log, properties), `dumpstate_board` (từ IDumpstateDevice HAL vendor).
- Tool: **Battery Historian**, **ChkBugReport**, tìm nhanh từ khoá: `FATAL`, `ANR in`, `Watchdog`, `tombstone`, `avc: denied`, `am_crash`, `am_anr`, `binder: ... failed`.

---

### Q10. Tình huống: "Sau khi flash firmware, LED không đổi màu khi standby". Debug thế nào?

1. `adb shell ps -AZ | grep daviddevicecontrol` → HAL có chạy? domain đúng?
2. `adb shell service list | grep -i david` → đã đăng ký?
3. `logcat -s daviddevicecontrol DeviceControl DavidExtService PowerLedManager NativeWrapperImpl` → client có gọi xuống?
4. `dmesg | grep avc | grep -i led` → bị SELinux chặn ghi sysfs?
5. Thử trực tiếp: `adb root; echo 1 > /sys/class/leds/<red>/brightness` → driver hoạt động?
6. Kiểm tra quyền DAC: `ls -l /sys/class/leds/*/` (ueventd.rc có set owner `system`?).
7. Kiểm tra VINTF: `adb shell lshal | grep david`, `vintf` → manifest có khai báo?
8. Kiểm tra sự kiện power: `dumpsys power`, `dumpsys hdmi_control` (STB standby thường đi qua CEC), log của `PowerStateManager`.
9. Viết lại bước tái hiện + root cause + fix (sepolicy/ueventd/logic).
