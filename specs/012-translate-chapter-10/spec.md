# Feature Specification: Dịch chương 10 (Nhất quán và đồng thuận)

**Feature Branch**: `012-translate-chapter-10`

**Created**: 2026-11-29

**Status**: Draft

**Input**: User description: "Dịch chương 10 theo dạng *thuật ngữ* (*english term*)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Người đọc đọc được chương 10 bằng tiếng Việt (Priority: P1)

Người đọc là dân IT/DEV mở `ch10` trên site và đọc toàn bộ bằng tiếng Việt tự nhiên, không còn chữ Hán, với đủ hình, bảng, tham chiếu chéo, chú thích và mục tham khảo như bản gốc.

**Why this priority**: Chương 1–9 đã dịch; chương 10 là chương tiếp theo (Nhất quán và đồng thuận).

**Independent Test**: Build site, mở `ch10`; đọc từ đầu đến cuối không gặp đoạn tiếng Trung, không link gãy, không hình thiếu.

**Acceptance Scenarios**:

1. **Given** `ch10` đã dịch và xuất bản, **When** người đọc mở trang, **Then** toàn bộ văn bản (tiêu đề, chú thích hình, trích dẫn, chú thích cuối trang) là tiếng Việt.
2. **Given** trang có tham chiếu chéo tới hình/bảng/chương khác, **When** người đọc bấm vào, **Then** đến đúng vị trí.
3. **Given** trang còn nháp, **When** build xuất bản, **Then** trang không lộ ra; chỉ lộ khi dịch xong và bỏ cờ nháp.

---

### User Story 2 - Thuật ngữ ghi dạng *thuật ngữ* (*english term*) (Priority: P1)

Ở lần đầu một thuật ngữ kỹ thuật xuất hiện trong trang, người đọc thấy bản Việt in nghiêng kèm term tiếng Anh gốc trong ngoặc ASCII, in nghiêng: `*độ trễ* (*latency*)`. Các lần sau chỉ dùng tiếng Việt (trừ mục `show_en` trong bảng thuật ngữ, luôn kèm tiếng Anh).

**Why this priority**: Yêu cầu trọng tâm của người dùng và Nguyên tắc III của constitution.

**Independent Test**: `python3 bin/glossary-vi.py check` qua với `ch10`; mọi cặp ở lần đầu khớp bảng thuật ngữ.

**Acceptance Scenarios**:

1. **Given** thuật ngữ kỹ thuật xuất hiện lần đầu trong một trang, **When** người đọc gặp nó, **Then** thấy `*thuật ngữ* (*english term*)`, bản Việt khớp bảng.
2. **Given** thuật ngữ đó xuất hiện lần sau trong cùng trang, **Then** chỉ thấy bản Việt (hoặc kèm tiếng Anh nếu `show_en`).
3. **Given** thuật ngữ chưa có trong bảng, **When** người dịch cần dùng, **Then** thêm vào bảng trước rồi mới dùng.

---

### User Story 3 - Cấu trúc tương đương bản zh và qua kiểm tra tự động (Priority: P2)

Người bảo trì chạy kiểm tra và nhận kết quả sạch: cấu trúc 1:1 với bản zh, build và kiểm tra link thành công.

**Why this priority**: Nguyên tắc II và IV của constitution; cho phép đồng bộ khi upstream đổi.

**Independent Test**: `make vi-check` và `python3 bin/glossary-vi.py check` đều qua.

**Acceptance Scenarios**:

1. **Given** `ch10` đã dịch, **When** chạy kiểm tra cấu trúc, **Then** số heading, hình, xref, footnote bằng bản zh; ID, anchor, thuộc tính shortcode, URL không đổi.
2. **Given** bản dịch dùng thuật ngữ khác bảng, **When** chạy kiểm tra thuật ngữ, **Then** bị chặn kèm chỉ rõ vị trí.

---

### Edge Cases

- Bản zh và bản gốc tiếng Anh lệch nghĩa → theo bản tiếng Anh.
- Thuật ngữ không có bản dịch phổ biến (tên sản phẩm, viết tắt như SLA, p99) → giữ nguyên tiếng Anh.
- Thuật ngữ lần đầu trong tiêu đề, chú thích hình hay trích dẫn → áp dụng cùng quy ước, không làm hỏng anchor tiêu đề.
- Chương 10 không có khối mã, shortcode `eg` hay `tbl`; chỉ cần giữ nguyên `fig`/`xref`, anchor và thuộc tính.
- Chương dài, nhiều bảng, hình và footnote → mọi số hình/bảng/footnote giữ nguyên.
- Giá trị front matter chứa `: ` không có ngoặc → bọc ngoặc kép hoặc dùng `—`.
- Ngoặc full-width `（ ）` quanh term trong zh → chuyển thành ngoặc ASCII.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Toàn bộ văn bản của `ch10` (thân bài, tiêu đề, chú thích hình/bảng, chú thích cuối trang) MUST được dịch sang tiếng Việt; không còn ký tự Hán ngoài URL/tên riêng.
- **FR-002**: Ở lần xuất hiện đầu của mỗi thuật ngữ kỹ thuật trong từng trang, văn bản MUST ghi `*thuật ngữ* (*english term*)` với ngoặc ASCII; các lần sau chỉ dùng tiếng Việt, trừ mục `show_en`.
- **FR-003**: Mọi thuật ngữ dùng MUST khớp bảng chung `data/glossary/vi.toml`; term mới MUST được thêm vào bảng trước khi dùng.
- **FR-004**: Thuật ngữ không có bản dịch phổ biến MUST giữ nguyên tiếng Anh theo chính sách của bảng.
- **FR-005**: Cấu trúc MUST tương đương 1:1 với bản zh: heading, hình, xref, footnote bằng nhau; anchor/ID, shortcode, thuộc tính, số footnote, URL giữ nguyên; chỉ dịch `caption` và text hiển thị.
- **FR-006**: Front matter MUST giữ nguyên `translationKey`, `weight`, `book_kind`, `book_number`, `book_part`; chỉ `title` là tiếng Việt, khớp dữ liệu trang chủ.
- **FR-007**: Nội dung kỹ thuật MUST trung thành với nguồn, không thêm/bớt/diễn giải lại; ghi chú của người dịch phải đánh dấu rõ.
- **FR-008**: Khi dịch xong, trang MUST bỏ cờ nháp và qua kiểm tra tự động (cấu trúc, thuật ngữ, build, link) không lỗi.
- **FR-009**: Văn phong MUST là tiếng Việt tự nhiên, không dịch word-by-word từ tiếng Trung.

### Key Entities

- **Trang nội dung** (`ch10`): văn bản, hình, xref, chú thích; trạng thái nháp/đã xuất bản.
- **Mục thuật ngữ**: cặp Anh–Việt trong bảng chung, có thể kèm `show_en` và ghi chú ngữ cảnh.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% đoạn văn của chương hiển thị bằng tiếng Việt; 0 ký tự Hán ngoài URL/tên riêng.
- **SC-002**: 100% thuật ngữ ở lần đầu trong ch10 có dạng `*thuật ngữ* (*english term*)` và khớp bảng, 0 cảnh báo `show-en` (kiểm tự động).
- **SC-003**: 0 chênh lệch cấu trúc so với bản zh và 0 link gãy khi build.
- **SC-004**: Bộ kiểm tra của dự án chạy sạch trước khi bỏ cờ nháp.
- **SC-005**: Lấy mẫu 10 thuật ngữ bất kỳ chương, người đọc tìm được term tiếng Anh gốc ngay tại chỗ ở 10/10 trường hợp.

## Assumptions

- Nguồn dịch là `content/zh/ch10.md` (đang là bản sao nháp ở `content/vi/`), đối chiếu `content/en/` khi mơ hồ.
- Phạm vi chỉ gồm `ch10`; các trang khác nằm ngoài phạm vi.
- Bảng thuật ngữ đã điền cho ch1–14 (feature 002); chỉ bổ sung term còn thiếu.
- "Thuật ngữ kỹ thuật" là term có trong bảng; từ thông thường không cần kèm tiếng Anh.
- Tên chương 10 tiếng Việt ("Nhất quán và đồng thuận") lấy từ `data/home/vi.yaml`.
- Hình ảnh raster dùng lại nguyên của upstream.
