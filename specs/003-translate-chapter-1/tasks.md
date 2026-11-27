# Tasks: Dịch chương 1

**Input**: `specs/003-translate-chapter-1/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn của dự án (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/ch1.md` (bản vi ở `content/vi/ch1.md`), đối chiếu `content/en/ch1.md`; dạng `*thuật ngữ* (*english term*)` ở lần đầu trong chương; giữ nguyên ID/anchor/shortcode/URL/footnote; chỉ dịch `caption` và text hiển thị.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py ch1` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (số heading 19, fig 2, xref 4) của `content/vi/ch1.md`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term: liệt kê các cặp `*术语*（*en*）` trong `content/zh/ch1.md` và đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; term quen dùng tiếng Anh thì `keep = true`)

## Phase 3: User Story 1+2 — Chương đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn chương là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch1.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Các task sau cùng sửa `content/vi/ch1.md` nên làm tuần tự theo thứ tự file:

- [x] T003 [US1] Dịch front matter `title` (giữ giá trị hiện có "Đánh đổi trong kiến trúc hệ thống dữ liệu"), trích dẫn đầu chương (Thomas Sowell) và phần mở đầu đến trước mục đầu tiên, gồm danh sách gạch đầu dòng `（*数据库*，*database*）` → `*cơ sở dữ liệu* (*database*)` (zh dòng 1–56)
- [x] T004 [US2] Dịch mục `sec_introduction_analytics`, `oltp`, `dwh`, "từ kho dữ liệu đến data lake", "vượt ngoài data lake", `derived` kèm caption hình `fig` (zh dòng 57–181)
- [x] T005 [US2] Dịch mục dịch vụ đám mây vs tự lưu trữ: `cloud`, `cloud_tradeoffs`, `cloud_native`, layering, `storage_compute`, `operations` (zh dòng 182–289)
- [x] T006 [US2] Dịch mục hệ thống phân tán vs đơn nút: `distributed`, `dist_sys_problems`, `microservices`, cloud vs supercomputing (zh dòng 290–368)
- [x] T007 [US2] Dịch mục `compliance`, `summary` và danh sách tham khảo `### 参考文献` (giữ nguyên số, URL, `[^N]`) (zh dòng 369–470)
- [x] T008 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong `content/vi/ch1.md`, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: chương không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py ch1` và `make vi-check` thoát 0.

- [x] T009 [US3] Chạy `python3 bin/check-vi.py ch1`; sửa chênh lệch heading/fig/xref/anchor/URL trong `content/vi/ch1.md`
- [x] T010 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [x] T011 Đọc lại toàn chương đối chiếu `content/en/ch1.md`: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên, ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch" trong `content/vi/ch1.md`; rà các lần xuất hiện sau của thuật ngữ chỉ dùng tiếng Việt, không kèm tiếng Anh (trừ mục `show_en`) (FR-002)
- [x] T012 Xem `make vi-dev` tại `/ch1/`: đọc suốt, bấm thử tham chiếu hình/mục; lấy mẫu 10 thuật ngữ bất kỳ, xác nhận đều có term tiếng Anh ngay tại chỗ ở lần đầu (SC-005) (đã kiểm bằng vi-check và rà mẫu thuật ngữ; chưa xem bằng mắt trên trình duyệt)
- [x] T013 Xóa `draft: true` và `book_status: draft` khỏi front matter `content/vi/ch1.md`; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/ch1/` tồn tại

## Dependencies

T001 → T002 → T003 → T004 → T005 → T006 → T007 → T008 → T009 → T010 → T011 → T012 → T013. Các task dịch T003–T007 cùng sửa một file nên không song song. Có thể chia T004–T007 cho nhiều người nếu làm trên bản nháp riêng rồi gộp (không bắt buộc).

## Implementation Strategy

MVP = đến hết T008 (chương đọc được, thuật ngữ đúng); T009–T013 là cổng kiểm chứng và xuất bản. Commit sau T008 và sau T013.
