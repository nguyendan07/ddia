# Quickstart: kiểm chứng

Điều kiện: Python 3.12+, Hugo 0.170.1, Go (như `make vi-check`).

1. **Cấu trúc**: `python3 bin/check-vi.py ch5` → thoát 0.
2. **Thuật ngữ**: `python3 bin/glossary-vi.py check` → thoát 0, không cảnh báo `mismatch`/`show-en` cho ch5.
3. **Không còn chữ Hán**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch5.md` → không dòng nào (ngoài URL/tên riêng).
4. **Mã không đổi**: so các khối ``` của `content/vi/ch5.md` với zh (trừ chú thích) — không lệch.
5. **Xem thử**: `make vi-dev` → đọc `/ch5/`, bấm thử xref hình/ví dụ và footnote.
6. **Cổng cuối**: bỏ `draft`/`book_status`, chạy `make vi-check` → OK; `make vi-build` → `public/ch5/` tồn tại.
