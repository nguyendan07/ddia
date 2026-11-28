# Research: Dịch chương 7

Kế thừa quyết định của [feature 003](../003-translate-chapter-1/research.md), [004](../004-translate-preface-part-i-ch2/research.md) và [007](../007-translate-chapter-5/research.md) (quy ước thuật ngữ, ngoặc full-width → ASCII, dịch tại chỗ không dùng `bin/translate.py`, thêm term thiếu vào bảng trước, bỏ cờ nháp ở cuối). Bổ sung:

- **Không có khối mã/`eg`** — Decision: chỉ cần giữ nguyên shortcode `fig`/`xref` và footnote. Rationale: không có mã cần bảo toàn.
- **34 footnote, 14 xref, 10 hình** — Decision: giữ nguyên ID, số, URL; dịch theo từng mục `##` rồi chạy `check-vi.py ch7` để bắt lệch sớm.
- **Thuật ngữ đặc thù (sharding, partition, hot spot, skew, rebalancing, consistent hashing, request routing, secondary index, scatter/gather…)** — Decision: kiểm `data/glossary/vi.toml` trước; term thiếu thì thêm vào bảng trước khi dùng.
- **Tên sản phẩm và viết tắt (Cassandra, MongoDB, Kafka, ZooKeeper, DynamoDB…)** — Decision: giữ nguyên tiếng Anh.

Không còn NEEDS CLARIFICATION.
