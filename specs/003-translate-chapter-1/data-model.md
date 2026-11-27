# Data Model: Dịch chương 1

- **Trang chương** (`content/vi/ch1.md`): front matter (`title` đổi; `translationKey`, `weight`, `book_*` giữ), cờ `draft`/`book_status` (nháp → xuất bản), thân Markdown với shortcode `fig`/`xref`, anchor, footnote.
  - Bất biến: số heading = 19, `fig` = 2, `xref` = 4 (theo zh); ID và URL không đổi.
  - Chuyển trạng thái: `draft` → `published` khi `vi-check` và `glossary-vi.py check` sạch.
- **Mục thuật ngữ** (`data/glossary/vi.toml`): `en`, `vi`, `keep`, `show_en`, `ctx`, `note`, `status`. Quy tắc: một `en` một bản dịch (trừ `ctx`); cặp ở lần đầu trong chương phải khớp `en`/`vi`.
