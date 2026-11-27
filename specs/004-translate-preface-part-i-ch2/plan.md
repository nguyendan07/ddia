# Implementation Plan: Dịch lời nói đầu, Phần I và chương 2

**Branch**: `004-translate-preface-part-i-ch2` | **Date**: 2026-11-27 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/004-translate-preface-part-i-ch2/spec.md`

## Summary

Dịch tại chỗ `content/vi/preface.md`, `content/vi/part-i.md`, `content/vi/ch2.md` (bản sao zh, đang `draft: true`) sang tiếng Việt theo quy ước `*thuật ngữ* (*english term*)` ở lần đầu mỗi trang, đối chiếu `content/en/`. Không viết code mới: dùng nguyên `bin/check-vi.py`, `bin/glossary-vi.py`, `make vi-check`. Term thiếu thì thêm vào `data/glossary/vi.toml` trước khi dùng. Xong từng trang thì bỏ `draft` và `book_status`.

## Technical Context

**Language/Version**: Markdown + shortcode OINK; Python 3.12 (script kiểm tra có sẵn)

**Primary Dependencies**: Hugo 0.170.1, theme OINK v1.6.0 (không đổi)

**Storage**: file — ba trang trong `content/vi/`, `data/glossary/vi.toml`

**Testing**: `python3 bin/check-vi.py preface part-i ch2`, `python3 bin/glossary-vi.py check`, `make vi-check`

**Target Platform**: GitHub Pages dưới `/ddia/`

**Project Type**: site tĩnh / bản dịch sách

**Constraints**: cấu trúc 1:1 với zh (ch2: ~6 hình, ~8 xref, nhiều footnote; preface/part-i không hình/xref); ID/URL giữ nguyên; front matter bất biến trừ `title`. Số chính xác do `check-vi.py` so với zh.

**Giới hạn kiểm tra**: `bin/glossary-vi.py` chỉ quét `content/vi/ch*.md`; `preface` và `part-i` kiểm thuật ngữ thủ công (không sửa script ở feature này, theo Nguyên tắc V).

**Scale/Scope**: 3 trang, ~680 dòng (ch2 ≈ 500 dòng chiếm phần lớn)

## Constitution Check

| Nguyên tắc | Đánh giá |
|---|---|
| I. Trung thành với nguồn | Đạt — dịch từ zh, đối chiếu en, en thắng khi lệch; lời cảm ơn/tên riêng giữ nguyên |
| II. Tương đương 1:1 | Đạt — sửa tại chỗ; `check-vi.py` kiểm |
| III. Thuật ngữ nhất quán | Đạt — term mới vào bảng trước; `show_en` luôn kèm Anh |
| IV. Kiểm chứng tự động | Đạt — `make vi-check` + `glossary-vi.py check` trước merge |
| V. Tái sử dụng upstream | Đạt — không thêm code, không sửa file upstream |

Không có vi phạm; không cần Complexity Tracking. Re-check sau thiết kế: vẫn đạt.

## Project Structure

### Documentation (this feature)

```text
specs/004-translate-preface-part-i-ch2/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── tasks.md        # /speckit-tasks
```

(Không có `contracts/`: không thêm giao diện mới.)

### Source Code (repository root)

```text
content/vi/preface.md
content/vi/part-i.md
content/vi/ch2.md          # dịch tại chỗ, bỏ draft khi xong
data/glossary/vi.toml      # chỉ bổ sung term còn thiếu / sửa nếu lệch
```

**Structure Decision**: chỉ sửa các file trên; không tạo file nguồn mới. Thứ tự: part-i (ngắn) → preface → ch2 (lớn, dịch theo mục).
