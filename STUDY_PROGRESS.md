# Tiến trình ôn tập

> File này để tiếp tục ôn trên máy khác. Mở Claude Code trong repo này và nói:
> "Đọc STUDY_PROGRESS.md rồi tiếp tục ôn từ câu tiếp theo."

## Cách học (đã thống nhất)

- Tài liệu: `AOSP_Interview_Prep+VN/` (bản tiếng Việt).
- Theo **lộ trình 7 ngày** trong `AOSP_Interview_Prep+VN/README.md`.
- **Chế độ học**: mỗi lượt Claude đưa **1 câu** kèm đáp án (rút gọn ý chính, theo công thức
  định nghĩa → cơ chế → ví dụ → pitfall). Người học gõ `next` mới sang câu tiếp.
- **Hết chương** thì Claude mới hỏi lại (quiz), người học trả lời, Claude chấm x/10 và bổ sung chỗ thiếu.

## Vị trí hiện tại

- **Ngày 1 (xong)**: Chương 11, mục 11.1 → 11.5, **Q1 → Q19**
  - 11.1 Lifecycle: Q1 config change vs process death · Q2 đổi độ phân giải HDMI · Q3 SavedStateRegistry
  - 11.2 Task: Q4 launchMode · Q5 `CLEAR_TOP` vs `CLEAR_TASK` · Q6 StrandHogg/`taskAffinity` · Q7 `TaskStackBuilder`
  - 11.3 Service: Q8 thread + ANR timeout · Q9 started+bound destroy · Q10 FGS A14 · Q11 start service từ background
  - 11.4 Broadcast: Q12 `RECEIVER_(NOT_)EXPORTED` · Q13 ordered/abort · Q14 sticky · Q15 implicit ban A8
  - 11.5 Provider: Q16 provider `onCreate` trước Application · Q17 CursorWindow 2MB · Q18 FileProvider · Q19 file 100MB
  - Câu hỏi phụ đã giải thích thêm: AMS vs ATMS · truyền fd qua binder (`BINDER_TYPE_FD`) ·
    cấu trúc ContentProvider (Transport/Proxy, ContentProviderConnection stable/unstable, ContentService) ·
    thread-safe là gì · `file://` bị chặn ở đâu (`Intent.prepareToLeaveProcess` → StrictMode)
- **Ngày 2 (đang học)**: Chương 11 phần còn lại, từ 11.6
  - Đã học: Q20 MessageQueue ngủ/thức (epoll, eventfd) · Q21 sync barrier
  - **Câu tiếp theo: Q22**: IdleHandler.

## Quiz cuối chương

- Chương 11: chưa làm (làm sau khi học hết chương; có thể làm mini-quiz Ngày 1 nếu muốn).
