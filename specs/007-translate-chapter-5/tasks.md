# Tasks: Dịch chương 5

**Input**: `specs/007-translate-chapter-5/` (plan, spec, research, data-model, quickstart)
**Tests**: không thêm test mới; dùng kiểm tra có sẵn (`check-vi.py`, `glossary-vi.py`, `make vi-check`).
**Quy ước chung cho mọi task dịch**: dịch từ `content/zh/ch5.md` vào `content/vi/ch5.md`, đối chiếu `content/en/ch5.md` (en thắng khi lệch); dạng `*thuật ngữ* (*english term*)` ở lần đầu trong trang; giữ nguyên ID/anchor/shortcode/URL/footnote/tên riêng và toàn bộ nội dung khối mã (JSON, Protobuf, Avro schema…); chỉ dịch `caption` và text hiển thị; front matter chỉ đổi `title` (đã khớp `data/home/vi.yaml`). Các cặp `*术语*（*en*）` → ngoặc ASCII.

## Phase 1: Setup

- [x] T001 Chạy `python3 bin/check-vi.py ch5` và `python3 bin/glossary-vi.py check` để ghi lại trạng thái xuất phát (7 fig, 18 xref, 5 eg, 51 định nghĩa footnote theo zh) của `content/vi/ch5.md`

## Phase 2: Foundational

- [x] T002 Chuẩn bị term (đã rà thêm các cặp có chú thích kèm trong ngoặc, thêm 10 term vào bảng): liệt kê các cặp `*术语*（*en*）` trong `content/zh/ch5.md`, đối chiếu `data/glossary/vi.toml`; thêm mục còn thiếu theo `specs/002-glossary-en-vi/contracts/glossary-file.md` (một bản dịch/term; tên ngôn ngữ/sản phẩm/viết tắt thì `keep = true`)

## Phase 3: User Story 1+2 — Chương 5 đọc được bằng tiếng Việt, thuật ngữ đúng dạng (P1) 🎯 MVP

**Goal**: toàn bộ văn bản là tiếng Việt, thuật ngữ lần đầu ghi `*vi* (*en*)`.
**Independent Test**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch5.md` rỗng; `python3 bin/glossary-vi.py check` thoát 0.

Tất cả task cùng sửa `content/vi/ch5.md` nên làm tuần tự; chạy `python3 bin/check-vi.py ch5` sau mỗi task để bắt lệch sớm.

- [x] T003 [US1] Dịch phần mở đầu chương (trích dẫn, giới thiệu) và `sec_encoding_formats` đến hết `id96` (đặc thù ngôn ngữ) (zh dòng 1–78)
- [x] T004 [US2] Dịch `sec_encoding_json` (kèm `json-schema`, `binary-encoding`) (zh dòng 79–151; giữ nguyên mã JSON/hex)
- [x] T005 [US2] Dịch `sec_encoding_protobuf` (kèm `field-tags-and-schema-evolution`) và `sec_encoding_avro` (kèm `the-writers-schema-and-the-readers-schema`, `schema-evolution-rules`, `but-what-is-the-writers-schema`, `dynamically-generated-schemas`) (zh dòng 152–298)
- [x] T006 [US2] Dịch `sec_encoding_schemas` và mở đầu `sec_encoding_dataflow`, `sec_encoding_dataflow_db` (kèm `different-values-written-at-different-times`, `archival-storage`) (zh dòng 299–358)
- [x] T007 [US2] Dịch `sec_encoding_dataflow_rpc` (kèm `sec_web_services`, `sec_problems_with_rpc`, `sec_encoding_service_discovery`, `data-encoding-and-evolution-for-rpc`) (zh dòng 359–477)
- [x] T008 [US2] Dịch `sec_encoding_dataflow_workflows` (kèm `durable-execution`) và `sec_encoding_dataflow_msg` (kèm `message-brokers`, `distributed-actor-frameworks`) (zh dòng 478–575)
- [x] T009 [US2] Dịch `summary` và danh sách `### 参考文献` (giữ nguyên số, URL, `[^N]`) (zh dòng 576–653)
- [x] T010 [US2] Chạy `python3 bin/glossary-vi.py check`; sửa mọi `mismatch`/`show-en` trong `content/vi/ch5.md`, thêm term thiếu vào `data/glossary/vi.toml`

**Checkpoint**: không còn chữ Hán, thuật ngữ khớp bảng.

## Phase 4: User Story 3 — Cấu trúc 1:1 và qua kiểm tra tự động (P2)

**Goal**: không chênh lệch cấu trúc, build và link sạch.
**Independent Test**: `python3 bin/check-vi.py ch5` và `make vi-check` thoát 0.

- [x] T011 [US3] Chạy `python3 bin/check-vi.py ch5`; sửa chênh lệch heading/fig/xref/eg/anchor/URL/footnote trong `content/vi/ch5.md`; xác nhận các khối mã không đổi so với zh (quickstart bước 4)
- [x] T012 [US3] Chạy `make vi-check`; sửa lỗi build/link phát sinh (front matter YAML không có `: ` trần, base path `/ddia/`)

## Phase 5: Polish & xuất bản

- [x] T013 Đọc lại `content/vi/ch5.md` đối chiếu `content/en/ch5.md`: sửa chỗ lệch nghĩa (en thắng), văn phong tự nhiên; ghi chú người dịch (nếu có) đánh dấu "Ghi chú người dịch"; các lần xuất hiện sau của thuật ngữ chỉ dùng tiếng Việt (trừ `show_en`) (FR-002, FR-007)
- [ ] T014 Xem `make vi-dev` tại `/ch5/`: bấm thử xref hình/ví dụ và footnote; lấy mẫu 10 thuật ngữ, xác nhận có term tiếng Anh ngay tại chỗ (SC-005)
- [x] T015 Xóa `draft: true` và `book_status: draft` khỏi front matter `content/vi/ch5.md`; chạy lại `make vi-check` và `make vi-build`, xác nhận `public/ch5/` tồn tại

## Dependencies

T001 → T002 → T003 → … → T015 tuần tự. Không có task song song vì cùng sửa một file.

## Implementation Strategy

MVP = đến hết T010 (chương đọc được, thuật ngữ đúng); T011–T015 là cổng kiểm chứng và xuất bản. Commit sau T010 và sau T015.
