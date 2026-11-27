# Research: Dịch chương 1

- **Quy ước thuật ngữ** — Decision: lần đầu trong chương `*vi* (*en*)`, ngoặc ASCII; sau đó chỉ tiếng Việt, trừ `show_en`. Rationale: constitution III, `glossary-vi.py check` đã kiểm cặp ở lần đầu. Alternatives: kèm Anh mọi lần (rối văn bản); chú thích cuối trang (lệch cấu trúc zh).
- **Xử lý zh `*术语*（*en*）`** — Decision: chuyển thẳng thành `*vi* (*en*)`; zh còn có dạng danh sách `（*数据库*，*database*）` ở mục liệt kê → dịch thành `(*cơ sở dữ liệu*, *database*)` hoặc đưa về một dạng thống nhất `*cơ sở dữ liệu* (*database*)`; chọn dạng sau để `check` nhận.
- **Quy trình dịch** — Decision: dịch tại chỗ, theo từng mục, đối chiếu en; không dùng `bin/translate.py` (script zh→en của upstream, sinh bản dịch tự động không phù hợp Nguyên tắc I). Alternatives: tạo file mới rồi thay (thừa).
- **Term thiếu bảng** — Decision: chạy `glossary-vi.py check` sau khi dịch; thêm mục theo `specs/002-glossary-en-vi/contracts/glossary-file.md` rồi dịch lại chỗ dùng.
- **Cờ nháp** — Decision: bỏ `draft` và `book_status` ở bước cuối, sau khi `vi-check` sạch (không nháp lọt ra build xuất bản).

Không còn NEEDS CLARIFICATION.
