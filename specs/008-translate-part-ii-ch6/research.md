# Research: Dịch Phần II và chương 6

Kế thừa quyết định của [feature 003](../003-translate-chapter-1/research.md), [004](../004-translate-preface-part-i-ch2/research.md) và [007](../007-translate-chapter-5/research.md) (quy ước thuật ngữ, ngoặc full-width → ASCII, dịch tại chỗ không dùng `bin/translate.py`, thêm term thiếu vào bảng trước, bỏ cờ nháp ở cuối). Bổ sung:

- **Không có khối mã/`eg`** — Decision: chỉ cần giữ nguyên shortcode `fig`/`xref` và footnote. Rationale: không có mã cần bảo toàn.
- **Nhiều footnote (64) và xref (40), 16 hình ở ch6** — Decision: giữ nguyên ID, số, URL; dịch theo từng mục `##` rồi chạy `check-vi.py part-ii ch6` để bắt lệch sớm.
- **Thuật ngữ đặc thù (replication, leader/follower, replication lag, quorum, conflict resolution, CRDT, multi-leader, leaderless…)** — Decision: kiểm `data/glossary/vi.toml` trước; term thiếu thì thêm vào bảng trước khi dùng.
- **Tên sản phẩm và viết tắt (PostgreSQL, Raft, CRDT, SLA…)** — Decision: giữ nguyên tiếng Anh.

Không còn NEEDS CLARIFICATION.
