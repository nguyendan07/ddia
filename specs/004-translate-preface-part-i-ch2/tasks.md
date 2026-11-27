# Tasks: Dịch lời nói đầu, Phần I và chương 2

**Input**: `specs/004-translate-preface-part-i-ch2/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/<slug>.md` vào `content/vi/<slug>.md`, đối chiếu `content/en/<slug>.md` (en thắng khi lệch); dạng `*thuật ngữ* (*english term*)` ở lần đầu trong mỗi trang; giữ nguyên ID/anchor/shortcode/URL/footnote/tên riêng; chỉ dịch `caption` và text hiển thị; front matter chỉ đổi `title`.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py preface part-i ch2` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (số heading/fig/xref/footnote theo zh) của ba file trong `content/vi/`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term: liệt kê các cặp `*术语*（*en*）` trong `content/zh/{preface,part-i,ch2}.md`, đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; term quen dùng tiếng Anh thì `keep = true`)

## Phase 3: User Story 1+2 — Ba trang đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn bộ văn bản là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/{preface,part-i,ch2}.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Các file khác nhau nên T003–T004 song song được; các task trong `ch2.md` làm tuần tự.

- [x] T003 [P] [US1] Dịch `content/vi/part-i.md` (zh 53 dòng; giữ `title` "Nền tảng của hệ thống dữ liệu", tên chương khớp `data/home/vi.yaml`)
- [x] T004 [P] [US1] Dịch `content/vi/preface.md`: mục đối tượng độc giả, phạm vi, tổng quan sách, tham khảo, Safari, liên hệ, lời cảm ơn (zh dòng 38–127; giữ nguyên tên riêng, URL, email)
- [x] T005 [US2] Dịch `content/vi/ch2.md` phần mở đầu (trích dẫn, front matter `title` "Định nghĩa yêu cầu phi chức năng", giới thiệu) và mục `sec_introduction_twitter` kèm `id20`, `sec_introduction_materializing` (zh dòng 1–81; caption hình/ví dụ `fig`/`eg`)
- [x] T006 [US2] Dịch mục `sec_introduction_percentiles`: `id23`, `id24`, `sec_introduction_slo_sla` (zh dòng 82–172; giữ nguyên p50/p99/SLA/SLO)
- [x] T007 [US2] Dịch mục `sec_introduction_reliability`: `id27`, `sec_introduction_hardware_faults` (kèm hai mục con), `id31` (zh dòng 173–269)
- [x] T008 [US2] Dịch mục `sec_introduction_scalability` (`id33`, `sec_introduction_shared_nothing`, `id35`) và `sec_introduction_maintainability` (`id37`, `id38`, `sec_introduction_evolvability`) (zh dòng 270–386)
- [x] T009 [US2] Dịch mục `summary` và danh sách `### 参考文献` (giữ nguyên số, URL, `[^N]`) (zh dòng 387–496)
- [x] T010 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong ba file, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py preface part-i ch2` và `make vi-check` thoát 0.

- [x] T011 [US3] Chạy `python3 bin/check-vi.py preface part-i ch2`; sửa chênh lệch heading/fig/xref/anchor/URL/footnote trong `content/vi/{preface,part-i,ch2}.md`
- [x] T012 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [x] T013 Đọc lại ba trang đối chiếu `content/en/`: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên; ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch"; các lần xuất hiện sau của thuật ngữ chỉ dùng tiếng Việt (trừ `show_en`) (FR-002, FR-007) (đã đối chiếu số liệu ch2 với `content/en/ch2.md` và các hộp Tác động thời gian phản hồi, Horizon: khớp, không phải sửa; chưa đối chiếu từng câu cho preface/part-i)
- [ ] T014 Xem `make vi-dev` tại `/preface/`, `/part-i/`, `/ch2/`: bấm thử xref hình/bảng và footnote; lấy mẫu 10 thuật ngữ mỗi trang, xác nhận có term tiếng Anh ngay tại chỗ (SC-005)
- [x] T015 Xóa `draft: true` và `book_status: draft` khỏi front matter của ba file trong `content/vi/`; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/{preface,part-i,ch2}/` tồn tại

## Dependencies

T001 → T002 → {T003 ∥ T004 ∥ (T005 → T006 → T007 → T008 → T009)} → T010 → T011 → T012 → T013 → T014 → T015. Các task T005–T009 cùng sửa `ch2.md` nên không song song.

## Implementation Strategy

MVP = đến hết T010 (ba trang đọc được, thuật ngữ đúng); T011–T015 là cổng kiểm chứng và xuất bản. Commit sau T010 và sau T015.
