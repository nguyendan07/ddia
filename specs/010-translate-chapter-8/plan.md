# Implementation Plan: Dịch chương 8

**Branch**: `010-translate-chapter-8` | **Date**: 2026-11-28 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/010-translate-chapter-8/spec.md`

## Summary

Dịch tại chỗ `content/vi/ch8.md` (bản sao zh, đang `draft: true`) sang tiếng Việt theo quy ước `*thuật ngữ* (*english term*)` ở lần đầu trong trang, đối chiếu `content/en/ch8.md`. Không viết code mới: dùng `bin/check-vi.py`, `bin/glossary-vi.py`, `make vi-check`. Term thiếu thì thêm vào `data/glossary/vi.toml` trước khi dùng. Xong thì bỏ `draft` và `book_status`.

## Technical Context

**Language/Version**: Markdown + shortcode OINK; Python 3.12 (script kiểm tra có sẵn)

**Primary Dependencies**: Hugo 0.170.1, theme OINK v1.6.0 (không đổi)

**Storage**: file — `content/vi/ch8.md`, `data/glossary/vi.toml`

**Testing**: `python3 bin/check-vi.py ch8`, `python3 bin/glossary-vi.py check`, `make vi-check`

**Target Platform**: GitHub Pages dưới `/ddia/`

**Project Type**: site tĩnh / bản dịch sách

**Constraints**: cấu trúc 1:1 với zh (đo được — 14 `fig`, 35 `xref`, 2 `eg` (8-1, 8-2), 85 định nghĩa footnote, 5 khối mã). ID/URL giữ nguyên; front matter bất biến trừ `title` (đã khớp `data/home/vi.yaml`).

**Scale/Scope**: 1 trang (~1147 dòng) — chương 8 của Phần II, dài nhất trong các chương đã dịch.

## Constitution Check

| Nguyên tắc | Đánh giá |
|---|---|
| I. Trung thành với nguồn | Đạt — dịch từ zh, đối chiếu en, en thắng khi lệch |
| II. Tương đương 1:1 | Đạt — sửa tại chỗ; `check-vi.py` kiểm |
| III. Thuật ngữ nhất quán | Đạt — term mới vào bảng trước; `show_en` luôn kèm Anh |
| IV. Kiểm chứng tự động | Đạt — `make vi-check` + `glossary-vi.py check` trước merge |
| V. Tái sử dụng upstream | Đạt — không thêm code, không sửa file upstream |

Không vi phạm; không cần Complexity Tracking. Re-check sau thiết kế: vẫn đạt.

## Project Structure

```text
specs/010-translate-chapter-8/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── tasks.md        # /speckit-tasks
```

(Không có `contracts/`: không thêm giao diện mới.)

```text
content/vi/ch8.md          # dịch tại chỗ, bỏ draft khi xong
data/glossary/vi.toml      # chỉ bổ sung term còn thiếu / sửa nếu lệch
```

**Structure Decision**: chỉ sửa các file trên. Chương dài (5 mục `##`) nên dịch theo từng mục, chạy `check-vi.py ch8` sau mỗi mục.
