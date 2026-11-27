# Feature Specification: Dịch chương 1 (Đánh đổi trong kiến trúc hệ thống dữ liệu)

**Feature Branch**: `003-translate-chapter-1`

**Created**: 2026-11-27

**Status**: Draft

**Input**: User description: "Dịch chương đầu tiên theo dạng *thuật ngữ* (*english term*)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Người đọc đọc được chương 1 bằng tiếng Việt (Priority: P1)

Người đọc là dân IT/DEV mở chương 1 trên site và đọc toàn bộ chương bằng tiếng Việt tự nhiên, không còn chữ Hán, với đủ hình, bảng, tham chiếu chéo và chú thích như bản gốc.

**Why this priority**: Đây là chương mở đầu của sách; không có nó, site vi chưa có nội dung thật nào để xuất bản.

**Independent Test**: Build site, mở trang chương 1; đọc từ đầu đến cuối không gặp đoạn tiếng Trung, không link gãy, không hình thiếu.

**Acceptance Scenarios**:

1. **Given** chương 1 đã dịch và xuất bản, **When** người đọc mở trang, **Then** toàn bộ văn bản (kể cả chú thích hình, tiêu đề, trích dẫn đầu chương) là tiếng Việt.
2. **Given** chương có tham chiếu chéo tới hình/bảng/chương khác, **When** người đọc bấm vào, **Then** đến đúng vị trí.
3. **Given** chương còn ở trạng thái nháp, **When** build xuất bản, **Then** chương không lộ ra; chỉ lộ khi đã dịch xong và bỏ cờ nháp.

---

### User Story 2 - Thuật ngữ ghi dạng *thuật ngữ* (*english term*) (Priority: P1)

Ở lần đầu một thuật ngữ kỹ thuật xuất hiện trong chương, người đọc thấy bản dịch Việt in nghiêng kèm term tiếng Anh gốc trong ngoặc, in nghiêng: `*cơ sở dữ liệu* (*database*)`. Các lần sau chỉ dùng tiếng Việt (trừ mục `show_en` trong bảng thuật ngữ, luôn kèm tiếng Anh).

**Why this priority**: Là yêu cầu trọng tâm của người dùng và Nguyên tắc III của constitution; dân IT cần biết term gốc để tra tài liệu.

**Independent Test**: Chạy kiểm tra thuật ngữ của dự án trên chương 1: mọi cặp `*thuật ngữ* (*english term*)` ở lần đầu đều khớp bảng thuật ngữ, không cảnh báo thiếu tiếng Anh với mục `show_en`.

**Acceptance Scenarios**:

1. **Given** một thuật ngữ kỹ thuật xuất hiện lần đầu trong chương, **When** người đọc gặp nó, **Then** thấy dạng `*thuật ngữ* (*english term*)` dùng ngoặc ASCII, bản Việt khớp bảng thuật ngữ.
2. **Given** thuật ngữ đó xuất hiện lần sau, **When** người đọc gặp lại, **Then** chỉ thấy bản Việt (hoặc kèm tiếng Anh nếu là mục `show_en`).
3. **Given** thuật ngữ chưa có trong bảng, **When** người dịch cần dùng, **Then** thuật ngữ được thêm vào bảng trước (một bản dịch duy nhất) rồi mới dùng trong chương.

---

### User Story 3 - Chương 1 tương đương cấu trúc với bản zh và qua kiểm tra tự động (Priority: P2)

Người bảo trì chạy kiểm tra của dự án và nhận kết quả sạch cho chương 1: cấu trúc 1:1 với bản zh, build và kiểm tra link thành công.

**Why this priority**: Nguyên tắc II và IV của constitution; cho phép đồng bộ khi upstream đổi.

**Independent Test**: `make vi-check` và `python3 bin/glossary-vi.py check` đều qua với chương 1.

**Acceptance Scenarios**:

1. **Given** chương 1 đã dịch, **When** chạy kiểm tra cấu trúc, **Then** số heading, hình, tham chiếu chéo, chú thích bằng bản zh; ID, anchor, thuộc tính shortcode và URL không đổi.
2. **Given** bản dịch dùng thuật ngữ khác bảng, **When** chạy kiểm tra thuật ngữ, **Then** bị chặn kèm chỉ rõ vị trí.

---

### Edge Cases

- Bản zh và bản gốc tiếng Anh lệch nghĩa → theo bản tiếng Anh (constitution I).
- Thuật ngữ không có bản dịch phổ biến (tên sản phẩm, viết tắt như OLTP, ETL) → giữ nguyên tiếng Anh, không tự chế.
- Thuật ngữ xuất hiện lần đầu trong tiêu đề, chú thích hình hoặc trích dẫn → áp dụng cùng quy ước ở lần xuất hiện đầu trong văn xuôi, không làm hỏng anchor tiêu đề.
- Giá trị front matter chứa `: ` không có ngoặc → phải bọc ngoặc kép hoặc dùng `—`.
- Câu zh dùng ngoặc full-width `（ ）` quanh term → chuyển thành ngoặc ASCII ở bản vi.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Toàn bộ văn bản chương 1 (trích dẫn đầu chương, thân bài, tiêu đề, chú thích hình, đoạn mô tả bảng) MUST được dịch sang tiếng Việt; không còn ký tự Hán ngoài URL/tên riêng.
- **FR-002**: Ở lần xuất hiện đầu tiên của mỗi thuật ngữ kỹ thuật trong chương, văn bản MUST ghi `*thuật ngữ* (*english term*)` với ngoặc ASCII; các lần sau chỉ dùng thuật ngữ tiếng Việt, trừ mục `show_en` luôn kèm tiếng Anh.
- **FR-003**: Mọi cặp thuật ngữ dùng trong chương MUST khớp bảng thuật ngữ chung (`data/glossary/vi.toml`); term mới MUST được thêm vào bảng trước khi dùng.
- **FR-004**: Thuật ngữ không có bản dịch phổ biến MUST giữ nguyên tiếng Anh theo chính sách của bảng.
- **FR-005**: Cấu trúc MUST tương đương 1:1 với bản zh: số heading, hình, tham chiếu chéo, chú thích bằng nhau; mọi anchor/ID, shortcode và thuộc tính, số footnote, URL giữ nguyên; chỉ dịch `caption` và text hiển thị.
- **FR-006**: Front matter MUST giữ nguyên `translationKey`, `weight`, `book_kind`, `book_number`, `book_part`; chỉ `title` là tiếng Việt, khớp tên chương ở dữ liệu trang chủ.
- **FR-007**: Nội dung kỹ thuật MUST trung thành với nguồn, không thêm/bớt/diễn giải lại; mọi ghi chú của người dịch phải đánh dấu rõ.
- **FR-008**: Khi dịch xong, trang MUST bỏ cờ nháp để được xuất bản, và chạy qua các kiểm tra tự động của dự án (cấu trúc, thuật ngữ, build, link) mà không lỗi.
- **FR-009**: Văn phong MUST là tiếng Việt tự nhiên, không dịch word-by-word từ tiếng Trung.

### Key Entities

- **Trang chương 1**: nội dung chương gồm các mục, hình, tham chiếu chéo và chú thích; có trạng thái nháp/đã xuất bản.
- **Mục thuật ngữ**: cặp Anh–Việt trong bảng chung, có thể kèm cờ `show_en` và ghi chú ngữ cảnh.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% đoạn văn của chương 1 hiển thị bằng tiếng Việt; 0 ký tự Hán ngoài URL/tên riêng.
- **SC-002**: 100% thuật ngữ ở lần xuất hiện đầu có dạng `*thuật ngữ* (*english term*)` và khớp bảng thuật ngữ; 0 cảnh báo `show-en` còn lại.
- **SC-003**: 0 chênh lệch cấu trúc so với bản zh (heading, hình, xref, footnote) và 0 link gãy khi build.
- **SC-004**: Bộ kiểm tra của dự án chạy sạch cho chương 1 trước khi bỏ cờ nháp.
- **SC-005**: Người đọc IT/DEV lấy mẫu 10 thuật ngữ bất kỳ trong chương và tìm được term tiếng Anh gốc ngay tại chỗ, không cần mở trang khác, ở 10/10 trường hợp.

## Assumptions

- Nguồn dịch là `content/zh/ch1.md` (đang là bản sao nháp ở `content/vi/ch1.md`), đối chiếu `content/en/ch1.md` khi mơ hồ.
- Phạm vi chỉ gồm chương 1; các chương và trang khác (preface, glossary, …) nằm ngoài phạm vi.
- Bảng thuật ngữ đã được điền cho các chương 1–14 (feature 002); chỉ bổ sung term còn thiếu.
- "Thuật ngữ kỹ thuật" là term có trong bảng thuật ngữ; từ thông thường không cần kèm tiếng Anh.
- Tên chương tiếng Việt đã chốt: "Đánh đổi trong kiến trúc hệ thống dữ liệu".
- Hình ảnh raster dùng lại nguyên của upstream, không dịch.
