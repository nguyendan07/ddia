# Quickstart: kiểm chứng chương 1

Điều kiện: Python 3.12+, Hugo 0.170.1, Go (như `make vi-check`).

1. **Kiểm cấu trúc**: `python3 bin/check-vi.py ch1` → thoát 0 (heading/hình/xref/footnote bằng zh).
2. **Kiểm thuật ngữ**: `python3 bin/glossary-vi.py check` → thoát 0, không cảnh báo `mismatch`/`show-en` cho `ch1`.
3. **Không còn chữ Hán**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/ch1.md` → không có dòng nào (ngoài URL/tên riêng nếu có).
4. **Xem thử**: `make vi-dev` → mở `/ch1/`, đọc suốt chương, bấm thử các tham chiếu hình.
5. **Cổng cuối**: bỏ `draft`/`book_status`, chạy `make vi-check` → OK; `make vi-build` → `public/ch1/` tồn tại.
