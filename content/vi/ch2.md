---
translationKey: ddia-v2-ch2
title: Định nghĩa yêu cầu phi chức năng
book_kind: chapter
book_number: "2"
book_part: I
weight: 102
breadcrumbs: false
---

<a id="ch_nonfunctional"></a>

![](/map/ch01.png)

> *Internet được làm tốt đến mức hầu hết mọi người coi nó như một nguồn tài nguyên thiên nhiên như Thái Bình Dương, chứ không phải một sản phẩm do con người tạo ra. Lần gần nhất có một công nghệ ở quy mô như vậy mà gần như không mắc lỗi là khi nào?*
>
> [Alan Kay](https://web.archive.org/web/20120712231854/http://www.drdobbs.com/architecture-and-design/interview-with-alan-kay/240003442),
> trả lời phỏng vấn *Dr Dobb’s Journal* (2012)

Khi xây dựng một ứng dụng, bạn sẽ đối mặt với một danh sách yêu cầu. Đứng đầu danh sách đó rất có thể là các chức năng mà ứng dụng phải cung cấp: cần những màn hình và nút bấm nào, mỗi thao tác phải làm được gì để đạt mục đích của phần mềm. Đó là các *yêu cầu chức năng* (*functional requirements*).

Ngoài ra, bạn có thể còn có những *yêu cầu phi chức năng* (*nonfunctional requirements*): chẳng hạn, ứng dụng phải nhanh, đáng tin cậy, an toàn, tuân thủ pháp luật và dễ bảo trì. Những yêu cầu này thường không được viết ra rõ ràng vì chúng có vẻ hiển nhiên, nhưng tầm quan trọng của chúng không hề kém chức năng của ứng dụng: một ứng dụng chậm đến mức không chịu nổi, hoặc quá thiếu tin cậy, gần như coi như không tồn tại.

Nhiều yêu cầu phi chức năng, như bảo mật, nằm ngoài phạm vi của cuốn sách này. Tuy nhiên, chương này sẽ bàn một số yêu cầu trong đó và giúp bạn diễn đạt chính xác hệ thống của mình cần đạt những gì:

* Cách định nghĩa và đo *hiệu năng* (*performance*) của hệ thống (xem [“Mô tả hiệu năng”](/ch2#sec_introduction_percentiles));
* Một dịch vụ *đáng tin cậy* (*reliable*) nghĩa là gì — tức là vẫn tiếp tục hoạt động đúng ngay cả khi có sự cố (xem [“Độ tin cậy và khả năng chịu lỗi”](/ch2#sec_introduction_reliability));
* Khả năng tăng năng lực tính toán một cách hiệu quả khi tải của hệ thống tăng, giúp hệ thống có *khả năng mở rộng* (*scalability*, xem [“Khả năng mở rộng”](/ch2#sec_introduction_scalability)); và
* Cách làm cho hệ thống dễ bảo trì hơn trong quá trình sử dụng lâu dài (xem [“Khả năng bảo trì”](/ch2#sec_introduction_maintainability)).

Các thuật ngữ giới thiệu trong chương này còn được dùng lại ở những chương sau, khi ta đi sâu vào chi tiết triển khai của các hệ thống chuyên sâu về dữ liệu. Tuy nhiên, những định nghĩa trừu tượng khó tránh khỏi khô khan. Để các khái niệm cụ thể hơn, ta bắt đầu bằng một ví dụ triển khai dịch vụ mạng xã hội, qua đó minh họa hiệu năng và khả năng mở rộng có ý nghĩa gì trong thực tế.


## Nghiên cứu tình huống: dòng thời gian trang chủ của mạng xã hội {#sec_introduction_twitter}

Giả sử nhiệm vụ của bạn là xây dựng một mạng xã hội giống X (trước đây là Twitter), trong đó người dùng có thể đăng bài và theo dõi người dùng khác. Cách triển khai ở đây sẽ đơn giản hơn nhiều so với dịch vụ thật [^1] [^2] [^3], nhưng cũng đủ để minh họa một số vấn đề mà hệ thống quy mô lớn gặp phải.

Giả sử mỗi ngày người dùng đăng 500 triệu bài, trung bình 5.700 bài mỗi giây. Thỉnh thoảng, tốc độ đăng bài có thể vọt lên 150.000 bài mỗi giây [^4]. Giả sử thêm rằng mỗi người dùng trung bình theo dõi 200 người và cũng có 200 người theo dõi (thực tế phân bố rất rộng: đa số chỉ có vài người theo dõi, còn một số ít người nổi tiếng như Barack Obama có hơn một trăm triệu người theo dõi).

### Biểu diễn người dùng, bài đăng và quan hệ theo dõi {#id20}

Giả sử ta lưu toàn bộ dữ liệu trong một cơ sở dữ liệu quan hệ, như {{< xref fig="2-1" page="/ch2" anchor="fig_twitter_relational" >}}Hình 2-1{{< /xref >}}: một bảng lưu người dùng, một bảng lưu bài đăng và một bảng lưu quan hệ theo dõi.

{{< fig num="2-1" id="fig_twitter_relational" src="/fig/ddia_0201.png" caption="Lược đồ quan hệ đơn giản cho một mạng xã hội cho phép người dùng theo dõi lẫn nhau." class="ddia-figure ddia-figure--panorama" width="1772" height="542" />}}

Giả sử thao tác đọc chính mà mạng xã hội này cần hỗ trợ là *home timeline* — hiển thị các bài đăng gần đây của những người bạn theo dõi (để đơn giản, ta bỏ qua quảng cáo, bài gợi ý từ người dùng chưa theo dõi và các tính năng mở rộng khác). Có thể lấy home timeline của một người dùng bằng truy vấn SQL sau:

```sql
SELECT posts.*, users.* FROM posts
    JOIN follows ON posts.sender_id = follows.followee_id
    JOIN users ON posts.sender_id = users.id
    WHERE follows.follower_id = current_user
    ORDER BY posts.timestamp DESC
    LIMIT 1000
```

Khi thực thi truy vấn này, cơ sở dữ liệu dùng bảng `follows` để tìm tất cả những người mà `current_user` theo dõi, tra các bài đăng gần đây của những người dùng đó, rồi sắp xếp theo dấu thời gian để lấy 1.000 bài mới nhất.

Bài đăng rất cần tính kịp thời, nên giả sử sau khi ai đó đăng bài, ta muốn những người theo dõi họ thấy bài trong vòng 5 giây. Một cách là mỗi khi người dùng còn trực tuyến, client lặp lại truy vấn trên mỗi 5 giây (gọi là *polling*). Nếu có 10 triệu người dùng đang trực tuyến và đăng nhập cùng lúc, điều đó có nghĩa là mỗi giây phải chạy 2 triệu truy vấn. Kể cả kéo dài khoảng cách giữa các lần polling, con số này vẫn rất lớn.

Hơn nữa, bản thân truy vấn này cũng không rẻ: nếu bạn theo dõi 200 người, cơ sở dữ liệu phải lấy các bài đăng gần đây của từng người trong 200 người đó, rồi gộp các danh sách lại. Với 2 triệu truy vấn timeline mỗi giây, cơ sở dữ liệu phải tra bài đăng gần đây của một người đăng 400 triệu lần mỗi giây — một con số kinh người, mà đó còn mới chỉ là trường hợp trung bình. Có người dùng theo dõi hàng chục nghìn tài khoản. Chạy truy vấn này cho họ cực kỳ tốn kém và cũng khó đảm bảo tốc độ.

### Vật chất hóa và cập nhật timeline {#sec_introduction_materializing}

Làm sao để làm tốt hơn? Thứ nhất, thay vì để client polling, tốt hơn là để máy chủ chủ động đẩy bài đăng mới đến những người theo dõi đang trực tuyến. Thứ hai, ta nên tính trước kết quả của truy vấn trên, để yêu cầu lấy home timeline được phục vụ trực tiếp từ cache.

Hãy hình dung ta lưu cho mỗi người dùng một cấu trúc dữ liệu chứa home timeline của họ, tức các bài đăng gần đây của những người họ theo dõi. Mỗi khi có người đăng bài, ta tìm tất cả những người theo dõi họ rồi chèn bài đăng đó vào home timeline của từng người, giống như bỏ thư vào từng hộp thư. Như vậy, khi người dùng đăng nhập, ta chỉ cần đưa họ timeline đã tính sẵn. Để nhận thông báo về bài mới trên timeline, client chỉ cần đăng ký theo dõi luồng bài đăng liên tục được thêm vào home timeline của người đó.

Nhược điểm của cách này là mỗi lần đăng bài phải làm nhiều việc hơn, vì home timeline là *dữ liệu dẫn xuất* (*derived data*) cần được cập nhật theo. Toàn bộ quá trình được minh họa ở {{< xref fig="2-2" page="/ch2" anchor="fig_twitter_timelines" >}}Hình 2-2{{< /xref >}}. Khi một yêu cầu ban đầu kéo theo nhiều yêu cầu phía sau, ta dùng *fan-out* để chỉ hệ số khuếch đại số lượng yêu cầu.

{{< fig num="2-2" id="fig_twitter_timelines" src="/fig/ddia_0202.png" caption="Fan-out — chuyển bài đăng mới đến từng người theo dõi của người đăng." class="ddia-figure ddia-figure--wide" width="1772" height="638" />}}

Với tốc độ 5.700 bài mỗi giây, nếu mỗi bài trung bình được gửi đến 200 người theo dõi (tức hệ số fan-out là 200), mỗi giây cần thực hiện hơn 1 triệu lần ghi home timeline một chút. Con số này vẫn lớn, nhưng so với 400 triệu lần tra bài đăng gần đây theo người đăng mỗi giây của phương án kia thì đã tiết kiệm được nhiều công việc.

Nếu một sự kiện đặc biệt khiến tốc độ đăng bài tăng vọt, ta không cần chuyển phát mọi timeline ngay lập tức. Có thể đưa các tác vụ chuyển phát vào hàng đợi và chấp nhận việc bài đăng tạm thời sẽ xuất hiện chậm hơn một chút trên timeline của người theo dõi. Ngay cả khi có đỉnh tải như vậy, timeline vẫn tải nhanh, vì việc đọc chỉ cần truy cập cache.

Quá trình tính trước và liên tục cập nhật kết quả truy vấn như vậy gọi là *vật chất hóa* (*materialization*), và cache timeline là một ví dụ về *khung nhìn vật chất hóa* (*materialized view*) (ta sẽ bàn thêm khái niệm này ở [“Duy trì khung nhìn vật chất hóa”](/ch12#sec_stream_mat_view)). Khung nhìn vật chất hóa giúp đọc nhanh hơn, đổi lại là khi ghi phải làm nhiều việc hơn. Với đa số người dùng, chi phí ghi không cao, nhưng mạng xã hội còn phải tính đến một số trường hợp cực đoan:

* Nếu một người dùng theo dõi rất nhiều tài khoản và các tài khoản đó đăng bài thường xuyên, tốc độ ghi vào timeline vật chất hóa của họ sẽ rất cao. Tuy nhiên, người dùng này nhiều khả năng cũng không đọc hết các bài trong timeline, nên có thể bỏ bớt một phần các lần ghi timeline và chỉ hiển thị cho họ một mẫu các bài đăng từ những tài khoản họ theo dõi [^5].
* Nếu một người nổi tiếng có lượng người theo dõi khổng lồ đăng bài, ta phải làm rất nhiều việc để chèn bài đó vào home timeline của hàng triệu người. Trong trường hợp này, bỏ bớt các lần ghi là không chấp nhận được. Một giải pháp là xử lý bài đăng của người nổi tiếng tách khỏi bài đăng của người khác: thay vì cố công thêm bài của họ vào hàng triệu timeline, ta lưu riêng và gộp với timeline vật chất hóa khi đọc. Dù đã áp dụng những tối ưu như vậy, để phục vụ các tài khoản nổi tiếng, mạng xã hội vẫn có thể cần rất nhiều hạ tầng [^6].

## Mô tả hiệu năng {#sec_introduction_percentiles}

Khi bàn về hiệu năng phần mềm, người ta thường xét hai loại chỉ số chính:

*thời gian phản hồi* (*response time*)
: Khoảng thời gian từ lúc người dùng gửi yêu cầu đến lúc nhận được phản hồi cần thiết. Đơn vị đo là giây (hoặc mili giây, micro giây).

*thông lượng* (*throughput*)
: Số yêu cầu hoặc lượng dữ liệu hệ thống xử lý được mỗi giây. Với một lượng tài nguyên phần cứng cho trước, thông lượng hệ thống xử lý được có giới hạn trên, gọi là *thông lượng tối đa* (*maximum throughput*). Đơn vị đo thường viết dưới dạng “bao nhiêu … mỗi giây”.

Trong tình huống mạng xã hội, “số bài đăng mỗi giây” và “số lần ghi timeline mỗi giây” là các chỉ số thông lượng, còn “thời gian cần để tải home timeline” và “thời gian để bài đăng đến được người theo dõi” là các chỉ số thời gian phản hồi.

Giữa thông lượng và thời gian phản hồi thường có mối liên hệ. {{< xref fig="2-3" page="/ch2" anchor="fig_throughput" >}}Hình 2-3{{< /xref >}} phác họa một mối quan hệ điển hình ở dịch vụ trực tuyến. Khi thông lượng yêu cầu thấp, thời gian phản hồi của dịch vụ ngắn. Khi tải tăng, thời gian phản hồi tăng theo. Nguyên nhân là *xếp hàng* (*queueing*): khi một yêu cầu đến một hệ thống đang tải cao, rất có thể CPU đang xử lý các yêu cầu trước đó, nên yêu cầu mới phải chờ đến khi yêu cầu trước xử lý xong. Khi thông lượng tiến gần giới hạn xử lý của phần cứng, độ trễ xếp hàng tăng vọt.

{{< fig num="2-3" id="fig_throughput" src="/fig/ddia_0203.png" caption="Khi thông lượng của dịch vụ tiến gần giới hạn năng lực xử lý, xếp hàng khiến thời gian phản hồi tăng vọt." class="ddia-figure ddia-figure--wide" width="2953" height="1018" />}}


<a id="sidebar_metastable"></a>

> [!TIP] Khi hệ thống quá tải không thể phục hồi
>
> Khi hệ thống gần quá tải và thông lượng bị đẩy đến sát giới hạn, đôi khi nó rơi vào một vòng luẩn quẩn: hệ thống ngày càng kém hiệu quả nên càng quá tải. Chẳng hạn, nếu hàng dài các yêu cầu đang chờ xử lý, thời gian phản hồi có thể tăng đến mức client hết thời gian chờ (timeout) và gửi lại yêu cầu. Tốc độ yêu cầu vì thế tăng thêm, khiến vấn đề càng trầm trọng — đây là *retry storm*. Ngay cả khi tải sau đó giảm xuống, hệ thống vẫn có thể mắc kẹt trong trạng thái quá tải cho đến khi được khởi động lại hoặc đặt lại bằng cách khác. Hiện tượng này gọi là *metastable failure*, và có thể gây gián đoạn nghiêm trọng cho hệ thống production [^7] [^8].
>
> Để tránh việc gửi lại làm sập dịch vụ, có thể tăng dần và ngẫu nhiên hóa khoảng chờ giữa các lần thử lại liên tiếp ở phía client (*exponential backoff* [^9] [^10]). Cũng có thể dùng thuật toán *circuit breaker* [^11] [^12] hoặc *token bucket* [^13] để tạm ngừng gửi yêu cầu đến dịch vụ vừa trả lỗi hoặc timeout gần đây. Máy chủ cũng có thể chủ động từ chối yêu cầu khi nhận thấy mình sắp quá tải (*load shedding* [^14]), và trong phản hồi yêu cầu client giảm tốc độ gửi (*backpressure* [^1] [^15]). Việc chọn thuật toán xếp hàng và cân bằng tải cũng có ảnh hưởng [^16].

Trong các chỉ số hiệu năng, người dùng thường quan tâm nhất đến thời gian phản hồi, còn thông lượng quyết định lượng tài nguyên tính toán cần thiết (ví dụ số máy chủ), do đó quyết định chi phí để xử lý một khối lượng công việc nhất định. Nếu thông lượng có thể tăng vượt khả năng xử lý của phần cứng hiện tại, cần mở rộng năng lực. Nếu việc tăng tài nguyên tính toán giúp nâng đáng kể thông lượng tối đa của hệ thống, ta nói hệ thống có *khả năng mở rộng* (*scalability*).

Phần này chủ yếu tập trung vào thời gian phản hồi. Ta sẽ quay lại thông lượng và khả năng mở rộng ở mục [“Khả năng mở rộng”](/ch2#sec_introduction_scalability).

### Độ trễ và thời gian phản hồi {#id23}

“Độ trễ” và “thời gian phản hồi” đôi khi bị dùng lẫn lộn, nhưng cuốn sách này dùng các thuật ngữ đó theo nghĩa cụ thể sau (như {{< xref fig="2-4" page="/ch2" anchor="fig_response_time" >}}Hình 2-4{{< /xref >}}):

* *Thời gian phản hồi* (*response time*) là thời gian mà client nhìn thấy, bao gồm mọi độ trễ phát sinh ở bất cứ đâu trong hệ thống.
* *Thời gian phục vụ* (*service time*) là khoảng thời gian dịch vụ thực sự dùng để xử lý yêu cầu của người dùng.
* *Độ trễ xếp hàng* (*queueing delay*) có thể xảy ra ở nhiều chỗ trong quy trình. Chẳng hạn, sau khi yêu cầu đến, có thể nó phải đợi đến khi CPU rảnh mới bắt đầu được xử lý. Gói dữ liệu phản hồi cũng có thể phải chờ trong bộ đệm trước, nếu các tác vụ khác trên cùng máy đang gửi lượng lớn dữ liệu qua giao diện mạng đầu ra.
* *Độ trễ* (*latency*) nói chung là khoảng thời gian yêu cầu không được xử lý thực sự, tức thời gian yêu cầu ở trạng thái *tiềm ẩn* (*latent*). Cụ thể, *độ trễ mạng* (*network latency*, còn gọi là *network delay*) là thời gian yêu cầu và phản hồi di chuyển trong mạng.

{{< fig num="2-4" id="fig_response_time" src="/fig/ddia_0204.png" caption="Thời gian phản hồi, thời gian phục vụ, độ trễ mạng và độ trễ xếp hàng." class="ddia-figure ddia-figure--wide" width="2953" height="1018" />}}

Trong {{< xref fig="2-4" page="/ch2" anchor="fig_response_time" >}}Hình 2-4{{< /xref >}}, thời gian trôi từ trái sang phải. Mỗi node tham gia giao tiếp được vẽ bằng một đường ngang, còn thông điệp yêu cầu hoặc phản hồi được vẽ thành mũi tên xiên đậm từ node này sang node khác. Ta sẽ thường gặp cách vẽ này ở các phần sau của sách.

Ngay cả khi gửi đi gửi lại cùng một yêu cầu, thời gian phản hồi mỗi lần cũng có thể chênh lệch rất nhiều. Nhiều yếu tố có thể gây thêm độ trễ ngẫu nhiên, chẳng hạn: chuyển ngữ cảnh sang một tiến trình nền, mất gói mạng và truyền lại TCP, tạm dừng do thu gom rác, lỗi trang buộc hệ thống đọc dữ liệu từ đĩa, rung cơ học của giá đặt máy chủ [^17], v.v. Ta sẽ bàn thêm vấn đề này ở [“Timeout và độ trễ không giới hạn”](/ch9#sec_distributed_queueing).

Phần lớn dao động của thời gian phản hồi thường đến từ độ trễ xếp hàng. Máy chủ chỉ xử lý được đồng thời một số tác vụ có hạn (ví dụ bị giới hạn bởi số nhân CPU), nên chỉ cần vài yêu cầu chậm là đủ chặn các yêu cầu phía sau, hiện tượng này gọi là *head-of-line blocking*. Dù các yêu cầu sau tự thân có thời gian phục vụ ngắn, thời gian phản hồi tổng thể mà client thấy vẫn dài, vì chúng phải chờ các yêu cầu trước hoàn tất. Độ trễ xếp hàng không thuộc thời gian phục vụ, nên việc đo thời gian phản hồi ở phía client là rất quan trọng.

### Trung bình, trung vị và phân vị {#id24}

Vì thời gian phản hồi mỗi lần yêu cầu đều khác nhau, ta không nên chỉ coi nó là một con số, mà phải xem nó là một *phân bố* (*distribution*) các giá trị đo được. Trong {{< xref fig="2-5" page="/ch2" anchor="fig_lognormal" >}}Hình 2-5{{< /xref >}}, mỗi thanh màu xám đại diện cho một yêu cầu dịch vụ, chiều cao thanh cho biết yêu cầu đó mất bao lâu. Hầu hết yêu cầu khá nhanh, nhưng thỉnh thoảng có những *giá trị ngoại lai* (*outlier*) mất lâu hơn nhiều. Sự biến thiên của độ trễ mạng còn gọi là *jitter*.

{{< fig num="2-5" id="fig_lognormal" src="/fig/ddia_0205.png" caption="Minh họa trung bình và phân vị bằng mẫu thời gian phản hồi của 100 yêu cầu dịch vụ." class="ddia-figure ddia-figure--panorama" width="2953" height="902" />}}

Dịch vụ thường báo cáo thời gian phản hồi *trung bình* (*mean*), tức *trung bình cộng* (*arithmetic mean*): cộng mọi thời gian phản hồi rồi chia cho số yêu cầu. Thời gian phản hồi trung bình giúp ước tính giới hạn trên của thông lượng [^18]. Tuy nhiên, nếu muốn biết thời gian phản hồi “điển hình”, trung bình không phải chỉ số tốt, vì nó không cho biết thực sự có bao nhiêu người dùng đã phải chờ như vậy.

Thông thường, dùng *phân vị* (*percentile*) sẽ phù hợp hơn. Sắp xếp thời gian phản hồi từ nhanh đến chậm, *trung vị* (*median*) là giá trị nằm chính giữa. Chẳng hạn, nếu trung vị của thời gian phản hồi là 200 mili giây, nghĩa là một nửa số yêu cầu mất dưới 200 mili giây, nửa còn lại mất lâu hơn. Vì vậy, nếu muốn biết người dùng thường phải chờ bao lâu, trung vị là chỉ số rất tốt. Trung vị còn gọi là *phân vị thứ 50*, đôi khi viết tắt là *p50*.

Để thấy các giá trị ngoại lai tệ đến mức nào, có thể xem các phân vị cao hơn. Thường dùng phân vị thứ *95*, *99* và *99,9*, viết tắt là *p95*, *p99* và *p999*. Chúng ứng với ngưỡng thời gian phản hồi mà lần lượt 95%, 99% hoặc 99,9% số yêu cầu nhanh hơn. Chẳng hạn, nếu thời gian phản hồi ở phân vị thứ 95 là 1,5 giây, nghĩa là trong 100 yêu cầu, 95 yêu cầu mất dưới 1,5 giây, 5 yêu cầu còn lại mất 1,5 giây hoặc lâu hơn. {{< xref fig="2-5" page="/ch2" anchor="fig_lognormal" >}}Hình 2-5{{< /xref >}} minh họa điều này.

Các phân vị cao của thời gian phản hồi còn gọi là *độ trễ đuôi* (*tail latencies*). Chúng rất quan trọng vì tác động trực tiếp đến trải nghiệm của người dùng với dịch vụ. Chẳng hạn, Amazon dùng phân vị thứ 99,9 để mô tả yêu cầu thời gian phản hồi của các dịch vụ nội bộ, dù nó chỉ ảnh hưởng đến 1 trong 1.000 yêu cầu. Lý do là những khách hàng có phản hồi chậm nhất thường là những người có nhiều dữ liệu nhất trong tài khoản — họ đã mua nhiều hàng, tức là những khách hàng giá trị nhất [^19]. Đảm bảo trang web cũng nhanh với họ là cách quan trọng để giữ họ hài lòng.

Mặt khác, Amazon cho rằng tối ưu cho phân vị thứ 99,99 (yêu cầu chậm nhất trong 10.000 yêu cầu) có chi phí quá cao mà lợi ích không đủ. Giảm thời gian phản hồi ở các phân vị rất cao là điều rất khó, vì nó dễ bị ảnh hưởng bởi những sự kiện ngẫu nhiên không kiểm soát được, và càng về sau lợi ích thu được càng nhỏ.

> [!TIP] Tác động của thời gian phản hồi đến người dùng
>
> Theo trực giác, dịch vụ nhanh dĩ nhiên được người dùng ưa chuộng hơn dịch vụ chậm [^20]. Tuy nhiên, để có dữ liệu đáng tin cậy định lượng tác động của độ trễ lên hành vi người dùng lại khó đến bất ngờ.
>
> Một số số liệu hay được trích dẫn không đáng tin. Năm 2006, Google báo cáo rằng khi thời gian phản hồi của kết quả tìm kiếm tăng từ 400 mili giây lên 900 mili giây thì lưu lượng và doanh thu giảm 20% [^21]. Nhưng một nghiên cứu khác của Google năm 2009 lại báo cáo rằng độ trễ tăng thêm 400 mili giây chỉ làm số lượt tìm kiếm mỗi ngày giảm 0,6% [^22]. Cùng năm đó, Bing phát hiện thời gian tải tăng 2 giây làm doanh thu quảng cáo giảm 4,3% [^23]. Các công ty này dường như chưa công bố số liệu mới hơn.
>
> Một nghiên cứu mới hơn của Akamai [^24] tuyên bố rằng thời gian phản hồi tăng 100 mili giây có thể làm tỷ lệ chuyển đổi của trang thương mại điện tử giảm tới 7%. Nhưng xem kỹ sẽ thấy cũng nghiên cứu đó cho thấy các trang tải *rất* nhanh cũng gắn với tỷ lệ chuyển đổi thấp hơn! Kết quả có vẻ mâu thuẫn này có thể giải thích như sau: những trang tải nhanh nhất thường là trang không có nội dung hữu ích, chẳng hạn trang lỗi 404. Nghiên cứu không cố tách tác động của nội dung trang khỏi tác động của thời gian tải, nên kết quả e rằng không có nhiều ý nghĩa.
>
> Một nghiên cứu của Yahoo [^25] so sánh tỷ lệ nhấp chuột giữa kết quả tìm kiếm tải nhanh và tải chậm, trong điều kiện kiểm soát chất lượng kết quả tìm kiếm. Nghiên cứu thấy rằng khi phản hồi nhanh và chậm chênh nhau từ 1,25 giây trở lên, tìm kiếm nhanh nhận được nhiều lượt nhấp hơn 20%–30%.

### Ứng dụng các chỉ số thời gian phản hồi {#sec_introduction_slo_sla}

Nếu dịch vụ backend phải thực hiện nhiều lời gọi cho một yêu cầu của người dùng cuối, các phân vị cao càng quan trọng. Dù các lời gọi này được gửi song song, yêu cầu của người dùng cuối vẫn phải chờ lời gọi chậm nhất hoàn tất. Như {{< xref fig="2-6" page="/ch2" anchor="fig_tail_amplification" >}}Hình 2-6{{< /xref >}} cho thấy, chỉ cần một lời gọi chậm là đủ làm chậm cả yêu cầu của người dùng cuối. Dù chỉ một tỷ lệ nhỏ các lời gọi backend chậm, một yêu cầu của người dùng cuối cần càng nhiều lời gọi backend thì xác suất xuất hiện lời gọi chậm trong đó càng cao, và cuối cùng một tỷ lệ cao hơn các yêu cầu người dùng bị chậm. Hiệu ứng này gọi là *khuếch đại độ trễ đuôi* (*tail latency amplification*) [^26].

{{< fig num="2-6" id="fig_tail_amplification" src="/fig/ddia_0206.png" caption="Khi một yêu cầu cần nhiều lời gọi backend, chỉ một lời gọi chậm là đủ làm chậm cả yêu cầu của người dùng cuối." class="ddia-figure ddia-figure--wide" width="2880" height="1304" />}}

Phân vị thường xuất hiện trong *SLO* (*service level objective* — mục tiêu mức dịch vụ) và *SLA* (*service level agreement* — thỏa thuận mức dịch vụ), dùng để quy định hiệu năng và tính sẵn sàng mà dịch vụ phải đạt [^27]. Chẳng hạn, một SLO có thể yêu cầu thời gian phản hồi trung vị của dịch vụ dưới 200 mili giây, thời gian phản hồi ở phân vị thứ 99 dưới 1 giây, và ít nhất 99,9% số yêu cầu hợp lệ nhận được phản hồi không lỗi. SLA là một hợp đồng quy định điều gì xảy ra nếu không đạt SLO (ví dụ khách hàng có thể được quyền hoàn tiền). Đó ít nhất là ý tưởng cơ bản. Trong thực tế, việc định nghĩa các chỉ số tính sẵn sàng tốt cho SLO và SLA không hề đơn giản [^28] [^29].


<a id="sidebar_percentiles"></a>

> [!TIP] Tính phân vị
>
> Nếu muốn đưa phân vị thời gian phản hồi vào bảng điều khiển giám sát của dịch vụ, bạn cần tính các chỉ số này liên tục và hiệu quả. Chẳng hạn, có thể duy trì một cửa sổ trượt ghi lại thời gian phản hồi của mọi yêu cầu trong 10 phút gần nhất. Mỗi phút, tính trung vị và các phân vị khác của các giá trị trong cửa sổ, rồi vẽ các chỉ số đó lên biểu đồ.
>
> Cách cài đặt đơn giản nhất là lưu danh sách thời gian phản hồi của mọi yêu cầu trong cửa sổ thời gian, và sắp xếp nó mỗi phút một lần. Nếu cách này kém hiệu quả, có những thuật toán tính xấp xỉ phân vị khá chính xác với chi phí CPU và bộ nhớ rất thấp. Các thư viện mã nguồn mở để ước lượng phân vị gồm HdrHistogram, t-digest [^30] [^31], OpenHistogram [^32] và DDSketch [^33].
>
> Lưu ý rằng lấy trung bình các phân vị — ví dụ để giảm độ phân giải thời gian, hoặc gộp dữ liệu từ nhiều máy — là vô nghĩa về mặt toán học. Cách đúng để tổng hợp dữ liệu thời gian phản hồi là cộng các histogram lại [^34].

## Độ tin cậy và khả năng chịu lỗi {#sec_introduction_reliability}

Ai cũng có cảm nhận trực giác về việc một thứ có đáng tin cậy hay không. Những kỳ vọng điển hình đối với phần mềm đáng tin cậy gồm:

* Ứng dụng thực hiện đúng chức năng mà người dùng mong đợi.
* Cho phép người dùng mắc lỗi, hoặc dùng phần mềm theo những cách không lường trước.
* Hiệu năng đủ tốt cho trường hợp sử dụng cần thiết, dưới tải và lượng dữ liệu dự kiến.
* Hệ thống ngăn chặn truy cập trái phép và lạm dụng.

Nếu gộp tất cả những điều này lại thành “hoạt động đúng”, thì *độ tin cậy* (*reliability*) có thể hiểu đại khái là “tiếp tục hoạt động đúng ngay cả khi có sự cố”. Để mô tả chính xác hơn thế nào là “có sự cố”, ta cần phân biệt *lỗi* (*fault*) và *hỏng hóc* (*failure*) [^35] [^36] [^37] như sau:

Lỗi
: Một *bộ phận* của hệ thống ngừng hoạt động bình thường. Ví dụ, một ổ đĩa bị lỗi, một máy bị sập, hoặc một dịch vụ bên ngoài mà hệ thống phụ thuộc bị gián đoạn.

Hỏng hóc
: *Toàn bộ hệ thống* ngừng cung cấp dịch vụ cần thiết cho người dùng. Nói cách khác, hệ thống không đạt được mục tiêu mức dịch vụ (SLO).

Lỗi và hỏng hóc dễ bị nhầm lẫn vì thực ra chúng là cùng một sự việc, chỉ khác tầng quan sát. Chẳng hạn, nếu một ổ đĩa ngừng hoạt động, ta nói ổ đĩa đó hỏng: nếu cả hệ thống chỉ có đúng ổ đĩa ấy thì hệ thống cũng ngừng cung cấp dịch vụ cần thiết. Tuy nhiên, nếu ta nói về một hệ thống gồm nhiều ổ đĩa, thì việc một ổ đĩa hỏng chỉ là một lỗi xét từ góc độ toàn hệ thống: chỉ cần dữ liệu còn bản sao trên ổ đĩa khác, cả hệ thống có thể chịu được lỗi này.

### Khả năng chịu lỗi {#id27}

Nếu hệ thống vẫn tiếp tục cung cấp dịch vụ cần thiết cho người dùng khi một số lỗi xảy ra, ta nói nó *chịu lỗi* (*fault-tolerant*). Nếu hệ thống không chịu được lỗi ở một bộ phận nào đó, bộ phận đó gọi là *SPOF* (*single point of failure*, điểm hỏng duy nhất), vì một khi nó gặp lỗi, lỗi sẽ leo thang thành hỏng hóc của cả hệ thống.

Lấy tình huống mạng xã hội làm ví dụ: trong quá trình fan-out có thể xảy ra lỗi khi một máy chịu trách nhiệm cập nhật timeline vật chất hóa bị sập hoặc không khả dụng. Để quá trình này chịu lỗi, phải bảo đảm một máy khác có thể tiếp quản nhiệm vụ, không bỏ sót bài đăng nào lẽ ra phải chuyển phát, cũng không chuyển phát trùng lặp. (Ý tưởng này gọi là *ngữ nghĩa exactly-once* (*exactly-once semantics*), sẽ được bàn chi tiết ở [“Nguyên lý end-to-end của cơ sở dữ liệu”](/ch13#sec_future_end_to_end).)

Khả năng chịu lỗi luôn nhắm đến những loại lỗi và số lượng lỗi cụ thể. Chẳng hạn, một hệ thống có thể chịu được tối đa hai ổ đĩa hỏng cùng lúc, hoặc một trong ba node bị sập. Yêu cầu hệ thống chịu được số lượng lỗi tùy ý là vô nghĩa: nếu mọi node đều sập, không cách nào cứu vãn. Nếu cả Trái Đất (cùng mọi máy chủ trên đó) bị hố đen nuốt chửng, để chịu được lỗi đó thì phải đặt website trên không gian vũ trụ — chúc bạn may mắn, xem ngân sách này có được duyệt không.

Phản trực giác là, trong các hệ thống chịu lỗi như vậy, đôi khi cố ý kích hoạt lỗi để *tăng* tỷ lệ lỗi lại hợp lý, ví dụ giết ngẫu nhiên một tiến trình mà không báo trước. Kỹ thuật này gọi là *chèn lỗi* (*fault injection*). Nhiều lỗi nghiêm trọng thực ra bắt nguồn từ xử lý lỗi kém [^38]. Cố ý gây lỗi giúp cơ chế chịu lỗi liên tục được diễn tập và kiểm nghiệm, từ đó tăng niềm tin rằng khi lỗi xảy ra tự nhiên, hệ thống thực sự xử lý đúng. *chaos engineering* là một ngành dùng các thí nghiệm như chèn lỗi để tăng niềm tin của con người vào cơ chế chịu lỗi [^39].

Dù thường ta ưu tiên chịu lỗi hơn là ngăn ngừa lỗi, đôi khi phòng bệnh quả thực hơn chữa bệnh (ví dụ khi không có thuốc chữa). Vấn đề bảo mật là như vậy: nếu kẻ tấn công đã xâm nhập hệ thống và lấy được dữ liệu nhạy cảm, không thể hoàn tác việc đó. Tuy nhiên, cuốn sách này chủ yếu bàn về các loại lỗi có thể khắc phục, như các mục sau sẽ làm rõ.

### Lỗi phần cứng và phần mềm {#sec_introduction_hardware_faults}

Khi nói đến nguyên nhân hỏng hóc hệ thống, người ta dễ nghĩ đầu tiên đến lỗi phần cứng:

* Mỗi năm có khoảng 2%–5% ổ đĩa cơ bị hỏng [^40] [^41], do đó, trong một cụm lưu trữ có 10.000 ổ đĩa, trung bình mỗi ngày sẽ có một ổ hỏng. Dữ liệu gần đây cho thấy ổ đĩa ngày càng đáng tin cậy, nhưng tỷ lệ hỏng vẫn không thể bỏ qua [^42].
* Mỗi năm có khoảng 0,5%–1% ổ cứng thể rắn (*SSD*) bị hỏng [^43]. Một số ít lỗi bit được tự động sửa [^44], nhưng mỗi ổ đĩa vẫn gặp khoảng một lỗi không thể sửa mỗi năm, kể cả ổ khá mới (tức ít bị mài mòn). Tỷ lệ lỗi này cao hơn ổ đĩa cơ [^45] [^46].
* Các thành phần phần cứng khác như nguồn điện, bộ điều khiển RAID và mô-đun bộ nhớ cũng có thể hỏng, chỉ là không thường xuyên bằng ổ đĩa [^47] [^48].
* Cứ khoảng 1.000 máy thì có một máy có một nhân CPU thỉnh thoảng tính ra kết quả sai, nhiều khả năng do lỗi chế tạo [^49] [^50] [^51]. Tính toán sai đôi khi gây sập, nhưng đôi khi chỉ khiến chương trình trả về kết quả sai.
* Dữ liệu trong RAM cũng có thể bị hỏng, do các sự kiện ngẫu nhiên như tia vũ trụ, hoặc do khiếm khuyết vật lý vĩnh viễn. Ngay cả khi dùng bộ nhớ có mã sửa lỗi (*ECC*), mỗi năm vẫn có hơn 1% số máy gặp lỗi không thể sửa, thường khiến máy sập và cần thay mô-đun bộ nhớ bị ảnh hưởng [^52]. Ngoài ra, một số kiểu truy cập bộ nhớ bệnh lý rất có thể gây lật bit [^53].
* Cả một trung tâm dữ liệu có thể trở nên không khả dụng (ví dụ do mất điện hoặc cấu hình mạng sai), thậm chí bị phá hủy vĩnh viễn (ví dụ do hỏa hoạn, lũ lụt hay động đất [^54]). Bão Mặt Trời — khi Mặt Trời phun ra lượng lớn hạt mang điện — có thể cảm ứng dòng điện rất mạnh trong các đường dây dài, làm hỏng lưới điện và cáp mạng dưới biển [^55]. Những sự cố quy mô lớn này tuy hiếm, nhưng nếu dịch vụ không chịu được việc mất cả trung tâm dữ liệu thì hậu quả có thể là thảm họa [^56].

Những sự kiện này khá hiếm, nên với hệ thống nhỏ, miễn là phần cứng hỏng dễ thay thì thường không cần lo lắng quá mức. Tuy nhiên, trong hệ thống quy mô lớn, lỗi phần cứng xảy ra đủ thường xuyên để trở thành một phần của hoạt động bình thường của hệ thống.

#### Chịu lỗi phần cứng bằng dư thừa {#tolerating-hardware-faults-through-redundancy}

Trước phần cứng không đáng tin cậy, phản ứng đầu tiên của ta thường là bổ sung *dư thừa* (*redundancy*) cho từng thành phần phần cứng để giảm tỷ lệ hỏng hóc của cả hệ thống. Đĩa có thể ghép thành RAID (phân tán dữ liệu trên nhiều đĩa của cùng một máy để một đĩa hỏng không gây mất dữ liệu); máy chủ có thể trang bị nguồn kép và CPU cắm nóng; trung tâm dữ liệu có thể dùng pin và máy phát điện diesel làm nguồn dự phòng. Những biện pháp dư thừa này thường giúp một máy chạy liên tục nhiều năm không gián đoạn.

Dư thừa hiệu quả nhất khi các lỗi của từng thành phần độc lập với nhau. Độc lập nghĩa là việc xảy ra một lỗi không làm thay đổi xác suất xảy ra lỗi khác. Tuy nhiên, thực tế cho thấy các hỏng hóc của thành phần thường có tương quan đáng kể [^41] [^57] [^58]. Tình huống cả giá máy chủ, thậm chí cả trung tâm dữ liệu không khả dụng vẫn phổ biến hơn ta mong muốn.

Dư thừa phần cứng có thể tăng thời gian hoạt động liên tục của từng máy. Tuy nhiên, như đã nói ở [“Hệ thống phân tán và hệ thống một node”](/ch1#sec_introduction_distributed), dùng hệ thống phân tán còn có những lợi ích khác, chẳng hạn chịu được việc cả trung tâm dữ liệu gián đoạn. Vì vậy, hệ thống đám mây thường không quá chú trọng độ tin cậy của từng máy, mà cố gắng chịu được lỗi node ở tầng phần mềm để dịch vụ đạt tính sẵn sàng cao. Nhà cung cấp đám mây dùng *availability zone* để cho biết những tài nguyên nào nằm cùng một chỗ về mặt vật lý. So với các tài nguyên phân tán về địa lý, tài nguyên ở cùng một chỗ có khả năng hỏng cùng lúc cao hơn.

Các kỹ thuật chịu lỗi được bàn trong sách này nhằm chịu được việc mất cả máy, cả giá máy hay cả availability zone. Chúng thường cho phép máy trong một trung tâm dữ liệu đảm nhận công việc khi máy ở trung tâm dữ liệu khác gặp lỗi hoặc không thể truy cập. Ta sẽ bàn những kỹ thuật chịu lỗi này ở [Chương 6](/ch6#ch_replication), [Chương 10](/ch10#ch_consistency) và nhiều chỗ khác trong sách.

Hệ thống chịu được việc mất cả máy cũng có lợi thế về vận hành. Nếu cần khởi động lại máy (ví dụ để cài bản vá bảo mật hệ điều hành), hệ thống một máy chủ phải lên lịch ngừng hoạt động, còn hệ thống nhiều node chịu lỗi có thể khởi động lại từng node để cài bản vá mà không ảnh hưởng đến việc phục vụ người dùng. Cách này gọi là *rolling upgrade*, sẽ được bàn thêm ở [Chương 5](/ch5#ch_encoding).

#### Lỗi phần mềm {#software-faults}

Dù các hỏng hóc phần cứng có thể có tương quan yếu, nhìn chung chúng vẫn độc lập với nhau. Chẳng hạn, khi một ổ đĩa hỏng, các ổ đĩa khác trên cùng máy nhiều khả năng vẫn hoạt động bình thường thêm một thời gian. Ngược lại, lỗi phần mềm thường có tương quan rất cao, vì nhiều node thường chạy cùng một bộ phần mềm nên cũng mang cùng khiếm khuyết [^59] [^60]. Loại lỗi này khó lường trước hơn, và thường gây ra hỏng hóc hệ thống nhiều hơn hẳn so với lỗi phần cứng không liên quan [^47]. Ví dụ:

* Một lỗi phần mềm khiến mọi node hỏng cùng lúc trong một tình huống nhất định. Chẳng hạn, giây nhuận ngày 30 tháng 6 năm 2012 đã kích hoạt một lỗi trong nhân Linux, khiến nhiều ứng dụng Java treo đồng loạt, kéo theo nhiều dịch vụ internet gián đoạn [^61]. Một ví dụ khác: do lỗi firmware, một số mẫu SSD đột ngột hỏng đồng loạt sau đúng 32.768 giờ hoạt động (chưa đến 4 năm), và dữ liệu trên đĩa không thể khôi phục [^62].
* Một tiến trình mất kiểm soát ngốn hết các tài nguyên dùng chung và hữu hạn như thời gian CPU, bộ nhớ, dung lượng đĩa, băng thông mạng hay luồng [^63]. Chẳng hạn, tiến trình tiêu thụ quá nhiều bộ nhớ khi xử lý một yêu cầu lớn có thể bị hệ điều hành giết. Lỗi trong thư viện client cũng có thể sinh ra lượng yêu cầu cao hơn dự kiến nhiều [^64].
* Một dịch vụ mà hệ thống phụ thuộc trở nên chậm, mất phản hồi, hoặc bắt đầu trả về phản hồi bị hỏng nội dung.
* Tương tác giữa các hệ thống tạo ra hành vi nảy sinh (*emergent behavior*) mà khi kiểm thử riêng từng hệ thống thì không xuất hiện [^65].
* Hỏng hóc dây chuyền: vấn đề ở một thành phần khiến thành phần khác quá tải và chậm đi, thành phần này lại kéo sập thành phần kế tiếp [^66] [^67].

Những lỗi gây ra sự cố phần mềm như vậy thường nằm im rất lâu, cho đến khi một tập điều kiện bất thường kích hoạt chúng. Khi đó người ta mới nhận ra phần mềm đã giả định điều gì đó về môi trường chạy. Giả định này thường đúng, nhưng rồi vì lý do nào đó không còn đúng nữa [^68] [^69].

Với lỗi hệ thống trong phần mềm không có thuốc đặc trị, nhưng nhiều việc nhỏ có thể giúp ích: suy nghĩ kỹ về các giả định và tương tác trong hệ thống, kiểm thử toàn diện, cô lập tiến trình, cho phép tiến trình sập và khởi động lại, tránh các vòng phản hồi như retry storm (xem [“Khi hệ thống quá tải không thể phục hồi”](/ch2#sidebar_metastable)), và đo lường, giám sát, phân tích hành vi của hệ thống trong môi trường production.

### Con người và độ tin cậy {#id31}

Hệ thống phần mềm do con người thiết kế và xây dựng, và những người vận hành giữ cho hệ thống chạy cũng là con người. Khác với máy móc, con người không chỉ làm theo quy trình. Điểm mạnh của họ chính là khả năng sáng tạo và ứng biến để hoàn thành công việc. Nhưng đặc điểm này cũng đem lại tính khó đoán: dù xuất phát điểm tốt đến đâu, con người vẫn mắc lỗi, đôi khi dẫn đến hỏng hóc hệ thống. Ví dụ, một nghiên cứu về các dịch vụ internet lớn thấy rằng việc người vận hành thay đổi cấu hình là nguyên nhân hàng đầu gây gián đoạn dịch vụ, còn lỗi phần cứng (máy chủ hoặc mạng) chỉ đóng vai trò trong 10%–25% số lần gián đoạn [^70].

Người ta rất dễ quy những vấn đề này cho “lỗi của con người”, và ảo tưởng rằng có thể giải quyết bằng quy trình chặt hơn, quy tắc ngặt hơn để ràng buộc hành vi con người. Tuy nhiên, đổ lỗi cho cá nhân thường phản tác dụng. Cái gọi là “lỗi của con người” thực ra không phải nguyên nhân gốc của sự cố, mà là triệu chứng cho thấy *hệ thống xã hội–kỹ thuật* (*sociotechnical system*) do con người và công nghệ cùng tạo nên có vấn đề. Những người ở trong đó chỉ đang cố hết sức làm tốt công việc [^71]. Hệ thống phức tạp cũng thường thể hiện hành vi nảy sinh, và những tương tác bất ngờ giữa các thành phần cũng có thể dẫn đến hỏng hóc [^72].

Nhiều biện pháp kỹ thuật có thể giảm tác động của sai sót con người, gồm: kiểm thử toàn diện (cả kiểm thử viết tay lẫn *kiểm thử dựa trên thuộc tính*, *property-based testing*, với lượng lớn đầu vào ngẫu nhiên) [^38]; có cơ chế rollback để nhanh chóng hoàn tác thay đổi cấu hình; phát hành mã mới dần dần; giám sát chi tiết, rõ ràng cùng các công cụ quan sát được (*observability*) để chẩn đoán vấn đề trên production (xem [“Các vấn đề của hệ thống phân tán”](/ch1#sec_introduction_dist_sys_problems)); thiết kế giao diện cẩn thận để “làm điều đúng” dễ hơn và “làm điều sai” khó hơn.

Tuy nhiên, những biện pháp này đều tốn thời gian và tiền bạc. Dưới áp lực thực tế của hoạt động hằng ngày, tổ chức thường ưu tiên công việc tạo ra doanh thu hơn là các biện pháp nâng cao khả năng chống sai sót của chính mình. Nếu phải chọn giữa phát triển thêm tính năng và kiểm thử thêm, nhiều tổ chức chọn tính năng cũng dễ hiểu. Đã chọn như vậy, thì khi những sai sót lẽ ra tránh được chắc chắn xảy ra, việc trách người mắc lỗi là vô lý — vấn đề thực sự nằm ở cách tổ chức đặt ưu tiên.

Ngày càng nhiều tổ chức xây dựng văn hóa *postmortem không đổ lỗi* (*blameless postmortem*): sau sự cố, mọi người tham gia được khuyến khích kể rõ đầu đuôi sự việc mà không lo bị trừng phạt, nhờ vậy, những người khác trong tổ chức mới học được từ đó và tránh để vấn đề tương tự tái diễn [^73]. Quá trình postmortem có thể phát hiện rằng cần điều chỉnh ưu tiên kinh doanh, cần đầu tư vào những mảng bị bỏ quên lâu, cần thay đổi cơ chế khuyến khích của những người liên quan, hoặc còn các vấn đề mang tính hệ thống khác cần báo cho ban quản lý.

Nói chung, khi điều tra sự cố nên cảnh giác với những câu trả lời quá đơn giản. “Bob lẽ ra phải cẩn thận hơn khi triển khai thay đổi này” không giúp giải quyết vấn đề, “chúng ta phải viết lại backend bằng Haskell” cũng vậy. Ban quản lý nên nắm lấy cơ hội để hiểu từ những người ở tuyến đầu — những người dùng hệ thống xã hội–kỹ thuật này mỗi ngày — xem nó thực sự vận hành ra sao, rồi dựa trên phản hồi đó hành động để cải thiện [^71].


<a id="sidebar_reliability_importance"></a>

> [!TIP] Độ tin cậy quan trọng đến mức nào?
>
> Độ tin cậy không chỉ quan trọng với nhà máy điện hạt nhân và hệ thống điều khiển không lưu. Người ta cũng mong những ứng dụng bình thường hơn hoạt động đáng tin cậy. Lỗi trong ứng dụng doanh nghiệp làm giảm năng suất (và gây rủi ro pháp lý nếu số liệu báo cáo sai), còn việc website thương mại điện tử gián đoạn có thể gây tổn thất doanh thu lớn và làm tổn hại danh tiếng doanh nghiệp.
>
> Với nhiều ứng dụng, gián đoạn tạm thời vài phút thậm chí vài giờ vẫn chịu được [^74], nhưng mất hoặc hỏng dữ liệu vĩnh viễn thì là thảm họa. Hãy hình dung một phụ huynh lưu toàn bộ ảnh và video của con mình trong ứng dụng ảnh của bạn [^75]. Nếu cơ sở dữ liệu đột nhiên bị hỏng, họ sẽ cảm thấy thế nào? Họ có biết cách khôi phục từ bản sao lưu không?
>
> Vụ bê bối Horizon của Bưu điện Anh là một ví dụ khác về phần mềm không đáng tin cậy gây hại cho con người. Từ năm 1999 đến 2019, hàng trăm người điều hành các điểm giao dịch của Bưu điện Anh bị kết tội trộm cắp hoặc gian lận chỉ vì phần mềm kế toán báo tài khoản của họ bị thiếu hụt. Cuối cùng người ta phát hiện nhiều khoản thiếu hụt thực ra do lỗi phần mềm, và nhiều bản án sau đó đã bị hủy [^76]. Vụ bất công tư pháp có lẽ lớn nhất trong lịch sử nước Anh này xảy ra vì luật pháp Anh giả định máy tính hoạt động đúng, do đó cũng giả định bằng chứng do máy tính tạo ra là đáng tin, trừ khi ai đó đưa ra được bằng chứng ngược lại [^77]. Kỹ sư phần mềm có thể thấy ý nghĩ “phần mềm không có lỗi nào” thật nực cười, nhưng điều đó chẳng an ủi được chút nào những người bị kết án oan, bị bỏ tù, bị phá sản, thậm chí tự tử vì hệ thống máy tính không đáng tin cậy.
>
> Trong một số trường hợp, ta có thể hy sinh độ tin cậy để giảm chi phí phát triển (ví dụ làm nguyên mẫu sản phẩm cho một thị trường chưa được kiểm chứng). Nhưng ta phải ý thức rõ khi nào mình đang đi đường tắt, và luôn nhớ hậu quả có thể gây ra.

## Khả năng mở rộng {#sec_introduction_scalability}

Hệ thống hôm nay chạy đáng tin cậy không có nghĩa là tương lai cũng chắc chắn như vậy. Một nguyên nhân phổ biến khiến hệ thống suy giảm là tải tăng: có thể số người dùng đồng thời tăng từ 10.000 lên 100.000, hoặc từ 1 triệu lên 10 triệu. Cũng có thể hệ thống nay xử lý lượng dữ liệu lớn hơn nhiều so với trước.

*Khả năng mở rộng* (*scalability*) là thuật ngữ mô tả khả năng của hệ thống ứng phó với tải tăng. Khi bàn về khả năng mở rộng, đôi khi người ta nói: “Bạn đâu phải Google hay Amazon. Đừng lo chuyện quy mô nữa, cứ dùng cơ sở dữ liệu quan hệ.” Câu này có hợp với bạn hay không tùy thuộc vào loại ứng dụng bạn đang xây dựng.

Nếu bạn đang xây dựng một sản phẩm mới hiện chưa có nhiều người dùng, chẳng hạn một mảng kinh doanh mới của công ty khởi nghiệp, mục tiêu kỹ thuật bao trùm thường là giữ hệ thống đơn giản và linh hoạt nhất có thể, nhờ vậy, khi dần hiểu nhu cầu của khách hàng, bạn dễ dàng sửa đổi, điều chỉnh tính năng sản phẩm [^78]. Trong môi trường này, lo lắng về quy mô giả định mà tương lai có thể mới cần chỉ phản tác dụng: nhẹ thì công sức đầu tư cho khả năng mở rộng là uổng phí và tối ưu hóa sớm, nặng thì nó nhốt bạn vào một thiết kế thiếu linh hoạt, khiến ứng dụng khó tiến hóa hơn.

Đó là vì khả năng mở rộng không phải một nhãn một chiều. Nói đơn giản “X có khả năng mở rộng” hay “Y không mở rộng được” là vô nghĩa. Khi bàn về khả năng mở rộng, điều thực sự cần cân nhắc là những câu hỏi sau:

* “Nếu hệ thống tăng trưởng theo một cách nào đó, ta có những lựa chọn ứng phó nào?”
* “Làm thế nào để thêm tài nguyên tính toán đáp ứng tải phát sinh?”
* “Với kỳ vọng tăng trưởng hiện tại, khi nào ta chạm giới hạn của kiến trúc hiện có?”

Nếu ứng dụng rất được ưa chuộng và do đó phải xử lý tải ngày càng tăng, bạn sẽ dần biết nút thắt hiệu năng nằm ở đâu, và hiểu hệ thống cần mở rộng theo những chiều nào. Đến lúc đó mới bắt đầu nghiêm túc cân nhắc các kỹ thuật mở rộng cũng chưa muộn.

### Mô tả tải {#id33}

Trước hết, ta cần mô tả ngắn gọn tải hiện tại của hệ thống, chỉ khi đó mới bàn tiếp được vấn đề tăng trưởng (ví dụ điều gì xảy ra khi tải tăng gấp đôi). Mô tả này thường là một chỉ số thông lượng nào đó, chẳng hạn số yêu cầu dịch vụ nhận mỗi giây, lượng GB dữ liệu mới mỗi ngày, hay số lần thanh toán giỏ hàng hoàn tất mỗi giờ. Đôi khi ta quan tâm đến giá trị đỉnh của một biến, ví dụ số người dùng trực tuyến đồng thời trong [“Nghiên cứu tình huống: dòng thời gian trang chủ của mạng xã hội”](/ch2#sec_introduction_twitter).

Tải thường còn có các đặc trưng thống kê khác, cũng ảnh hưởng đến mẫu truy cập và do đó ảnh hưởng đến yêu cầu mở rộng của hệ thống. Chẳng hạn, bạn có thể cần biết tỷ lệ đọc/ghi của cơ sở dữ liệu, tỷ lệ cache trúng (cache hit), hoặc số mục dữ liệu mỗi người dùng có (ví dụ số người theo dõi trong tình huống mạng xã hội). Đôi khi trường hợp trung bình là quan trọng nhất, đôi khi nút thắt lại bị chi phối bởi một số ít trường hợp cực đoan. Tất cả tùy thuộc chi tiết của ứng dụng cụ thể.

Sau khi mô tả được tải hệ thống, ta có thể nghiên cứu điều gì xảy ra khi tải tăng. Có thể xét từ hai góc độ:

* Tăng tải theo một cách nào đó trong khi tài nguyên hệ thống (CPU, bộ nhớ, băng thông mạng, v.v.) giữ nguyên, hiệu năng hệ thống bị ảnh hưởng ra sao?
* Tăng tải theo một cách nào đó và bạn muốn hiệu năng giữ nguyên, cần tăng thêm bao nhiêu tài nguyên?

Thông thường, mục tiêu của ta là giữ chi phí vận hành hệ thống ở mức thấp nhất có thể trong khi vẫn đáp ứng yêu cầu hiệu năng của SLA (xem [“Ứng dụng các chỉ số thời gian phản hồi”](/ch2#sec_introduction_slo_sla)). Cần càng nhiều tài nguyên tính toán thì chi phí càng cao. Một số phần cứng có thể có hiệu quả chi phí tốt hơn số khác, và khi phần cứng mới xuất hiện, những yếu tố này cũng thay đổi theo thời gian.

Nếu tăng gấp đôi tài nguyên mà xử lý được gấp đôi tải với hiệu năng không đổi, ta nói hệ thống có *khả năng mở rộng tuyến tính* (*linear scalability*), và thường đó là điều tốt. Đôi khi, nhờ lợi thế kinh tế theo quy mô hoặc tải đỉnh phân bố đều hơn, chưa đến gấp đôi tài nguyên cũng xử lý được gấp đôi tải [^79] [^80]. Phổ biến hơn là chi phí tăng nhanh hơn tuyến tính, và nguyên nhân gây ra sự kém hiệu quả này có thể rất nhiều. Chẳng hạn, khi hệ thống có lượng dữ liệu lớn, công việc cần để xử lý một lần ghi có thể nhiều hơn khi dữ liệu ít, dù bản thân yêu cầu ghi có cùng kích thước.

### Kiến trúc chia sẻ bộ nhớ, chia sẻ đĩa và shared-nothing {#sec_introduction_shared_nothing}

Cách đơn giản nhất để tăng tài nguyên phần cứng cho dịch vụ là chuyển dịch vụ sang một máy mạnh hơn. Tốc độ của từng nhân CPU không còn tăng đáng kể, nhưng bạn vẫn có thể mua (hoặc thuê trên đám mây) máy có nhiều nhân CPU hơn, RAM lớn hơn và nhiều dung lượng đĩa hơn. Cách này gọi là *mở rộng theo chiều dọc* (*vertical scaling*), còn gọi là *scaling up*.

Chạy nhiều tiến trình hoặc luồng trên một máy cho phép xử lý song song. Mọi luồng trong cùng một tiến trình đều truy cập cùng một vùng RAM, nên cách này còn gọi là *kiến trúc chia sẻ bộ nhớ* (*shared-memory architecture*). Vấn đề của phương án chia sẻ bộ nhớ là chi phí tăng nhanh hơn tuyến tính: máy cao cấp có tài nguyên phần cứng gấp đôi thường đắt hơn gấp đôi nhiều, và do các nút thắt khác nhau, một máy có quy mô gấp đôi thường không xử lý nổi gấp đôi tải.

Một phương án khác là *kiến trúc chia sẻ đĩa* (*shared-disk architecture*): nhiều máy có CPU và RAM riêng nhưng lưu dữ liệu trên một mảng đĩa dùng chung, máy và đĩa được nối bằng mạng tốc độ cao, ví dụ *NAS* (*network-attached storage*, lưu trữ gắn mạng) hoặc *SAN* (*storage area network*, mạng vùng lưu trữ). Kiến trúc này trước đây thường dùng cho khối lượng công việc kho dữ liệu triển khai tại chỗ, nhưng tranh chấp tài nguyên và chi phí khóa đã hạn chế khả năng mở rộng của phương án chia sẻ đĩa [^81].

Ngược lại, *kiến trúc shared-nothing* (*shared-nothing architecture*) [^82] (còn gọi là *mở rộng theo chiều ngang* (*horizontal scaling*) hay *scaling out*) đã trở nên rất phổ biến. Phương án này dùng một hệ thống phân tán gồm nhiều node, mỗi node có CPU, RAM và đĩa riêng. Mọi sự phối hợp giữa các node đều thực hiện ở tầng phần mềm qua mạng thông thường.

Ưu điểm của kiến trúc shared-nothing là: nó có triển vọng mở rộng tuyến tính; có thể dùng bất kỳ phần cứng nào có hiệu quả chi phí tốt nhất, nhất là trên đám mây; dễ điều chỉnh tài nguyên phần cứng khi tải tăng giảm; và có thể phân bố hệ thống trên nhiều trung tâm dữ liệu và vùng địa lý để chịu lỗi tốt hơn. Nhược điểm là phải phân mảnh (sharding) tường minh (xem [Chương 7](/ch7#ch_sharding)) và phải đối mặt với toàn bộ độ phức tạp của hệ thống phân tán (xem [Chương 9](/ch9#ch_distributed)).

Một số hệ cơ sở dữ liệu cloud-native tách lưu trữ và thực thi giao dịch thành các dịch vụ khác nhau (xem [“Tách rời lưu trữ và tính toán”](/ch1#sec_introduction_storage_compute)), để nhiều node tính toán dùng chung một dịch vụ lưu trữ. Mô hình này có phần giống kiến trúc chia sẻ đĩa nhưng tránh được các vấn đề về khả năng mở rộng của hệ thống cũ: dịch vụ lưu trữ không cung cấp trừu tượng hệ thống tệp (NAS) hay thiết bị khối (SAN), mà cung cấp một bộ API chuyên dụng được thiết kế cho nhu cầu cụ thể của cơ sở dữ liệu [^83].

### Các nguyên tắc về khả năng mở rộng {#id35}

Kiến trúc của hệ thống quy mô lớn thường phụ thuộc rất nhiều vào ứng dụng cụ thể. Không có một kiến trúc mở rộng chung, dùng được cho mọi trường hợp (thường gọi là *magic scaling sauce*, “nước sốt thần kỳ”). Chẳng hạn, một hệ thống xử lý 100.000 yêu cầu mỗi giây, mỗi yêu cầu 1 kB, trông sẽ rất khác một hệ thống chỉ xử lý 3 yêu cầu mỗi phút, mỗi yêu cầu 2 GB — dù cả hai có cùng thông lượng dữ liệu 100 MB/s.

Hơn nữa, kiến trúc phù hợp với một mức tải thường không đáp ứng nổi tải gấp mười lần. Nếu bạn đang phát triển một dịch vụ tăng trưởng nhanh, rất có thể mỗi khi tải tăng một bậc độ lớn, bạn lại phải xem xét lại kiến trúc. Bản thân yêu cầu của ứng dụng cũng nhiều khả năng liên tục thay đổi, nên lập kế hoạch trước cho nhu cầu mở rộng vượt quá một bậc độ lớn thường không đáng.

Có một nguyên tắc chung tốt cho khả năng mở rộng: chia hệ thống thành các thành phần nhỏ hơn để chúng có thể vận hành gần như độc lập với nhau. Đây là nguyên tắc chung đằng sau microservices (xem [“Microservices và serverless”](/ch1#sec_introduction_microservices)), phân mảnh ([Chương 7](/ch7#ch_sharding)), xử lý luồng ([Chương 12](/ch12#ch_stream)) và kiến trúc shared-nothing. Tuy nhiên, thách thức thực sự là xác định cái gì nên đặt cùng nhau, cái gì nên tách ra. Các sách khác trình bày nguyên tắc thiết kế microservices [^84], còn sách này sẽ bàn về phân mảnh trong hệ thống shared-nothing ở [Chương 7](/ch7#ch_sharding).

Một nguyên tắc tốt khác là đừng làm hệ thống phức tạp hơn mức cần thiết. Nếu một cơ sở dữ liệu trên một máy đủ làm việc, nó nhiều khả năng đáng chọn hơn một cấu hình phân tán phức tạp. Hệ thống tự động mở rộng (tự thêm hoặc bớt tài nguyên theo nhu cầu) quả thực rất hay, nhưng nếu tải khá dễ dự đoán, hệ thống mở rộng thủ công có thể ít gây bất ngờ hơn khi vận hành (xem [“Vận hành: cân bằng lại tự động/thủ công”](/ch7#sec_sharding_operations)). Hệ thống gồm 5 dịch vụ đơn giản hơn hệ thống gồm 50 dịch vụ. Một kiến trúc tốt thường là sự pha trộn thực dụng của nhiều phương án.

## Khả năng bảo trì {#sec_introduction_maintainability}

Phần mềm không bị mài mòn, cũng không bị mỏi vật liệu như thiết bị cơ khí, nên nó không hỏng theo cùng cách. Tuy nhiên, yêu cầu của ứng dụng thường xuyên thay đổi, môi trường chạy phần mềm cũng thay đổi (ví dụ các thư viện phụ thuộc và nền tảng bên dưới), và trong phần mềm luôn có lỗi cần sửa.

Ai cũng biết phần lớn chi phí của phần mềm không nằm ở giai đoạn phát triển ban đầu, mà ở giai đoạn bảo trì liên tục: sửa lỗi, giữ cho hệ thống chạy ổn định, điều tra hỏng hóc, thích ứng với nền tảng mới, sửa đổi cho các trường hợp sử dụng mới, trả nợ kỹ thuật và thêm tính năng mới [^85] [^86].

Tuy nhiên, bản thân việc bảo trì cũng khó. Một hệ thống chạy thành công nhiều năm rất có thể vẫn dùng công nghệ lỗi thời mà nay chẳng còn mấy kỹ sư hiểu, ví dụ máy tính lớn (mainframe) và mã COBOL; khi nhân sự rời tổ chức, tri thức tổ chức về lý do và cách thiết kế hệ thống có thể đã mất; người bảo trì có khi phải sửa sai sót do người đi trước để lại. Hơn nữa, hệ thống máy tính thường gắn chặt với tổ chức mà nó phục vụ, nghĩa là việc bảo trì một hệ thống *legacy* như vậy vừa là vấn đề của con người vừa là vấn đề kỹ thuật [^87].

Mọi hệ thống ta xây dựng hôm nay, nếu đủ giá trị để tồn tại lâu dài, rồi sẽ có ngày trở thành hệ thống legacy. Để giảm tối đa nỗi khổ của những người bảo trì phần mềm về sau, ta nên cân nhắc vấn đề bảo trì ngay khi thiết kế. Dù không thể biết trước quyết định nào sẽ gây rắc rối bảo trì trong tương lai, cuốn sách này đặc biệt chú ý đến một số nguyên tắc có tính áp dụng rộng:

*khả năng vận hành* (*operability*)
: Giúp tổ chức dễ dàng giữ cho hệ thống chạy trơn tru.

*tính đơn giản* (*simplicity*)
: Dùng các mẫu và cấu trúc quen thuộc, nhất quán, tránh độ phức tạp không cần thiết, để kỹ sư mới cũng dễ hiểu hệ thống.

*khả năng tiến hóa* (*evolvability*)
: Giúp kỹ sư dễ sửa đổi hệ thống trong tương lai, điều chỉnh và mở rộng hệ thống khi yêu cầu thay đổi, để thích ứng với các trường hợp sử dụng chưa lường trước.

### Khả năng vận hành: giúp việc vận hành nhẹ nhàng hơn {#id37}

Ta đã bàn về vai trò của vận hành ở [“Vận hành trong thời đại đám mây”](/ch1#sec_introduction_operations), và thấy rằng để vận hành đáng tin cậy, quy trình của con người quan trọng ít nhất ngang với công cụ phần mềm. Thực tế có ý kiến cho rằng: “vận hành tốt thường có thể vượt qua giới hạn của phần mềm tồi (hoặc chưa đầy đủ), nhưng dù phần mềm tốt, vận hành tồi cũng không thể khiến nó chạy đáng tin cậy” [^60].

Hệ thống quy mô lớn gồm hàng nghìn máy, nếu chỉ bảo trì thủ công thì chi phí cao đến mức không kham nổi, nên tự động hóa là thiết yếu. Tuy nhiên, tự động hóa là con dao hai lưỡi: luôn có những trường hợp biên (ví dụ kịch bản lỗi hiếm gặp) cần đội vận hành can thiệp thủ công. Những gì tự động hóa không xử lý được lại chính là các vấn đề phức tạp nhất, nên mức độ tự động hóa càng cao thì càng cần một đội vận hành có kỹ năng *giỏi hơn* để giải quyết những vấn đề đó [^88].

Hơn nữa, khi một hệ thống tự động mắc lỗi, thường khó điều tra hơn hệ thống mà người vận hành làm tay một số thao tác. Vì vậy, đối với khả năng vận hành, tự động hóa không phải lúc nào cũng càng nhiều càng tốt. Một mức tự động hóa nhất định vẫn rất quan trọng, còn điểm cân bằng tối ưu tùy thuộc tình hình của từng ứng dụng và tổ chức.

Khả năng vận hành tốt nghĩa là làm cho công việc thường ngày nhẹ nhàng hơn, để đội vận hành tập trung sức vào các nhiệm vụ giá trị cao. Hệ thống dữ liệu có thể đơn giản hóa công việc công việc hằng ngày theo nhiều cách [^89] như sau:

* Cho phép công cụ giám sát kiểm tra các chỉ số then chốt của hệ thống, và hỗ trợ các công cụ quan sát được (xem [“Các vấn đề của hệ thống phân tán”](/ch1#sec_introduction_dist_sys_problems)) để hiểu sâu hành vi của hệ thống khi chạy. Nhiều công cụ thương mại và mã nguồn mở có thể giúp ích ở mặt này [^90].
* Tránh phụ thuộc vào bất kỳ máy nào, để có thể cho máy ngừng hoạt động để bảo trì mà toàn hệ thống vẫn chạy liên tục.
* Có tài liệu tốt và mô hình vận hành dễ hiểu (“nếu tôi làm X thì Y sẽ xảy ra”).
* Có hành vi mặc định tốt, đồng thời cho phép quản trị viên ghi đè các thiết lập mặc định khi cần.
* Tự sửa chữa khi thích hợp, đồng thời cho phép quản trị viên điều khiển thủ công trạng thái hệ thống khi cần.
* Thể hiện hành vi dễ dự đoán, hạn chế những điều bất ngờ.

### Tính đơn giản: quản lý độ phức tạp {#id38}

Dự án phần mềm nhỏ có thể có mã đơn giản, dễ chịu, giàu tính biểu đạt, nhưng khi dự án lớn dần, mã thường trở nên rất phức tạp và khó hiểu. Độ phức tạp này làm chậm mọi người phải làm việc trên hệ thống, và làm chi phí bảo trì tăng thêm. Một dự án phần mềm sa vào vũng lầy phức tạp đôi khi được gọi là *big ball of mud* (“cục bùn khổng lồ”) [^91].

Khi độ phức tạp khiến việc bảo trì trở nên khó khăn, ngân sách thường bị vượt và tiến độ thường bị chậm. Sửa đổi phần mềm phức tạp cũng dễ gây ra lỗi hơn: hệ thống càng khó hiểu và khó suy luận thì lập trình viên càng dễ bỏ sót các giả định ẩn, hậu quả ngoài ý muốn và tương tác bất ngờ [^69]. Ngược lại, giảm độ phức tạp có thể nâng cao đáng kể khả năng bảo trì của phần mềm, nên tính đơn giản cần là một mục tiêu then chốt khi ta xây dựng hệ thống.

Hệ thống đơn giản dễ hiểu hơn, nên ta cần giải quyết vấn đề đã cho bằng cách đơn giản nhất có thể. Tiếc là nói thì dễ, làm thì khó. Một thứ có đơn giản hay không thường phụ thuộc vào khẩu vị chủ quan, và không có tiêu chuẩn khách quan nào để đo tính đơn giản [^92]. Chẳng hạn, một hệ thống có thể giấu cách cài đặt phức tạp sau giao diện đơn giản, còn hệ thống khác có cách cài đặt tự thân đơn giản nhưng để lộ nhiều chi tiết nội bộ hơn cho người dùng — hệ thống nào đơn giản hơn?

Người ta từng cố chia độ phức tạp thành hai loại, *độ phức tạp thiết yếu* (*essential complexity*) và *độ phức tạp phát sinh* (*accidental complexity*), để suy luận về nó [^93]. Theo cách nghĩ này, độ phức tạp thiết yếu vốn có trong miền vấn đề của ứng dụng, còn độ phức tạp phát sinh chỉ nảy sinh do giới hạn của công cụ. Không may, sự phân biệt này cũng có khiếm khuyết, vì khi công cụ không ngừng tiến hóa, ranh giới giữa độ phức tạp thiết yếu và phát sinh cũng thay đổi [^94].

Một trong những công cụ tốt nhất để quản lý độ phức tạp là *trừu tượng hóa* (*abstraction*). Một trừu tượng tốt có thể giấu vô số chi tiết cài đặt sau một bề ngoài sạch sẽ, dễ hiểu, và cũng có thể dùng rộng rãi cho nhiều ứng dụng khác nhau. Tái sử dụng trừu tượng không chỉ hiệu quả hơn so với cài đặt lại các chức năng tương tự hết lần này đến lần khác, mà còn mang lại phần mềm chất lượng cao hơn, vì khi chất lượng của thành phần trừu tượng được cải thiện, mọi ứng dụng dùng nó đều được hưởng lợi.

Ví dụ, ngôn ngữ lập trình bậc cao là một trừu tượng, che giấu mã máy, thanh ghi CPU và lời gọi hệ thống. SQL cũng là một trừu tượng, che giấu các cấu trúc dữ liệu phức tạp trên đĩa và trong bộ nhớ, các yêu cầu đồng thời do client khác gửi đến, và những bất nhất phát sinh sau khi sập. Dĩ nhiên, khi lập trình bằng ngôn ngữ bậc cao, ta vẫn đang dùng mã máy, chỉ là ta không dùng nó *trực tiếp*, vì trừu tượng của ngôn ngữ lập trình giúp ta không phải nghĩ đến các chi tiết đó.

Để giảm độ phức tạp của mã ứng dụng, có thể dùng các phương pháp như *mẫu thiết kế* (*design pattern*) [^95] và *thiết kế hướng miền* (*domain-driven design*, *DDD*) [^96] để xây dựng các trừu tượng. Cuốn sách này không bàn về những trừu tượng dành riêng cho ứng dụng như vậy, mà bàn về các trừu tượng đa dụng như giao dịch cơ sở dữ liệu, chỉ mục (*index*) và nhật ký sự kiện. Bạn có thể xây dựng ứng dụng trên chúng. Nếu muốn áp dụng các phương pháp như DDD, bạn cũng có thể cài đặt chúng trên nền tảng được mô tả trong sách.

### Khả năng tiến hóa: giúp thay đổi dễ dàng hơn {#sec_introduction_evolvability}

Gần như không thể có chuyện yêu cầu của hệ thống mãi không đổi. Khả năng cao hơn là yêu cầu liên tục thay đổi: bạn biết thêm sự thật mới, xuất hiện các trường hợp sử dụng chưa lường trước, ưu tiên kinh doanh thay đổi, người dùng yêu cầu tính năng mới, nền tảng mới thay thế nền tảng cũ, yêu cầu pháp lý hoặc quy định thay đổi, sự tăng trưởng của hệ thống buộc kiến trúc phải thay đổi, v.v.

Về quy trình tổ chức, mô hình làm việc *Agile* cung cấp khung để thích ứng với thay đổi. Cộng đồng Agile cũng phát triển các công cụ kỹ thuật và quy trình phù hợp để phát triển phần mềm trong môi trường thay đổi thường xuyên, như phát triển hướng kiểm thử (*TDD*) và tái cấu trúc (refactoring). Cuốn sách này tìm cách nâng cao tính linh hoạt ở cấp hệ thống gồm nhiều ứng dụng hoặc dịch vụ có đặc tính khác nhau.

Việc sửa đổi hệ thống dữ liệu để thích ứng với yêu cầu thay đổi dễ hay khó liên quan chặt chẽ đến tính đơn giản và trừu tượng của hệ thống: hệ thống kết nối lỏng, đơn giản thường dễ sửa đổi hơn hệ thống kết nối chặt, phức tạp. Khái niệm này quan trọng đến mức ta dùng một từ khác để chỉ tính linh hoạt ở cấp hệ thống dữ liệu: *khả năng tiến hóa* (*evolvability*) [^97].

Một số thao tác trong hệ thống lớn là không thể đảo ngược, nên phải thực hiện hết sức thận trọng. Đây là một yếu tố chính khiến việc thay đổi trở nên khó khăn [^98]. Ví dụ, giả sử bạn cần di chuyển từ cơ sở dữ liệu này sang cơ sở dữ liệu khác: nếu hệ thống mới gặp vấn đề mà không thể chuyển lại hệ thống cũ, rủi ro cao hơn nhiều so với trường hợp có thể dễ dàng quay lui. Giảm thiểu tính không thể đảo ngược sẽ tăng tính linh hoạt của hệ thống.

## Tóm tắt {#summary}

Chương này đã xem xét một số yêu cầu phi chức năng: hiệu năng, độ tin cậy, khả năng mở rộng và khả năng bảo trì. Trong quá trình bàn các chủ đề này, ta cũng gặp một số nguyên tắc và thuật ngữ sẽ được dùng xuyên suốt cuốn sách. Chương bắt đầu từ tình huống triển khai home timeline của mạng xã hội, qua đó minh họa một số thách thức xuất hiện khi quy mô hệ thống tăng.

Ta đã bàn cách đo hiệu năng (ví dụ dùng phân vị của thời gian phản hồi), cách đo tải hệ thống (ví dụ dùng các chỉ số thông lượng), và cách dùng các chỉ số này trong SLA. Khả năng mở rộng có liên quan chặt chẽ: khi tải tăng, làm sao để hiệu năng giữ nguyên. Ta đã thấy một số nguyên tắc chung về khả năng mở rộng, chẳng hạn chia một tác vụ thành các phần nhỏ hơn có thể chạy độc lập với nhau. Các chương sau sẽ đi sâu vào chi tiết kỹ thuật để đạt khả năng mở rộng.

Để đạt độ tin cậy, có thể dùng các kỹ thuật chịu lỗi, giúp hệ thống tiếp tục cung cấp dịch vụ ngay cả khi một thành phần nào đó (ví dụ ổ đĩa, máy hoặc một dịch vụ khác) gặp lỗi. Ta đã xem xét các loại lỗi phần cứng có thể xảy ra và phân biệt chúng với lỗi phần mềm. Lỗi phần mềm thường có tương quan cao nên khó xử lý hơn. Một khía cạnh khác của việc nâng cao độ tin cậy là tăng khả năng chống chịu của hệ thống trước sai sót của con người. Ta cũng thấy postmortem không đổ lỗi giúp tổ chức học hỏi từ sự cố.

Cuối cùng, ta bàn về một số khía cạnh của khả năng bảo trì, gồm hỗ trợ công việc của đội vận hành, quản lý độ phức tạp, và làm cho chức năng của ứng dụng dễ tiến hóa theo thời gian. Không có câu trả lời đơn giản để đạt các mục tiêu này, nhưng việc xây dựng ứng dụng từ những khối quen thuộc, cung cấp các trừu tượng thực dụng, quả thực có ích. Phần còn lại của sách sẽ giới thiệu một loạt khối xây dựng đã được chứng minh là có giá trị trong thực tế.

### Tài liệu tham khảo

[^1]: Mike Cvet. [How We Learned to Stop Worrying and Love Fan-In at Twitter](https://www.youtube.com/watch?v=WEgCjwyXvwc). At *QCon San Francisco*, December 2016. 
[^2]: Raffi Krikorian. [Timelines at Scale](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/). At *QCon San Francisco*, November 2012. Archived at [perma.cc/V9G5-KLYK](https://perma.cc/V9G5-KLYK) 
[^3]: Twitter. [Twitter’s Recommendation Algorithm](https://blog.twitter.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm). *blog.twitter.com*, March 2023. Archived at [perma.cc/L5GT-229T](https://perma.cc/L5GT-229T)
[^4]: Raffi Krikorian. [New Tweets per second record, and how!](https://blog.twitter.com/engineering/en_us/a/2013/new-tweets-per-second-record-and-how) *blog.twitter.com*, August 2013. Archived at [perma.cc/6JZN-XJYN](https://perma.cc/6JZN-XJYN) 
[^5]: Jaz Volpert. [When Imperfect Systems are Good, Actually: Bluesky’s Lossy Timelines](https://jazco.dev/2025/02/19/imperfection/). *jazco.dev*, February 2025. Archived at [perma.cc/2PVE-L2MX](https://perma.cc/2PVE-L2MX)
[^6]: Samuel Axon. [3% of Twitter’s Servers Dedicated to Justin Bieber](https://mashable.com/archive/justin-bieber-twitter). *mashable.com*, September 2010. Archived at [perma.cc/F35N-CGVX](https://perma.cc/F35N-CGVX)
[^7]: Nathan Bronson, Abutalib Aghayev, Aleksey Charapko, and Timothy Zhu. [Metastable Failures in Distributed Systems](https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf). At *Workshop on Hot Topics in Operating Systems* (HotOS), May 2021. [doi:10.1145/3458336.3465286](https://doi.org/10.1145/3458336.3465286) 
[^8]: Marc Brooker. [Metastability and Distributed Systems](https://brooker.co.za/blog/2021/05/24/metastable.html). *brooker.co.za*, May 2021. Archived at [perma.cc/7FGJ-7XRK](https://perma.cc/7FGJ-7XRK) 
[^9]: Marc Brooker. [Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/). *aws.amazon.com*, March 2015. Archived at [perma.cc/R6MS-AZKH](https://perma.cc/R6MS-AZKH) 
[^10]: Marc Brooker. [What is Backoff For?](https://brooker.co.za/blog/2022/08/11/backoff.html) *brooker.co.za*, August 2022. Archived at [perma.cc/PW9N-55Q5](https://perma.cc/PW9N-55Q5) 
[^11]: Michael T. Nygard. [*Release It!*](https://learning.oreilly.com/library/view/release-it-2nd/9781680504552/), 2nd Edition. Pragmatic Bookshelf, January 2018. ISBN: 9781680502398 
[^12]: Frank Chen. [Slowing Down to Speed Up – Circuit Breakers for Slack’s CI/CD](https://slack.engineering/circuit-breakers/). *slack.engineering*, August 2022. Archived at [perma.cc/5FGS-ZPH3](https://perma.cc/5FGS-ZPH3)
[^13]: Marc Brooker. [Fixing retries with token buckets and circuit breakers](https://brooker.co.za/blog/2022/02/28/retries.html). *brooker.co.za*, February 2022. Archived at [perma.cc/MD6N-GW26](https://perma.cc/MD6N-GW26) 
[^14]: David Yanacek. [Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/). Amazon Builders’ Library, *aws.amazon.com*. Archived at [perma.cc/9SAW-68MP](https://perma.cc/9SAW-68MP)
[^15]: Matthew Sackman. [Pushing Back](https://wellquite.org/posts/lshift/pushing_back/). *wellquite.org*, May 2016. Archived at [perma.cc/3KCZ-RUFY](https://perma.cc/3KCZ-RUFY) 
[^16]: Dmitry Kopytkov and Patrick Lee. [Meet Bandaid, the Dropbox service proxy](https://dropbox.tech/infrastructure/meet-bandaid-the-dropbox-service-proxy). *dropbox.tech*, March 2018. Archived at [perma.cc/KUU6-YG4S](https://perma.cc/KUU6-YG4S) 
[^17]: Haryadi S. Gunawi, Riza O. Suminto, Russell Sears, Casey Golliher, Swaminathan Sundararaman, Xing Lin, Tim Emami, Weiguang Sheng, Nematollah Bidokhti, Caitie McCaffrey, Gary Grider, Parks M. Fields, Kevin Harms, Robert B. Ross, Andree Jacobson, Robert Ricci, Kirk Webb, Peter Alvaro, H. Birali Runesha, Mingzhe Hao, and Huaicheng Li. [Fail-Slow at Scale: Evidence of Hardware Performance Faults in Large Production Systems](https://www.usenix.org/system/files/conference/fast18/fast18-gunawi.pdf). At *16th USENIX Conference on File and Storage Technologies*, February 2018. 
[^18]: Marc Brooker. [Is the Mean Really Useless?](https://brooker.co.za/blog/2017/12/28/mean.html) *brooker.co.za*, December 2017. Archived at [perma.cc/U5AE-CVEM](https://perma.cc/U5AE-CVEM) 
[^19]: Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, Gunavardhan Kakulapati, Avinash Lakshman, Alex Pilchin, Swaminathan Sivasubramanian, Peter Vosshall, and Werner Vogels. [Dynamo: Amazon’s Highly Available Key-Value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf). At *21st ACM Symposium on Operating Systems Principles* (SOSP), October 2007. [doi:10.1145/1294261.1294281](https://doi.org/10.1145/1294261.1294281)
[^20]: Kathryn Whitenton. [The Need for Speed, 23 Years Later](https://www.nngroup.com/articles/the-need-for-speed/). *nngroup.com*, May 2020. Archived at [perma.cc/C4ER-LZYA](https://perma.cc/C4ER-LZYA) 
[^21]: Greg Linden. [Marissa Mayer at Web 2.0](https://glinden.blogspot.com/2006/11/marissa-mayer-at-web-20.html). *glinden.blogspot.com*, November 2005. Archived at [perma.cc/V7EA-3VXB](https://perma.cc/V7EA-3VXB) 
[^22]: Jake Brutlag. [Speed Matters for Google Web Search](https://services.google.com/fh/files/blogs/google_delayexp.pdf). *services.google.com*, June 2009. Archived at [perma.cc/BK7R-X7M2](https://perma.cc/BK7R-X7M2) 
[^23]: Eric Schurman and Jake Brutlag. [Performance Related Changes and their User Impact](https://www.youtube.com/watch?v=bQSE51-gr2s). Talk at *Velocity 2009*. 
[^24]: Akamai Technologies, Inc. [The State of Online Retail Performance](https://web.archive.org/web/20210729180749/https%3A//www.akamai.com/us/en/multimedia/documents/report/akamai-state-of-online-retail-performance-spring-2017.pdf). *akamai.com*, April 2017. Archived at [perma.cc/UEK2-HYCS](https://perma.cc/UEK2-HYCS) 
[^25]: Xiao Bai, Ioannis Arapakis, B. Barla Cambazoglu, and Ana Freire. [Understanding and Leveraging the Impact of Response Latency on User Behaviour in Web Search](https://doi.org/10.1145/3106372). *ACM Transactions on Information Systems*, volume 36, issue 2, article 21, April 2018. [doi:10.1145/3106372](https://doi.org/10.1145/3106372)
[^26]: Jeffrey Dean and Luiz André Barroso. [The Tail at Scale](https://cacm.acm.org/research/the-tail-at-scale/). *Communications of the ACM*, volume 56, issue 2, pages 74–80, February 2013. [doi:10.1145/2408776.2408794](https://doi.org/10.1145/2408776.2408794) 
[^27]: Alex Hidalgo. [*Implementing Service Level Objectives: A Practical Guide to SLIs, SLOs, and Error Budgets*](https://www.oreilly.com/library/view/implementing-service-level/9781492076803/). O’Reilly Media, September 2020. ISBN: 1492076813
[^28]: Jeffrey C. Mogul and John Wilkes. [Nines are Not Enough: Meaningful Metrics for Clouds](https://research.google/pubs/pub48033/). At *17th Workshop on Hot Topics in Operating Systems* (HotOS), May 2019. [doi:10.1145/3317550.3321432](https://doi.org/10.1145/3317550.3321432) 
[^29]: Tamás Hauer, Philipp Hoffmann, John Lunney, Dan Ardelean, and Amer Diwan. [Meaningful Availability](https://www.usenix.org/conference/nsdi20/presentation/hauer). At *17th USENIX Symposium on Networked Systems Design and Implementation* (NSDI), February 2020. 
[^30]: Ted Dunning. [The t-digest: Efficient estimates of distributions](https://www.sciencedirect.com/science/article/pii/S2665963820300403). *Software Impacts*, volume 7, article 100049, February 2021. [doi:10.1016/j.simpa.2020.100049](https://doi.org/10.1016/j.simpa.2020.100049) 
[^31]: David Kohn. [How percentile approximation works (and why it’s more useful than averages)](https://www.timescale.com/blog/how-percentile-approximation-works-and-why-its-more-useful-than-averages/). *timescale.com*, September 2021. Archived at [perma.cc/3PDP-NR8B](https://perma.cc/3PDP-NR8B)
[^32]: Heinrich Hartmann and Theo Schlossnagle. [Circllhist — A Log-Linear Histogram Data Structure for IT Infrastructure Monitoring](https://arxiv.org/pdf/2001.06561.pdf). *arxiv.org*, January 2020. 
[^33]: Charles Masson, Jee E. Rim, and Homin K. Lee. [DDSketch: A Fast and Fully-Mergeable Quantile Sketch with Relative-Error Guarantees](https://www.vldb.org/pvldb/vol12/p2195-masson.pdf). *Proceedings of the VLDB Endowment*, volume 12, issue 12, pages 2195–2205, August 2019. [doi:10.14778/3352063.3352135](https://doi.org/10.14778/3352063.3352135) 
[^34]: Baron Schwartz. [Why Percentiles Don’t Work the Way You Think](https://orangematter.solarwinds.com/2016/11/18/why-percentiles-dont-work-the-way-you-think/). *solarwinds.com*, November 2016. Archived at [perma.cc/469T-6UGB](https://perma.cc/469T-6UGB)
[^35]: Walter L. Heimerdinger and Charles B. Weinstock. [A Conceptual Framework for System Fault Tolerance](https://resources.sei.cmu.edu/asset_files/TechnicalReport/1992_005_001_16112.pdf). Technical Report CMU/SEI-92-TR-033, Software Engineering Institute, Carnegie Mellon University, October 1992. Archived at [perma.cc/GD2V-DMJW](https://perma.cc/GD2V-DMJW) 
[^36]: Felix C. Gärtner. [Fundamentals of fault-tolerant distributed computing in asynchronous environments](https://dl.acm.org/doi/pdf/10.1145/311531.311532). *ACM Computing Surveys*, volume 31, issue 1, pages 1–26, March 1999. [doi:10.1145/311531.311532](https://doi.org/10.1145/311531.311532) 
[^37]: Algirdas Avižienis, Jean-Claude Laprie, Brian Randell, and Carl Landwehr. [Basic Concepts and Taxonomy of Dependable and Secure Computing](https://hdl.handle.net/1903/6459). *IEEE Transactions on Dependable and Secure Computing*, volume 1, issue 1, January 2004. [doi:10.1109/TDSC.2004.2](https://doi.org/10.1109/TDSC.2004.2) 
[^38]: Ding Yuan, Yu Luo, Xin Zhuang, Guilherme Renna Rodrigues, Xu Zhao, Yongle Zhang, Pranay U. Jain, and Michael Stumm. [Simple Testing Can Prevent Most Critical Failures: An Analysis of Production Failures in Distributed Data-Intensive Systems](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-yuan.pdf). At *11th USENIX Symposium on Operating Systems Design and Implementation* (OSDI), October 2014. 
[^39]: Casey Rosenthal and Nora Jones. [*Chaos Engineering*](https://learning.oreilly.com/library/view/chaos-engineering/9781492043850/). O’Reilly Media, April 2020. ISBN: 9781492043867
[^40]: Eduardo Pinheiro, Wolf-Dietrich Weber, and Luiz Andre Barroso. [Failure Trends in a Large Disk Drive Population](https://www.usenix.org/legacy/events/fast07/tech/full_papers/pinheiro/pinheiro_old.pdf). At *5th USENIX Conference on File and Storage Technologies* (FAST), February 2007. 
[^41]: Bianca Schroeder and Garth A. Gibson. [Disk failures in the real world: What does an MTTF of 1,000,000 hours mean to you?](https://www.usenix.org/legacy/events/fast07/tech/schroeder/schroeder.pdf) At *5th USENIX Conference on File and Storage Technologies* (FAST), February 2007. 
[^42]: Andy Klein. [Backblaze Drive Stats for Q2 2021](https://www.backblaze.com/blog/backblaze-drive-stats-for-q2-2021/). *backblaze.com*, August 2021. Archived at [perma.cc/2943-UD5E](https://perma.cc/2943-UD5E) 
[^43]: Iyswarya Narayanan, Di Wang, Myeongjae Jeon, Bikash Sharma, Laura Caulfield, Anand Sivasubramaniam, Ben Cutler, Jie Liu, Badriddine Khessib, and Kushagra Vaid. [SSD Failures in Datacenters: What? When? and Why?](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/08/a7-narayanan.pdf) At *9th ACM International on Systems and Storage Conference* (SYSTOR), June 2016. [doi:10.1145/2928275.2928278](https://doi.org/10.1145/2928275.2928278) 
[^44]: Alibaba Cloud Storage Team. [Storage System Design Analysis: Factors Affecting NVMe SSD Performance (1)](https://www.alibabacloud.com/blog/594375). *alibabacloud.com*, January 2019. Archived at [archive.org](https://web.archive.org/web/20230522005034/https%3A//www.alibabacloud.com/blog/594375) 
[^45]: Bianca Schroeder, Raghav Lagisetty, and Arif Merchant. [Flash Reliability in Production: The Expected and the Unexpected](https://www.usenix.org/system/files/conference/fast16/fast16-papers-schroeder.pdf). At *14th USENIX Conference on File and Storage Technologies* (FAST), February 2016. 
[^46]: Jacob Alter, Ji Xue, Alma Dimnaku, and Evgenia Smirni. [SSD failures in the field: symptoms, causes, and prediction models](https://dl.acm.org/doi/pdf/10.1145/3295500.3356172). At *International Conference for High Performance Computing, Networking, Storage and Analysis* (SC), November 2019. [doi:10.1145/3295500.3356172](https://doi.org/10.1145/3295500.3356172) 
[^47]: Daniel Ford, François Labelle, Florentina I. Popovici, Murray Stokely, Van-Anh Truong, Luiz Barroso, Carrie Grimes, and Sean Quinlan. [Availability in Globally Distributed Storage Systems](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Ford.pdf). At *9th USENIX Symposium on Operating Systems Design and Implementation* (OSDI), October 2010. 
[^48]: Kashi Venkatesh Vishwanath and Nachiappan Nagappan. [Characterizing Cloud Computing Hardware Reliability](https://www.microsoft.com/en-us/research/wp-content/uploads/2010/06/socc088-vishwanath.pdf). At *1st ACM Symposium on Cloud Computing* (SoCC), June 2010. [doi:10.1145/1807128.1807161](https://doi.org/10.1145/1807128.1807161) 
[^49]: Peter H. Hochschild, Paul Turner, Jeffrey C. Mogul, Rama Govindaraju, Parthasarathy Ranganathan, David E. Culler, and Amin Vahdat. [Cores that don’t count](https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s01-hochschild.pdf). At *Workshop on Hot Topics in Operating Systems* (HotOS), June 2021. [doi:10.1145/3458336.3465297](https://doi.org/10.1145/3458336.3465297)
[^50]: Harish Dattatraya Dixit, Sneha Pendharkar, Matt Beadon, Chris Mason, Tejasvi Chakravarthy, Bharath Muthiah, and Sriram Sankar. [Silent Data Corruptions at Scale](https://arxiv.org/abs/2102.11245). *arXiv:2102.11245*, February 2021. 
[^51]: Diogo Behrens, Marco Serafini, Sergei Arnautov, Flavio P. Junqueira, and Christof Fetzer. [Scalable Error Isolation for Distributed Systems](https://www.usenix.org/conference/nsdi15/technical-sessions/presentation/behrens). At *12th USENIX Symposium on Networked Systems Design and Implementation* (NSDI), May 2015. 
[^52]: Bianca Schroeder, Eduardo Pinheiro, and Wolf-Dietrich Weber. [DRAM Errors in the Wild: A Large-Scale Field Study](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/35162.pdf). At *11th International Joint Conference on Measurement and Modeling of Computer Systems* (SIGMETRICS), June 2009. [doi:10.1145/1555349.1555372](https://doi.org/10.1145/1555349.1555372) 
[^53]: Yoongu Kim, Ross Daly, Jeremie Kim, Chris Fallin, Ji Hye Lee, Donghyuk Lee, Chris Wilkerson, Konrad Lai, and Onur Mutlu. [Flipping Bits in Memory Without Accessing Them: An Experimental Study of DRAM Disturbance Errors](https://users.ece.cmu.edu/~yoonguk/papers/kim-isca14.pdf). At *41st Annual International Symposium on Computer Architecture* (ISCA), June 2014. [doi:10.1145/2678373.2665726](https://doi.org/10.1145/2678373.2665726)
[^54]: Tim Bray. [Worst Case](https://www.tbray.org/ongoing/When/202x/2021/10/08/The-WOrst-Case). *tbray.org*, October 2021. Archived at [perma.cc/4QQM-RTHN](https://perma.cc/4QQM-RTHN) 
[^55]: Sangeetha Abdu Jyothi. [Solar Superstorms: Planning for an Internet Apocalypse](https://ics.uci.edu/~sabdujyo/papers/sigcomm21-cme.pdf). At *ACM SIGCOMM Conferene*, August 2021. [doi:10.1145/3452296.3472916](https://doi.org/10.1145/3452296.3472916) 
[^56]: Adrian Cockcroft. [Failure Modes and Continuous Resilience](https://adrianco.medium.com/failure-modes-and-continuous-resilience-6553078caad5). *adrianco.medium.com*, November 2019. Archived at [perma.cc/7SYS-BVJP](https://perma.cc/7SYS-BVJP) 
[^57]: Shujie Han, Patrick P. C. Lee, Fan Xu, Yi Liu, Cheng He, and Jiongzhou Liu. [An In-Depth Study of Correlated Failures in Production SSD-Based Data Centers](https://www.usenix.org/conference/fast21/presentation/han). At *19th USENIX Conference on File and Storage Technologies* (FAST), February 2021. 
[^58]: Edmund B. Nightingale, John R. Douceur, and Vince Orgovan. [Cycles, Cells and Platters: An Empirical Analysis of Hardware Failures on a Million Consumer PCs](https://eurosys2011.cs.uni-salzburg.at/pdf/eurosys2011-nightingale.pdf). At *6th European Conference on Computer Systems* (EuroSys), April 2011. [doi:10.1145/1966445.1966477](https://doi.org/10.1145/1966445.1966477) 
[^59]: Haryadi S. Gunawi, Mingzhe Hao, Tanakorn Leesatapornwongsa, Tiratat Patana-anake, Thanh Do, Jeffry Adityatama, Kurnia J. Eliazar, Agung Laksono, Jeffrey F. Lukman, Vincentius Martin, and Anang D. Satria. [What Bugs Live in the Cloud?](https://ucare.cs.uchicago.edu/pdf/socc14-cbs.pdf) At *5th ACM Symposium on Cloud Computing* (SoCC), November 2014. [doi:10.1145/2670979.2670986](https://doi.org/10.1145/2670979.2670986) 
[^60]: Jay Kreps. [Getting Real About Distributed System Reliability](https://blog.empathybox.com/post/19574936361/getting-real-about-distributed-system-reliability). *blog.empathybox.com*, March 2012. Archived at [perma.cc/9B5Q-AEBW](https://perma.cc/9B5Q-AEBW) 
[^61]: Nelson Minar. [Leap Second Crashes Half the Internet](https://www.somebits.com/weblog/tech/bad/leap-second-2012.html). *somebits.com*, July 2012. Archived at [perma.cc/2WB8-D6EU](https://perma.cc/2WB8-D6EU) 
[^62]: Hewlett Packard Enterprise. [Support Alerts – Customer Bulletin a00092491en\_us](https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-a00092491en_us). *support.hpe.com*, November 2019. Archived at [perma.cc/S5F6-7ZAC](https://perma.cc/S5F6-7ZAC) 
[^63]: Lorin Hochstein. [awesome limits](https://github.com/lorin/awesome-limits). *github.com*, November 2020. Archived at [perma.cc/3R5M-E5Q4](https://perma.cc/3R5M-E5Q4) 
[^64]: Caitie McCaffrey. [Clients Are Jerks: AKA How Halo 4 DoSed the Services at Launch & How We Survived](https://www.caitiem.com/2015/06/23/clients-are-jerks-aka-how-halo-4-dosed-the-services-at-launch-how-we-survived/). *caitiem.com*, June 2015. Archived at [perma.cc/MXX4-W373](https://perma.cc/MXX4-W373) 
[^65]: Lilia Tang, Chaitanya Bhandari, Yongle Zhang, Anna Karanika, Shuyang Ji, Indranil Gupta, and Tianyin Xu. [Fail through the Cracks: Cross-System Interaction Failures in Modern Cloud Systems](https://tianyin.github.io/pub/csi-failures.pdf). At *18th European Conference on Computer Systems* (EuroSys), May 2023. [doi:10.1145/3552326.3587448](https://doi.org/10.1145/3552326.3587448) 
[^66]: Mike Ulrich. [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/). In Betsy Beyer, Jennifer Petoff, Chris Jones, and Niall Richard Murphy (ed). [*Site Reliability Engineering: How Google Runs Production Systems*](https://www.oreilly.com/library/view/site-reliability-engineering/9781491929117/). O’Reilly Media, 2016. ISBN: 9781491929124
[^67]: Harri Faßbender. [Cascading failures in large-scale distributed systems](https://blog.mi.hdm-stuttgart.de/index.php/2022/03/03/cascading-failures-in-large-scale-distributed-systems/). *blog.mi.hdm-stuttgart.de*, March 2022. Archived at [perma.cc/K7VY-YJRX](https://perma.cc/K7VY-YJRX) 
[^68]: Richard I. Cook. [How Complex Systems Fail](https://www.adaptivecapacitylabs.com/HowComplexSystemsFail.pdf). Cognitive Technologies Laboratory, April 2000. Archived at [perma.cc/RDS6-2YVA](https://perma.cc/RDS6-2YVA) 
[^69]: David D. Woods. [STELLA: Report from the SNAFUcatchers Workshop on Coping With Complexity](https://snafucatchers.github.io/). *snafucatchers.github.io*, March 2017. Archived at [archive.org](https://web.archive.org/web/20230306130131/https%3A//snafucatchers.github.io/) 
[^70]: David Oppenheimer, Archana Ganapathi, and David A. Patterson. [Why Do Internet Services Fail, and What Can Be Done About It?](https://static.usenix.org/events/usits03/tech/full_papers/oppenheimer/oppenheimer.pdf) At *4th USENIX Symposium on Internet Technologies and Systems* (USITS), March 2003. 
[^71]: Sidney Dekker. [*The Field Guide to Understanding ‘Human Error’, 3rd Edition*](https://learning.oreilly.com/library/view/the-field-guide/9781317031833/). CRC Press, November 2017. ISBN: 9781472439055
[^72]: Sidney Dekker. [*Drift into Failure: From Hunting Broken Components to Understanding Complex Systems*](https://doi.org/10.1201/9781315257396). CRC Press, 2011. ISBN: 9781315257396
[^73]: John Allspaw. [Blameless PostMortems and a Just Culture](https://www.etsy.com/codeascraft/blameless-postmortems/). *etsy.com*, May 2012. Archived at [perma.cc/YMJ7-NTAP](https://perma.cc/YMJ7-NTAP) 
[^74]: Itzy Sabo. [Uptime Guarantees — A Pragmatic Perspective](https://world.hey.com/itzy/uptime-guarantees-a-pragmatic-perspective-736d7ea4). *world.hey.com*, March 2023. Archived at [perma.cc/F7TU-78JB](https://perma.cc/F7TU-78JB) 
[^75]: Michael Jurewitz. [The Human Impact of Bugs](http://jury.me/blog/2013/3/14/the-human-impact-of-bugs). *jury.me*, March 2013. Archived at [perma.cc/5KQ4-VDYL](https://perma.cc/5KQ4-VDYL) 
[^76]: Mark Halper. [How Software Bugs led to ‘One of the Greatest Miscarriages of Justice’ in British History](https://cacm.acm.org/news/how-software-bugs-led-to-one-of-the-greatest-miscarriages-of-justice-in-british-history/). *Communications of the ACM*, January 2025. [doi:10.1145/3703779](https://doi.org/10.1145/3703779)
[^77]: Nicholas Bohm, James Christie, Peter Bernard Ladkin, Bev Littlewood, Paul Marshall, Stephen Mason, Martin Newby, Steven J. Murdoch, Harold Thimbleby, and Martyn Thomas. [The legal rule that computers are presumed to be operating correctly – unforeseen and unjust consequences](https://www.benthamsgaze.org/wp-content/uploads/2022/06/briefing-presumption-that-computers-are-reliable.pdf). Briefing note, *benthamsgaze.org*, June 2022. Archived at [perma.cc/WQ6X-TMW4](https://perma.cc/WQ6X-TMW4) 
[^78]: Dan McKinley. [Choose Boring Technology](https://mcfunley.com/choose-boring-technology). *mcfunley.com*, March 2015. Archived at [perma.cc/7QW7-J4YP](https://perma.cc/7QW7-J4YP) 
[^79]: Andy Warfield. [Building and operating a pretty big storage system called S3](https://www.allthingsdistributed.com/2023/07/building-and-operating-a-pretty-big-storage-system.html). *allthingsdistributed.com*, July 2023. Archived at [perma.cc/7LPK-TP7V](https://perma.cc/7LPK-TP7V) 
[^80]: Marc Brooker. [Surprising Scalability of Multitenancy](https://brooker.co.za/blog/2023/03/23/economics.html). *brooker.co.za*, March 2023. Archived at [perma.cc/ZZD9-VV8T](https://perma.cc/ZZD9-VV8T) 
[^81]: Ben Stopford. [Shared Nothing vs. Shared Disk Architectures: An Independent View](http://www.benstopford.com/2009/11/24/understanding-the-shared-nothing-architecture/). *benstopford.com*, November 2009. Archived at [perma.cc/7BXH-EDUR](https://perma.cc/7BXH-EDUR) 
[^82]: Michael Stonebraker. [The Case for Shared Nothing](https://dsf.berkeley.edu/papers/hpts85-nothing.pdf). *IEEE Database Engineering Bulletin*, volume 9, issue 1, pages 4–9, March 1986. 
[^83]: Panagiotis Antonopoulos, Alex Budovski, Cristian Diaconu, Alejandro Hernandez Saenz, Jack Hu, Hanuma Kodavalla, Donald Kossmann, Sandeep Lingam, Umar Farooq Minhas, Naveen Prakash, Vijendra Purohit, Hugh Qu, Chaitanya Sreenivas Ravella, Krystyna Reisteter, Sheetal Shrotri, Dixin Tang, and Vikram Wakade. [Socrates: The New SQL Server in the Cloud](https://www.microsoft.com/en-us/research/uploads/prod/2019/05/socrates.pdf). At *ACM International Conference on Management of Data* (SIGMOD), pages 1743–1756, June 2019. [doi:10.1145/3299869.3314047](https://doi.org/10.1145/3299869.3314047) 
[^84]: Sam Newman. [*Building Microservices*, second edition](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/). O’Reilly Media, 2021. ISBN: 9781492034025
[^85]: Nathan Ensmenger. [When Good Software Goes Bad: The Surprising Durability of an Ephemeral Technology](https://themaintainers.wpengine.com/wp-content/uploads/2021/04/ensmenger-maintainers-v2.pdf). At *The Maintainers Conference*, April 2016. Archived at [perma.cc/ZXT4-HGZB](https://perma.cc/ZXT4-HGZB) 
[^86]: Robert L. Glass. [*Facts and Fallacies of Software Engineering*](https://learning.oreilly.com/library/view/facts-and-fallacies/0321117425/). Addison-Wesley Professional, October 2002. ISBN: 9780321117427 
[^87]: Marianne Bellotti. [*Kill It with Fire*](https://learning.oreilly.com/library/view/kill-it-with/9781098128883/). No Starch Press, April 2021. ISBN: 9781718501188 
[^88]: Lisanne Bainbridge. [Ironies of automation](https://www.adaptivecapacitylabs.com/IroniesOfAutomation-Bainbridge83.pdf). *Automatica*, volume 19, issue 6, pages 775–779, November 1983. [doi:10.1016/0005-1098(83)90046-8](https://doi.org/10.1016/0005-1098%2883%2990046-8) 
[^89]: James Hamilton. [On Designing and Deploying Internet-Scale Services](https://www.usenix.org/legacy/events/lisa07/tech/full_papers/hamilton/hamilton.pdf). At *21st Large Installation System Administration Conference* (LISA), November 2007. 
[^90]: Dotan Horovits. [Open Source for Better Observability](https://horovits.medium.com/open-source-for-better-observability-8c65b5630561). *horovits.medium.com*, October 2021. Archived at [perma.cc/R2HD-U2ZT](https://perma.cc/R2HD-U2ZT) 
[^91]: Brian Foote and Joseph Yoder. [Big Ball of Mud](http://www.laputan.org/pub/foote/mud.pdf). At *4th Conference on Pattern Languages of Programs* (PLoP), September 1997. Archived at [perma.cc/4GUP-2PBV](https://perma.cc/4GUP-2PBV) 
[^92]: Marc Brooker. [What is a simple system?](https://brooker.co.za/blog/2022/05/03/simplicity.html) *brooker.co.za*, May 2022. Archived at [perma.cc/U72T-BFVE](https://perma.cc/U72T-BFVE) 
[^93]: Frederick P. Brooks. [No Silver Bullet – Essence and Accident in Software Engineering](https://worrydream.com/refs/Brooks_1986_-_No_Silver_Bullet.pdf). In [*The Mythical Man-Month*](https://www.oreilly.com/library/view/mythical-man-month-the/0201835959/), Anniversary edition, Addison-Wesley, 1995. ISBN: 9780201835953 
[^94]: Dan Luu. [Against essential and accidental complexity](https://danluu.com/essential-complexity/). *danluu.com*, December 2020. Archived at [perma.cc/H5ES-69KC](https://perma.cc/H5ES-69KC) 
[^95]: Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides. [*Design Patterns: Elements of Reusable Object-Oriented Software*](https://learning.oreilly.com/library/view/design-patterns-elements/0201633612/). Addison-Wesley Professional, October 1994. ISBN: 9780201633610 
[^96]: Eric Evans. [*Domain-Driven Design: Tackling Complexity in the Heart of Software*](https://learning.oreilly.com/library/view/domain-driven-design-tackling/0321125215/). Addison-Wesley Professional, August 2003. ISBN: 9780321125217 
[^97]: Hongyu Pei Breivold, Ivica Crnkovic, and Peter J. Eriksson. [Analyzing Software Evolvability](https://www.es.mdh.se/pdf_publications/1251.pdf). at *32nd Annual IEEE International Computer Software and Applications Conference* (COMPSAC), July 2008. [doi:10.1109/COMPSAC.2008.50](https://doi.org/10.1109/COMPSAC.2008.50) 
[^98]: Enrico Zaninotto. [From X programming to the X organisation](https://martinfowler.com/articles/zaninotto.pdf). At *XP Conference*, May 2002. Archived at [perma.cc/R9AR-QCKZ](https://perma.cc/R9AR-QCKZ)
