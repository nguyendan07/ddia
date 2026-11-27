---
translationKey: ddia-v2-part-i
title: Nền tảng của hệ thống dữ liệu
book_kind: part
book_number: I
search_exclude: true
weight: 100
breadcrumbs: false
---

Năm chương đầu giới thiệu các khái niệm nền tảng ở tầng dưới của hệ thống dữ liệu, áp dụng cho cả hệ thống đơn lẻ chạy trên một máy lẫn hệ thống phân tán trải trên nhiều máy.

1. [Chương 1](/ch1) bàn về *những đánh đổi trong kiến trúc hệ thống dữ liệu*. Ta sẽ thảo luận các loại hệ thống dữ liệu khác nhau (chẳng hạn hệ thống phân tích và hệ thống tác nghiệp) cùng cách chúng vận hành trong môi trường đám mây.
2. [Chương 2](/ch2) trình bày cách định nghĩa các yêu cầu phi chức năng. *Độ tin cậy, khả năng mở rộng và khả năng bảo trì* thực sự có nghĩa là gì? Làm thế nào để đạt được những mục tiêu này?
3. [Chương 3](/ch3) so sánh một số *mô hình dữ liệu và ngôn ngữ truy vấn*. Từ góc nhìn của lập trình viên, đây là điểm khác biệt dễ thấy nhất giữa các cơ sở dữ liệu. Mỗi mô hình dữ liệu phù hợp với những tình huống ứng dụng khác nhau.
4. [Chương 4](/ch4) đi sâu vào bên trong *storage engine*, xem cơ sở dữ liệu sắp xếp dữ liệu trên đĩa như thế nào. Mỗi công cụ lưu trữ được tối ưu cho một loại tải khác nhau, và việc chọn đúng công cụ ảnh hưởng rất lớn đến hiệu năng của hệ thống.
5. [Chương 5](/ch5) so sánh một số *định dạng mã hóa dữ liệu*. Chương đặc biệt xem xét các định dạng này hoạt động ra sao trong môi trường mà yêu cầu của ứng dụng thường xuyên thay đổi và lược đồ phải tiến hóa theo thời gian.

[Phần II](/part-ii) sẽ dành riêng cho những vấn đề đặc thù của *hệ thống dữ liệu phân tán*.


## [1. Đánh đổi trong kiến trúc hệ thống dữ liệu](/ch1)
- [Hệ thống phân tích và hệ thống tác nghiệp](/ch1#sec_introduction_analytics)
- [Dịch vụ đám mây và self-hosted](/ch1#sec_introduction_cloud)
- [Hệ thống phân tán và hệ thống một node](/ch1#sec_introduction_distributed)
- [Hệ thống dữ liệu, pháp luật và xã hội](/ch1#sec_introduction_compliance)
- [Tóm tắt](/ch1#summary)

## [2. Định nghĩa yêu cầu phi chức năng](/ch2)
- [Nghiên cứu tình huống: dòng thời gian trang chủ của mạng xã hội](/ch2#sec_introduction_twitter)
- [Mô tả hiệu năng](/ch2#sec_introduction_percentiles)
- [Độ tin cậy và khả năng chịu lỗi](/ch2#sec_introduction_reliability)
- [Khả năng mở rộng](/ch2#sec_introduction_scalability)
- [Khả năng bảo trì](/ch2#sec_introduction_maintainability)
- [Tóm tắt](/ch2#summary)

## [3. Mô hình dữ liệu và ngôn ngữ truy vấn](/ch3)
- [Mô hình quan hệ và mô hình tài liệu](/ch3#sec_datamodels_history)
- [Mô hình dữ liệu đồ thị](/ch3#sec_datamodels_graph)
- [Event sourcing và CQRS](/ch3#sec_datamodels_events)
- [Dataframe, ma trận và mảng](/ch3#sec_datamodels_dataframes)
- [Tóm tắt](/ch3#summary)

## [4. Lưu trữ và truy xuất](/ch4)
- [Lưu trữ và lập chỉ mục cho hệ thống OLTP](/ch4#sec_storage_oltp)
- [Kho lưu trữ dữ liệu phân tích](/ch4#sec_storage_analytics)
- [Chỉ mục đa chiều và chỉ mục toàn văn](/ch4#sec_storage_multidimensional)
- [Tóm tắt](/ch4#summary)

## [5. Mã hóa và tiến hóa](/ch5)
- [Các định dạng mã hóa dữ liệu](/ch5#sec_encoding_formats)
- [Các mô hình luồng dữ liệu](/ch5#sec_encoding_dataflow)
- [Tóm tắt](/ch5#summary)
