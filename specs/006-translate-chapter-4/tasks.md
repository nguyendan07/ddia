# Tasks: Dịch chương 4

**Input**: `specs/006-translate-chapter-4/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/ch4.md` vào `content/vi/ch4.md`, đối chiếu `content/en/ch4.md` (en thắng khi lệch); dạng `*thuật ngữ* (*english term*)` ở lần đầu trong trang; giữ nguyên ID/anchor/shortcode/URL/footnote/tên riêng và toàn bộ nội dung khối mã (shell, SQL); chỉ dịch `caption` và text hiển thị; front matter chỉ đổi `title` (đã khớp `data/home/vi.yaml`). Các cặp `*术语*（*en*）` → ngoặc ASCII.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py ch4` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (11 fig, 27 xref, 1 eg, 104 định nghĩa footnote theo zh) của `content/vi/ch4.md`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term: liệt kê các cặp `*术语*（*en*）` trong `content/zh/ch4.md`, đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; tên ngôn ngữ/sản phẩm/viết tắt thì `keep = true`)

## Phase 3: User Story 1+2 — Chương 4 đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn bộ văn bản là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch4.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Tất cả task cùng sửa `content/vi/ch4.md` nên làm tuần tự; chạy `python3 bin/check-vi.py ch4` sau mỗi task để bắt lệch sớm.

- [x] T003 [US1] Dịch phần mở đầu chương (trích dẫn, giới thiệu) và mục `sec_storage_oltp` đến hết `the-sstable-file-format` (zh dòng 1–116; giữ nguyên mã)
- [x] T004 [US2] Dịch `constructing-and-merging-sstables`, `bloom-filters`, `sec_storage_lsm_compaction` (zh dòng 117–191)
- [x] T005 [US2] Dịch `sec_storage_b_trees` (kèm `sec_storage_btree_wal`, `b-tree-variants`) và `sec_storage_btree_lsm_comparison` (read performance, sequential, write amplification, disk space) (zh dòng 192–293)
- [x] T006 [US2] Dịch `sec_storage_index_multicolumn`, `sec_storage_index_heap`, `sec_storage_inmemory`, phần mở đầu `sec_storage_analytics` và `sec_cloud_data_warehouses` (zh dòng 294–360)
- [x] T007 [US2] Dịch `sec_storage_column` (kèm `sec_storage_column_compression`, `sort-order-in-column-storage`, `writing-to-column-oriented-storage`) (zh dòng 361–455)
- [x] T008 [US2] Dịch `sec_storage_vectorized`, `sec_storage_materialized_views`, `sec_storage_multidimensional`, `sec_storage_full_text`, `id92` (vector embeddings) (zh dòng 456–570)
- [x] T009 [US2] Dịch `summary` và danh sách `### 参考文献` (giữ nguyên số, URL, `[^N]`) (zh dòng 571–698)
- [x] T010 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong `content/vi/ch4.md`, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py ch4` và `make vi-check` thoát 0.

- [x] T011 [US3] Chạy `python3 bin/check-vi.py ch4`; sửa chênh lệch heading/fig/xref/eg/anchor/URL/footnote trong `content/vi/ch4.md`; xác nhận các khối mã không đổi so với zh (quickstart bước 4)
- [x] T012 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [ ] T013 Đọc lại `content/vi/ch4.md` đối chiếu `content/en/ch4.md`: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên; ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch"; các lần xuất hiện sau của thuật ngữ chỉ dùng tiếng Việt (trừ `show_en`) (FR-002, FR-007) (đã so số liệu chính; còn đọc đối chiếu từng mục `##`)
- [ ] T014 Xem `make vi-dev` tại `/ch4/`: bấm thử xref hình/ví dụ và footnote; lấy mẫu 10 thuật ngữ, xác nhận có term tiếng Anh ngay tại chỗ (SC-005)
- [x] T015 Xóa `draft: true` và `book_status: draft` khỏi front matter `content/vi/ch4.md`; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/ch4/` tồn tại

## Dependencies

T001 → T002 → T003 → T004 → T005 → T006 → T007 → T008 → T009 → T010 → T011 → T012 → T013 → T014 → T015. Không có task song song vì cùng sửa một file.

## Implementation Strategy

MVP = đến hết T010 (chương đọc được, thuật ngữ đúng); T011–T015 là cổng kiểm chứng và xuất bản. Commit sau T010 và sau T015.
