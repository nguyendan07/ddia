# Quickstart: kiểm chứng

Điều kiện: Python 3.12+, Hugo 0.170.1, Go (như `make vi-check`).

1. **Cấu trúc**: `python3 bin/check-vi.py preface part-i ch2` → thoát 0.
2. **Thuật ngữ**: `python3 bin/glossary-vi.py check` → thoát 0, không cảnh báo `mismatch`/`show-en` cho ba trang.
3. **Không còn chữ Hán**: `grep -nP '[\x{4e00}-\x{9fff}]' content/vi/{preface,part-i,ch2}.md` → không dòng nào (ngoài URL/tên riêng).
4. **Xem thử**: `make vi-dev` → đọc `/preface/`, `/part-i/`, `/ch2/`, bấm thử xref hình/bảng và footnote.
5. **Cổng cuối**: bỏ `draft`/`book_status` từng trang, chạy `make vi-check` → OK; `make vi-build` → `public/{preface,part-i,ch2}/` tồn tại.
