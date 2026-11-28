# Tasks: Dịch chương 7

**Input**: `specs/009-translate-chapter-7/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/ch7.md` vào `content/vi/ch7.md`, đối chiếu `content/en/ch7.md` (en thắng khi lệch); dạng `*thuật ngữ* (*english term*)` ở lần đầu trong trang; giữ nguyên ID/anchor/shortcode/URL/footnote/tên riêng; chỉ dịch `caption` và text hiển thị; front matter chỉ đổi `title` (đã khớp `data/home/vi.yaml`). Các cặp `*术语*（*en*）` → ngoặc ASCII.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py ch7` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (ch7: 10 fig, 14 xref, 34 footnote theo zh) của `content/vi/ch7.md`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term: liệt kê các cặp `*术语*（*en*）` trong `content/zh/ch7.md`, đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; tên sản phẩm/viết tắt thì `keep = true`)

## Phase 3: User Story 1+2 — Chương 7 đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn bộ văn bản là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch7.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Các task cùng sửa một file nên làm tuần tự; chạy `python3 bin/check-vi.py ch7` sau mỗi task để bắt lệch sớm.

- [x] T003 [US1] Dịch phần mở đầu và `sec_sharding_reasons` kèm `sec_sharding_multitenancy` trong `content/vi/ch7.md` (zh dòng 1–94)
- [x] T004 [US2] Dịch `sec_sharding_key_value`, `sec_sharding_key_range` và `rebalancing-key-range-sharded-data` (zh dòng 95–136)
- [x] T005 [US2] Dịch `sec_sharding_hash` cùng các mục con: hash-modulo, fixed-number-of-shards, sharding-by-hash-range, `sec_sharding_consistent_hashing` (zh dòng 137–205)
- [x] T006 [US2] Dịch `sec_sharding_skew`, `sec_sharding_operations` và `sec_sharding_routing` (zh dòng 206–269)
- [x] T007 [US2] Dịch `sec_sharding_secondary_indexes` kèm `id166`, `id167` (zh dòng 270–320)
- [x] T008 [US2] Dịch `summary` và danh sách `### 参考文献` (giữ nguyên số, URL, `[^N]`) (zh dòng 321–385)
- [x] T009 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong `content/vi/ch7.md`, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py ch7` và `make vi-check` thoát 0.

- [x] T010 [US3] Chạy `python3 bin/check-vi.py ch7`; sửa chênh lệch heading/fig/xref/anchor/URL/footnote trong `content/vi/ch7.md`
- [x] T011 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [x] T012 Đọc lại `content/vi/ch7.md` đối chiếu bản en: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên; ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch"; các lần sau của thuật ngữ chỉ dùng tiếng Việt (trừ `show_en`) (FR-002, FR-007)
- [ ] T013 Xem `make vi-dev` tại `/ch7/`: bấm thử xref hình và footnote; lấy mẫu 10 thuật ngữ, xác nhận có term tiếng Anh ngay tại chỗ (SC-005)
- [x] T014 Xóa `draft: true` và `book_status: draft` khỏi front matter `content/vi/ch7.md`; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/ch7/` tồn tại

## Dependencies

T001 → T002 → T003 → … → T014 tuần tự. Không có task song song vì cùng sửa một file.

## Implementation Strategy

MVP = đến hết T009 (chương đọc được, thuật ngữ đúng); T010–T014 là cổng kiểm chứng và xuất bản. Commit sau T009 và sau T014.
