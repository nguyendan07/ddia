# Tasks: Dịch chương 8

**Input**: `specs/010-translate-chapter-8/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/ch8.md` vào `content/vi/ch8.md`, đối chiếu `content/en/ch8.md` (en thắng khi lệch); dạng `*thuật ngữ* (*english term*)` ở lần đầu trong trang; giữ nguyên ID/anchor/shortcode (`fig`, `xref`, `eg`)/URL/footnote/tên riêng; khối mã SQL giữ nguyên, chỉ dịch `caption` và chú thích ❶; front matter chỉ đổi `title` (đã khớp `data/home/vi.yaml`). Các cặp `*术语*（*en*）` → ngoặc ASCII.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py ch8` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (ch8: 14 fig, 35 xref, 2 eg, 85 footnote, 5 khối mã theo zh) của `content/vi/ch8.md`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term: liệt kê các cặp `*术语*（*en*）` trong `content/zh/ch8.md`, đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu (vd. atomicity, dirty read/write, serializability, two-phase locking/commit, optimistic concurrency control) theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; tên sản phẩm/viết tắt/mức cô lập thì `keep = true`)

## Phase 3: User Story 1+2 — Chương 8 đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn bộ văn bản là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch8.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Các task cùng sửa một file nên làm tuần tự; chạy `python3 bin/check-vi.py ch8` sau mỗi task để bắt lệch sớm. Số dòng theo `content/zh/ch8.md`.

- [x] T003 [US1] Dịch phần mở đầu và `sec_transactions_overview` cùng `sec_transactions_acid` (atomicity, consistency, isolation, durability) trong `content/vi/ch8.md` (dòng 1–133)
- [x] T004 [US2] Dịch `sec_transactions_multi_object`: đơn đối tượng, nhu cầu đa đối tượng, xử lý lỗi và hủy (dòng 134–214)
- [x] T005 [US2] Dịch `sec_transactions_isolation_levels` mở đầu, `sec_transactions_read_committed` (dirty read/write, cách cài đặt) (dòng 215–285)
- [x] T006 [US2] Dịch `sec_transactions_snapshot_isolation`: MVCC, quy tắc hiển thị, chỉ mục, nhầm lẫn tên gọi (dòng 286–374)
- [x] T007 [US2] Dịch `sec_transactions_lost_update` kèm ví dụ `eg` 8-1 và khối mã SQL (dòng 375–466)
- [x] T008 [US2] Dịch `sec_transactions_write_skew`: write skew, phantom, materializing conflicts, kèm `eg` 8-2 (dòng 467–574)
- [x] T009 [US2] Dịch `sec_transactions_serializability` và `sec_transactions_serial` (stored procedure, sharding, tóm tắt) (dòng 575–659)
- [x] T010 [US2] Dịch `sec_transactions_2pl` (cài đặt, hiệu năng, predicate lock, index-range lock) (dòng 660–741)
- [x] T011 [US2] Dịch `sec_transactions_ssi` (pessimistic vs optimistic, MVCC cũ, hiệu năng) (dòng 742–818)
- [x] T012 [US2] Dịch `sec_transactions_distributed`, `sec_transactions_2pc` (lời hứa, coordinator hỏng, 3PC) (dòng 819–901)
- [x] T013 [US2] Dịch `sec_transactions_xa` (exactly-once, XA, giữ khóa khi in-doubt, phục hồi) và `sec_transactions_internal` (dòng 902–1009)
- [x] T014 [US2] Dịch `summary` và danh sách `### 参考文献` (giữ nguyên số, URL, `[^N]`) (dòng 1010–1147)
- [x] T015 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong `content/vi/ch8.md`, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py ch8` và `make vi-check` thoát 0.

- [x] T016 [US3] Chạy `python3 bin/check-vi.py ch8`; sửa chênh lệch heading/fig/xref/eg/anchor/URL/footnote/khối mã trong `content/vi/ch8.md`
- [x] T017 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [x] T018 Đọc lại `content/vi/ch8.md` đối chiếu bản en: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên; ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch"; các lần sau của thuật ngữ chỉ dùng tiếng Việt (trừ `show_en`) (FR-002, FR-007)
- [ ] T019 Xem `make vi-dev` tại `/ch8/`: bấm thử xref hình, ví dụ `eg` và footnote; lấy mẫu 10 thuật ngữ, xác nhận có term tiếng Anh ngay tại chỗ (SC-005)
- [x] T020 Xóa `draft: true` và `book_status: draft` khỏi front matter `content/vi/ch8.md`; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/ch8/` tồn tại

## Dependencies

T001 → T002 → T003 → … → T020 tuần tự. Không có task song song vì cùng sửa một file.

## Implementation Strategy

MVP = đến hết T015 (chương đọc được, thuật ngữ đúng); T016–T020 là cổng kiểm chứng và xuất bản. Chương dài nên commit sau mỗi vài mục (vd. sau T008, T013), sau T015 và sau T020.
