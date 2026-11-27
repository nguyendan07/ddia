# Tasks: Dịch chương 3

**Input**: `specs/005-translate-chapter-3/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/ch3.md` vào `content/vi/ch3.md`, đối chiếu `content/en/ch3.md` (en thắng khi lệch); dạng `*thuật ngữ* (*english term*)` ở lần đầu trong trang; giữ nguyên ID/anchor/shortcode/URL/footnote/tên riêng và toàn bộ nội dung khối mã (SQL, Cypher, SPARQL, Datalog, GraphQL, JSON); chỉ dịch `caption` và text hiển thị; front matter chỉ đổi `title` (đã khớp `data/home/vi.yaml`). Các cặp `*术语*（*en*）` → ngoặc ASCII.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py ch3` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (9 fig, 46 xref, 14 eg, 69 định nghĩa footnote theo zh) của `content/vi/ch3.md`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term: liệt kê các cặp `*术语*（*en*）` trong `content/zh/ch3.md`, đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; tên ngôn ngữ/sản phẩm/viết tắt thì `keep = true`)

## Phase 3: User Story 1+2 — Chương 3 đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn bộ văn bản là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch3.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Tất cả task cùng sửa `content/vi/ch3.md` nên làm tuần tự; chạy `python3 bin/check-vi.py ch3` sau mỗi task để bắt lệch sớm.

- [x] T003 [US1] Dịch phần mở đầu chương (trích dẫn, giới thiệu) và mục `sec_datamodels_history` đến hết `sec_datamodels_document` cùng `object-relational-mapping-orm` và `the-document-data-model-for-one-to-many-relationships` (zh dòng 1–128)
- [x] T004 [US2] Dịch `sec_datamodels_normalization` (kèm `trade-offs-of-normalization`, `denormalization-in-the-social-networking-case-study`) và `sec_datamodels_many_to_many` (zh dòng 129–238)
- [x] T005 [US2] Dịch `sec_datamodels_analytics` và `sec_datamodels_document_summary` kèm bốn mục con: schema flexibility, data locality, query languages, convergence (zh dòng 239–369)
- [x] T006 [US2] Dịch phần mở đầu `sec_datamodels_graph`, `id56` (property graph), `id57` (Cypher), `id58` (SQL graph query) (zh dòng 370–581; giữ nguyên mã)
- [x] T007 [US2] Dịch `id59` (triple store) kèm `the-rdf-data-model`, `the-sparql-query-language`, và `id62` (Datalog) (zh dòng 582–777)
- [x] T008 [US2] Dịch `id63` (GraphQL), `sec_datamodels_events` (event sourcing và CQRS), `sec_datamodels_dataframes` (zh dòng 778–914)
- [x] T009 [US2] Dịch `summary` và danh sách `### 参考文献` (giữ nguyên số, URL, `[^N]`) (zh dòng 915–1013)
- [x] T010 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong `content/vi/ch3.md`, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py ch3` và `make vi-check` thoát 0.

- [x] T011 [US3] Chạy `python3 bin/check-vi.py ch3`; sửa chênh lệch heading/fig/xref/eg/anchor/URL/footnote trong `content/vi/ch3.md`; xác nhận các khối mã không đổi so với zh (quickstart bước 4)
- [x] T012 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [x] T013 Đọc lại `content/vi/ch3.md` đối chiếu `content/en/ch3.md`: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên; ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch"; các lần xuất hiện sau của thuật ngữ chỉ dùng tiếng Việt (trừ `show_en`) (FR-002, FR-007)
- [x] T014 Xem `make vi-dev` tại `/ch3/`: bấm thử xref hình/ví dụ và footnote; lấy mẫu 10 thuật ngữ, xác nhận có term tiếng Anh ngay tại chỗ (SC-005)
- [x] T015 Xóa `draft: true` và `book_status: draft` khỏi front matter `content/vi/ch3.md`; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/ch3/` tồn tại

## Dependencies

T001 → T002 → T003 → T004 → T005 → T006 → T007 → T008 → T009 → T010 → T011 → T012 → T013 → T014 → T015. Không có task song song vì cùng sửa một file.

## Implementation Strategy

MVP = đến hết T010 (chương đọc được, thuật ngữ đúng); T011–T015 là cổng kiểm chứng và xuất bản. Commit sau T010 và sau T015.
