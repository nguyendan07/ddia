# Research: Dịch lời nói đầu, Phần I và chương 2

Kế thừa quyết định của [feature 003](../003-translate-chapter-1/research.md) (quy ước thuật ngữ, xử lý ngoặc full-width, dịch tại chỗ không dùng `bin/translate.py`, thêm term thiếu vào bảng trước, bỏ cờ nháp ở cuối). Bổ sung:

- **Phạm vi thuật ngữ cho preface/part-i** — Decision: chỉ áp dụng cặp `*vi* (*en*)` cho term có trong bảng; văn xuôi giới thiệu ít term nên chấp nhận ít cặp. Alternatives: bổ sung term hàng loạt (vượt phạm vi).
- **Tên riêng và lời cảm ơn trong preface** — Decision: giữ nguyên tên người/tổ chức/sản phẩm; chỉ dịch câu văn. Rationale: constitution I.
- **Chương 2 (nhiều footnote, 98 dòng định nghĩa `[^N]`)** — Decision: giữ nguyên số footnote và URL, chỉ dịch phần chữ; dịch theo từng mục rồi chạy `check-vi.py ch2` sau mỗi mục lớn để bắt lệch sớm.
- **Viết tắt/chỉ số (SLA, SLO, p50/p99…)** — Decision: giữ nguyên tiếng Anh.
- **Tên chương trong `part-i`** — Decision: khớp `data/home/vi.yaml`.

Không còn NEEDS CLARIFICATION.
