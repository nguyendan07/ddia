# Quickstart: kiểm chứng

Điều kiện: Python 3.12+, Hugo 0.170.1, Go (như `make vi-check`).

1. **Cấu trúc**: `python3 bin/check-vi.py ch8` → thoát 0.
2. **Thuật ngữ**: `python3 bin/glossary-vi.py check` → thoát 0, không cảnh báo `mismatch`/`show-en` cho ch8.
3. **Không còn chữ Hán**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch8.md` → không dòng nào (ngoài URL/tên riêng).
4. **Xem thử**: `make vi-dev` → đọc `/ch8/`, bấm thử xref hình và footnote.
5. **Cổng cuối**: bỏ `draft`/`book_status` ở ch8, chạy `make vi-check` → OK; `make vi-build` → `public/ch8/` tồn tại.
