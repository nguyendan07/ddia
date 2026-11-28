# Research: Dịch chương 10

Kế thừa quyết định của [feature 003](../003-translate-chapter-1/research.md), [004](../004-translate-preface-part-i-ch2/research.md), [007](../007-translate-chapter-5/research.md), [009](../009-translate-chapter-7/research.md), [010](../010-translate-chapter-8/research.md) và [011](../011-translate-chapter-9/research.md) (quy ước thuật ngữ, ngoặc full-width → ASCII, dịch tại chỗ không dùng `bin/translate.py`, thêm term thiếu vào bảng trước, bỏ cờ nháp ở cuối). Bổ sung:

- **Không có shortcode `eg`/`tbl`, không có khối mã** — Decision: chỉ cần giữ nguyên shortcode `fig`/`xref`, anchor và thuộc tính.
- **91 footnote, 24 dòng xref, 10 hình** — Decision: giữ nguyên ID, số, URL; dịch theo 4 mục `##` rồi chạy `check-vi.py ch10` để bắt lệch sớm.
- **Thuật ngữ đặc thù** (linearizability, consensus, total order broadcast, logical clock, Lamport timestamp, hybrid logical clock, compare-and-set, atomic commit, leader election, epoch/ballot, quorum, coordination service…) — Decision: kiểm `data/glossary/vi.toml` trước; term thiếu thì thêm trước khi dùng. Tên thuật toán/sản phẩm (Paxos, Raft, ZooKeeper, etcd, Zab, Viewstamped Replication) giữ nguyên.
- **Tên riêng, viết tắt** — Decision: giữ nguyên.

Không còn NEEDS CLARIFICATION.
