# Research: Dịch chương 4

Kế thừa quyết định của [feature 003](../003-translate-chapter-1/research.md) và [004](../004-translate-preface-part-i-ch2/research.md) (quy ước thuật ngữ, ngoặc full-width → ASCII, dịch tại chỗ không dùng `bin/translate.py`, thêm term thiếu vào bảng trước, bỏ cờ nháp ở cuối). Bổ sung:

- **Mã ví dụ (shell, SQL)** — Decision: giữ nguyên mã; chỉ dịch `caption` của `eg` và văn bản xung quanh. Rationale: mã phải chạy được, constitution I.
- **Nhiều footnote (104), 27 xref, 11 hình** — Decision: giữ nguyên ID, số, URL; chỉ dịch chữ hiển thị. Dịch theo từng mục `##` rồi chạy `check-vi.py ch4` để bắt lệch sớm.
- **Thuật ngữ đặc thù chương 4 (LSM-tree, B-tree, SSTable, column store, vector index, compaction…)** — Decision: kiểm `data/glossary/vi.toml` trước; term thiếu thì thêm vào bảng trước khi dùng.
- **Tên ngôn ngữ/sản phẩm và viết tắt (SQL, OLTP, OLAP, HNSW…)** — Decision: giữ nguyên tiếng Anh.

Không còn NEEDS CLARIFICATION.
