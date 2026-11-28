# Implementation Plan: Dịch chương 10

**Branch**: `012-translate-chapter-10` | **Date**: 2026-11-29 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/012-translate-chapter-10/spec.md`

## Summary

Dịch tại chỗ `content/vi/ch10.md` (bản sao zh, đang `draft: true`) sang tiếng Việt theo quy ước `*thuật ngữ* (*english term*)` ở lần đầu trong trang, đối chiếu `content/en/ch10.md`. Không viết code mới: dùng `bin/check-vi.py`, `bin/glossary-vi.py`, `make vi-check`. Term thiếu thì thêm vào `data/glossary/vi.toml` trước khi dùng. Xong thì bỏ `draft` và `book_status`.

## Technical Context

**Language/Version**: Markdown + shortcode OINK; Python 3.12 (script kiểm tra có sẵn)

**Primary Dependencies**: Hugo 0.170.1, theme OINK v1.6.0 (không đổi)

**Storage**: file — `content/vi/ch10.md`, `data/glossary/vi.toml`

**Testing**: `python3 bin/check-vi.py ch10`, `python3 bin/glossary-vi.py check`, `make vi-check`

**Target Platform**: GitHub Pages dưới `/ddia/`

**Project Type**: site tĩnh / bản dịch sách

**Constraints**: cấu trúc 1:1 với zh (đo được — 10 `fig`, 24 dòng `xref`, 0 `eg`, 0 `tbl`, 91 định nghĩa footnote, 0 khối mã). ID/URL giữ nguyên; front matter bất biến trừ `title` (đã khớp `data/home/vi.yaml`).

**Scale/Scope**: 1 trang (~848 dòng) — chương 10 của Phần II, dài nhất trong các chương đã dịch.

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
specs/012-translate-chapter-10/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── tasks.md        # /speckit-tasks
```

(Không có `contracts/`: không thêm giao diện mới.)

```text
content/vi/ch10.md          # dịch tại chỗ, bỏ draft khi xong
data/glossary/vi.toml      # chỉ bổ sung term còn thiếu / sửa nếu lệch
```

**Structure Decision**: chỉ sửa các file trên. Chương dài (4 mục `##`: Linearizability, Bộ sinh ID và đồng hồ logic, Đồng thuận, Tóm tắt) nên dịch theo từng mục, chạy `check-vi.py ch10` sau mỗi mục.
