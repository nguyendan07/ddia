# Research: Dịch chương 3

Kế thừa quyết định của [feature 003](../003-translate-chapter-1/research.md) và [004](../004-translate-preface-part-i-ch2/research.md) (quy ước thuật ngữ, ngoặc full-width → ASCII, dịch tại chỗ không dùng `bin/translate.py`, thêm term thiếu vào bảng trước, bỏ cờ nháp ở cuối). Bổ sung:

- **Mã ví dụ (SQL, Cypher, SPARQL, Datalog, GraphQL, JSON…)** — Decision: giữ nguyên mã; chỉ dịch `caption` của `eg` và văn bản xung quanh. Rationale: mã phải chạy được, constitution I.
- **Nhiều xref (46) và footnote (69)** — Decision: giữ nguyên ID, số, URL; chỉ dịch chữ hiển thị. Dịch theo từng mục `##` rồi chạy `check-vi.py ch3` để bắt lệch sớm.
- **Thuật ngữ đặc thù chương 3 (graph, property graph, triple, schema-on-read…)** — Decision: kiểm `data/glossary/vi.toml` trước; term thiếu thì thêm vào bảng trước khi dùng.
- **Tên ngôn ngữ/sản phẩm và viết tắt (SQL, ORM, GraphQL, RDF…)** — Decision: giữ nguyên tiếng Anh.

Không còn NEEDS CLARIFICATION.
