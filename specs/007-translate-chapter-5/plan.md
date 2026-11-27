# Implementation Plan: Dịch chương 5

**Branch**: `007-translate-chapter-5` | **Date**: 2026-11-28 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/007-translate-chapter-5/spec.md`

## Summary

Dịch tại chỗ `content/vi/ch5.md` (bản sao zh, đang `draft: true`) sang tiếng Việt theo quy ước `*thuật ngữ* (*english term*)` ở lần đầu trong trang, đối chiếu `content/en/ch5.md`. Không viết code mới: dùng `bin/check-vi.py`, `bin/glossary-vi.py`, `make vi-check`. Term thiếu thì thêm vào `data/glossary/vi.toml` trước khi dùng. Xong thì bỏ `draft` và `book_status`.

## Technical Context

**Language/Version**: Markdown + shortcode OINK; Python 3.12 (script kiểm tra có sẵn)

**Primary Dependencies**: Hugo 0.170.1, theme OINK v1.6.0 (không đổi)

**Storage**: file — `content/vi/ch5.md`, `data/glossary/vi.toml`

**Testing**: `python3 bin/check-vi.py ch5`, `python3 bin/glossary-vi.py check`, `make vi-check`

**Target Platform**: GitHub Pages dưới `/ddia/`

**Project Type**: site tĩnh / bản dịch sách

**Constraints**: cấu trúc 1:1 với zh (ch5 đo được: 7 `fig`, 18 `xref`, 5 `eg`, 51 định nghĩa footnote, 16 dòng rào mã ```). Mã ví dụ (JSON, Protobuf, Avro, schema…) giữ nguyên, chỉ dịch chú thích trong mã nếu có và `caption`. ID/URL giữ nguyên; front matter bất biến trừ `title` (đã khớp `data/home/vi.yaml`).

**Scale/Scope**: 1 trang, 653 dòng — chương cuối Phần I.

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
specs/007-translate-chapter-5/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── tasks.md        # /speckit-tasks
```

(Không có `contracts/`: không thêm giao diện mới.)

```text
content/vi/ch5.md          # dịch tại chỗ, bỏ draft khi xong
data/glossary/vi.toml      # chỉ bổ sung term còn thiếu / sửa nếu lệch
```

**Structure Decision**: chỉ sửa hai file trên. Dịch theo từng mục `##`, chạy `check-vi.py ch5` sau mỗi mục lớn.
