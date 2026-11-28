# Tasks: Dịch Phần II và chương 6

**Input**: `specs/008-translate-part-ii-ch6/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/<trang>.md` vào `content/vi/<trang>.md`, đối chiếu `content/en/<trang>.md` (en thắng khi lệch); dạng `*thuật ngữ* (*english term*)` ở lần đầu trong trang; giữ nguyên ID/anchor/shortcode/URL/footnote/tên riêng; chỉ dịch `caption` và text hiển thị; front matter chỉ đổi `title` (đã khớp `data/home/vi.yaml`). Các cặp `*术语*（*en*）` → ngoặc ASCII.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py part-ii ch6` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (part-ii: 1 fig, 4 footnote; ch6: 16 fig, 40 xref, 64 footnote theo zh) của `content/vi/part-ii.md` và `content/vi/ch6.md`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term: liệt kê các cặp `*术语*（*en*）` trong `content/zh/part-ii.md` và `content/zh/ch6.md`, đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; tên sản phẩm/viết tắt thì `keep = true`)

## Phase 3: User Story 1+2 — Phần II và chương 6 đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn bộ văn bản là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/part-ii.md content/vi/ch6.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Các task cùng sửa một file nên làm tuần tự; chạy `python3 bin/check-vi.py part-ii ch6` sau mỗi task để bắt lệch sớm.

- [x] T003 [US1] Dịch `content/vi/part-ii.md` (zh 125 dòng, kèm hình và 4 footnote)
- [x] T004 [US2] Dịch phần mở đầu `content/vi/ch6.md` và `sec_replication_leader` đến hết `sec_replication_sync_async`, `sec_replication_new_replica` (zh dòng 1–121)
- [x] T005 [US2] Dịch `sec_replication_failover` và `sec_replication_implementation` (zh dòng 122–207)
- [x] T006 [US2] Dịch `sec_replication_lag` kèm `sec_replication_ryw`, `sec_replication_monotonic_reads`, `sec_replication_consistent_prefix`, `id131` (zh dòng 208–313)
- [x] T007 [US2] Dịch `sec_replication_multi_leader` kèm `sec_replication_multi_dc`, `sec_replication_offline_clients` (zh dòng 314–422)
- [x] T008 [US2] Dịch `sec_replication_write_conflicts` và `sec_replication_crdts` (zh dòng 423–518)
- [x] T009 [US2] Dịch `sec_replication_leaderless` kèm `id287`, `sec_replication_quorum_limitations`, `sec_replication_leaderless_perf` (zh dòng 519–649)
- [x] T010 [US2] Dịch `sec_replication_concurrent`, `summary` và danh sách `### 参考文献` (giữ nguyên số, URL, `[^N]`) (zh dòng 650–854)
- [x] T011 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong hai trang, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py part-ii ch6` và `make vi-check` thoát 0.

- [x] T012 [US3] Chạy `python3 bin/check-vi.py part-ii ch6`; sửa chênh lệch heading/fig/xref/anchor/URL/footnote trong `content/vi/part-ii.md`, `content/vi/ch6.md`
- [x] T013 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [x] T014 Đọc lại hai trang đối chiếu bản en: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên; ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch"; các lần xuất hiện sau của thuật ngữ chỉ dùng tiếng Việt (trừ `show_en`) (FR-002, FR-007)
- [ ] T015 Xem `make vi-dev` tại `/part-ii/` và `/ch6/`: bấm thử xref hình và footnote; lấy mẫu 10 thuật ngữ, xác nhận có term tiếng Anh ngay tại chỗ (SC-005)
- [x] T016 Xóa `draft: true` và `book_status: draft` khỏi front matter cả hai trang; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/part-ii/` và `public/ch6/` tồn tại

## Dependencies

T001 → T002 → T003 → … → T016 tuần tự. Không có task song song vì cùng sửa một file (part-ii và ch6 có thể làm song song về lý thuyết nhưng nên giữ tuần tự để dùng chung term mới).

## Implementation Strategy

MVP = đến hết T011 (hai trang đọc được, thuật ngữ đúng); T012–T016 là cổng kiểm chứng và xuất bản. Commit sau T011 và sau T016.
