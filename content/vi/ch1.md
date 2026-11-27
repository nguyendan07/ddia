---
translationKey: ddia-v2-ch1
title: Đánh đổi trong kiến trúc hệ thống dữ liệu
book_kind: chapter
book_number: "1"
book_part: I
weight: 101
breadcrumbs: false
---

<a id="ch_tradeoffs"></a>

> *Không có giải pháp nào, chỉ có những sự đánh đổi. […] Bạn chỉ có thể cố gắng đạt được sự đánh đổi tốt nhất có thể, và cũng chỉ có thể hy vọng như vậy.*
>
> [Thomas Sowell](https://www.youtube.com/watch?v=2YUtKr8-_Fg), trả lời phỏng vấn Fred Barnes (2005)

Ngày nay, dữ liệu nằm ở trung tâm của rất nhiều ứng dụng. Cùng với sự phổ biến của ứng dụng web, ứng dụng di động, phần mềm như một dịch vụ (SaaS) và các dịch vụ đám mây, việc lưu dữ liệu của nhiều người dùng trong một hạ tầng dữ liệu dùng chung phía máy chủ đã trở nên quen thuộc. Dữ liệu từ hoạt động của người dùng, giao dịch kinh doanh, thiết bị và cảm biến đều cần được lưu lại để phục vụ phân tích. Khi người dùng tương tác với ứng dụng, họ vừa đọc dữ liệu đã lưu, vừa tạo ra thêm dữ liệu.

Một lượng dữ liệu nhỏ có thể lưu trữ và xử lý trên một máy duy nhất, thường khá dễ đối phó. Tuy nhiên, khi khối lượng dữ liệu hoặc tốc độ truy vấn tăng lên, dữ liệu cần được phân bổ trên nhiều máy, kéo theo nhiều thách thức. Khi yêu cầu của ứng dụng trở nên phức tạp hơn, việc đặt toàn bộ dữ liệu vào một hệ thống cũng không còn đủ. Có thể cần kết hợp nhiều hệ thống lưu trữ hoặc xử lý với những khả năng khác nhau.

Nếu quản lý dữ liệu là một trong những thách thức chính khi phát triển ứng dụng, ta gọi đó là ứng dụng *chuyên sâu về dữ liệu* (*data-intensive*) [^1]. Với hệ thống *chuyên sâu về tính toán* (*compute-intensive*), khó khăn nằm ở việc song song hóa một phép tính cực lớn, còn với ứng dụng chuyên sâu về dữ liệu, ta thường quan tâm hơn đến việc lưu trữ và xử lý lượng dữ liệu khổng lồ, quản lý các thay đổi của dữ liệu, bảo đảm tính nhất quán khi có lỗi và đồng thời xảy ra, cũng như duy trì tính sẵn sàng cao của dịch vụ.

Các ứng dụng như vậy thường được xây dựng từ những khối chuẩn, cung cấp các chức năng thông dụng. Chẳng hạn, nhiều ứng dụng cần:

* Lưu dữ liệu để chính chúng, hoặc ứng dụng khác, có thể tìm lại sau này (*cơ sở dữ liệu* (*database*))
* Ghi nhớ kết quả của một thao tác tốn kém để đọc nhanh hơn (*cache*)
* Cho phép người dùng tìm dữ liệu theo từ khóa, hoặc lọc dữ liệu theo nhiều cách (*chỉ mục tìm kiếm* (*search index*))
* Xử lý ngay khi có sự kiện hoặc thay đổi dữ liệu (*xử lý luồng* (*stream processing*))
* Định kỳ xử lý một lượng lớn dữ liệu đã tích lũy (*xử lý theo lô* (*batch processing*))

Khi xây dựng ứng dụng, ta thường chọn vài hệ thống phần mềm hoặc dịch vụ (ví dụ cơ sở dữ liệu và API) rồi ghép chúng lại bằng mã ứng dụng. Nếu mục đích sử dụng của bạn đúng với điều các hệ thống dữ liệu này vốn được thiết kế cho, việc đó có thể khá dễ dàng.

Nhưng khi mục tiêu của ứng dụng ngày càng cao, thách thức cũng xuất hiện. Có rất nhiều hệ thống cơ sở dữ liệu với đặc tính khác nhau, phù hợp với những mục đích khác nhau — nên chọn cái nào? Cache có nhiều cách làm, chỉ mục tìm kiếm (*search index*) cũng có nhiều cách xây dựng, và còn nhiều lựa chọn tương tự — làm sao cân nhắc? Bạn phải xác định công cụ và cách tiếp cận nào phù hợp nhất với công việc trước mắt, và khi một việc không thể do một công cụ đơn lẻ đảm nhiệm, việc kết hợp nhiều công cụ cũng có thể khó khăn.

Cuốn sách này sẽ giúp bạn quyết định dùng công nghệ nào và kết hợp chúng ra sao. Như bạn sẽ thấy, không có cách tiếp cận nào vượt trội hẳn so với các cách còn lại. Mỗi phương án đều có ưu và nhược điểm. Cuốn sách sẽ dạy bạn đặt những câu hỏi đúng để đánh giá và so sánh các hệ thống dữ liệu, từ đó tìm ra phương án đáp ứng tốt nhất nhu cầu của một ứng dụng cụ thể.

Hành trình của chúng ta bắt đầu từ một số cách điển hình mà các tổ chức ngày nay sử dụng dữ liệu. Nhiều ý tưởng ở đây bắt nguồn từ *phần mềm doanh nghiệp* (*enterprise software*), tức nhu cầu phần mềm và thực tiễn kỹ thuật của các tổ chức lớn như tập đoàn và cơ quan chính phủ. Vì trước đây, chỉ các tổ chức lớn mới có lượng dữ liệu đồ sộ đến mức cần những giải pháp công nghệ phức tạp, còn khi dữ liệu đủ nhỏ thì cứ lưu bằng bảng tính! Tuy nhiên, những năm gần đây, các công ty nhỏ và startup quản lý khối lượng dữ liệu lớn, xây dựng hệ thống chuyên sâu về dữ liệu, cũng đã trở nên rất phổ biến.

Một thách thức then chốt của hệ thống dữ liệu là: những người khác nhau cần làm những việc hoàn toàn khác nhau với dữ liệu. Trong một công ty, bạn và nhóm của bạn có những ưu tiên riêng. Một nhóm khác, dù đang xử lý cùng một dữ liệu, có thể có mục tiêu hoàn toàn khác. Hơn nữa, các mục tiêu này thường không được nói rõ, nên rất dễ gây hiểu lầm và tranh cãi về hướng đi đúng.

Để giúp bạn hiểu các lựa chọn, chương này so sánh một số cặp khái niệm đối lập và bàn về sự đánh đổi giữa chúng:

* Hệ thống tác nghiệp và hệ thống phân tích khác nhau thế nào ([“Hệ thống phân tích và hệ thống tác nghiệp”](/ch1#sec_introduction_analytics));
* Dịch vụ đám mây và hệ thống self-hosted có những ưu nhược điểm gì ([“Dịch vụ đám mây và self-hosted”](/ch1#sec_introduction_cloud));
* Khi nào nên chuyển từ hệ thống một node sang hệ thống phân tán ([“Hệ thống phân tán và hệ thống một node”](/ch1#sec_introduction_distributed)); và
* Cân bằng giữa nhu cầu kinh doanh và quyền của người dùng ra sao ([“Hệ thống dữ liệu, pháp luật và xã hội”](/ch1#sec_introduction_compliance)).

Ngoài ra, chương này cũng cung cấp các thuật ngữ cần thiết để đọc phần còn lại của cuốn sách.

> [!TIP] Thuật ngữ: frontend và backend
>
> Nhiều nội dung trong cuốn sách này liên quan đến *phát triển backend* (*backend development*). Lấy ứng dụng web làm ví dụ, mã phía client chạy trong trình duyệt được gọi là *frontend*, còn mã phía máy chủ xử lý yêu cầu của người dùng được gọi là *backend*. Ứng dụng di động giống frontend ở chỗ: chúng cung cấp giao diện người dùng và thường giao tiếp với backend phía máy chủ qua Internet. Đôi khi frontend cũng quản lý dữ liệu cục bộ trên thiết bị của người dùng [^2], nhưng thách thức lớn nhất của hạ tầng dữ liệu thường nằm ở backend: frontend chỉ xử lý dữ liệu của một người dùng, còn backend quản lý dữ liệu thay mặt *tất cả* người dùng.
>
> Dịch vụ backend thường truy cập được qua HTTP (đôi khi là WebSocket). Nó thường gồm một số mã ứng dụng đọc ghi dữ liệu trong một hoặc nhiều cơ sở dữ liệu, và đôi khi tương tác với các hệ thống dữ liệu khác như cache, hàng đợi tin nhắn. Các hệ thống này có thể gọi chung là *hạ tầng dữ liệu* (*data infrastructure*). Mã ứng dụng thường là *stateless*, nghĩa là sau khi xử lý xong một yêu cầu HTTP, nó quên mọi thứ về yêu cầu đó. Bất kỳ thông tin nào cần giữ lại giữa các yêu cầu đều phải lưu ở client hoặc trong hạ tầng dữ liệu phía máy chủ.


## Hệ thống phân tích và hệ thống tác nghiệp {#sec_introduction_analytics}

Nếu bạn làm việc với hệ thống dữ liệu trong một doanh nghiệp, rất có thể bạn sẽ gặp vài nhóm người làm việc với dữ liệu. Nhóm đầu tiên là *kỹ sư backend* (*backend engineer*), người xây dựng các dịch vụ xử lý yêu cầu đọc và cập nhật dữ liệu. Các dịch vụ này thường phục vụ trực tiếp người dùng bên ngoài, hoặc gián tiếp thông qua các dịch vụ khác (xem [“Microservices và serverless”](/ch1#sec_introduction_microservices)), đôi khi chúng chỉ dành cho các bộ phận khác trong tổ chức dùng.

Ngoài các nhóm quản lý dịch vụ backend, thường còn hai nhóm người khác cần truy cập dữ liệu của tổ chức: *chuyên viên phân tích nghiệp vụ* (*business analyst*) tạo báo cáo dựa trên hoạt động của tổ chức để giúp ban quản lý ra quyết định tốt hơn — đây là *business intelligence* (BI); còn *nhà khoa học dữ liệu* (*data scientist*) tìm kiếm những hiểu biết mới từ dữ liệu, hoặc dùng phân tích dữ liệu và học máy/AI để xây dựng các tính năng hướng tới người dùng, chẳng hạn gợi ý kiểu “người mua X cũng mua Y” trên trang thương mại điện tử, phân tích dự đoán như chấm điểm rủi ro hay lọc thư rác, và xếp hạng kết quả tìm kiếm.

Chuyên viên phân tích nghiệp vụ và nhà khoa học dữ liệu dùng các công cụ khác nhau và làm việc theo cách khác nhau, nhưng vẫn có điểm chung: cả hai đều thực hiện *phân tích dữ liệu* (*analytics*), nghĩa là xem xét dữ liệu do người dùng và các dịch vụ backend tạo ra, nhưng thường không sửa đổi dữ liệu đó (có thể trừ việc sửa lỗi). Họ có thể tạo các tập dữ liệu dẫn xuất, xử lý dữ liệu gốc theo một cách nào đó. Điều này dẫn đến hai loại hệ thống tách biệt nhau — sự phân biệt mà cuốn sách này sẽ dùng xuyên suốt:

* *Hệ thống tác nghiệp* (*operational system*) gồm các dịch vụ backend tạo ra dữ liệu và hạ tầng dữ liệu đi kèm, ví dụ phục vụ trực tiếp người dùng bên ngoài. Mã ứng dụng đọc và sửa dữ liệu trong cơ sở dữ liệu dựa trên các thao tác của người dùng.
* *Hệ thống phân tích* (*analytical system*) phục vụ chuyên viên phân tích nghiệp vụ và nhà khoa học dữ liệu. Chúng giữ bản sao chỉ đọc của dữ liệu từ các hệ thống tác nghiệp, và được tối ưu cho các kiểu xử lý dữ liệu mà phân tích cần.

Như phần tiếp theo sẽ giải thích, hệ thống tác nghiệp và hệ thống phân tích thường có lý do chính đáng để tách rời nhau. Khi hai loại hệ thống này ngày càng trưởng thành, hai vai trò chuyên biệt mới xuất hiện: *kỹ sư dữ liệu* (*data engineer*) và *analytics engineer*. Kỹ sư dữ liệu biết cách tích hợp hệ thống tác nghiệp với hệ thống phân tích, và chịu trách nhiệm rộng hơn về hạ tầng dữ liệu của tổ chức [^3]. Analytics engineer mô hình hóa và biến đổi dữ liệu để chuyên viên phân tích nghiệp vụ và nhà khoa học dữ liệu trong tổ chức dễ dùng hơn [^4].

Nhiều kỹ sư chuyên về một trong hai loại hệ thống. Tuy nhiên, cuốn sách này sẽ đề cập cả hai, vì cả hai đều đóng vai trò quan trọng trong vòng đời dữ liệu của một tổ chức. Chúng ta sẽ đi sâu vào hạ tầng dữ liệu cần có để phục vụ người dùng nội bộ và bên ngoài, giúp bạn phối hợp tốt hơn với đồng nghiệp ở phía bên kia ranh giới.

### Đặc điểm của xử lý giao dịch và phân tích {#sec_introduction_oltp}

Trong những ngày đầu của xử lý dữ liệu thương mại, mỗi lần ghi vào cơ sở dữ liệu thường tương ứng với một *giao dịch kinh doanh*: hoàn tất một lần bán hàng, đặt hàng với nhà cung cấp, trả lương nhân viên, v.v. Về sau, cơ sở dữ liệu được dùng cho cả những lĩnh vực không liên quan đến tiền bạc, nhưng tên gọi *giao dịch* (*transaction*) vẫn được giữ lại để chỉ một nhóm thao tác đọc và ghi tạo thành một đơn vị logic.

> [!NOTE]
> [Chương 8](/ch8#ch_transactions) sẽ bàn chi tiết ý nghĩa của “giao dịch”. Chương này dùng từ đó theo nghĩa rộng, chỉ các thao tác đọc và ghi có độ trễ thấp.

Mặc dù cơ sở dữ liệu bắt đầu được dùng cho đủ loại dữ liệu — bài đăng mạng xã hội, hành động trong trò chơi, liên hệ trong sổ địa chỉ, v.v. — mô hình truy cập cơ bản vẫn giống với việc xử lý giao dịch kinh doanh. Hệ thống tác nghiệp thường tra cứu một lượng nhỏ bản ghi theo một khóa nào đó (gọi là *truy vấn điểm*, *point query*), rồi chèn, cập nhật hoặc xóa bản ghi dựa trên đầu vào của người dùng. Vì các ứng dụng này có tính tương tác, mô hình truy cập đó được gọi là *xử lý giao dịch trực tuyến* (OLTP).

Trong khi đó, cơ sở dữ liệu cũng ngày càng được dùng để phân tích, và mô hình truy cập của phân tích rất khác OLTP. Truy vấn phân tích thường quét một lượng lớn bản ghi và tính các thống kê tổng hợp như đếm, tổng hoặc trung bình, thay vì trả từng bản ghi cho người dùng. Chẳng hạn, chuyên viên phân tích nghiệp vụ của một chuỗi siêu thị có thể muốn trả lời các câu hỏi sau:

* Tổng doanh thu của từng cửa hàng trong tháng Một là bao nhiêu?
* Trong đợt khuyến mãi gần đây, chúng ta bán được nhiều hơn bình thường bao nhiêu chuối?
* Nhãn hiệu thức ăn trẻ em nào thường được mua cùng nhất với tã nhãn hiệu X?

Báo cáo do các truy vấn này tạo ra là cơ sở quan trọng của business intelligence, giúp ban quản lý quyết định hành động tiếp theo. Để phân biệt kiểu dùng cơ sở dữ liệu này với xử lý giao dịch, người ta gọi nó là *xử lý phân tích trực tuyến* (OLAP) [^5]. Ranh giới giữa OLTP và phân tích không phải lúc nào cũng rõ ràng, nhưng {{< xref tbl="1-1" page="/ch1" anchor="tab_oltp_vs_olap" >}}Bảng 1-1{{< /xref >}} liệt kê một số đặc điểm điển hình của hai loại này.

| Thuộc tính | Hệ thống tác nghiệp (OLTP) | Hệ thống phân tích (OLAP) |
|------------|----------------------------|---------------------------|
| Kiểu đọc chính | Truy vấn điểm (đọc từng bản ghi theo khóa) | Tổng hợp trên nhiều bản ghi |
| Kiểu ghi chính | Tạo, cập nhật và xóa từng bản ghi | Nhập hàng loạt (ETL) hoặc luồng sự kiện |
| Ví dụ người dùng là con người | Người dùng cuối của ứng dụng web hoặc di động | Chuyên viên phân tích nội bộ, hỗ trợ ra quyết định |
| Ví dụ mục đích sử dụng của máy | Kiểm tra một thao tác có được ủy quyền hay không | Phát hiện mẫu gian lận hoặc lạm dụng |
| Kiểu truy vấn | Một tập truy vấn cố định, do ứng dụng định sẵn | Chuyên viên phân tích có thể truy vấn tùy ý |
| Dữ liệu biểu diễn | Trạng thái mới nhất của dữ liệu (thời điểm hiện tại) | Lịch sử các sự kiện đã xảy ra theo thời gian |
| Kích thước tập dữ liệu | GB đến TB | TB đến PB |
{#tab_oltp_vs_olap num="1-1" caption="So sánh đặc điểm của hệ thống tác nghiệp và hệ thống phân tích"}

> [!NOTE]
> Nghĩa của *trực tuyến* (online) trong OLAP không rõ ràng. Có lẽ nó ám chỉ rằng truy vấn không chỉ dùng cho các báo cáo định sẵn, mà chuyên viên phân tích còn dùng hệ thống OLAP một cách tương tác để truy vấn khám phá.

Hệ thống tác nghiệp thường không cho phép người dùng tự viết truy vấn SQL rồi gửi cho cơ sở dữ liệu thực thi, vì như vậy họ có thể đọc hoặc sửa dữ liệu mà họ không có quyền truy cập. Người dùng cũng có thể viết những truy vấn có chi phí thực thi cao, ảnh hưởng đến việc dùng cơ sở dữ liệu của người dùng khác. Vì vậy, hệ thống OLTP chủ yếu thực thi một tập truy vấn cố định được viết sẵn trong mã ứng dụng, chỉ thỉnh thoảng mới chạy truy vấn tùy chỉnh một lần khi bảo trì hoặc xử lý sự cố. Cơ sở dữ liệu phân tích thì khác: thường cho phép người dùng tự do viết tay các truy vấn SQL tùy ý, hoặc tự động sinh truy vấn thông qua các công cụ trực quan hóa dữ liệu hay bảng điều khiển như Tableau, Looker hoặc Microsoft Power BI.

Còn có một loại hệ thống được thiết kế cho tải phân tích (tức truy vấn tổng hợp nhiều bản ghi) nhưng lại nhúng vào sản phẩm hướng tới người dùng. Cách dùng này gọi là *phân tích sản phẩm* (*product analytics*) hoặc *phân tích thời gian thực* (*real-time analytics*). Các hệ thống được thiết kế cho mục đích đó gồm Pinot, Druid và ClickHouse [^6].

### Kho dữ liệu {#sec_introduction_dwh}

Ban đầu, cùng một cơ sở dữ liệu được dùng cho cả xử lý giao dịch lẫn truy vấn phân tích. SQL tỏ ra rất linh hoạt, làm tốt cả hai loại truy vấn. Tuy nhiên, vào cuối những năm 1980 và đầu những năm 1990, các doanh nghiệp bắt đầu ngừng dùng hệ thống OLTP để phân tích, mà chuyển sang chạy truy vấn phân tích trên một hệ thống cơ sở dữ liệu riêng. Cơ sở dữ liệu riêng này được gọi là *kho dữ liệu* (*data warehouse*).

Một doanh nghiệp lớn có thể có hàng chục, thậm chí hàng trăm hệ thống xử lý giao dịch trực tuyến: hệ thống hỗ trợ website hướng tới khách hàng, hệ thống điều khiển điểm bán hàng (máy tính tiền) ở cửa hàng, hệ thống theo dõi tồn kho trong kho hàng, hệ thống lập lộ trình cho đội xe, hệ thống quản lý nhà cung cấp và nhân viên, cùng nhiều tác vụ khác. Mỗi hệ thống đều phức tạp, cần một nhóm chuyên trách bảo trì, nên cuối cùng phần lớn chúng vận hành độc lập với nhau.

Thông thường, không nên để chuyên viên phân tích nghiệp vụ và nhà khoa học dữ liệu truy vấn trực tiếp các hệ thống OLTP này, vì những lý do sau:

* Dữ liệu cần dùng có thể nằm rải rác ở nhiều hệ thống tác nghiệp, khó kết hợp các tập dữ liệu đó trong một truy vấn duy nhất. Vấn đề này gọi là *silo dữ liệu* (*data silo*).
* Lược đồ và cách bố trí dữ liệu phù hợp với OLTP lại không phù hợp lắm với phân tích (xem [“Mô hình hình sao và bông tuyết: lược đồ cho phân tích”](/ch3#sec_datamodels_analytics)).
* Truy vấn phân tích có thể rất tốn kém: chạy chúng trên cơ sở dữ liệu OLTP sẽ ảnh hưởng đến hiệu năng của những người dùng khác.
* Vì lý do bảo mật hoặc tuân thủ, hệ thống OLTP có thể nằm trong một mạng riêng không cho người dùng truy cập trực tiếp.

Ngược lại, *kho dữ liệu* là một cơ sở dữ liệu riêng mà chuyên viên phân tích có thể truy vấn thoải mái mà không ảnh hưởng đến hoạt động của hệ thống OLTP [^7]. Như [Chương 4](/ch4#ch_storage) sẽ giới thiệu, kho dữ liệu thường lưu trữ dữ liệu theo cách rất khác cơ sở dữ liệu OLTP, nhằm tối ưu cho các kiểu truy vấn phổ biến trong phân tích.

Kho dữ liệu chứa bản sao chỉ đọc của dữ liệu từ các hệ thống OLTP khác nhau của doanh nghiệp. Dữ liệu trước tiên được trích xuất từ cơ sở dữ liệu OLTP (bằng cách kết xuất định kỳ hoặc qua luồng cập nhật liên tục), rồi được chuyển đổi sang lược đồ thuận tiện cho phân tích và được làm sạch, cuối cùng được nạp vào kho dữ liệu. Quá trình đưa dữ liệu vào kho dữ liệu này gọi là *trích xuất—chuyển đổi—nạp* (ETL), như minh họa ở {{< xref fig="1-1" page="/ch1" anchor="fig_dwh_etl" >}}Hình 1-1{{< /xref >}}. Đôi khi thứ tự của hai bước *chuyển đổi* và *nạp* bị đảo lại, tức nạp dữ liệu trước rồi mới chuyển đổi bên trong kho dữ liệu. Cách này được gọi là *ELT*.

{{< fig num="1-1" id="fig_dwh_etl" src="/fig/ddia_0101.png" caption="Sơ đồ đơn giản hóa việc đưa dữ liệu vào kho dữ liệu bằng ETL." class="ddia-figure ddia-figure--standard" width="2953" height="2099" />}}

Đôi khi nguồn dữ liệu của quy trình ETL là các sản phẩm SaaS bên ngoài, ví dụ hệ thống quản lý quan hệ khách hàng (CRM), tiếp thị qua email hoặc xử lý thẻ tín dụng. Khi đó, bạn không thể truy cập trực tiếp cơ sở dữ liệu gốc, mà chỉ lấy được dữ liệu qua API của nhà cung cấp phần mềm. Đưa dữ liệu của các hệ thống bên ngoài này vào kho dữ liệu của chính mình cho phép thực hiện những phân tích mà bản thân API của SaaS không hỗ trợ. ETL cho API của SaaS thường được thực hiện bởi các dịch vụ kết nối dữ liệu chuyên dụng như Fivetran, Singer hoặc AirByte.

Một số hệ thống cơ sở dữ liệu cung cấp *xử lý giao dịch/phân tích kết hợp* (HTAP), với mục tiêu hỗ trợ cả OLTP lẫn phân tích trong một hệ thống duy nhất, không cần ETL để chuyển dữ liệu từ hệ thống này sang hệ thống khác [^8] [^9]. Tuy nhiên, nhiều hệ thống HTAP bên trong vẫn gồm một hệ thống OLTP và một hệ thống phân tích riêng được ghép lại, chỉ là cùng ẩn sau một giao diện thống nhất. Vì vậy, để hiểu cách các hệ thống này hoạt động, sự khác biệt giữa hai loại vẫn rất quan trọng.

Hơn nữa, ngay cả khi đã có HTAP, việc tách riêng hệ thống tác nghiệp và hệ thống phân tích vẫn rất phổ biến, vì mục tiêu và yêu cầu của chúng khác nhau. Đặc biệt, việc mỗi hệ thống tác nghiệp có cơ sở dữ liệu riêng thường được coi là thực hành tốt (xem [“Microservices và serverless”](/ch1#sec_introduction_microservices)), có thể dẫn đến hàng trăm cơ sở dữ liệu tác nghiệp riêng biệt, còn doanh nghiệp thường chỉ có một kho dữ liệu, để chuyên viên phân tích nghiệp vụ kết hợp dữ liệu từ nhiều hệ thống tác nghiệp trong một truy vấn.

Do đó, HTAP không thay thế được kho dữ liệu. Nó phù hợp với một loại tình huống khác: cùng một ứng dụng vừa cần chạy truy vấn phân tích quét nhiều dòng dữ liệu, vừa cần đọc và cập nhật từng bản ghi với độ trễ thấp. Chẳng hạn, phát hiện gian lận có thể có tải công việc như vậy [^10].

Việc tách hệ thống tác nghiệp khỏi hệ thống phân tích phản ánh một xu hướng rộng hơn: khi yêu cầu của tải công việc ngày càng cao, hệ thống ngày càng chuyên biệt hóa, được tối ưu cho từng loại tải cụ thể. Hệ thống đa dụng đủ sức xử lý lượng dữ liệu nhỏ, nhưng quy mô càng lớn, hệ thống thường càng chuyên biệt [^11].

#### Từ kho dữ liệu đến data lake {#from-data-warehouse-to-data-lake}

Kho dữ liệu thường dùng mô hình dữ liệu *quan hệ* và được truy vấn bằng SQL (xem [Chương 3](/ch3#ch_datamodels)), đôi khi kèm phần mềm business intelligence chuyên dụng. Mô hình này rất phù hợp với các truy vấn mà chuyên viên phân tích nghiệp vụ cần, nhưng lại không phù hợp lắm với nhu cầu của nhà khoa học dữ liệu, những người có thể phải làm các việc sau:

* Chuyển đổi dữ liệu thành dạng thích hợp để huấn luyện mô hình học máy. Việc này thường đòi hỏi biến các hàng và cột của bảng cơ sở dữ liệu thành vector hoặc ma trận số, trong đó các giá trị được gọi là *đặc trưng* (*feature*). Thực hiện phép chuyển đổi này sao cho hiệu năng của mô hình sau huấn luyện cao nhất có thể được gọi là *kỹ thuật đặc trưng* (*feature engineering*). Nó thường cần viết mã tùy biến khó diễn đạt bằng SQL.
* Áp dụng kỹ thuật xử lý ngôn ngữ tự nhiên lên dữ liệu văn bản (ví dụ đánh giá sản phẩm) để cố gắng trích xuất thông tin có cấu trúc (ví dụ cảm xúc tác giả bày tỏ, hoặc các chủ đề được nhắc đến). Tương tự, họ cũng có thể cần dùng thị giác máy tính để trích xuất thông tin có cấu trúc từ ảnh.

Dù đã có nhiều nỗ lực đưa các toán tử học máy vào mô hình dữ liệu SQL [^12], cũng như xây dựng hệ thống học máy hiệu quả trên nền mô hình quan hệ [^13], nhiều nhà khoa học dữ liệu vẫn không muốn làm việc trong cơ sở dữ liệu quan hệ như kho dữ liệu. Họ thường thích các thư viện phân tích dữ liệu Python như pandas và scikit-learn, các ngôn ngữ phân tích thống kê như R, và các framework phân tích phân tán như Spark [^14]. Chúng ta sẽ bàn thêm về các công cụ này ở [“DataFrame, ma trận và mảng”](/ch3#sec_datamodels_dataframes).

Vì vậy, tổ chức cần cung cấp dữ liệu dưới dạng phù hợp với nhà khoa học dữ liệu. Giải pháp là *data lake*: một kho dữ liệu tập trung, chứa bản sao của mọi dữ liệu có thể hữu ích cho phân tích, lấy từ các hệ thống tác nghiệp qua quy trình ETL. Điểm khác biệt giữa data lake và kho dữ liệu là data lake chỉ chứa tệp, không áp đặt định dạng tệp hay mô hình dữ liệu cụ thể. Tệp trong data lake có thể là một tập bản ghi cơ sở dữ liệu được mã hóa theo định dạng như Avro hay Parquet (xem [Chương 5](/ch5#ch_encoding)), mà cũng hoàn toàn có thể là văn bản, hình ảnh, video, số liệu cảm biến, ma trận thưa, vector đặc trưng, chuỗi bộ gen hoặc bất kỳ loại dữ liệu nào khác [^15]. Data lake không chỉ linh hoạt hơn mà còn thường rẻ hơn kho lưu trữ dữ liệu quan hệ, vì có thể dùng kho lưu trữ tệp thông dụng, giá rẻ như lưu trữ đối tượng (xem [“Kiến trúc hệ thống cloud-native”](/ch1#sec_introduction_cloud_native)).

Quy trình ETL đã được tổng quát hóa thành *pipeline dữ liệu* (*data pipeline*). Trong một số trường hợp, data lake trở thành một chặng trên đường từ hệ thống tác nghiệp đến kho dữ liệu. Data lake lưu dữ liệu ở dạng “thô” do hệ thống tác nghiệp tạo ra, không chuyển đổi chúng sang lược đồ của kho dữ liệu quan hệ. Lợi ích của cách làm này là mỗi bên tiêu thụ dữ liệu có thể biến dữ liệu thô thành dạng phù hợp nhất với nhu cầu của mình. Cách làm này có một cái tên hóm hỉnh — *nguyên tắc sushi* (*sushi principle*): “dữ liệu thô thì tốt hơn” [^16].

Ngoài việc nạp dữ liệu từ data lake vào kho dữ liệu riêng, còn có thể chạy trực tiếp các tải công việc kiểu kho dữ liệu (truy vấn SQL và phân tích nghiệp vụ) trên các tệp trong data lake, song song với các tải khoa học dữ liệu và học máy. Kiến trúc này gọi là *data lakehouse*. Nó cần thêm công cụ thực thi truy vấn và một tầng siêu dữ liệu (ví dụ quản lý lược đồ) bên trên kho lưu trữ tệp của data lake [^17].

Apache Hive, Spark SQL, Presto và Trino đều dùng cách tiếp cận này.

#### Vượt ra ngoài data lake {#beyond-the-data-lake}

Khi thực tiễn phân tích ngày càng trưởng thành, các tổ chức cũng chú trọng hơn đến việc quản lý và vận hành hệ thống phân tích cùng pipeline dữ liệu, thể hiện ở Tuyên ngôn DataOps chẳng hạn [^18]. Điều này bao gồm quản trị, quyền riêng tư và việc tuân thủ các quy định như GDPR và CCPA. Chúng ta sẽ bàn về các vấn đề này ở [“Hệ thống dữ liệu, pháp luật và xã hội”](/ch1#sec_introduction_compliance) và [“Lập pháp và tự điều chỉnh”](/ch14#sec_future_legislation).

Ngoài ra, dữ liệu phân tích ngày càng được cung cấp không chỉ dưới dạng tệp và bảng quan hệ mà còn dưới dạng luồng sự kiện (xem [Chương 12](/ch12#ch_stream)). Với phân tích dựa trên tệp, có thể chạy lại phân tích định kỳ (ví dụ mỗi ngày) để phản ứng với thay đổi của dữ liệu, còn xử lý luồng cho phép hệ thống phân tích phản hồi nhanh hơn nhiều, thường chỉ trong vài giây sau khi có sự kiện. Có nên dùng xử lý luồng hay không còn phụ thuộc vào yêu cầu về tính kịp thời của ứng dụng. Chẳng hạn, nó có thể được dùng để nhận diện và ngăn chặn hoạt động gian lận hoặc lạm dụng tiềm ẩn.

Đôi khi, đầu ra của hệ thống phân tích còn được cung cấp ngược lại cho hệ thống tác nghiệp, quá trình này đôi khi được gọi là *reverse ETL* [^19]. Ví dụ, một mô hình học máy được huấn luyện trong hệ thống phân tích có thể được triển khai lên môi trường production để sinh gợi ý kiểu “người mua X cũng mua Y” cho người dùng cuối. Những đầu ra của hệ thống phân tích được đưa vào dùng thực tế như vậy còn được gọi là *sản phẩm dữ liệu* (*data product*) [^20]. Mô hình học máy có thể được triển khai lên hệ thống tác nghiệp bằng các công cụ chuyên dụng như TFX, Kubeflow hoặc MLflow.

### Hệ thống ghi nhận gốc và dữ liệu dẫn xuất {#sec_introduction_derived}

Bên cạnh việc phân biệt hệ thống tác nghiệp và hệ thống phân tích, cuốn sách này còn phân biệt *hệ thống ghi nhận gốc* (*system of record*) với *hệ thống dữ liệu dẫn xuất* (*derived data system*). Cặp thuật ngữ này hữu ích, giúp bạn nắm rõ luồng dữ liệu đi qua các hệ thống:

Hệ thống ghi nhận gốc
:   Hệ thống ghi nhận gốc, còn gọi là *nguồn sự thật* (*source of truth*), giữ phiên bản có thẩm quyền hay phiên bản *chuẩn* (*canonical*) của một loại dữ liệu. Khi có dữ liệu mới đến, ví dụ do người dùng nhập, nó được ghi vào đây đầu tiên. Mỗi sự kiện chỉ được biểu diễn một lần, thường theo dạng *chuẩn hóa* (*normalized*) (xem [“Chuẩn hóa, phi chuẩn hóa và phép nối”](/ch3#sec_datamodels_normalization)). Nếu một hệ thống khác không khớp với hệ thống ghi nhận gốc, thì theo định nghĩa, giá trị trong hệ thống ghi nhận gốc là đúng.

Hệ thống dữ liệu dẫn xuất
:   Dữ liệu trong hệ thống dẫn xuất là kết quả của việc lấy dữ liệu sẵn có từ một hệ thống khác rồi chuyển đổi hoặc xử lý theo một cách nào đó. Nếu dữ liệu dẫn xuất bị mất, bạn có thể tạo lại nó từ nguồn dữ liệu gốc. Cache là một ví dụ điển hình: nếu dữ liệu có trong cache thì trả về ngay, còn nếu cache không có thì quay lại đọc từ cơ sở dữ liệu bên dưới. Giá trị phi chuẩn hóa, chỉ mục (*index*), khung nhìn vật chất hóa, biểu diễn dữ liệu đã chuyển đổi và các mô hình được huấn luyện trên tập dữ liệu cũng thuộc loại này.

Về mặt kỹ thuật, dữ liệu dẫn xuất là *dư thừa* (*redundant*), vì nó lặp lại thông tin đã có. Nhưng để truy vấn đọc có hiệu năng tốt, sự dư thừa này thường không thể thiếu. Bạn có thể dẫn xuất nhiều tập dữ liệu khác nhau từ cùng một nguồn, cho phép nhìn dữ liệu từ các “góc độ” khác nhau.

Hệ thống phân tích thường là hệ thống dẫn xuất, vì chúng tiêu thụ dữ liệu được tạo ra ở nơi khác. Dịch vụ tác nghiệp có thể gồm cả hệ thống ghi nhận gốc lẫn hệ thống dẫn xuất: hệ thống ghi nhận gốc là cơ sở dữ liệu chính nơi dữ liệu được ghi trước tiên, còn hệ thống dẫn xuất là các chỉ mục (*index*) và cache giúp tăng tốc các thao tác đọc phổ biến, nhất là với những truy vấn mà hệ thống ghi nhận gốc không trả lời hiệu quả được.

Đa số cơ sở dữ liệu, công cụ lưu trữ và ngôn ngữ truy vấn không tự nhiên thuộc loại ghi nhận gốc hay dẫn xuất. Cơ sở dữ liệu chỉ là công cụ, dùng thế nào là do bạn quyết định. Một hệ thống thuộc loại nào phụ thuộc vào cách nó được dùng trong ứng dụng của bạn, chứ không phụ thuộc vào công cụ nó dùng. Làm rõ dữ liệu nào dẫn xuất từ dữ liệu nào có thể khiến một kiến trúc hệ thống vốn rối rắm trở nên rõ ràng.

Nếu dữ liệu của một hệ thống dẫn xuất từ hệ thống khác, thì mỗi khi dữ liệu gốc trong hệ thống ghi nhận gốc thay đổi, cần có quy trình tương ứng để cập nhật dữ liệu dẫn xuất. Đáng tiếc là nhiều cơ sở dữ liệu được thiết kế với giả định rằng ứng dụng chỉ dùng mỗi nó, nên rất khó tích hợp nhiều hệ thống và lan truyền các cập nhật như vậy. Chúng ta sẽ bàn về các cách *tích hợp dữ liệu* (*data integration*) ở [“Tích hợp dữ liệu”](/ch13#sec_future_integration), nhờ đó có thể kết hợp nhiều hệ thống dữ liệu để làm những việc mà một hệ thống đơn lẻ không tự làm được.

Đến đây, phần so sánh giữa phân tích và xử lý giao dịch của chúng ta kết thúc. Phần tiếp theo bàn về một sự đánh đổi khác mà có lẽ bạn đã thấy nhiều người tranh luận mãi.


## Dịch vụ đám mây và self-hosted {#sec_introduction_cloud}

Bất kể tổ chức cần làm gì, một trong những câu hỏi đầu tiên luôn là: nên tự làm bên trong, hay thuê ngoài? Nên tự xây dựng, hay mua?

Suy cho cùng, điều này tùy thuộc vào ưu tiên kinh doanh. Quan điểm quản trị thông thường cho rằng những việc thuộc năng lực cốt lõi hay lợi thế cạnh tranh của tổ chức nên làm nội bộ, còn những việc không cốt lõi, thường lệ hoặc phổ biến thì nên giao cho nhà cung cấp [^21]. Lấy một ví dụ cực đoan: hầu hết các công ty không tự phát điện (trừ công ty năng lượng, và ở đây cũng chưa tính nguồn điện dự phòng khẩn cấp), vì mua điện từ lưới điện rẻ hơn.

Với phần mềm, có hai quyết định quan trọng: ai xây dựng, và ai triển khai. Cả hai việc đều có thể thuê ngoài ở các mức độ khác nhau, tạo nên một dải liên tục như {{< xref fig="1-2" page="/ch1" anchor="fig_cloud_spectrum" >}}Hình 1-2{{< /xref >}}. Ở một đầu là phần mềm hoàn toàn tùy biến, do bạn tự viết và tự chạy nội bộ, còn ở đầu kia là dịch vụ đám mây hay sản phẩm phần mềm như một dịch vụ (SaaS) được dùng rộng rãi, do nhà cung cấp bên ngoài phát triển và vận hành, bạn chỉ dùng được qua giao diện web hoặc API.

{{< fig num="1-2" id="fig_cloud_spectrum" src="/fig/ddia_0102.png" caption="Dải liên tục của các loại phần mềm và cách vận hành chúng." class="ddia-figure ddia-figure--panorama" width="1772" height="392" />}}

Nằm giữa dải này là phần mềm có sẵn (có thể là mã nguồn mở hoặc thương mại) mà bạn tự vận hành theo kiểu *self-hosted*, tức tự mình triển khai. Ví dụ, bạn tải MySQL về và cài lên một máy chủ do bạn kiểm soát. Máy chủ này có thể là phần cứng của chính bạn — thường gọi là *on-premises*, ngay cả khi thực tế nó nằm trong tủ rack của một trung tâm dữ liệu thuê chứ không thật sự ở cơ sở của bạn — hoặc là máy ảo trên đám mây, tức *hạ tầng như một dịch vụ* (IaaS). Trên dải này còn nhiều vị trí trung gian khác, ví dụ dùng phần mềm mã nguồn mở nhưng chạy phiên bản do chính bạn sửa đổi.

Một câu hỏi độc lập với dải liên tục này là: triển khai dịch vụ *như thế nào*, dù trên đám mây hay tại chỗ, chẳng hạn có dùng framework điều phối như Kubernetes hay không. Tuy nhiên, việc chọn công cụ triển khai nằm ngoài phạm vi cuốn sách này, vì còn những yếu tố khác ảnh hưởng đến kiến trúc hệ thống dữ liệu nhiều hơn.

### Ưu và nhược điểm của dịch vụ đám mây {#sec_introduction_cloud_tradeoffs}

Dùng dịch vụ đám mây thay vì tự chạy phần mềm tương tự, về bản chất là thuê nhà cung cấp đám mây vận hành phần mềm đó. Có cả những lý do thuyết phục để ủng hộ lẫn phản đối việc dùng dịch vụ đám mây. Các nhà cung cấp đám mây tuyên bố rằng dùng dịch vụ của họ giúp tiết kiệm thời gian và tiền bạc, đồng thời cho phép bạn hành động nhanh hơn so với tự xây dựng hạ tầng.

Dịch vụ đám mây có thực sự rẻ hơn và đỡ công hơn self-hosted hay không phụ thuộc rất nhiều vào kỹ năng bạn có và tải công việc của hệ thống. Nếu bạn đã quen triển khai và vận hành hệ thống cần dùng, và tải khá dễ dự đoán (tức số máy cần thiết không dao động mạnh), thì mua máy riêng và tự chạy phần mềm thường rẻ hơn [^22] [^23].

Ngược lại, nếu bạn cần một hệ thống mà mình chưa biết triển khai và vận hành, thì dùng dịch vụ đám mây thường dễ và nhanh hơn so với tự học cách quản lý từ đầu. Nếu còn phải riêng tuyển dụng và đào tạo nhân sự để bảo trì, vận hành hệ thống, chi phí có thể rất cao. Dùng dịch vụ đám mây vẫn cần một đội vận hành (xem [“Vận hành trong thời đại đám mây”](/ch1#sec_introduction_operations)), nhưng thuê ngoài việc quản trị hệ thống nền tảng giúp đội tập trung vào các vấn đề ở tầng cao hơn.

Thuê ngoài việc vận hành hệ thống cho công ty chuyên cung cấp dịch vụ đó cũng có thể cho chất lượng tốt hơn, vì nhà cung cấp tích lũy được nhiều kinh nghiệm vận hành khi phục vụ rất nhiều khách hàng. Mặt khác, nếu tự vận hành, bạn có thể cấu hình và tinh chỉnh dịch vụ cho đúng tải công việc riêng của mình. Nhà cung cấp đám mây khó có khả năng sẵn lòng tùy biến như vậy cho bạn.

Dịch vụ đám mây đặc biệt có giá trị nếu tải của hệ thống biến động mạnh theo thời gian. Nếu máy được cấp phát theo tải đỉnh, nhưng tài nguyên tính toán phần lớn thời gian nằm không, hiệu quả chi phí của hệ thống sẽ rất kém. Trong trường hợp này, ưu điểm của dịch vụ đám mây là có thể dễ dàng tăng hoặc giảm tài nguyên tính toán khi nhu cầu thay đổi.

Chẳng hạn, tải của hệ thống phân tích thường biến thiên rất lớn: để thực thi nhanh một truy vấn phân tích lớn, cần huy động đồng thời rất nhiều tài nguyên tính toán, nhưng khi truy vấn xong, các tài nguyên này nằm không cho đến khi người dùng gửi truy vấn tiếp theo. Truy vấn định sẵn (ví dụ báo cáo hằng ngày) có thể xếp hàng và lập lịch để làm phẳng tải, nhưng với truy vấn tương tác, càng muốn có kết quả nhanh thì tải công việc càng biến động. Nếu tập dữ liệu lớn đến mức phải dùng nhiều tài nguyên tính toán mới truy vấn nhanh được, dùng dịch vụ đám mây có thể tiết kiệm chi phí, vì tài nguyên nhàn rỗi có thể trả lại cho nhà cung cấp thay vì để không. Với tập dữ liệu nhỏ, sự khác biệt này không đáng kể.

Nhược điểm lớn nhất của dịch vụ đám mây là bạn không kiểm soát được nó:

* Nếu dịch vụ thiếu tính năng bạn cần, bạn chỉ có thể lịch sự hỏi nhà cung cấp có sẵn lòng bổ sung không, nhưng thường bạn không thể tự triển khai.
* Nếu dịch vụ ngừng hoạt động, bạn chỉ còn cách chờ nó phục hồi.
* Nếu cách dùng của bạn kích hoạt lỗi hoặc vấn đề hiệu năng, việc chẩn đoán sẽ rất khó. Với phần mềm tự chạy, bạn có thể lấy số liệu hiệu năng và thông tin gỡ lỗi từ hệ điều hành để hiểu hành vi của nó, và có thể xem nhật ký máy chủ, nhưng khi dịch vụ do nhà cung cấp lưu trữ, bạn thường không tiếp cận được những thông tin nội bộ này.
* Nếu dịch vụ ngừng hoạt động, giá cao đến mức không chấp nhận được, hoặc nhà cung cấp thay đổi sản phẩm theo cách bạn không thích, bạn đành phải chịu sự chi phối — tiếp tục chạy phiên bản cũ thường không khả thi, nên buộc phải chuyển sang dịch vụ khác [^24]. Nếu có dịch vụ thay thế cung cấp API tương thích thì rủi ro này nhỏ hơn, nhưng nhiều dịch vụ đám mây không có API chuẩn, chi phí chuyển đổi cao, dẫn đến vấn đề bị khóa chặt vào nhà cung cấp (vendor lock-in).
* Bạn phải tin tưởng nhà cung cấp đám mây bảo vệ an toàn cho dữ liệu, điều này có thể làm việc tuân thủ các quy định về quyền riêng tư và bảo mật khó hơn.

Bất chấp các rủi ro này, việc các tổ chức xây dựng ứng dụng mới trên nền dịch vụ đám mây, hoặc dùng phương án lai chỉ dùng dịch vụ đám mây cho một số phần của hệ thống, ngày càng phổ biến. Tuy nhiên, dịch vụ đám mây sẽ không thay thế mọi hệ thống dữ liệu nội bộ: nhiều hệ thống cũ ra đời trước điện toán đám mây, và với bất kỳ dịch vụ nào có yêu cầu đặc thù mà dịch vụ đám mây hiện có không đáp ứng được, hệ thống nội bộ vẫn không thể thiếu. Ví dụ, các ứng dụng cực kỳ nhạy với độ trễ như giao dịch tần suất cao cần kiểm soát hoàn toàn phần cứng.

### Kiến trúc hệ thống cloud-native {#sec_introduction_cloud_native}

Điện toán đám mây không chỉ mang đến một mô hình kinh tế khác — thuê bao dịch vụ thay vì mua phần cứng và giấy phép phần mềm rồi tự chạy — mà sự trỗi dậy của nó còn tác động sâu sắc về mặt kỹ thuật đến cách hiện thực hóa các hệ thống dữ liệu. Thuật ngữ *cloud-native* dùng để mô tả kiến trúc được thiết kế để tận dụng ưu thế của dịch vụ đám mây.

Về nguyên tắc, hầu như mọi phần mềm có thể self-hosted đều có thể được cung cấp như dịch vụ đám mây. Thực tế, nhiều hệ thống dữ liệu phổ biến hiện nay đã có dịch vụ được quản lý tương ứng. Tuy nhiên, những hệ thống thiết kế theo tư duy cloud-native ngay từ đầu đã cho thấy một số ưu thế: hiệu năng tốt hơn trên cùng phần cứng, phục hồi sau lỗi nhanh hơn, có thể nhanh chóng điều chỉnh tài nguyên tính toán cho khớp với tải, và hỗ trợ tập dữ liệu lớn hơn [^25] [^26] [^27]. {{< xref tbl="1-2" page="/ch1" anchor="tab_cloud_native_dbs" >}}Bảng 1-2{{< /xref >}} liệt kê một số ví dụ của hai loại hệ thống này.

| Loại                | Hệ thống self-hosted         | Hệ thống cloud-native                                                 |
|---------------------|-----------------------------|-----------------------------------------------------------------------|
| Tác nghiệp/OLTP     | MySQL, PostgreSQL, MongoDB  | AWS Aurora [^25], Azure SQL DB Hyperscale [^26], Google Cloud Spanner |
| Phân tích/OLAP      | Teradata, ClickHouse, Spark | Snowflake [^27], Google BigQuery, Azure Synapse Analytics             |
{#tab_cloud_native_dbs num="1-2" caption="Ví dụ về hệ thống cơ sở dữ liệu self-hosted và cloud-native"}

#### Các tầng của dịch vụ đám mây {#layering-of-cloud-services}

Nhiều hệ thống dữ liệu self-hosted có yêu cầu rất đơn giản đối với môi trường chạy: chạy trên hệ điều hành thông thường như Linux hoặc Windows, lưu dữ liệu thành các tệp trong hệ thống tệp, và giao tiếp qua giao thức mạng chuẩn như TCP/IP. Một số ít hệ thống phụ thuộc vào phần cứng đặc biệt như GPU (cho học máy) hoặc card mạng RDMA, nhưng nhìn chung, phần mềm self-hosted dùng những tài nguyên tính toán rất thông dụng: CPU, bộ nhớ, hệ thống tệp và mạng IP.

Trên đám mây, loại phần mềm này có thể chạy trong môi trường hạ tầng như một dịch vụ (IaaS), dùng một hoặc nhiều máy ảo (còn gọi là *instance*), mỗi instance được cấp một lượng CPU, bộ nhớ, đĩa và băng thông mạng nhất định. So với máy vật lý, instance trên đám mây được cấp phát nhanh hơn và có nhiều cấu hình để chọn hơn, nhưng ngoài ra, chúng giống máy tính truyền thống: bạn có thể tùy ý chạy bất kỳ phần mềm nào, và cũng phải tự chịu trách nhiệm quản lý.

Ngược lại, ý tưởng then chốt của dịch vụ cloud-native là: không chỉ dùng tài nguyên tính toán do hệ điều hành quản lý, mà còn xây dựng các dịch vụ tầng cao hơn trên nền các dịch vụ đám mây tầng thấp. Ví dụ:

* Các dịch vụ *lưu trữ đối tượng* (*object storage*) như Amazon S3, Azure Blob Storage và Cloudflare R2 dùng để lưu các tệp lớn. API của chúng hạn chế hơn hệ thống tệp thông thường, chỉ cung cấp thao tác đọc và ghi tệp cơ bản, nhưng ưu điểm là che giấu máy vật lý bên dưới. Dịch vụ tự động phân bổ dữ liệu trên nhiều máy, bạn không phải lo đĩa của một máy nào đó đầy. Ngay cả khi một số máy hoặc đĩa của chúng hỏng hẳn, dữ liệu cũng không bị mất.
* Nhiều dịch vụ khác lại được xây dựng trên nền lưu trữ đối tượng và các dịch vụ đám mây khác. Ví dụ, Snowflake là cơ sở dữ liệu phân tích (kho dữ liệu) trên đám mây, dựa vào S3 để lưu dữ liệu [^27], và có những dịch vụ khác tiếp tục được xây dựng trên nền Snowflake.

Trong khoa học máy tính, trừu tượng hóa vốn luôn như vậy: không có câu trả lời duy nhất đúng cho việc nên chọn tầng nào. Nói chung, trừu tượng ở tầng càng cao thì thường càng hướng đến trường hợp sử dụng cụ thể. Nếu nhu cầu của bạn khớp với kịch bản mà một hệ thống tầng cao được thiết kế, thì dùng thẳng hệ thống có sẵn thường đỡ phiền hơn nhiều so với tự xây dựng bằng hệ thống tầng thấp, và cũng đủ đáp ứng. Ngược lại, nếu không hệ thống tầng cao nào phù hợp, bạn đành tự xây dựng từ các thành phần tầng thấp.

#### Tách rời lưu trữ và tính toán {#sec_introduction_storage_compute}

Trong điện toán truyền thống, lưu trữ trên đĩa được coi là lưu trữ bền vững: ta giả định dữ liệu một khi đã ghi vào đĩa thì sẽ không mất. Để chịu được lỗi của một ổ cứng đơn lẻ, người ta thường dùng RAID (mảng đĩa dự phòng độc lập), lưu bản sao dữ liệu trên nhiều đĩa gắn vào cùng một máy. RAID có thể được hiện thực bằng phần cứng, hoặc bằng phần mềm trong hệ điều hành. Với ứng dụng truy cập hệ thống tệp, tất cả những điều này đều trong suốt.

Máy tính toán (máy ảo) trên đám mây cũng có thể gắn đĩa cục bộ, nhưng hệ thống cloud-native thường muốn coi chúng như bộ nhớ đệm tạm thời, chứ không phải lưu trữ lâu dài. Lý do là: khi instance tương ứng gặp lỗi, đĩa cục bộ sẽ không truy cập được, và khi cần đổi instance sang một máy vật lý khác, lớn hơn hoặc nhỏ hơn để thích ứng với thay đổi của tải, đĩa cục bộ cũng không truy cập được.

Để thay cho đĩa cục bộ, dịch vụ đám mây còn cung cấp lưu trữ đĩa ảo, có thể tháo khỏi một instance rồi gắn vào instance khác, ví dụ Amazon EBS, Azure Managed Disks và Google Cloud Persistent Disk. Đĩa ảo này không phải đĩa vật lý thực sự, mà là một dịch vụ đám mây do một nhóm máy khác cung cấp để mô phỏng hành vi của đĩa — tức *thiết bị khối* (*block device*), trong đó mỗi khối thường là 4 KiB. Công nghệ này cho phép phần mềm đĩa truyền thống chạy trên đám mây, nhưng việc mô phỏng thiết bị khối sinh thêm chi phí phụ trội. Nếu hệ thống được thiết kế cho đám mây ngay từ đầu thì có thể tránh được các chi phí này [^25]. Nó cũng khiến ứng dụng rất nhạy với bất thường của mạng, vì mỗi thao tác I/O trên thiết bị khối ảo thực chất là một lần gọi mạng [^28].

Để giải quyết vấn đề này, dịch vụ cloud-native thường tránh dùng đĩa ảo, mà xây dựng trên các dịch vụ lưu trữ chuyên dụng được tối ưu cho từng tải công việc cụ thể. Các dịch vụ lưu trữ đối tượng như S3 phù hợp để lưu lâu dài các tệp tương đối lớn, kích thước từ vài trăm KB đến vài GB. Một hàng hay một giá trị đơn lẻ trong cơ sở dữ liệu thường nhỏ hơn nhiều so với khoảng này, vì thế cơ sở dữ liệu đám mây thường quản lý các giá trị nhỏ trong một dịch vụ riêng, và lưu các khối dữ liệu lớn hơn, chứa nhiều giá trị, vào lưu trữ đối tượng [^26] [^29]. Chúng ta sẽ giới thiệu các cách hiện thực tương ứng ở [Chương 4](/ch4#ch_storage).

Trong kiến trúc hệ thống truyền thống, cùng một máy tính đảm nhiệm cả lưu trữ (đĩa) lẫn tính toán (CPU và bộ nhớ), còn trong hệ thống cloud-native, hai trách nhiệm này ở một mức độ nào đó được tách riêng, hay nói cách khác là *tách rời* (*disaggregated*) [^9] [^27] [^30] [^31]. Ví dụ, S3 chỉ chịu trách nhiệm lưu tệp. Nếu muốn phân tích dữ liệu trong đó, bạn phải chạy mã phân tích bên ngoài S3. Điều này cũng có nghĩa là dữ liệu cần được truyền qua mạng, mà chúng ta sẽ bàn thêm ở [“Hệ thống phân tán và hệ thống một node”](/ch1#sec_introduction_distributed).

Ngoài ra, hệ thống cloud-native thường dùng mô hình *multitenant*: thay vì cấp riêng một máy cho mỗi khách hàng, cùng một dịch vụ xử lý dữ liệu và tính toán của nhiều khách hàng trên phần cứng dùng chung [^32].

Multitenant có thể nâng cao mức sử dụng phần cứng, dễ đạt khả năng mở rộng hơn và thuận tiện hơn cho nhà cung cấp đám mây khi quản lý, nhưng để bảo đảm hoạt động của khách hàng này không ảnh hưởng đến hiệu năng hay an ninh của khách hàng khác thì phải có thiết kế kỹ thuật cẩn thận [^33].

### Vận hành trong thời đại đám mây {#sec_introduction_operations}

Theo truyền thống, những người quản lý hạ tầng dữ liệu phía máy chủ của tổ chức được gọi là *quản trị viên cơ sở dữ liệu* (DBA) hoặc *quản trị viên hệ thống* (sysadmin). Những năm gần đây, nhiều tổ chức cố gắng gộp vai trò phát triển phần mềm và vận hành vào cùng một nhóm, để nhóm cùng chịu trách nhiệm về dịch vụ backend và hạ tầng dữ liệu. Triết lý *DevOps* thúc đẩy xu hướng này. *Kỹ sư độ tin cậy trang web* (SRE) là cách Google hiện thực hóa triết lý đó [^34].

Trách nhiệm của vận hành là bảo đảm dịch vụ được cung cấp đến người dùng một cách đáng tin cậy, gồm cấu hình hạ tầng và triển khai ứng dụng, đồng thời giữ môi trường production ổn định, gồm giám sát và chẩn đoán mọi vấn đề có thể ảnh hưởng đến độ tin cậy. Với hệ thống self-hosted, vận hành truyền thống có rất nhiều việc đổ dồn lên từng máy riêng lẻ, chẳng hạn lập kế hoạch dung lượng (theo dõi dung lượng đĩa còn trống và thêm đĩa trước khi hết), cấp phát máy mới, di chuyển dịch vụ giữa các máy và cài bản vá hệ điều hành.

Nhiều dịch vụ đám mây che giấu các máy cụ thể đang chạy dịch vụ đó sau API. Ví dụ, lưu trữ đám mây không còn cung cấp đĩa dung lượng cố định mà dùng *tính phí theo mức sử dụng* (*metered billing*): bạn không cần lập kế hoạch dung lượng trước, vẫn có thể lưu dữ liệu, rồi trả tiền theo dung lượng thực tế đã dùng. Ngoài ra, nhiều dịch vụ đám mây vẫn duy trì được tính sẵn sàng cao ngay cả khi từng máy riêng lẻ gặp lỗi (xem [“Độ tin cậy và khả năng chịu lỗi”](/ch2#sec_introduction_reliability)).

Việc chuyển trọng tâm từ từng máy sang dịch vụ cũng đi kèm thay đổi vai trò vận hành. Mục tiêu cấp cao là cung cấp dịch vụ đáng tin cậy không đổi, nhưng quy trình và công cụ đã tiến hóa. Triết lý DevOps/SRE nhấn mạnh hơn:

* Tự động hóa — dùng quy trình lặp lại được thay vì làm thủ công các tác vụ một lần;
* Dùng máy ảo và dịch vụ tạm thời thay vì máy chủ chạy lâu dài;
* Hỗ trợ cập nhật ứng dụng thường xuyên;
* Rút kinh nghiệm từ sự cố; và
* Lưu giữ hiểu biết của tổ chức về hệ thống, ngay cả khi nhân sự đến rồi đi [^35].

Cùng với sự trỗi dậy của dịch vụ đám mây, vai trò vận hành cũng phân hóa: đội vận hành của các công ty hạ tầng chuyên về cách cung cấp dịch vụ đáng tin cậy cho số lượng lớn khách hàng, còn khách hàng của dịch vụ đám mây cố gắng giảm thiểu thời gian và công sức đầu tư vào hạ tầng [^36].

Khách hàng của dịch vụ đám mây vẫn cần vận hành, chỉ là mối quan tâm khác đi, ví dụ chọn dịch vụ phù hợp nhất cho một tác vụ, tích hợp các dịch vụ khác nhau, và di chuyển từ dịch vụ này sang dịch vụ khác. Dù tính phí theo mức sử dụng loại bỏ việc lập kế hoạch dung lượng theo nghĩa truyền thống, bạn vẫn phải biết rõ tài nguyên nào được dùng vào đâu, kẻo tiêu tiền vô ích cho tài nguyên đám mây không cần thiết: lập kế hoạch dung lượng trở thành lập kế hoạch tài chính, và tối ưu hiệu năng trở thành tối ưu chi phí [^37].

Hơn nữa, dịch vụ đám mây vẫn có giới hạn tài nguyên hay *hạn mức* (*quota*), ví dụ số tiến trình tối đa chạy đồng thời. Bạn phải biết trước và lên kế hoạch, không thể đợi đến khi chạm trần mới xử lý [^38].

Dùng dịch vụ đám mây có thể dễ và nhanh hơn tự chạy hạ tầng, nhưng học cách dùng vẫn tốn công, đôi khi còn phải tìm cách vòng qua các giới hạn của nó. Khi ngày càng có nhiều nhà cung cấp, và dịch vụ đám mây cho đủ loại trường hợp sử dụng xuất hiện dày đặc, việc tích hợp các dịch vụ khác nhau trở thành một thách thức đặc biệt hóc búa [^39] [^40].

ETL (xem [“Kho dữ liệu”](/ch1#sec_introduction_dwh)) chỉ là một phần của vấn đề. Các dịch vụ đám mây tác nghiệp cũng cần tích hợp với nhau. Hiện còn thiếu tiêu chuẩn để đơn giản hóa việc tích hợp này, nên thường phải đầu tư nhiều công sức thủ công.

Cũng có những công việc vận hành không thể thuê ngoài hoàn toàn cho dịch vụ đám mây, chẳng hạn giữ an toàn cho ứng dụng và các thư viện phụ thuộc của nó, quản lý tương tác giữa các dịch vụ của chính mình, giám sát tải của dịch vụ, và truy tìm gốc rễ của các vấn đề như hiệu năng suy giảm hay dịch vụ gián đoạn. Điện toán đám mây đang thay đổi vai trò của vận hành, nhưng vận hành vẫn quan trọng như bao giờ hết.


## Hệ thống phân tán và hệ thống một node {#sec_introduction_distributed}

Hệ thống gồm nhiều máy giao tiếp với nhau qua mạng được gọi là *hệ thống phân tán* (*distributed system*). Mỗi tiến trình tham gia hệ thống phân tán được gọi là một *node*. Có nhiều lý do để dùng hệ thống phân tán:

Hệ thống phân tán vốn có
:   Nếu một ứng dụng có hai hay nhiều người dùng tương tác với nhau, và mỗi người dùng thiết bị của riêng mình, thì hệ thống đó tất yếu là phân tán: các thiết bị chỉ có thể giao tiếp với nhau qua mạng.

Yêu cầu giữa các dịch vụ đám mây
:   Nếu dữ liệu lưu ở một dịch vụ nhưng do dịch vụ khác xử lý, thì bắt buộc phải truyền dữ liệu từ dịch vụ này sang dịch vụ kia qua mạng.

Chịu lỗi/tính sẵn sàng cao
:   Nếu ứng dụng cần tiếp tục chạy khi một máy (thậm chí nhiều máy, mạng hoặc cả trung tâm dữ liệu) gặp lỗi, bạn có thể dùng nhiều máy để dự phòng: khi một máy hỏng, máy khác tiếp quản. Xem [“Độ tin cậy và khả năng chịu lỗi”](/ch2#sec_introduction_reliability) và phần thảo luận về sao chép (*replication*) ở [Chương 6](/ch6#ch_replication).

Khả năng mở rộng
:   Nếu khối lượng dữ liệu hoặc nhu cầu tính toán tăng vượt quá sức một máy, có thể phân tán tải sang nhiều máy. Xem [“Khả năng mở rộng”](/ch2#sec_introduction_scalability).

Độ trễ
:   Nếu người dùng ở khắp nơi trên thế giới, bạn có thể muốn triển khai máy chủ ở nhiều khu vực địa lý, để mỗi người dùng được phục vụ bởi máy chủ gần mình. Nhờ vậy, người dùng không phải chờ gói tin mạng đi vòng nửa Trái Đất mới nhận được phản hồi. Xem [“Mô tả hiệu năng”](/ch2#sec_introduction_percentiles).

Tính co giãn
:   Nếu ứng dụng lúc bận lúc rảnh, việc triển khai trên đám mây có thể co giãn theo nhu cầu, để bạn chỉ trả tiền cho tài nguyên đang dùng. Điều này khó làm trên một máy đơn lẻ: dù phần lớn thời gian gần như không có tải, bạn vẫn phải cấu hình sẵn máy theo tải tối đa.

Dùng phần cứng chuyên dụng
:   Các phần của hệ thống có thể dùng phần cứng phù hợp với tải công việc riêng. Ví dụ, lưu trữ đối tượng có thể dùng máy có nhiều đĩa nhưng ít CPU; hệ thống phân tích dữ liệu có thể dùng máy có nhiều CPU và bộ nhớ nhưng không có đĩa; hệ thống học máy có thể dùng máy trang bị GPU — khi huấn luyện mạng nơ-ron sâu và thực hiện các tác vụ học máy khác, GPU hiệu quả hơn CPU rất nhiều.

Tuân thủ pháp luật
:   Một số quốc gia ban hành luật lưu trú dữ liệu, yêu cầu dữ liệu về những người thuộc phạm vi tài phán của nước đó phải được lưu trữ và xử lý trong lãnh thổ nước đó [^41]. Phạm vi áp dụng của các quy định này không giống nhau: có nơi chỉ nhắm vào dữ liệu y tế hoặc tài chính, có nơi rộng hơn. Do đó, nếu người dùng của một dịch vụ nằm ở nhiều khu vực tài phán như vậy, buộc phải phân tán dữ liệu người dùng lên máy chủ ở nhiều địa điểm.

Tính bền vững
:   Nếu thời điểm và địa điểm chạy công việc có độ linh hoạt nhất định, bạn có thể chọn thời gian và nơi có nhiều điện tái tạo, đồng thời tránh lúc lưới điện căng thẳng. Cách này vừa giảm phát thải carbon, vừa tận dụng được điện giá rẻ [^42] [^43].

Những lý do này áp dụng cho cả dịch vụ do bạn tự viết (mã ứng dụng) lẫn dịch vụ cấu thành từ phần mềm có sẵn như cơ sở dữ liệu.

### Các vấn đề của hệ thống phân tán {#sec_introduction_dist_sys_problems}

Hệ thống phân tán cũng có nhược điểm. Mọi yêu cầu và lời gọi API đi qua mạng đều phải đối mặt với khả năng thất bại: mạng có thể đứt, dịch vụ có thể quá tải hoặc sập, và bất kỳ yêu cầu nào cũng có thể hết thời gian chờ mà không nhận được phản hồi. Khi đó, ta không biết dịch vụ rốt cuộc đã nhận được yêu cầu hay chưa, và vội thử lại chưa chắc đã an toàn. Chúng ta sẽ bàn chi tiết các vấn đề này ở [Chương 9](/ch9#ch_distributed).

Dù mạng trong trung tâm dữ liệu rất nhanh, gọi một dịch vụ khác vẫn chậm hơn nhiều so với gọi hàm trong cùng một tiến trình [^44].

Khi xử lý lượng dữ liệu khổng lồ, thay vì chuyển dữ liệu từ nơi lưu sang máy khác để xử lý, đưa phép tính đến máy đang giữ dữ liệu có thể nhanh hơn [^45].

Nhiều node hơn cũng không nhất thiết nhanh hơn: trong một số trường hợp, một chương trình đơn luồng đơn giản trên một máy tính có thể vượt hẳn hiệu năng của cụm có hơn 100 lõi CPU [^46].

Hệ thống phân tán thường rất khó khắc phục sự cố: nếu hệ thống phản hồi chậm, làm sao tìm ra vấn đề nằm ở đâu? Các kỹ thuật dùng để chẩn đoán vấn đề của hệ thống phân tán được gọi chung là *observability* [^47] [^48]. Nó thu thập dữ liệu thực thi của hệ thống và cho phép truy vấn dữ liệu đó, nhờ vậy vừa phân tích được các chỉ số cấp cao, vừa lần theo từng sự kiện riêng lẻ. Các công cụ *tracing* như OpenTelemetry, Zipkin và Jaeger có thể ghi lại client nào gọi máy chủ nào để làm thao tác gì, và mỗi lần gọi mất bao lâu [^49].

Cơ sở dữ liệu cung cấp nhiều cơ chế bảo đảm tính nhất quán của dữ liệu, như chúng ta sẽ thấy ở [Chương 6](/ch6#ch_replication) và [Chương 8](/ch8#ch_transactions). Nhưng khi mỗi dịch vụ có cơ sở dữ liệu riêng, việc giữ dữ liệu nhất quán giữa các dịch vụ trở thành vấn đề của chính ứng dụng. Giao dịch phân tán (xem [Chương 8](/ch8#ch_transactions)) là một cách khả dĩ, nhưng hiếm khi dùng trong microservices, vì nó đi ngược mục tiêu để các dịch vụ độc lập với nhau, và nhiều cơ sở dữ liệu vốn không hỗ trợ giao dịch phân tán [^50].

Vì tất cả những lý do trên, nếu một việc có thể làm trên một máy đơn lẻ, thì thường đơn giản và rẻ hơn nhiều so với xây dựng hệ thống phân tán [^23] [^46] [^51]. CPU ngày càng nhanh, dung lượng bộ nhớ và đĩa ngày càng lớn, phần cứng cũng ngày càng đáng tin cậy. Cộng thêm các cơ sở dữ liệu một node như DuckDB, SQLite và KùzuDB, ngày nay nhiều tải công việc có thể chạy trên một node duy nhất. Chúng ta sẽ tiếp tục bàn chủ đề này ở [Chương 4](/ch4#ch_storage).

### Microservices và serverless {#sec_introduction_microservices}

Cách phổ biến nhất để phân tán hệ thống trên nhiều máy là chia nó thành client và máy chủ, để client gửi yêu cầu đến máy chủ. Giao tiếp này thường dùng HTTP nhất, mà chúng ta sẽ bàn thêm ở [“Luồng dữ liệu qua dịch vụ: REST và RPC”](/ch5#sec_encoding_dataflow_rpc). Cùng một tiến trình có thể vừa là máy chủ (xử lý yêu cầu đến), vừa là client (gửi yêu cầu đến dịch vụ khác).

Cách xây dựng ứng dụng này theo truyền thống gọi là *kiến trúc hướng dịch vụ* (SOA), và những năm gần đây, ý tưởng này tiếp tục phát triển thành kiến trúc *microservices* [^52] [^53]. Trong kiến trúc này, mỗi dịch vụ có mục đích rõ ràng (ví dụ mục đích của S3 là lưu tệp); dịch vụ mở khả năng của mình qua API để client gọi qua mạng; và mỗi dịch vụ do một nhóm chịu trách nhiệm bảo trì. Nhờ vậy, ứng dụng phức tạp có thể được tách thành nhiều dịch vụ tương tác với nhau, do các nhóm khác nhau quản lý.

Tách phần mềm phức tạp thành nhiều dịch vụ có vài lợi ích: từng dịch vụ có thể cập nhật độc lập, giảm phối hợp giữa các nhóm; mỗi dịch vụ có thể được cấp tài nguyên phần cứng phù hợp nhu cầu riêng; chi tiết hiện thực bị che sau API, nên người phụ trách dịch vụ có thể tự do thay đổi hiện thực mà không ảnh hưởng đến client. Về lưu trữ dữ liệu, thường mỗi dịch vụ có cơ sở dữ liệu riêng, và các dịch vụ không dùng chung cơ sở dữ liệu. Nếu không, toàn bộ cấu trúc cơ sở dữ liệu thực chất sẽ trở thành một phần của API dịch vụ, rất khó sửa đổi. Hơn nữa, truy vấn do một dịch vụ gửi ra cũng có thể làm chậm các dịch vụ khác.

Mặt khác, nhiều dịch vụ hơn cũng tự sinh thêm độ phức tạp: mỗi dịch vụ cần hạ tầng tương ứng để triển khai phiên bản mới, điều chỉnh tài nguyên phần cứng theo tải, thu thập nhật ký, giám sát tình trạng dịch vụ, và cảnh báo kỹ sư trực khi có sự cố. Các framework *điều phối* (*orchestration*) như Kubernetes cung cấp năng lực nền tảng cho loại hạ tầng này, nên trở thành cách phổ biến để triển khai dịch vụ. Kiểm thử một dịch vụ trong quá trình phát triển cũng có thể phiền toái, vì các dịch vụ khác mà nó phụ thuộc phải chạy cùng.

API của microservices cũng khó tiến hóa. Client gọi API sẽ kỳ vọng có một số trường nhất định, và khi yêu cầu kinh doanh thay đổi, nhà phát triển có thể muốn thêm hoặc bớt trường API, nhưng điều đó có thể làm client lỗi. Tệ hơn, những vấn đề như vậy thường chỉ được phát hiện muộn trong chu kỳ phát triển, khi API dịch vụ đã cập nhật được triển khai lên môi trường staging hoặc production. Các chuẩn mô tả API như OpenAPI và gRPC giúp quản lý mối quan hệ giữa API phía client và API phía máy chủ. Chúng ta sẽ bàn thêm ở [Chương 5](/ch5#ch_encoding).

Microservices chủ yếu là dùng biện pháp kỹ thuật để giải quyết vấn đề về con người: cho phép các nhóm khác nhau tiến hành công việc độc lập mà không cần phối hợp với nhau. Điều này có giá trị với công ty lớn, nhưng ở công ty nhỏ với ít nhóm, microservices rất có thể chỉ là chi phí phụ trội không cần thiết, khi đó tốt nhất nên hiện thực ứng dụng theo cách đơn giản nhất [^52].

*Serverless*, còn gọi là *hàm như một dịch vụ* (FaaS), là một cách triển khai dịch vụ khác: nó thuê ngoài thêm việc quản lý hạ tầng cho nhà cung cấp đám mây [^33]. Với máy ảo, bạn phải tự quyết định khi nào khởi động hay tắt instance, còn mô hình serverless để nhà cung cấp đám mây tự động cấp phát và thu hồi tài nguyên phần cứng dựa trên các yêu cầu gửi đến dịch vụ [^54]. Cách triển khai này chuyển thêm gánh nặng vận hành sang nhà cung cấp đám mây, và cho phép tính phí linh hoạt theo mức sử dụng thay vì theo instance máy. Để mang lại những lợi ích đó, nhiều hạ tầng serverless giới hạn thời gian thực thi của hàm và môi trường chạy, đồng thời hàm có thể khởi động chậm trong lần gọi đầu tiên. Cái tên “serverless” cũng dễ gây hiểu lầm: mỗi lần thực thi hàm serverless vẫn phải dùng một máy chủ, chỉ là lần thực thi sau có thể chuyển sang máy khác. Ngoài ra, BigQuery và nhiều sản phẩm Kafka cũng dùng cách nói “serverless” để chỉ rằng dịch vụ có thể tự động co giãn và tính phí theo mức sử dụng thay vì theo instance máy.

Cũng như lưu trữ đám mây dùng tính phí theo mức sử dụng để thay cho lập kế hoạch dung lượng (quyết định trước mua bao nhiêu đĩa), mô hình serverless cũng đưa tính phí theo mức sử dụng vào việc thực thi mã: bạn chỉ trả tiền cho thời gian mã ứng dụng thực sự chạy, không cần cấp phát tài nguyên trước.

### Điện toán đám mây và siêu tính toán {#id17}

Điện toán đám mây không phải cách duy nhất để xây dựng hệ thống tính toán quy mô lớn. Một hướng khác là *tính toán hiệu năng cao* (HPC), còn gọi là *siêu tính toán* (*supercomputing*). Dù hai bên có phần giao nhau, HPC thường có trọng tâm khác và dùng công nghệ khác so với điện toán đám mây và hệ thống trung tâm dữ liệu doanh nghiệp. Các khác biệt gồm:

* Siêu máy tính thường dùng cho các tác vụ tính toán khoa học chuyên sâu về tính toán, như dự báo thời tiết, mô hình hóa khí hậu, động lực học phân tử (mô phỏng chuyển động của nguyên tử và phân tử), các bài toán tối ưu phức tạp, và giải phương trình đạo hàm riêng. Còn điện toán đám mây thường dùng hơn cho dịch vụ trực tuyến, hệ thống dữ liệu kinh doanh, và các hệ thống khác cần đáp ứng yêu cầu của người dùng với tính sẵn sàng cao.
* Siêu máy tính thường chạy các tác vụ xử lý theo lô lớn, và thỉnh thoảng ghi trạng thái tính toán xuống đĩa dưới dạng checkpoint. Nếu một node gặp lỗi, cách làm phổ biến là dừng toàn bộ tải công việc của cả cụm, sửa node lỗi, rồi tính toán lại từ checkpoint gần nhất [^55] [^56]. Dịch vụ đám mây thường không nên dừng cả cụm như vậy, vì dịch vụ phải liên tục đáp ứng người dùng và hạn chế gián đoạn.
* Các node của siêu máy tính thường giao tiếp qua bộ nhớ chia sẻ và truy cập bộ nhớ trực tiếp từ xa (RDMA), vừa có băng thông cao vừa có độ trễ thấp, nhưng với điều kiện người dùng hệ thống rất tin cậy lẫn nhau [^57]. Trong điện toán đám mây, mạng và máy thường được chia sẻ bởi các tổ chức không tin nhau, nên cần cơ chế bảo mật mạnh hơn, như cách ly tài nguyên (dùng máy ảo, v.v.), mã hóa và xác thực.
* Mạng trung tâm dữ liệu đám mây thường dựa trên IP và Ethernet, dùng topo Clos để có băng thông bisection cao — một chỉ số thường dùng để đo hiệu năng tổng thể của mạng [^55] [^58]. Siêu máy tính thường dùng topo mạng chuyên dụng như lưới đa chiều và torus [^59]. Với các tải HPC có mẫu giao tiếp đã biết, các topo này cho hiệu năng tốt hơn.
* Điện toán đám mây cho phép các node phân bố ở nhiều khu vực địa lý, còn siêu máy tính thường giả định mọi node đều ở gần nhau.

Hệ thống phân tích quy mô lớn đôi khi cũng có một số đặc điểm của siêu tính toán, và nếu làm việc trong lĩnh vực này, hiểu biết về các kỹ thuật đó sẽ rất đáng giá. Tuy nhiên, cuốn sách này chủ yếu tập trung vào các dịch vụ phải luôn sẵn sàng, như đã bàn ở [“Độ tin cậy và khả năng chịu lỗi”](/ch2#sec_introduction_reliability).

## Hệ thống dữ liệu, pháp luật và xã hội {#sec_introduction_compliance}

Đến đây, chương này đã cho thấy kiến trúc hệ thống dữ liệu không chỉ chịu ảnh hưởng của mục tiêu và yêu cầu kỹ thuật, mà còn của nhu cầu con người trong tổ chức mà nó phục vụ. Ngày càng nhiều kỹ sư hệ thống dữ liệu nhận ra rằng chỉ đáp ứng nhu cầu của doanh nghiệp mình là chưa đủ: chúng ta còn có trách nhiệm với toàn xã hội.

Một vấn đề đặc biệt đáng quan tâm là các hệ thống lưu trữ dữ liệu cá nhân và hành vi của họ. Từ năm 2018, *Quy định chung về bảo vệ dữ liệu* (GDPR) trao cho cư dân nhiều quốc gia châu Âu quyền kiểm soát lớn hơn đối với dữ liệu cá nhân và nhiều quyền pháp lý hơn. Nhiều quốc gia và khu vực khác trên thế giới cũng ban hành các quy định về quyền riêng tư tương tự, ví dụ Đạo luật Quyền riêng tư của Người tiêu dùng California (CCPA). Các quy định nhắm vào AI như *Đạo luật AI của Liên minh châu Âu* lại hạn chế thêm cách dùng dữ liệu cá nhân.

Ngay cả ở những lĩnh vực không bị quản lý trực tiếp, người ta cũng ngày càng nhận ra tác động của hệ thống máy tính đối với cá nhân và xã hội. Mạng xã hội đã thay đổi cách mọi người tiếp cận tin tức, từ đó ảnh hưởng đến quan điểm chính trị, thậm chí có thể chi phối kết quả bầu cử. Hệ thống tự động cũng ngày càng đưa ra những quyết định ảnh hưởng sâu sắc đến cá nhân, ví dụ ai được vay tiền hay mua bảo hiểm, ai được mời phỏng vấn xin việc, và ai bị coi là nghi phạm tội phạm [^60].

Mỗi người tham gia vào các hệ thống như vậy đều có trách nhiệm cân nhắc tác động đạo đức của chúng và bảo đảm hệ thống tuân thủ pháp luật liên quan. Không phải ai cũng phải trở thành chuyên gia pháp lý và đạo đức, nhưng hiểu biết cơ bản về pháp luật và đạo đức cũng quan trọng không kém việc nắm kiến thức nền tảng về hệ thống phân tán.

Các yếu tố pháp lý đang ảnh hưởng đến những phần căn bản nhất của thiết kế hệ thống dữ liệu [^61]. Ví dụ, GDPR trao cho cá nhân quyền yêu cầu xóa dữ liệu của họ, đôi khi gọi là *quyền được lãng quên* (*right to be forgotten*). Tuy nhiên, như cuốn sách này sẽ giới thiệu, nhiều hệ thống dữ liệu được thiết kế dựa trên các cấu trúc bất biến như nhật ký chỉ ghi thêm. Một tệp lẽ ra bất biến thì làm sao xóa một số dữ liệu ở giữa? Nếu dữ liệu đã được đưa vào các tập dữ liệu dẫn xuất (xem [“Hệ thống ghi nhận gốc và dữ liệu dẫn xuất”](/ch1#sec_introduction_derived)), chẳng hạn trở thành dữ liệu huấn luyện của mô hình học máy, thì xóa thế nào? Trả lời những câu hỏi này đặt ra những thách thức kỹ thuật mới.

Hiện chưa có hướng dẫn rõ ràng về những kỹ thuật hay kiến trúc hệ thống cụ thể nào được coi là “tuân thủ GDPR”. Quy định cố ý không chỉ định công nghệ cụ thể, vì tiến bộ công nghệ có thể nhanh chóng làm các quy định ấy lỗi thời. Văn bản pháp luật chỉ đưa ra các nguyên tắc cấp cao cần được diễn giải. Do đó, việc tuân thủ quy định về quyền riêng tư không có câu trả lời đơn giản. Tuy nhiên, khi bàn về một số kỹ thuật trong cuốn sách này, chúng ta sẽ xem xét chúng từ góc độ đó.

Nói chung, chúng ta lưu trữ dữ liệu vì tin rằng giá trị của nó cao hơn chi phí lưu trữ. Nhưng đừng quên rằng chi phí lưu trữ không chỉ là hóa đơn trả cho Amazon S3 hay dịch vụ khác. Phân tích chi phí – lợi ích còn phải tính đến các rủi ro: dữ liệu bị rò rỉ, hoặc bị kẻ tấn công đánh cắp, phá hoại có thể phải chịu trách nhiệm pháp lý và thiệt hại về danh tiếng, và nếu dữ liệu được lưu trữ, xử lý không đúng quy định pháp luật, còn có thể phát sinh chi phí kiện tụng và tiền phạt [^51].

Chính phủ hoặc cảnh sát cũng có thể buộc doanh nghiệp giao nộp dữ liệu. Nếu dữ liệu có thể làm lộ những hành vi bị luật pháp địa phương coi là tội phạm — ví dụ, ở một số nước Trung Đông và châu Phi, đồng tính bị trừng phạt hình sự; ở một số bang của Mỹ, việc tìm cách phá thai cũng có thể bị truy cứu hình sự — thì lưu trữ dữ liệu đó mang lại rủi ro an toàn thực sự cho người dùng. Ví dụ, dữ liệu vị trí dễ dàng tiết lộ việc một người từng đến phòng khám phá thai. Thậm chí chỉ một bản nhật ký lịch sử địa chỉ IP của người dùng cũng có thể làm lộ vị trí đại khái của họ.

Sau khi cân nhắc mọi rủi ro, kết luận hợp lý có thể là: một số dữ liệu không đáng để lưu trữ, nên cần xóa đi. Nguyên tắc *tối thiểu hóa dữ liệu* (*data minimization*) (đôi khi còn được biểu đạt bằng từ tiếng Đức *Datensparsamkeit*) đi ngược quan niệm “dữ liệu lớn” vốn thiên về lưu thật nhiều dữ liệu trước, hy vọng sau này có thể hữu ích [^62]. Tuy nhiên, tối thiểu hóa dữ liệu phù hợp với GDPR: dữ liệu cá nhân chỉ được thu thập cho những mục đích cụ thể, rõ ràng, sau này không được dùng cho mục đích khác, và không được giữ lâu hơn thời hạn cần thiết cho mục đích ban đầu [^63].

Doanh nghiệp cũng bắt đầu coi trọng vấn đề quyền riêng tư và bảo mật. Các công ty thẻ tín dụng yêu cầu doanh nghiệp xử lý thanh toán tuân thủ tiêu chuẩn nghiêm ngặt của ngành thẻ thanh toán (PCI). Đơn vị xử lý thanh toán phải thường xuyên chịu đánh giá của các kiểm toán viên độc lập để xác minh họ tiếp tục tuân thủ. Nhà cung cấp phần mềm cũng chịu sự thẩm định ngày càng chặt, nhiều bên mua hiện yêu cầu nhà cung cấp phù hợp tiêu chuẩn Kiểm soát Tổ chức Dịch vụ (SOC) loại 2. Cũng như tuân thủ PCI, nhà cung cấp phải qua kiểm toán của bên thứ ba để xác minh đã đáp ứng các yêu cầu.

Tóm lại, cần cân bằng giữa nhu cầu kinh doanh và nhu cầu của những người có dữ liệu được thu thập, xử lý. Chủ đề này còn rộng hơn nhiều. [Chương 14](/ch14#ch_right_thing) sẽ đi sâu vào đạo đức và tuân thủ pháp luật, bao gồm cả thiên kiến và phân biệt đối xử.


## Tóm tắt {#summary}

Chủ đề của chương này là hiểu về sự đánh đổi: nhiều vấn đề không có một đáp án đúng duy nhất, mà có vài cách làm khác nhau, mỗi cách có ưu và nhược điểm. Chúng ta đã khảo sát một số lựa chọn quan trọng nhất ảnh hưởng đến kiến trúc hệ thống dữ liệu, và giới thiệu các thuật ngữ cần thiết để đọc phần còn lại của cuốn sách.

Trước tiên, chúng ta phân biệt hệ thống tác nghiệp (xử lý giao dịch, tức OLTP) với hệ thống phân tích (OLAP), thấy rằng hai loại này không chỉ quản lý các loại dữ liệu có mô hình truy cập khác nhau, mà còn phục vụ những nhóm đối tượng khác nhau. Chúng ta cũng làm quen với kho dữ liệu và data lake, nhận dữ liệu từ hệ thống tác nghiệp qua ETL. [Chương 4](/ch4#ch_storage) sẽ giải thích rằng, vì loại truy vấn cần phục vụ khác nhau, cách bố trí dữ liệu bên trong hệ thống tác nghiệp và hệ thống phân tích thường khác nhau rất nhiều.

Tiếp đó, chúng ta so sánh dịch vụ đám mây, xuất hiện muộn hơn, với mô hình phần mềm self-hosted vốn thống trị kiến trúc hệ thống dữ liệu từ lâu. Cách nào hiệu quả chi phí hơn phụ thuộc nhiều vào hoàn cảnh cụ thể, nhưng không thể phủ nhận rằng cách tiếp cận cloud-native đang thay đổi sâu sắc kiến trúc hệ thống dữ liệu, ví dụ tách rời lưu trữ khỏi tính toán.

Hệ thống đám mây vốn là hệ thống phân tán, và chúng ta cũng đã sơ lược khảo sát một số đánh đổi giữa hệ thống phân tán và giải pháp một máy. Có những tình huống không thể tránh phân tán, nhưng nếu hệ thống vẫn có thể giữ trên một máy, thì không nên vội phân tán hóa nó. [Chương 9](/ch9#ch_distributed) sẽ bàn chi tiết hơn về những thách thức mà hệ thống phân tán mang lại.

Cuối cùng, chúng ta thấy rằng kiến trúc hệ thống dữ liệu không chỉ do nhu cầu của doanh nghiệp triển khai hệ thống quyết định, mà còn chịu ảnh hưởng của các quy định về quyền riêng tư, những quy định này bảo vệ quyền của những người liên quan đến việc xử lý dữ liệu, điều mà nhiều kỹ sư rất dễ bỏ qua. Cách chuyển yêu cầu pháp lý thành hiện thực kỹ thuật hiện vẫn chưa được hiểu đầy đủ, nhưng khi đọc các phần tiếp theo của cuốn sách, hãy luôn giữ vấn đề này trong đầu.

### Tài liệu tham khảo


[^1]: Richard T. Kouzes, Gordon A. Anderson, Stephen T. Elbert, Ian Gorton, and Deborah K. Gracio. [The Changing Paradigm of Data-Intensive Computing](http://www2.ic.uff.br/~boeres/slides_AP/papers/TheChanginParadigmDataIntensiveComputing_2009.pdf). *IEEE Computer*, volume 42, issue 1, January 2009. [doi:10.1109/MC.2009.26](https://doi.org/10.1109/MC.2009.26)
[^2]: Martin Kleppmann, Adam Wiggins, Peter van Hardenberg, and Mark McGranaghan. [Local-first software: you own your data, in spite of the cloud](https://www.inkandswitch.com/local-first/). At *2019 ACM SIGPLAN International Symposium on New Ideas, New Paradigms, and Reflections on Programming and Software* (Onward!), October 2019. [doi:10.1145/3359591.3359737](https://doi.org/10.1145/3359591.3359737)
[^3]: Joe Reis and Matt Housley. [*Fundamentals of Data Engineering*](https://www.oreilly.com/library/view/fundamentals-of-data/9781098108298/). O’Reilly Media, 2022. ISBN: 9781098108304
[^4]: Rui Pedro Machado and Helder Russa. [*Analytics Engineering with SQL and dbt*](https://www.oreilly.com/library/view/analytics-engineering-with/9781098142377/). O’Reilly Media, 2023. ISBN: 9781098142384
[^5]: Edgar F. Codd, S. B. Codd, and C. T. Salley. [Providing OLAP to User-Analysts: An IT Mandate](https://www.estgv.ipv.pt/PaginasPessoais/jloureiro/ESI_AID2007_2008/fichas/codd.pdf). E. F. Codd Associates, 1993. Archived at [perma.cc/RKX8-2GEE](https://perma.cc/RKX8-2GEE)
[^6]: Chinmay Soman and Neha Pawar. [Comparing Three Real-Time OLAP Databases: Apache Pinot, Apache Druid, and ClickHouse](https://startree.ai/blog/a-tale-of-three-real-time-olap-databases). *startree.ai*, April 2023. Archived at [perma.cc/8BZP-VWPA](https://perma.cc/8BZP-VWPA)
[^7]: Surajit Chaudhuri and Umeshwar Dayal. [An Overview of Data Warehousing and OLAP Technology](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/sigrecord.pdf). *ACM SIGMOD Record*, volume 26, issue 1, pages 65–74, March 1997. [doi:10.1145/248603.248616](https://doi.org/10.1145/248603.248616)
[^8]: Fatma Özcan, Yuanyuan Tian, and Pinar Tözün. [Hybrid Transactional/Analytical Processing: A Survey](https://humming80.github.io/papers/sigmod-htaptut.pdf). At *ACM International Conference on Management of Data* (SIGMOD), May 2017. [doi:10.1145/3035918.3054784](https://doi.org/10.1145/3035918.3054784)
[^9]: Adam Prout, Szu-Po Wang, Joseph Victor, Zhou Sun, Yongzhu Li, Jack Chen, Evan Bergeron, Eric Hanson, Robert Walzer, Rodrigo Gomes, and Nikita Shamgunov. [Cloud-Native Transactions and Analytics in SingleStore](https://dl.acm.org/doi/abs/10.1145/3514221.3526055). At *International Conference on Management of Data* (SIGMOD), June 2022. [doi:10.1145/3514221.3526055](https://doi.org/10.1145/3514221.3526055)
[^10]: Chao Zhang, Guoliang Li, Jintao Zhang, Xinning Zhang, and Jianhua Feng. [HTAP Databases: A Survey](https://arxiv.org/pdf/2404.15670). *IEEE Transactions on Knowledge and Data Engineering*, April 2024. [doi:10.1109/TKDE.2024.3389693](https://doi.org/10.1109/TKDE.2024.3389693)
[^11]: Michael Stonebraker and Uğur Çetintemel. [‘One Size Fits All’: An Idea Whose Time Has Come and Gone](https://pages.cs.wisc.edu/~shivaram/cs744-readings/fits_all.pdf). At *21st International Conference on Data Engineering* (ICDE), April 2005. [doi:10.1109/ICDE.2005.1](https://doi.org/10.1109/ICDE.2005.1)
[^12]: Jeffrey Cohen, Brian Dolan, Mark Dunlap, Joseph M. Hellerstein, and Caleb Welton. [MAD Skills: New Analysis Practices for Big Data](https://www.vldb.org/pvldb/vol2/vldb09-219.pdf). *Proceedings of the VLDB Endowment*, volume 2, issue 2, pages 1481–1492, August 2009. [doi:10.14778/1687553.1687576](https://doi.org/10.14778/1687553.1687576)
[^13]: Dan Olteanu. [The Relational Data Borg is Learning](https://www.vldb.org/pvldb/vol13/p3502-olteanu.pdf). *Proceedings of the VLDB Endowment*, volume 13, issue 12, August 2020. [doi:10.14778/3415478.3415572](https://doi.org/10.14778/3415478.3415572)
[^14]: Matt Bornstein, Martin Casado, and Jennifer Li. [Emerging Architectures for Modern Data Infrastructure: 2020](https://future.a16z.com/emerging-architectures-for-modern-data-infrastructure-2020/). *future.a16z.com*, October 2020. Archived at [perma.cc/LF8W-KDCC](https://perma.cc/LF8W-KDCC)
[^15]: Martin Fowler. [DataLake](https://www.martinfowler.com/bliki/DataLake.html). *martinfowler.com*, February 2015. Archived at [perma.cc/4WKN-CZUK](https://perma.cc/4WKN-CZUK)
[^16]: Bobby Johnson and Joseph Adler. [The Sushi Principle: Raw Data Is Better](https://learning.oreilly.com/videos/strata-hadoop/9781491924143/9781491924143-video210840/). At *Strata+Hadoop World*, February 2015.
[^17]: Michael Armbrust, Ali Ghodsi, Reynold Xin, and Matei Zaharia. [Lakehouse: A New Generation of Open Platforms that Unify Data Warehousing and Advanced Analytics](https://www.cidrdb.org/cidr2021/papers/cidr2021_paper17.pdf). At *11th Annual Conference on Innovative Data Systems Research* (CIDR), January 2021.
[^18]: DataKitchen, Inc. [The DataOps Manifesto](https://dataopsmanifesto.org/en/). *dataopsmanifesto.org*, 2017. Archived at [perma.cc/3F5N-FUQ4](https://perma.cc/3F5N-FUQ4)
[^19]: Tejas Manohar. [What is Reverse ETL: A Definition & Why It’s Taking Off](https://hightouch.io/blog/reverse-etl/). *hightouch.io*, November 2021. Archived at [perma.cc/A7TN-GLYJ](https://perma.cc/A7TN-GLYJ)
[^20]: Simon O’Regan. [Designing Data Products](https://towardsdatascience.com/designing-data-products-b6b93edf3d23). *towardsdatascience.com*, August 2018. Archived at [perma.cc/HU67-3RV8](https://perma.cc/HU67-3RV8)
[^21]: Camille Fournier. [Why is it so hard to decide to buy?](https://skamille.medium.com/why-is-it-so-hard-to-decide-to-buy-d86fee98e88e) *skamille.medium.com*, July 2021. Archived at [perma.cc/6VSG-HQ5X](https://perma.cc/6VSG-HQ5X)
[^22]: David Heinemeier Hansson. [Why we’re leaving the cloud](https://world.hey.com/dhh/why-we-re-leaving-the-cloud-654b47e0). *world.hey.com*, October 2022. Archived at [perma.cc/82E6-UJ65](https://perma.cc/82E6-UJ65)
[^23]: Nima Badizadegan. [Use One Big Server](https://specbranch.com/posts/one-big-server/). *specbranch.com*, August 2022. Archived at [perma.cc/M8NB-95UK](https://perma.cc/M8NB-95UK)
[^24]: Steve Yegge. [Dear Google Cloud: Your Deprecation Policy is Killing You](https://steve-yegge.medium.com/dear-google-cloud-your-deprecation-policy-is-killing-you-ee7525dc05dc). *steve-yegge.medium.com*, August 2020. Archived at [perma.cc/KQP9-SPGU](https://perma.cc/KQP9-SPGU)
[^25]: Alexandre Verbitski, Anurag Gupta, Debanjan Saha, Murali Brahmadesam, Kamal Gupta, Raman Mittal, Sailesh Krishnamurthy, Sandor Maurice, Tengiz Kharatishvili, and Xiaofeng Bao. [Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases](https://media.amazonwebservices.com/blog/2017/aurora-design-considerations-paper.pdf). At *ACM International Conference on Management of Data* (SIGMOD), pages 1041–1052, May 2017. [doi:10.1145/3035918.3056101](https://doi.org/10.1145/3035918.3056101)
[^26]: Panagiotis Antonopoulos, Alex Budovski, Cristian Diaconu, Alejandro Hernandez Saenz, Jack Hu, Hanuma Kodavalla, Donald Kossmann, Sandeep Lingam, Umar Farooq Minhas, Naveen Prakash, Vijendra Purohit, Hugh Qu, Chaitanya Sreenivas Ravella, Krystyna Reisteter, Sheetal Shrotri, Dixin Tang, and Vikram Wakade. [Socrates: The New SQL Server in the Cloud](https://www.microsoft.com/en-us/research/uploads/prod/2019/05/socrates.pdf). At *ACM International Conference on Management of Data* (SIGMOD), pages 1743–1756, June 2019. [doi:10.1145/3299869.3314047](https://doi.org/10.1145/3299869.3314047)
[^27]: Midhul Vuppalapati, Justin Miron, Rachit Agarwal, Dan Truong, Ashish Motivala, and Thierry Cruanes. [Building An Elastic Query Engine on Disaggregated Storage](https://www.usenix.org/system/files/nsdi20-paper-vuppalapati.pdf). At *17th USENIX Symposium on Networked Systems Design and Implementation* (NSDI), February 2020.
[^28]: Nick Van Wiggeren. [The Real Failure Rate of EBS](https://planetscale.com/blog/the-real-fail-rate-of-ebs). *planetscale.com*, March 2025. Archived at [perma.cc/43CR-SAH5](https://perma.cc/43CR-SAH5)
[^29]: Colin Breck. [Predicting the Future of Distributed Systems](https://blog.colinbreck.com/predicting-the-future-of-distributed-systems/). *blog.colinbreck.com*, August 2024. Archived at [perma.cc/K5FC-4XX2](https://perma.cc/K5FC-4XX2)
[^30]: Gwen Shapira. [Compute-Storage Separation Explained](https://perma.cc/QCV3-XJNZ). *thenile.dev*, January 2023. Archived at [perma.cc/QCV3-XJNZ](https://perma.cc/QCV3-XJNZ)
[^31]: Ravi Murthy and Gurmeet Goindi. [AlloyDB for PostgreSQL under the hood: Intelligent, database-aware storage](https://cloud.google.com/blog/products/databases/alloydb-for-postgresql-intelligent-scalable-storage). *cloud.google.com*, May 2022. Archived at [archive.org](https://web.archive.org/web/20220514021120/https%3A//cloud.google.com/blog/products/databases/alloydb-for-postgresql-intelligent-scalable-storage)
[^32]: Jack Vanlightly. [The Architecture of Serverless Data Systems](https://jack-vanlightly.com/blog/2023/11/14/the-architecture-of-serverless-data-systems). *jack-vanlightly.com*, November 2023. Archived at [perma.cc/UDV4-TNJ5](https://perma.cc/UDV4-TNJ5)
[^33]: Eric Jonas, Johann Schleier-Smith, Vikram Sreekanti, Chia-Che Tsai, Anurag Khandelwal, Qifan Pu, Vaishaal Shankar, Joao Carreira, Karl Krauth, Neeraja Yadwadkar, Joseph E. Gonzalez, Raluca Ada Popa, Ion Stoica, David A. Patterson. [Cloud Programming Simplified: A Berkeley View on Serverless Computing](https://arxiv.org/abs/1902.03383). *arxiv.org*, February 2019.
[^34]: Betsy Beyer, Jennifer Petoff, Chris Jones, and Niall Richard Murphy. [*Site Reliability Engineering: How Google Runs Production Systems*](https://www.oreilly.com/library/view/site-reliability-engineering/9781491929117/). O’Reilly Media, 2016. ISBN: 9781491929124
[^35]: Thomas Limoncelli. [The Time I Stole $10,000 from Bell Labs](https://queue.acm.org/detail.cfm?id=3434773). *ACM Queue*, volume 18, issue 5, November 2020. [doi:10.1145/3434571.3434773](https://doi.org/10.1145/3434571.3434773)
[^36]: Charity Majors. [The Future of Ops Jobs](https://acloudguru.com/blog/engineering/the-future-of-ops-jobs). *acloudguru.com*, August 2020. Archived at [perma.cc/GRU2-CZG3](https://perma.cc/GRU2-CZG3)
[^37]: Boris Cherkasky. [(Over)Pay As You Go for Your Datastore](https://medium.com/riskified-technology/over-pay-as-you-go-for-your-datastore-11a29ae49a8b). *medium.com*, September 2021. Archived at [perma.cc/Q8TV-2AM2](https://perma.cc/Q8TV-2AM2)
[^38]: Shlomi Kushchi. [Serverless Doesn’t Mean DevOpsLess or NoOps](https://thenewstack.io/serverless-doesnt-mean-devopsless-or-noops/). *thenewstack.io*, February 2023. Archived at [perma.cc/3NJR-AYYU](https://perma.cc/3NJR-AYYU)
[^39]: Erik Bernhardsson. [Storm in the stratosphere: how the cloud will be reshuffled](https://erikbern.com/2021/11/30/storm-in-the-stratosphere-how-the-cloud-will-be-reshuffled.html). *erikbern.com*, November 2021. Archived at [perma.cc/SYB2-99P3](https://perma.cc/SYB2-99P3)
[^40]: Benn Stancil. [The data OS](https://benn.substack.com/p/the-data-os). *benn.substack.com*, September 2021. Archived at [perma.cc/WQ43-FHS6](https://perma.cc/WQ43-FHS6)
[^41]: Maria Korolov. [Data residency laws pushing companies toward residency as a service](https://www.csoonline.com/article/3647761/data-residency-laws-pushing-companies-toward-residency-as-a-service.html). *csoonline.com*, January 2022. Archived at [perma.cc/CHE4-XZZ2](https://perma.cc/CHE4-XZZ2)
[^42]: Severin Borenstein. [Can Data Centers Flex Their Power Demand?](https://energyathaas.wordpress.com/2025/04/14/can-data-centers-flex-their-power-demand/) *energyathaas.wordpress.com*, April 2025. Archived at <https://perma.cc/MUD3-A6FF>
[^43]: Bilge Acun, Benjamin Lee, Fiodar Kazhamiaka, Aditya Sundarrajan, Kiwan Maeng, Manoj Chakkaravarthy, David Brooks, and Carole-Jean Wu. [Carbon Dependencies in Datacenter Design and Management](https://hotcarbon.org/assets/2022/pdf/hotcarbon22-acun.pdf). *ACM SIGENERGY Energy Informatics Review*, volume 3, issue 3, pages 21–26. [doi:10.1145/3630614.3630619](https://doi.org/10.1145/3630614.3630619)
[^44]: Kousik Nath. [These are the numbers every computer engineer should know](https://www.freecodecamp.org/news/must-know-numbers-for-every-computer-engineer/). *freecodecamp.org*, September 2019. Archived at [perma.cc/RW73-36RL](https://perma.cc/RW73-36RL)
[^45]: Joseph M. Hellerstein, Jose Faleiro, Joseph E. Gonzalez, Johann Schleier-Smith, Vikram Sreekanti, Alexey Tumanov, and Chenggang Wu. [Serverless Computing: One Step Forward, Two Steps Back](https://arxiv.org/abs/1812.03651). At *Conference on Innovative Data Systems Research* (CIDR), January 2019.
[^46]: Frank McSherry, Michael Isard, and Derek G. Murray. [Scalability! But at What COST?](https://www.usenix.org/system/files/conference/hotos15/hotos15-paper-mcsherry.pdf) At *15th USENIX Workshop on Hot Topics in Operating Systems* (HotOS), May 2015.
[^47]: Cindy Sridharan. *[Distributed Systems Observability: A Guide to Building Robust Systems](https://perma.cc/M6JL-XKCM)*. Report, O’Reilly Media, May 2018. Archived at [perma.cc/M6JL-XKCM](https://perma.cc/M6JL-XKCM)
[^48]: Charity Majors. [Observability — A 3-Year Retrospective](https://thenewstack.io/observability-a-3-year-retrospective/). *thenewstack.io*, August 2019. Archived at [perma.cc/CG62-TJWL](https://perma.cc/CG62-TJWL)
[^49]: Benjamin H. Sigelman, Luiz André Barroso, Mike Burrows, Pat Stephenson, Manoj Plakal, Donald Beaver, Saul Jaspan, and Chandan Shanbhag. [Dapper, a Large-Scale Distributed Systems Tracing Infrastructure](https://research.google/pubs/pub36356/). Google Technical Report dapper-2010-1, April 2010. Archived at [perma.cc/K7KU-2TMH](https://perma.cc/K7KU-2TMH)
[^50]: Rodrigo Laigner, Yongluan Zhou, Marcos Antonio Vaz Salles, Yijian Liu, and Marcos Kalinowski. [Data management in microservices: State of the practice, challenges, and research directions](https://www.vldb.org/pvldb/vol14/p3348-laigner.pdf). *Proceedings of the VLDB Endowment*, volume 14, issue 13, pages 3348–3361, September 2021. [doi:10.14778/3484224.3484232](https://doi.org/10.14778/3484224.3484232)
[^51]: Jordan Tigani. [Big Data is Dead](https://motherduck.com/blog/big-data-is-dead/). *motherduck.com*, February 2023. Archived at [perma.cc/HT4Q-K77U](https://perma.cc/HT4Q-K77U)
[^52]: Sam Newman. [*Building Microservices*, second edition](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/). O’Reilly Media, 2021. ISBN: 9781492034025
[^53]: Chris Richardson. [Microservices: Decomposing Applications for Deployability and Scalability](https://www.infoq.com/articles/microservices-intro/). *infoq.com*, May 2014. Archived at [perma.cc/CKN4-YEQ2](https://perma.cc/CKN4-YEQ2)
[^54]: Mohammad Shahrad, Rodrigo Fonseca, Íñigo Goiri, Gohar Chaudhry, Paul Batum, Jason Cooke, Eduardo Laureano, Colby Tresness, Mark Russinovich, Ricardo Bianchini. [Serverless in the Wild: Characterizing and Optimizing the Serverless Workload at a Large Cloud Provider](https://www.usenix.org/system/files/atc20-shahrad.pdf). At *USENIX Annual Technical Conference* (ATC), July 2020.
[^55]: Luiz André Barroso, Urs Hölzle, and Parthasarathy Ranganathan. [The Datacenter as a Computer: Designing Warehouse-Scale Machines](https://www.morganclaypool.com/doi/10.2200/S00874ED3V01Y201809CAC046), third edition. Morgan & Claypool Synthesis Lectures on Computer Architecture, October 2018. [doi:10.2200/S00874ED3V01Y201809CAC046](https://doi.org/10.2200/S00874ED3V01Y201809CAC046)
[^56]: David Fiala, Frank Mueller, Christian Engelmann, Rolf Riesen, Kurt Ferreira, and Ron Brightwell. [Detection and Correction of Silent Data Corruption for Large-Scale High-Performance Computing](https://arcb.csc.ncsu.edu/~mueller/ftp/pub/mueller/papers/sc12.pdf),” at *International Conference for High Performance Computing, Networking, Storage and Analysis* (SC), November 2012. [doi:10.1109/SC.2012.49](https://doi.org/10.1109/SC.2012.49)
[^57]: Anna Kornfeld Simpson, Adriana Szekeres, Jacob Nelson, and Irene Zhang. [Securing RDMA for High-Performance Datacenter Storage Systems](https://www.usenix.org/conference/hotcloud20/presentation/kornfeld-simpson). At *12th USENIX Workshop on Hot Topics in Cloud Computing* (HotCloud), July 2020.
[^58]: Arjun Singh, Joon Ong, Amit Agarwal, Glen Anderson, Ashby Armistead, Roy Bannon, Seb Boving, Gaurav Desai, Bob Felderman, Paulie Germano, Anand Kanagala, Jeff Provost, Jason Simmons, Eiichi Tanda, Jim Wanderer, Urs Hölzle, Stephen Stuart, and Amin Vahdat. [Jupiter Rising: A Decade of Clos Topologies and Centralized Control in Google’s Datacenter Network](https://conferences.sigcomm.org/sigcomm/2015/pdf/papers/p183.pdf). At *Annual Conference of the ACM Special Interest Group on Data Communication* (SIGCOMM), August 2015. [doi:10.1145/2785956.2787508](https://doi.org/10.1145/2785956.2787508)
[^59]: Glenn K. Lockwood. [Hadoop’s Uncomfortable Fit in HPC](https://blog.glennklockwood.com/2014/05/hadoops-uncomfortable-fit-in-hpc.html). *glennklockwood.blogspot.co.uk*, May 2014. Archived at [perma.cc/S8XX-Y67B](https://perma.cc/S8XX-Y67B)
[^60]: Cathy O’Neil: *Weapons of Math Destruction: How Big Data Increases Inequality and Threatens Democracy*. Crown Publishing, 2016. ISBN: 9780553418811
[^61]: Supreeth Shastri, Vinay Banakar, Melissa Wasserman, Arun Kumar, and Vijay Chidambaram. [Understanding and Benchmarking the Impact of GDPR on Database Systems](https://www.vldb.org/pvldb/vol13/p1064-shastri.pdf). *Proceedings of the VLDB Endowment*, volume 13, issue 7, pages 1064–1077, March 2020. [doi:10.14778/3384345.3384354](https://doi.org/10.14778/3384345.3384354)
[^62]: Martin Fowler. [Datensparsamkeit](https://www.martinfowler.com/bliki/Datensparsamkeit.html). *martinfowler.com*, December 2013. Archived at [perma.cc/R9QX-CME6](https://perma.cc/R9QX-CME6)
[^63]: [Regulation (EU) 2016/679 of the European Parliament and of the Council of 27 April 2016 (General Data Protection Regulation)](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679&from=EN). *Official Journal of the European Union* L 119/1, May 2016.
