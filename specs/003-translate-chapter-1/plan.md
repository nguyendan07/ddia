# Implementation Plan: Dịch chương 1

**Branch**: `003-translate-chapter-1` | **Date**: 2026-11-27 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/003-translate-chapter-1/spec.md`

## Summary

Dịch tại chỗ `content/vi/ch1.md` (bản sao zh, đang `draft: true`) sang tiếng Việt theo quy ước `*thuật ngữ* (*english term*)` ở lần đầu, đối chiếu `content/en/ch1.md`. Không viết code mới: dùng nguyên `bin/check-vi.py`, `bin/glossary-vi.py`, `make vi-check`. Term thiếu thì thêm vào `data/glossary/vi.toml` trước khi dùng. Xong thì bỏ `draft` và `book_status`.

## Technical Context

**Language/Version**: Markdown + shortcode OINK; Python 3.12 (script kiểm tra có sẵn)

**Primary Dependencies**: Hugo 0.170.1, theme OINK v1.6.0 (không đổi)

**Storage**: file — `content/vi/ch1.md`, `data/glossary/vi.toml`

**Testing**: `python3 bin/check-vi.py ch1`, `python3 bin/glossary-vi.py check`, `make vi-check`

**Target Platform**: GitHub Pages dưới `/ddia/`

**Project Type**: site tĩnh / bản dịch sách

**Performance Goals**: N/A

**Constraints**: cấu trúc 1:1 với zh (19 heading, 2 hình, 4 xref, ID/URL giữ nguyên); front matter bất biến trừ `title`

**Scale/Scope**: 1 chương, ~470 dòng

## Constitution Check

| Nguyên tắc | Đánh giá |
|---|---|
| I. Trung thành với nguồn | Đạt — dịch từ zh, đối chiếu en, en thắng khi lệch; ghi chú người dịch tách riêng |
| II. Tương đương 1:1 | Đạt — sửa tại chỗ file đã có; `check-vi.py` kiểm |
| III. Thuật ngữ nhất quán | Đạt — đúng dạng yêu cầu; term mới vào bảng trước; `show_en` luôn kèm Anh |
| IV. Kiểm chứng tự động | Đạt — `make vi-check` + `glossary-vi.py check` trước merge |
| V. Tái sử dụng upstream | Đạt — không thêm file code, không sửa file upstream |

Không có vi phạm; không cần Complexity Tracking. Re-check sau thiết kế: vẫn đạt.

## Project Structure

### Documentation (this feature)

```text
specs/003-translate-chapter-1/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── tasks.md        # /speckit-tasks
```

(Không có `contracts/`: không thêm giao diện mới.)

### Source Code (repository root)

```text
content/vi/ch1.md          # dịch tại chỗ, bỏ draft khi xong
data/glossary/vi.toml      # chỉ bổ sung term còn thiếu / sửa nếu lệch
```

**Structure Decision**: chỉ sửa hai file trên; không tạo file nguồn mới.
