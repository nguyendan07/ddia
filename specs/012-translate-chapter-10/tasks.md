# Tasks: Dịch chương 10

**Input**: `specs/012-translate-chapter-10/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/ch10.md` vào `content/vi/ch10.md`, đối chiếu `content/en/ch10.md` (en thắng khi lệch); dạng `*thuật ngữ* (*english term*)` ở lần đầu trong trang; giữ nguyên ID/anchor/shortcode (`fig`, `xref`)/URL/footnote/tên riêng; khối mã giữ nguyên, chỉ dịch `caption` và chú thích; front matter chỉ đổi `title` (đã khớp `data/home/vi.yaml`). Các cặp `*术语*（*en*）` → ngoặc ASCII.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py ch10` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (ch10: 10 fig, 24 dòng xref, 0 eg, 0 tbl, 91 footnote, 0 khối mã theo zh) của `content/vi/ch10.md`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term: liệt kê các cặp `*术语*（*en*）` trong `content/zh/ch10.md`, đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu (vd. linearizability, total order broadcast, Lamport timestamp, hybrid logical clock, compare-and-set, atomic commit, shared log, coordination service) theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; tên sản phẩm/viết tắt thì `keep = true`)

## Phase 3: User Story 1+2 — Chương 10 đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn bộ văn bản là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch10.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Các task cùng sửa một file nên làm tuần tự; chạy `python3 bin/check-vi.py ch10` sau mỗi task để bắt lệch sớm. Số dòng theo `content/zh/ch10.md`.

- [x] T003 [US1] Dịch phần mở đầu chương và `sec_consistency_linearizability` + `sec_consistency_lin_definition` trong `content/vi/ch10.md` (dòng 1–131)
- [x] T004 [US2] Dịch `sec_consistency_linearizability_usage`: khóa và bầu leader, ràng buộc/tính duy nhất, phụ thuộc thời gian giữa các kênh (dòng 132–181)
- [x] T005 [US2] Dịch `sec_consistency_implementing_linearizable` và `sec_consistency_quorum_linearizable` (dòng 182–222)
- [x] T006 [US2] Dịch `sec_linearizability_cost`: định lý CAP, linearizability và độ trễ mạng (dòng 223–281)
- [x] T007 [US2] Dịch `sec_consistency_logical` mở đầu và `sec_consistency_timestamps`: Lamport, hybrid logical clock, so với vector clock (dòng 282–368)
- [x] T008 [US2] Dịch `sec_consistency_linearizable_id`: bộ sinh ID linearizable, ép ràng buộc bằng đồng hồ logic (dòng 369–409)
- [x] T009 [US2] Dịch `sec_consistency_consensus` mở đầu và `sec_consistency_faces`: đồng thuận giá trị đơn, compare-and-set (dòng 410–488)
- [x] T010 [US2] Dịch shared log, fetch-and-add và atomic commit như đồng thuận (dòng 489–568)
- [x] T011 [US2] Dịch `sec_consistency_total_order`: dùng shared log, từ single-leader sang đồng thuận, điểm tinh tế, lợi và hại (dòng 569–645)
- [x] T012 [US2] Dịch `sec_consistency_coordination`: phân việc cho node, service discovery (dòng 646–695)
- [x] T013 [US2] Dịch `summary` và danh sách `### 参考文献` (giữ nguyên số, URL, `[^N]`) (dòng 696–828)
- [x] T014 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong `content/vi/ch10.md`, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py ch10` và `make vi-check` thoát 0.

- [x] T015 [US3] Chạy `python3 bin/check-vi.py ch10`; sửa chênh lệch heading/fig/xref/anchor/URL/footnote trong `content/vi/ch10.md`
- [x] T016 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [x] T017 Đọc lại `content/vi/ch10.md` đối chiếu bản en: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên; ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch"; các lần sau của thuật ngữ chỉ dùng tiếng Việt (trừ `show_en`) (FR-002, FR-007)
- [x] T018 Xem `make vi-dev` tại `/ch10/`: bấm thử xref hình và footnote; lấy mẫu 10 thuật ngữ, xác nhận có term tiếng Anh ngay tại chỗ (SC-005) (đã kiểm tự động trên public/ch10: 10 hình, 91 footnote có đủ ref và def, xref trỏ đúng; chưa duyệt bằng mắt trên trình duyệt)
- [x] T019 Xóa `draft: true` và `book_status: draft` khỏi front matter `content/vi/ch10.md`; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/ch10/` tồn tại

## Dependencies

T001 → T002 → T003 → … → T019 tuần tự. Không có task song song vì cùng sửa một file.

## Implementation Strategy

MVP = đến hết T014 (chương đọc được, thuật ngữ đúng); T015–T019 là cổng kiểm chứng và xuất bản. Commit sau mỗi vài mục (vd. sau T007, T012), sau T014 và sau T019.
