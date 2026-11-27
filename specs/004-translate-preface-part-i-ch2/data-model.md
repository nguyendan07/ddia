# Data Model: Dịch lời nói đầu, Phần I và chương 2

- **Trang** (`content/vi/{preface,part-i,ch2}.md`): front matter (`title` đổi; `translationKey`, `weight`, `book_*` giữ), cờ `draft`/`book_status` (nháp → xuất bản), thân Markdown với shortcode `fig`/`xref`, anchor, footnote.
  - Bất biến: số heading, `fig`, `xref`, footnote bằng zh; ID và URL không đổi.
  - Chuyển trạng thái (mỗi trang độc lập): `draft` → `published` khi `check-vi.py` và `glossary-vi.py check` sạch cho trang đó.
- **Mục thuật ngữ** (`data/glossary/vi.toml`): như feature 003 — `en`, `vi`, `keep`, `show_en`, `ctx`, `note`, `status`; một `en` một bản dịch; cặp ở lần đầu trong trang phải khớp.
