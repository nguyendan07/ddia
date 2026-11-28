# Research: Dịch chương 9

Kế thừa quyết định của [feature 003](../003-translate-chapter-1/research.md), [004](../004-translate-preface-part-i-ch2/research.md), [007](../007-translate-chapter-5/research.md), [009](../009-translate-chapter-7/research.md) và [010](../010-translate-chapter-8/research.md) (quy ước thuật ngữ, ngoặc full-width → ASCII, dịch tại chỗ không dùng `bin/translate.py`, thêm term thiếu vào bảng trước, bỏ cờ nháp ở cuối). Bổ sung:

- **Không có shortcode `eg`, chỉ 1 khối mã** — Decision: giữ nguyên mã và thuộc tính; chỉ dịch chú thích kèm theo.
- **134 footnote, 14 xref, 7 hình** — Decision: giữ nguyên ID, số, URL; dịch theo từng mục `##` (5 mục) rồi chạy `check-vi.py ch9` để bắt lệch sớm.
- **Thuật ngữ đặc thù** (partial failure, unreliable network, timeout, clock synchronization, monotonic clock, process pause, fencing token, Byzantine fault, system model, safety/liveness…) — Decision: kiểm `data/glossary/vi.toml` trước; term thiếu thì thêm trước khi dùng. Tên giao thức/viết tắt (NTP, GPS, TCP, p99, SLA) giữ nguyên.
- **Tên sản phẩm, viết tắt** — Decision: giữ nguyên.

Không còn NEEDS CLARIFICATION.
