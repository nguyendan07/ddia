# Tasks: Dịch chương 9

**Input**: `specs/011-translate-chapter-9/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/ch9.md` vào `content/vi/ch9.md`, đối chiếu `content/en/ch9.md` (en thắng khi lệch); dạng `*thuật ngữ* (*english term*)` ở lần đầu trong trang; giữ nguyên ID/anchor/shortcode (`fig`, `xref`)/URL/footnote/tên riêng; khối mã giữ nguyên, chỉ dịch `caption` và chú thích; front matter chỉ đổi `title` (đã khớp `data/home/vi.yaml`). Các cặp `*术语*（*en*）` → ngoặc ASCII.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py ch9` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (ch9: 7 fig, 14 xref, 0 eg, 134 footnote, 1 khối mã theo zh) của `content/vi/ch9.md`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term: liệt kê các cặp `*术语*（*en*）` trong `content/zh/ch9.md`, đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu (vd. partial failure, timeout, monotonic clock, process pause, fencing token, Byzantine fault, system model, safety/liveness) theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; tên sản phẩm/viết tắt thì `keep = true`)

## Phase 3: User Story 1+2 — Chương 9 đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn bộ văn bản là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch9.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Các task cùng sửa một file nên làm tuần tự; chạy `python3 bin/check-vi.py ch9` sau mỗi task để bắt lệch sớm. Số dòng theo `content/zh/ch9.md`.

- [x] T003 [US1] Dịch phần mở đầu và `sec_distributed_partial_failure` trong `content/vi/ch9.md` (dòng 1–46)
- [x] T004 [US2] Dịch `sec_distributed_networks` phần đầu: `sec_distributed_tcp`, `sec_distributed_network_faults`, `id307` (dòng 47–124)
- [x] T005 [US2] Dịch `sec_distributed_queueing` (kèm network congestion) và `sec_distributed_sync_networks` (dòng 125–217)
- [x] T006 [US2] Dịch `sec_distributed_clocks` mở đầu, `sec_distributed_monotonic_timeofday`, `sec_distributed_clock_accuracy` (dòng 218–277)
- [x] T007 [US2] Dịch `sec_distributed_clocks_relying`: `sec_distributed_lww`, khoảng tin cậy, `sec_distributed_spanner` (dòng 278–342)
- [x] T008 [US2] Dịch `sec_distributed_clocks_pauses`, `sec_distributed_clocks_realtime`, `sec_distributed_gc_impact` (dòng 343–421)
- [x] T009 [US2] Dịch `sec_distributed_truth` mở đầu, `sec_distributed_majority`, `sec_distributed_lock_fencing` (dòng 422–510)
- [x] T010 [US2] Dịch `sec_distributed_byzantine` và `weak-forms-of-lying` (dòng 511–550)
- [x] T011 [US2] Dịch `sec_distributed_system_model`: tính đúng đắn, `sec_distributed_safety_liveness`, ánh xạ vào thực tế (dòng 551–627)
- [x] T012 [US2] Dịch `sec_distributed_formal`: model checking, `sec_fault_injection`, deterministic simulation testing (dòng 628–688)
- [x] T013 [US2] Dịch `summary` và danh sách `### 参考文献` (giữ nguyên số, URL, `[^N]`) (dòng 689–848)
- [x] T014 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong `content/vi/ch9.md`, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py ch9` và `make vi-check` thoát 0.

- [x] T015 [US3] Chạy `python3 bin/check-vi.py ch9`; sửa chênh lệch heading/fig/xref/anchor/URL/footnote/khối mã trong `content/vi/ch9.md`
- [x] T016 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [x] T017 Đọc lại `content/vi/ch9.md` đối chiếu bản en: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên; ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch"; các lần sau của thuật ngữ chỉ dùng tiếng Việt (trừ `show_en`) (FR-002, FR-007)
- [ ] T018 Xem `make vi-dev` tại `/ch9/`: bấm thử xref hình và footnote; lấy mẫu 10 thuật ngữ, xác nhận có term tiếng Anh ngay tại chỗ (SC-005)
- [x] T019 Xóa `draft: true` và `book_status: draft` khỏi front matter `content/vi/ch9.md`; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/ch9/` tồn tại

## Dependencies

T001 → T002 → T003 → … → T019 tuần tự. Không có task song song vì cùng sửa một file.

## Implementation Strategy

MVP = đến hết T014 (chương đọc được, thuật ngữ đúng); T015–T019 là cổng kiểm chứng và xuất bản. Commit sau mỗi vài mục (vd. sau T008, T012), sau T014 và sau T019.
