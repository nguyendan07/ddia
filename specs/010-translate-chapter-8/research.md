# Research: Dịch chương 8

Kế thừa quyết định của [feature 003](../003-translate-chapter-1/research.md), [004](../004-translate-preface-part-i-ch2/research.md), [007](../007-translate-chapter-5/research.md) và [009](../009-translate-chapter-7/research.md) (quy ước thuật ngữ, ngoặc full-width → ASCII, dịch tại chỗ không dùng `bin/translate.py`, thêm term thiếu vào bảng trước, bỏ cờ nháp ở cuối). Bổ sung:

- **Có khối mã và 2 shortcode `eg`** (8-1 `fig_transactions_select_for_update`, 8-2 `fig_transactions_meeting_rooms`) — Decision: giữ nguyên SQL và thuộc tính `num`/`id`; chỉ dịch `caption` và chú thích ❶. Rationale: mã không dịch; `eg` nằm trong danh sách shortcode cần giữ.
- **85 footnote, 35 xref, 14 hình** — Decision: giữ nguyên ID, số, URL; dịch theo từng mục `##` rồi chạy `check-vi.py ch8` để bắt lệch sớm.
- **Thuật ngữ đặc thù** (ACID, atomicity, serializability, dirty read/write, two-phase locking, optimistic concurrency control, two-phase commit…) — Decision: kiểm `data/glossary/vi.toml` trước; term thiếu thì thêm trước khi dùng. Các mức cô lập (read committed, snapshot isolation, serializable…) giữ nguyên tiếng Anh theo bảng (`keep = true`). Lưu ý: "serializability" của giao dịch khác nghĩa "serialization" ở ch5 (đã ghi trong bảng).
- **Tên sản phẩm, viết tắt, tên câu lệnh SQL** — Decision: giữ nguyên.

Không còn NEEDS CLARIFICATION.
