# Data Model: Dịch chương 3

- **Trang** (`content/vi/ch3.md`): front matter (`title` đã khớp; `translationKey`, `weight`, `book_*` giữ), cờ `draft`/`book_status`, thân Markdown với shortcode `fig`/`xref`/`eg`, anchor, footnote, khối mã.
  - Bất biến: số heading, `fig`, `xref`, `eg`, footnote bằng zh; ID, URL và nội dung khối mã không đổi.
  - Chuyển trạng thái: `draft` → `published` khi `check-vi.py` và `glossary-vi.py check` sạch.
- **Mục thuật ngữ** (`data/glossary/vi.toml`): như feature 003 — `en`, `vi`, `keep`, `show_en`, `ctx`, `note`, `status`; một `en` một bản dịch; cặp ở lần đầu trong trang phải khớp.
