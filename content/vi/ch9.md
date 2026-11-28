---
translationKey: ddia-v2-ch9
title: Những rắc rối của hệ phân tán
book_kind: chapter
book_number: "9"
book_part: II
weight: 209
breadcrumbs: false
---

<a id="ch_distributed"></a>

![](/map/ch08.png)

> *Tai nạn là thứ buồn cười thật. Chưa gặp thì chẳng bao giờ có, mà đã gặp rồi thì nó cứ thế xảy ra.*
>
> A.A. Milne, *The House at Pooh Corner* (1928)

Như đã bàn ở ["Độ tin cậy và khả năng chịu lỗi"](/ch2#sec_introduction_reliability), làm cho một hệ thống đáng tin cậy nghĩa là bảo đảm cả hệ thống vẫn tiếp tục chạy ngay cả khi có sự cố (tức là khi xảy ra lỗi). Tuy nhiên, lường trước mọi lỗi có thể xảy ra rồi xử lý chúng không dễ chút nào. Là lập trình viên, ta rất dễ chỉ tập trung vào đường đi suôn sẻ (xét cho cùng, phần lớn thời gian mọi thứ chạy tốt!) và bỏ qua lỗi, vì lỗi kéo theo vô số trường hợp biên.

Nếu muốn hệ thống đáng tin cậy khi có lỗi, bạn phải thay đổi căn bản tư duy và tập trung vào những chỗ có thể hỏng, dù xác suất rất thấp. Một việc chỉ có một phần triệu khả năng hỏng không có nghĩa là cứ mặc kệ: khi hệ thống đủ lớn, sự kiện một phần triệu xảy ra mỗi ngày. Những người vận hành hệ thống có kinh nghiệm sẽ nói với bạn rằng bất cứ thứ gì *có thể* hỏng thì *rồi sẽ* hỏng.

Hơn nữa, làm việc với *hệ thống phân tán* (*distributed system*) khác căn bản so với viết phần mềm trên một máy tính duy nhất — khác biệt lớn nhất là có rất nhiều cách hỏng mới lạ và "thú vị" [^1] [^2]. Chương này sẽ đưa bạn làm quen với những vấn đề gặp phải trong thực tế, và giúp bạn hiểu cái gì có thể dựa vào, cái gì thì không.

Để hiểu thách thức trước mắt, ta hãy đẩy chủ nghĩa bi quan lên mức tối đa và xem xét đủ thứ có thể hỏng trong một hệ thống phân tán. Ta sẽ bàn về các vấn đề của mạng (["Mạng không đáng tin cậy"](/ch9#sec_distributed_networks)) cũng như của đồng hồ và thời gian (["Đồng hồ không đáng tin cậy"](/ch9#sec_distributed_clocks)). Hệ quả của những vấn đề này thường gây mất phương hướng, nên ta cũng sẽ khảo sát cách nhận biết trạng thái của một hệ thống phân tán và cách suy luận về những gì đã xảy ra (["Tri thức, sự thật và lời nói dối"](/ch9#sec_distributed_truth)). Sau đó, ở [Chương 10](/ch10#ch_consistency), ta sẽ xem qua một số ví dụ về cách đạt được khả năng chịu lỗi trước những sự cố này.

## Sự cố và hỏng hóc từng phần {#sec_distributed_partial_failure}

Khi viết chương trình trên một máy tính duy nhất, nó thường chạy theo cách khá dễ đoán: hoặc chạy đúng, hoặc hỏng hẳn. Phần mềm có lỗi đôi khi khiến ta có cảm giác máy tính "đang trục trặc" (và khởi động lại thường là xong), nhưng đó thường chỉ là hậu quả của phần mềm viết dở.

Về nguyên tắc, phần mềm trên một máy đơn không nên chạy lúc được lúc không: khi phần cứng bình thường, cùng một thao tác luôn cho cùng một kết quả (đây là tính *tất định* (*deterministic*)). Nếu phần cứng gặp vấn đề (ví dụ bộ nhớ hỏng hoặc đầu nối lỏng), hậu quả thường là cả hệ thống hỏng (chẳng hạn kernel panic, "màn hình xanh chết chóc", hoặc không khởi động được). Một máy tính chạy phần mềm tốt thường hoặc hoạt động đầy đủ, hoặc hỏng hoàn toàn, chứ không nằm lơ lửng giữa hai trạng thái đó.

Đây là lựa chọn có chủ ý trong thiết kế máy tính: khi có lỗi nội bộ, ta thà để máy sập hẳn còn hơn để nó trả về kết quả sai, vì kết quả sai vừa khó xử lý vừa dễ gây rối. Nhờ vậy máy tính che giấu thực tại vật lý mơ hồ, lộn xộn bên dưới và đưa ra một mô hình hệ thống lý tưởng hóa vận hành hoàn hảo như toán học. Một lệnh CPU lúc nào cũng làm cùng một việc. Dữ liệu ghi vào bộ nhớ hay đĩa được giữ nguyên, không bị hỏng ngẫu nhiên. Như đã bàn ở ["Lỗi phần cứng và phần mềm"](/ch2#sec_introduction_hardware_faults), thực tế không thật sự như vậy — dữ liệu hoàn toàn có thể hỏng mà không báo trước, và CPU đôi khi âm thầm cho kết quả sai — nhưng những trường hợp đó đủ hiếm để thường bỏ qua được.

Khi phần mềm chạy trên nhiều máy nối với nhau bằng mạng, tình hình hoàn toàn khác. Lỗi trong hệ thống phân tán xảy ra thường xuyên hơn nhiều, ta không thể làm ngơ nữa mà buộc phải đối mặt với thực tế lộn xộn của thế giới vật lý. Và trong thế giới vật lý, số thứ có thể hỏng nhiều đến kinh ngạc, như giai thoại sau đây cho thấy: [^3]

> Trong kinh nghiệm hạn chế của tôi, tôi đã phải xử lý các vụ network partition kéo dài trong một trung tâm dữ liệu (DC), lỗi PDU (bộ phân phối điện), lỗi switch, cả một rack bất ngờ mất điện rồi khởi động lại, lỗi mạng backbone của cả trung tâm dữ liệu, mất điện toàn bộ trung tâm dữ liệu, và một tài xế bị hạ đường huyết lái chiếc xe bán tải Ford đâm vào hệ thống HVAC (sưởi, thông gió và điều hòa) của trung tâm dữ liệu. Mà tôi còn chẳng phải là dân vận hành.
>
> — Coda Hale

Trong hệ thống phân tán, rất có thể một phần nào đó của hệ thống hỏng theo cách không thể đoán trước, dù các phần còn lại vẫn chạy bình thường. Đây gọi là *hỏng hóc từng phần* (*partial failure*). Điều khó chịu là hỏng hóc từng phần có tính *phi tất định* (*nondeterministic*): bất kỳ thao tác nào liên quan đến nhiều node và mạng của chúng đôi khi thành công, đôi khi lại thất bại một cách khó lường. Như ta sẽ thấy, thậm chí bạn có thể *không biết* một việc đã thành công hay chưa!

Chính tính phi tất định và khả năng xảy ra hỏng hóc từng phần khiến hệ thống phân tán khó làm việc đến vậy [^4]. Tuy nhiên, nếu hệ thống phân tán chịu được hỏng hóc từng phần thì nó cũng có được những khả năng mạnh mẽ. Chẳng hạn, bạn có thể thực hiện rolling upgrade: khởi động lại từng node một để cài bản cập nhật phần mềm, trong khi cả hệ thống vẫn chạy liên tục không gián đoạn. Như vậy, nhờ khả năng chịu lỗi, ta có thể xây dựng hệ thống phân tán đáng tin cậy hơn hệ thống một node, từ những thành phần không đáng tin cậy.

Nhưng trước khi hiện thực khả năng chịu lỗi, ta cần hiểu rõ hơn về những lỗi mà ta phải chịu đựng. Điều quan trọng là xem xét thật nhiều loại lỗi có thể xảy ra — kể cả những lỗi khá khó xảy ra — và cố tình tạo ra những tình huống đó trong môi trường kiểm thử để xem chuyện gì sẽ xảy ra. Trong hệ thống phân tán, đa nghi, bi quan và cả hoang tưởng đều đáng giá.

## Mạng không đáng tin cậy {#sec_distributed_networks}

Như đã bàn ở ["Kiến trúc chia sẻ bộ nhớ, chia sẻ đĩa và shared-nothing"](/ch2#sec_introduction_shared_nothing), các hệ thống phân tán trong cuốn sách này phần lớn là *hệ thống shared-nothing*, tức là một tập hợp các máy nối với nhau bằng mạng. Mạng là cách duy nhất để các máy này liên lạc với nhau — ta giả định mỗi máy có bộ nhớ và đĩa riêng, máy này không thể truy cập trực tiếp bộ nhớ hay đĩa của máy kia mà chỉ có thể gửi yêu cầu đến một dịch vụ qua mạng. Ngay cả khi lưu trữ được chia sẻ (như Amazon S3), các máy vẫn phải giao tiếp với dịch vụ lưu trữ chung qua mạng.

Internet và phần lớn các mạng nội bộ trong trung tâm dữ liệu (thường là Ethernet) đều là *mạng gói bất đồng bộ* (*asynchronous packet network*). Trong loại mạng này, một node có thể gửi thông điệp (một gói tin) cho node khác, nhưng mạng không bảo đảm khi nào thông điệp đến, cũng không bảo đảm nó có đến hay không. Nếu bạn gửi yêu cầu và chờ phản hồi, nhiều chuyện có thể xảy ra (một số được minh họa ở {{< xref fig="9-1" page="/ch9" anchor="fig_distributed_network" >}}Hình 9-1{{< /xref >}}):

1. Yêu cầu có thể đã bị mất (có thể ai đó rút dây mạng).
2. Yêu cầu có thể đang nằm chờ trong hàng đợi và sẽ được chuyển đi sau (có thể mạng hoặc bên nhận đang quá tải).
3. Node ở xa có thể đã hỏng (có thể nó sập, hoặc bị tắt).
4. Node ở xa có thể tạm thời ngừng phản hồi (có thể nó đang chịu một lần tạm dừng garbage collection kéo dài, xem ["Tạm dừng tiến trình"](/ch9#sec_distributed_clocks_pauses)), rồi sẽ phản hồi lại sau.
5. Node ở xa có thể đã xử lý yêu cầu, nhưng phản hồi bị mất trên đường (có thể một switch mạng bị cấu hình sai).
6. Node ở xa có thể đã xử lý yêu cầu, nhưng phản hồi bị trễ và sẽ đến sau (có thể mạng hoặc chính máy của bạn đang quá tải).

{{< fig num="9-1" id="fig_distributed_network" src="/fig/ddia_0901.png" caption="Khi gửi yêu cầu mà không nhận được phản hồi, không thể biết nguyên nhân là (a) yêu cầu bị mất, (b) node ở xa đã ngừng hoạt động, hay (c) phản hồi bị mất." class="ddia-figure ddia-figure--panorama" width="2880" height="898" />}}


Bên gửi thậm chí không thể biết gói tin đã đến nơi chưa: cách duy nhất là để bên nhận gửi lại một thông điệp phản hồi, mà phản hồi này cũng có thể bị mất hoặc bị trễ. Trong mạng bất đồng bộ, các tình huống này không thể phân biệt được: thông tin duy nhất bạn có là "chưa nhận được phản hồi". Khi gửi yêu cầu cho node khác mà không nhận được phản hồi, *không thể* biết nguyên nhân là gì.

Cách xử lý thông thường cho vấn đề này là đặt timeout: chờ một khoảng thời gian rồi bỏ cuộc, giả định rằng phản hồi sẽ không đến nữa. Tuy nhiên, dù đã hết timeout, bạn vẫn không biết node ở xa có nhận được yêu cầu hay không (nếu yêu cầu vẫn đang xếp hàng đâu đó, nó vẫn có thể đến bên nhận sau, dù bên gửi đã bỏ cuộc).

### Giới hạn của TCP {#sec_distributed_tcp}

Gói tin mạng có kích thước tối đa (thường chỉ vài nghìn byte), nhưng nhiều ứng dụng cần gửi những thông điệp không vừa trong một gói, chẳng hạn yêu cầu và phản hồi. Các ứng dụng này thường dùng TCP (Transmission Control Protocol) để thiết lập một *kết nối* (*connection*), chia luồng dữ liệu lớn thành từng gói tin rồi ghép lại ở đầu nhận.


> [!NOTE]
> Phần lớn nội dung bàn về TCP dưới đây cũng áp dụng cho các giải pháp thay thế mới hơn như QUIC, giao thức SCTP (Stream Control Transmission Protocol) dùng trong WebRTC, giao thức uTP của BitTorrent và các giao thức truyền tải khác. So sánh với UDP, xem ["TCP và UDP"](/ch9#sidebar_distributed_tcp_udp).


TCP thường được nói là cung cấp đường truyền "đáng tin cậy", với nghĩa: nó phát hiện và truyền lại gói tin bị mất, phát hiện gói tin sai thứ tự và khôi phục đúng thứ tự, đồng thời phát hiện gói tin hỏng bằng checksum đơn giản. TCP cũng quyết định nên gửi dữ liệu nhanh đến mức nào, để vừa truyền nhanh nhất có thể vừa không làm quá tải mạng hay node nhận. Việc này gọi là *kiểm soát tắc nghẽn* (*congestion control*), *kiểm soát luồng* (*flow control*) hay backpressure [^5].

Khi bạn "gửi" dữ liệu bằng cách ghi vào socket, dữ liệu không được gửi đi ngay mà trước hết đi vào bộ đệm do hệ điều hành quản lý. Khi thuật toán kiểm soát tắc nghẽn quyết định hiện có thể gửi gói tin, nó lấy từ bộ đệm một lượng dữ liệu bằng một gói tin và đưa cho card mạng. Gói tin đi qua một số switch và router, cuối cùng hệ điều hành của node nhận đặt dữ liệu vào bộ đệm nhận rồi gửi trả gói xác nhận cho bên gửi. Chỉ đến lúc đó hệ điều hành bên nhận mới báo cho ứng dụng rằng có thêm dữ liệu đến [^6].

Vậy TCP đã "đáng tin cậy" thì ta không cần lo mạng không đáng tin cậy nữa sao? Tiếc là không. Nếu sau một khoảng timeout nhất định không nhận được xác nhận, TCP cho rằng gói tin chắc chắn đã mất, nhưng nó cũng không thể biết cái mất là gói tin đã gửi hay gói xác nhận trả về. TCP có thể gửi lại, nhưng không bảo đảm gói gửi lại sẽ đến nơi. Nếu dây mạng bị rút, TCP không thể tự cắm lại giúp bạn. Cuối cùng, sau một timeout có thể cấu hình, TCP ngừng thử lại và báo lỗi cho ứng dụng.

Nếu kết nối TCP bị đóng do lỗi — có thể node ở xa sập, cũng có thể mạng bị đứt — bạn không thể biết node ở xa đã xử lý được bao nhiêu dữ liệu [^6]. Ngay cả khi TCP đã xác nhận một gói tin được chuyển đến, điều đó chỉ có nghĩa là kernel hệ điều hành của node ở xa đã nhận được nó, còn ứng dụng vẫn có thể sập trước khi kịp xử lý dữ liệu. Muốn biết chắc yêu cầu thành công, ứng dụng bên kia phải trả về một phản hồi thành công một cách tường minh [^7].

Dù vậy, TCP vẫn rất hữu ích vì nó giúp ta dễ dàng gửi và nhận những thông điệp không vừa trong một gói tin. Sau khi thiết lập kết nối TCP, ta còn có thể gửi nhiều yêu cầu và phản hồi qua cùng kết nối đó. Cách làm thông thường là gửi một header cho biết thông điệp theo sau dài bao nhiêu byte, rồi mới gửi chính thông điệp. HTTP và nhiều giao thức RPC (xem ["Luồng dữ liệu qua dịch vụ: REST và RPC"](/ch5#sec_encoding_dataflow_rpc)) hoạt động như vậy.

### Lỗi mạng trong thực tế {#sec_distributed_network_faults}

Ta đã xây dựng mạng máy tính từ nhiều thập kỷ nay, lẽ ra đã tìm ra cách làm cho mạng đáng tin cậy từ lâu. Tiếc là đến giờ ta vẫn chưa thành công. Các nghiên cứu có hệ thống lẫn nhiều bằng chứng giai thoại đều cho thấy, ngay cả trong môi trường được kiểm soát như trung tâm dữ liệu do một công ty vận hành, sự cố mạng cũng có thể thường xuyên đến bất ngờ: [^8]

* Một nghiên cứu về trung tâm dữ liệu cỡ vừa cho thấy mỗi tháng xảy ra khoảng 12 sự cố mạng, trong đó một nửa làm đứt kết nối một máy, nửa còn lại làm đứt cả một rack [^9].
* Một nghiên cứu khác đo tỷ lệ hỏng của các thành phần như switch top-of-rack, switch tổng hợp và bộ cân bằng tải [^10]. Nghiên cứu thấy rằng thêm thiết bị mạng dự phòng không giảm sự cố nhiều như kỳ vọng, vì nó không chống được lỗi do con người (như cấu hình switch sai), vốn là nguyên nhân chính gây gián đoạn dịch vụ.
* Các vụ đứt đường truyền cáp quang diện rộng từng bị quy cho bò [^11], hải ly [^12] và cá mập [^13] (dù nhờ cáp biển được bảo vệ tốt hơn, các vụ cá mập cắn cáp ngày càng ít [^14]). Dĩ nhiên con người cũng không vô can: cấu hình sai [^15], trộm cắp cáp [^16] và phá hoại [^17] đều từng gây sự cố.
* Với liên lạc giữa các vùng đám mây khác nhau, thời gian round-trip ở các phân vị cao có thể lên tới vài *phút* [^18]. Ngay trong một trung tâm dữ liệu, gói tin cũng có thể bị trễ hơn một phút khi nâng cấp phần mềm switch gặp vấn đề và kích hoạt cấu hình lại topology mạng [^19]. Vì vậy ta phải giả định rằng thông điệp có thể bị trễ lâu tùy ý.
* Đôi khi liên lạc chỉ bị đứt một phần, tùy thuộc hai bên đang giao tiếp là ai. Chẳng hạn A và B liên lạc được, B và C liên lạc được, nhưng A và C thì không [^20] [^21]. Có những lỗi còn bất ngờ hơn, ví dụ một card mạng đôi khi bỏ mọi gói tin đến nhưng vẫn gửi gói tin đi bình thường: [^22] đường truyền chạy được ở một chiều không bảo đảm chiều ngược lại cũng chạy được.
* Ngay cả khi mạng chỉ gián đoạn trong thời gian rất ngắn, hệ quả của nó có thể kéo dài hơn nhiều so với sự cố ban đầu [^8] [^20] [^23].


> [!TIP] Network partition
>
> Khi một phần của mạng bị cắt khỏi phần còn lại do sự cố mạng, tình huống này đôi khi được gọi là *network partition* hoặc *netsplit*, nhưng về bản chất nó không khác các loại gián đoạn mạng khác. Network partition không liên quan đến việc shard hóa một hệ thống lưu trữ, vốn đôi khi cũng gọi là *partition* (xem [Chương 7](/ch7#ch_sharding)).


Ngay cả khi môi trường của bạn hiếm khi gặp sự cố mạng, việc sự cố *có thể* xảy ra đồng nghĩa với việc phần mềm phải xử lý được chúng. Hễ giao tiếp qua mạng là có thể thất bại — không tránh được điều đó.

Nếu cách xử lý sự cố mạng không được định nghĩa và kiểm thử rõ ràng, hậu quả có thể tệ không đáy. Chẳng hạn, cluster có thể rơi vào deadlock và vĩnh viễn không phục vụ được yêu cầu nữa, kể cả khi mạng đã hồi phục [^24], thậm chí còn có thể xóa sạch dữ liệu của bạn [^25]. Khi phần mềm rơi vào tình huống mà người thiết kế không lường trước, nó có thể làm bất cứ điều gì bất ngờ.

Xử lý sự cố mạng không nhất thiết là *chịu đựng* nó. Nếu mạng thường khá đáng tin cậy, hiển thị thông báo lỗi cho người dùng khi mạng có vấn đề cũng có thể là chiến lược hợp lý. Tuy nhiên, bạn phải biết phần mềm sẽ phản ứng thế nào với sự cố mạng và bảo đảm hệ thống phục hồi được. Có thể cố tình gây ra sự cố mạng để kiểm thử xem hệ thống phản ứng ra sao (đây gọi là *chèn lỗi* (*fault injection*), xem ["Chèn lỗi"](/ch9#sec_fault_injection)).

### Phát hiện lỗi {#id307}

Nhiều hệ thống cần tự động phát hiện node bị hỏng. Ví dụ:

* Bộ cân bằng tải cần ngừng gửi yêu cầu đến các node đã chết (tức là *loại nó khỏi vòng xoay*, take out of rotation).
* Trong cơ sở dữ liệu phân tán dùng sao chép đơn leader (*single-leader replication*), nếu leader hỏng thì cần nâng một follower lên làm leader mới (xem ["Xử lý sự cố node"](/ch6#sec_replication_failover)).

Tiếc là sự bất định của mạng khiến rất khó biết một node còn hoạt động hay không. Trong một số trường hợp cụ thể, bạn có thể nhận được phản hồi rõ ràng cho biết có chỗ nào đó hỏng:

* Nếu máy chạy node vẫn truy cập được nhưng không có tiến trình nào lắng nghe ở cổng đích (ví dụ tiến trình đã sập), hệ điều hành sẽ đóng hoặc từ chối kết nối TCP bằng gói `RST` hoặc `FIN`.
* Nếu tiến trình của node sập (hoặc bị quản trị viên kill) nhưng hệ điều hành của node vẫn chạy, một script có thể thông báo sự cố cho các node khác để node khác tiếp quản nhanh chóng mà không phải chờ timeout. Ví dụ HBase làm như vậy [^26].
* Nếu có quyền truy cập giao diện quản trị của các switch mạng trong trung tâm dữ liệu, bạn có thể truy vấn switch để phát hiện lỗi đường truyền ở mức phần cứng (ví dụ máy ở xa đã mất điện chưa). Cách này không dùng được nếu bạn kết nối qua Internet, nếu bạn ở trong một trung tâm dữ liệu dùng chung mà không truy cập được chính các switch, hoặc nếu sự cố mạng khiến cả giao diện quản trị cũng không truy cập được.
* Nếu router biết chắc địa chỉ IP bạn định kết nối không truy cập được, nó có thể trả về gói ICMP "Destination Unreachable". Nhưng router cũng không có khả năng phát hiện lỗi thần kỳ nào — nó chịu cùng những hạn chế như mọi thành phần khác trong mạng.

Biết nhanh một node ở xa đã chết là hữu ích, nhưng không thể trông chờ lúc nào cũng có phản hồi như vậy. Khi có sự cố, bạn có thể nhận phản hồi lỗi từ một tầng nào đó của ngăn xếp giao thức, nhưng thường phải giả định là mình sẽ không nhận được gì. Bạn có thể thử lại vài lần, chờ timeout, và nếu trong thời gian timeout không có phản hồi thì cuối cùng tuyên bố node đã chết.

### Timeout và độ trễ không giới hạn {#sec_distributed_queueing}

Nếu timeout là cách đáng tin duy nhất để phát hiện lỗi, thì nên đặt timeout bao lâu? Tiếc là không có câu trả lời đơn giản.

Timeout dài nghĩa là phải chờ lâu mới tuyên bố một node đã chết (trong thời gian đó, người dùng có thể phải chờ hoặc liên tục thấy thông báo lỗi). Timeout ngắn phát hiện lỗi nhanh hơn, nhưng dễ tuyên bố nhầm một node chỉ đang chậm tạm thời (ví dụ do node hoặc mạng tăng tải đột ngột) là đã chết.

Tuyên bố node chết quá sớm là rắc rối: nếu node thật ra vẫn sống và đang thực hiện một thao tác (chẳng hạn gửi email), mà node khác lại tiếp quản công việc của nó, thì thao tác đó có thể bị thực hiện hai lần. Ta sẽ bàn chi tiết hơn về vấn đề này ở ["Tri thức, sự thật và lời nói dối"](/ch9#sec_distributed_truth), [Chương 10](/ch10#ch_consistency) và ["Nguyên lý end-to-end của cơ sở dữ liệu"](/ch13#sec_future_end_to_end).

Khi một node bị tuyên bố đã chết, trách nhiệm của nó phải chuyển sang các node khác, làm tăng tải cho các node đó và cho mạng. Nếu hệ thống vốn đã đang gồng mình dưới tải cao, việc tuyên bố node chết quá sớm có thể làm tình hình tệ hơn. Đặc biệt, có thể xảy ra tình huống node thật ra chưa chết, chỉ phản hồi chậm vì quá tải. Khi đó, chuyển tải của nó sang các node khác lại có thể gây lỗi dây chuyền (trong trường hợp cực đoan, mọi node đều tuyên bố nhau đã chết và cả hệ thống ngừng hoạt động hoàn toàn — xem ["Khi hệ thống quá tải không thể phục hồi"](/ch2#sidebar_metastable)).

Hãy hình dung một hệ thống giả tưởng có mạng bảo đảm độ trễ tối đa của gói tin: mỗi gói tin hoặc được chuyển đến trong vòng thời gian *d*, hoặc bị mất, không bao giờ đến sau quá *d*. Giả sử ta cũng có thể bảo đảm rằng bất kỳ node nào chưa hỏng đều xử lý xong một yêu cầu trong vòng thời gian *r*. Khi đó, mọi yêu cầu thành công đều được bảo đảm nhận phản hồi trong vòng 2*d* + *r*, và nếu quá thời gian đó vẫn chưa có phản hồi, ta biết chắc mạng hoặc node ở xa đang không hoạt động bình thường. Nếu các bảo đảm này thật sự đúng thì 2*d* + *r* sẽ là timeout hợp lý.

Tiếc là hầu hết các hệ thống ta dùng đều không có bảo đảm nào trong số trên: mạng bất đồng bộ có *độ trễ không giới hạn* (*unbounded delay*), tức là mạng cố gắng chuyển gói tin nhanh nhất có thể nhưng thời gian cần để gói tin đến không có cận trên, và phần lớn các bản hiện thực server cũng không thể bảo đảm xử lý xong yêu cầu trong một thời gian tối đa nào đó (xem ["Bảo đảm thời gian phản hồi"](/ch9#sec_distributed_clocks_realtime)). Với việc phát hiện lỗi, chỉ chạy nhanh ở phần lớn thời gian là chưa đủ: nếu đặt timeout thấp, một đợt tăng vọt ngắn của thời gian round-trip cũng đủ làm hệ thống rối loạn.

<a id="sec_distributed_congestion"></a>

#### Tắc nghẽn mạng và xếp hàng {#network-congestion-and-queueing}

Khi lái xe, kẹt xe thường là yếu tố lớn nhất khiến thời gian di chuyển dao động. Tương tự, trên mạng máy tính, sự biến thiên của độ trễ gói tin thường do xếp hàng gây ra: [^27]

* Nếu nhiều node cùng gửi gói tin đến một đích, switch mạng phải xếp chúng vào hàng đợi rồi đưa lần lượt vào đường truyền tới đích (như minh họa ở {{< xref fig="9-2" page="/ch9" anchor="fig_distributed_switch_queueing" >}}Hình 9-2{{< /xref >}}). Khi đường truyền bận, gói tin có thể phải chờ một lúc mới được gửi. Hiện tượng này gọi là *tắc nghẽn mạng* (*network congestion*). Nếu dữ liệu đến quá nhiều, hàng đợi của switch đầy và gói tin bị bỏ, phải gửi lại — dù mạng vẫn đang hoạt động bình thường.
* Khi gói tin đến máy đích, nếu mọi lõi CPU đều đang bận, hệ điều hành xếp yêu cầu từ mạng vào hàng đợi cho đến khi ứng dụng sẵn sàng xử lý. Thời gian chờ tùy thuộc tải của máy, có thể dài tùy ý [^28].
* Trong môi trường ảo hóa, khi một máy ảo khác chiếm một lõi CPU, hệ điều hành đang chạy thường bị tạm dừng vài chục mili giây. Trong thời gian đó, máy ảo không tiêu thụ được dữ liệu mạng nào, nên hypervisor xếp hàng (đệm) dữ liệu đến [^29], làm độ trễ mạng càng dao động mạnh.
* Như đã nói, để tránh làm mạng quá tải, TCP giới hạn tốc độ gửi dữ liệu. Nghĩa là dữ liệu còn phải xếp hàng thêm một lần ở bên gửi trước khi vào mạng.

{{< fig num="9-2" id="fig_distributed_switch_queueing" src="/fig/ddia_0902.png" caption="Khi nhiều máy gửi lưu lượng mạng đến cùng một đích, hàng đợi của switch có thể bị lấp đầy. Ở đây, các cổng 1, 2 và 4 đều đang cố gửi gói tin đến cổng 3." class="ddia-figure ddia-figure--wide" width="2880" height="1081" />}}

Ngoài ra, khi TCP phát hiện gói tin bị mất và tự động truyền lại, ứng dụng tuy không trực tiếp thấy việc mất gói nhưng vẫn cảm nhận được độ trễ do nó gây ra: trước hết chờ timeout, rồi chờ gói được truyền lại được xác nhận.


<a id="sidebar_distributed_tcp_udp"></a>

> [!TIP] TCP và UDP
>
> Một số ứng dụng nhạy cảm với độ trễ, như hội nghị truyền hình và thoại qua IP (VoIP), dùng UDP thay vì TCP. Đây là sự đánh đổi giữa độ tin cậy và độ dao động của độ trễ: UDP không kiểm soát luồng và không truyền lại gói tin bị mất, nên tránh được một phần nguyên nhân khiến độ trễ mạng dao động (tuy nhiên nó vẫn chịu ảnh hưởng của việc xếp hàng ở switch và độ trễ lập lịch).
>
> UDP là lựa chọn tốt khi dữ liệu trễ thì không còn giá trị. Chẳng hạn trong cuộc gọi VoIP, khi đến lúc phát đoạn âm thanh ra loa thì thường đã quá muộn để truyền lại gói bị mất. Khi đó, truyền lại là vô nghĩa — ứng dụng chỉ có thể điền khoảng im lặng vào khoảng thời gian của gói bị mất (nghe như âm thanh ngắt quãng một chút) rồi phát tiếp phần sau. Việc "thử lại" thực sự diễn ra ở tầng con người. ("Bạn nói lại được không? Vừa nãy bị ngắt tiếng.")


Mọi yếu tố trên đều góp phần làm độ trễ mạng dao động. Khi hệ thống gần đạt công suất tối đa, biên độ biến thiên của độ trễ xếp hàng đặc biệt lớn: hệ thống còn nhiều dư địa dễ dàng xả hết hàng đợi, còn hệ thống có mức sử dụng cao có thể nhanh chóng dồn thành hàng đợi dài.

Trong đám mây công cộng và các trung tâm dữ liệu đa người thuê, nhiều khách hàng dùng chung tài nguyên: đường truyền và switch mạng được dùng chung, thậm chí cả card mạng và CPU của từng máy (khi dùng máy ảo) cũng dùng chung. Khi xử lý lượng dữ liệu lớn, có thể dùng hết dung lượng của đường truyền, khiến nó đạt trạng thái *bão hòa* (*saturation*). Bạn không thể kiểm soát hay biết khách hàng khác dùng tài nguyên chung ra sao, nên nếu ai đó ở gần bạn (một *noisy neighbor*) đang tiêu thụ nhiều tài nguyên, độ trễ mạng có thể dao động dữ dội [^30] [^31].

Trong môi trường như vậy, chỉ có thể chọn timeout bằng thực nghiệm: đo phân phối thời gian round-trip của mạng giữa nhiều máy trong thời gian dài để xác định độ trễ thường dao động bao nhiêu. Sau đó, dựa trên đặc điểm của ứng dụng, xác định sự đánh đổi phù hợp giữa độ trễ phát hiện lỗi và rủi ro timeout sớm.

Tốt hơn nữa là không dùng timeout cấu hình cố định, mà để hệ thống liên tục đo thời gian phản hồi và độ dao động của nó (*jitter*), rồi tự điều chỉnh timeout theo phân phối thời gian phản hồi quan sát được. Phi accrual failure detector [^32] dùng cách này, và được các hệ thống như Akka và Cassandra sử dụng [^33]. Timeout truyền lại của TCP cũng hoạt động theo cách tương tự [^5].

### Mạng đồng bộ và mạng bất đồng bộ {#sec_distributed_sync_networks}

Nếu có thể trông cậy mạng chuyển gói tin trong một độ trễ tối đa cố định và không bao giờ mất gói, hệ thống phân tán sẽ đơn giản hơn nhiều. Tại sao không giải quyết vấn đề này ở tầng phần cứng, làm cho mạng đáng tin cậy để phần mềm khỏi phải bận tâm?

Để trả lời, hãy so sánh mạng trung tâm dữ liệu với mạng điện thoại cố định truyền thống (không phải di động, cũng không phải VoIP). Mạng điện thoại truyền thống cực kỳ đáng tin cậy: khung âm thanh bị trễ hay cuộc gọi bị rớt rất hiếm. Cuộc gọi cần độ trễ đầu cuối thấp ổn định và đủ băng thông để truyền mẫu giọng nói. Nếu mạng máy tính cũng có độ tin cậy và khả năng dự đoán như vậy thì tốt biết bao?

Khi gọi điện qua mạng điện thoại, mạng thiết lập một *circuit*: trên toàn bộ đường đi từ người gọi này đến người gọi kia, một lượng băng thông cố định và được bảo đảm được cấp riêng cho cuộc gọi. Circuit này được giữ đến khi cuộc gọi kết thúc [^34]. Ví dụ, mạng ISDN chạy ở tốc độ cố định 4.000 khung mỗi giây. Khi thiết lập cuộc gọi, cuộc gọi được cấp 16 bit chỗ trong mỗi khung (ở mỗi chiều). Vì vậy, trong suốt cuộc gọi, mỗi bên được bảo đảm gửi đúng 16 bit dữ liệu âm thanh mỗi 250 micro giây [^35].

Mạng kiểu này là *đồng bộ* (*synchronous*): dù dữ liệu đi qua nhiều router, nó cũng không phải xếp hàng, vì mỗi chặng đã dành sẵn 16 bit cho cuộc gọi. Vì không cần xếp hàng nên độ trễ đầu cuối tối đa của mạng là cố định. Ta gọi đó là *độ trễ có giới hạn* (*bounded delay*).

#### Sao không làm cho độ trễ mạng dự đoán được? {#can-we-not-simply-make-network-delays-predictable}

Lưu ý circuit trong mạng điện thoại rất khác kết nối TCP: circuit dành riêng một lượng băng thông cố định, người khác không dùng được phần đó khi circuit còn tồn tại, còn gói tin của kết nối TCP tận dụng bất cứ băng thông mạng nào đang rảnh. Bạn có thể đưa cho TCP một khối dữ liệu có kích thước bất kỳ (như email hay trang web) và nó sẽ cố truyền xong trong thời gian ngắn nhất. Khi kết nối TCP nhàn rỗi, nó không chiếm băng thông, trừ các gói keepalive thỉnh thoảng gửi.

Nếu mạng trung tâm dữ liệu và Internet dùng chuyển mạch kênh, ta có thể xác định một thời gian round-trip tối đa được bảo đảm ngay khi thiết lập circuit. Nhưng chúng không như vậy: Ethernet và IP là các giao thức chuyển mạch gói, gói tin phải xếp hàng nên độ trễ mạng không có cận trên. Trong các giao thức này hoàn toàn không có khái niệm "circuit".

Vì sao mạng trung tâm dữ liệu và Internet dùng chuyển mạch gói? Vì chúng được tối ưu cho *lưu lượng đột biến* (*bursty traffic*). Cuộc gọi âm thanh hay video truyền một lượng bit mỗi giây khá ổn định trong suốt cuộc gọi, rất hợp với circuit. Ngược lại, yêu cầu một trang web, gửi email hay truyền tệp không có yêu cầu băng thông cụ thể — ta chỉ muốn chúng hoàn tất càng nhanh càng tốt.

Nếu truyền tệp qua circuit, bạn phải đoán trước cần cấp bao nhiêu băng thông. Đoán thấp thì truyền chậm không cần thiết, lại còn bỏ phí dung lượng mạng. Đoán cao thì không thiết lập được circuit, vì mạng không cho phép tạo circuit khi không bảo đảm được phần băng thông đã cấp. Vì vậy, dùng circuit cho dữ liệu đột biến vừa lãng phí dung lượng mạng vừa làm việc truyền chậm đi vô ích. TCP thì khác: nó điều chỉnh động tốc độ truyền dữ liệu theo dung lượng mạng đang có.

Cũng đã có những nỗ lực xây dựng mạng lai kết hợp cả chuyển mạch kênh và chuyển mạch gói. *Asynchronous Transfer Mode* (ATM) từng cạnh tranh với Ethernet vào thập niên 1980, nhưng ngoài các switch lõi của mạng điện thoại, nó chưa bao giờ được dùng rộng rãi. InfiniBand có phần tương tự: [^36] nó hiện thực kiểm soát luồng đầu cuối ở tầng liên kết, giảm nhu cầu xếp hàng trong mạng, dù tắc nghẽn trên đường truyền vẫn có thể gây trễ [^37]. Nếu vận dụng cẩn thận *chất lượng dịch vụ* (*quality of service*, QoS, tức độ ưu tiên và lập lịch gói tin) và *kiểm soát tiếp nhận* (*admission control*, tức giới hạn tốc độ của bên gửi), ta có thể mô phỏng chuyển mạch kênh trên mạng chuyển mạch gói, hoặc cung cấp độ trễ có giới hạn về mặt thống kê [^27] [^34]. Các thuật toán mạng mới như Low Latency, Low Loss, and Scalable Throughput (L4S) cố giải quyết một phần vấn đề xếp hàng và kiểm soát tắc nghẽn từ cả phía client lẫn router. Traffic controller (TC) của Linux cũng cho phép ứng dụng sắp xếp lại độ ưu tiên gói tin để đạt QoS.


<a id="sidebar_distributed_latency_utilization"></a>

> [!TIP] Độ trễ và mức sử dụng tài nguyên
>
> Nói chung, có thể xem sự dao động của độ trễ là kết quả của việc phân chia tài nguyên một cách động.
>
> Giả sử có một đường dây giữa hai tổng đài điện thoại, chứa được tối đa 10.000 cuộc gọi đồng thời. Mỗi circuit được chuyển qua đường dây này chiếm một khe cuộc gọi. Vì vậy có thể coi đường dây là một tài nguyên được tối đa 10.000 người dùng đồng thời chia sẻ. Tài nguyên được chia theo cách *tĩnh*: dù lúc này cả đường dây chỉ có mỗi cuộc gọi của bạn, và 9.999 khe còn lại đều trống, circuit của bạn vẫn chỉ nhận phần băng thông cố định đó, y hệt lúc đường dây đầy tải.
>
> Ngược lại, Internet chia sẻ băng thông mạng theo cách *động*. Các bên gửi chen lấn nhau để đưa gói tin của mình lên đường dây nhanh nhất có thể, còn switch mạng quyết định tại mỗi thời điểm gửi gói nào tiếp theo, tức là cách phân bổ băng thông. Cách này có nhược điểm là gây xếp hàng, nhưng ưu điểm là tận dụng tối đa đường dây. Đường dây có chi phí cố định, nên mức sử dụng càng cao thì mỗi byte gửi qua nó càng rẻ.
>
> CPU cũng có tình huống tương tự: nếu nhiều thread chia sẻ động một lõi CPU thì đôi khi một thread phải chờ trong hàng đợi chạy của hệ điều hành trong lúc thread khác đang chạy, nên có thể bị tạm dừng trong những khoảng thời gian dài ngắn khác nhau [^38]. Tuy nhiên, so với việc cấp tĩnh cho mỗi thread một lượng chu kỳ CPU cố định, cách này tận dụng phần cứng tốt hơn (xem ["Bảo đảm thời gian phản hồi"](/ch9#sec_distributed_clocks_realtime)). Để nâng mức sử dụng phần cứng, nền tảng đám mây cũng chạy nhiều máy ảo của các khách hàng khác nhau trên cùng một máy vật lý.
>
> Trong một số môi trường, chỉ cần chia tĩnh tài nguyên (ví dụ dùng phần cứng chuyên dụng với băng thông độc quyền) là có thể bảo đảm độ trễ. Nhưng cái giá là mức sử dụng tài nguyên thấp hơn — nói cách khác là chi phí cao hơn. Mô hình đa người thuê chia tài nguyên động có mức sử dụng tốt hơn nên rẻ hơn, đổi lại độ trễ dao động.
>
> Độ trễ mạng dao động không phải quy luật tự nhiên, chỉ là kết quả của sự đánh đổi giữa chi phí và lợi ích.


Tuy nhiên, các trung tâm dữ liệu đa người thuê, đám mây công cộng và giao tiếp qua Internet hiện chưa bật các cơ chế chất lượng dịch vụ như vậy. Công nghệ triển khai hiện nay không cho phép ta bảo đảm bất cứ điều gì về độ trễ hay độ tin cậy của mạng: phải giả định tắc nghẽn mạng, xếp hàng và độ trễ không giới hạn đều sẽ xảy ra. Do đó, không có giá trị timeout "đúng", mà phải xác định bằng thực nghiệm.

Các thỏa thuận peering giữa nhà cung cấp dịch vụ Internet và cách thiết lập định tuyến qua Border Gateway Protocol (BGP) giống chuyển mạch kênh hơn so với chính IP. Ở tầng này, quả thật có thể mua băng thông dành riêng. Tuy nhiên, định tuyến Internet hoạt động ở cấp mạng chứ không phải cấp từng kết nối giữa các host, và trên thang thời gian dài hơn nhiều.

## Đồng hồ không đáng tin cậy {#sec_distributed_clocks}

Đồng hồ và thời gian rất quan trọng. Ứng dụng dựa vào đồng hồ theo đủ kiểu để trả lời những câu hỏi như:

1. Yêu cầu này đã timeout chưa?
2. Phân vị thứ 99 của thời gian phản hồi của dịch vụ này là bao nhiêu?
3. Trong năm phút qua, trung bình dịch vụ này xử lý bao nhiêu truy vấn mỗi giây?
4. Người dùng ở lại trang web của chúng ta bao lâu?
5. Bài viết này được đăng lúc nào?
6. Email nhắc nhở nên gửi vào ngày nào, giờ nào?
7. Mục cache này hết hạn khi nào?
8. Timestamp của thông báo lỗi này trong file log là gì?

Câu hỏi 1–4 đo một *khoảng thời gian* (*duration*), ví dụ khoảng cách từ lúc gửi yêu cầu đến lúc nhận phản hồi,, còn câu hỏi 5–8 mô tả một *thời điểm* (*point in time*), tức là sự kiện xảy ra vào một ngày giờ cụ thể.

Trong hệ thống phân tán, thời gian là chuyện rắc rối vì giao tiếp không diễn ra tức thời: một thông điệp đi từ máy này sang máy khác qua mạng cần thời gian. Thời điểm nhận thông điệp luôn muộn hơn thời điểm gửi, nhưng vì độ trễ mạng thay đổi nên ta không biết muộn hơn bao nhiêu. Khi có nhiều máy tham gia, sự thật này đôi khi khiến việc xác định thứ tự các sự kiện trở nên khó khăn.

Hơn nữa, mỗi máy trong mạng có đồng hồ riêng, và đó là một thiết bị phần cứng thực thụ, thường là bộ dao động tinh thể thạch anh. Các thiết bị này không thật chính xác, nên mỗi máy có một khái niệm riêng về thời gian, có thể chạy nhanh hơn hoặc chậm hơn đôi chút so với máy khác. Có thể đồng bộ đồng hồ ở một mức độ nhất định: cơ chế phổ biến nhất là Network Time Protocol (NTP), cho phép chỉnh đồng hồ máy tính theo thời gian do một nhóm server báo cáo [^39]. Các server này lại lấy thời gian từ nguồn chính xác hơn, như bộ thu GPS.

### Đồng hồ đơn điệu và đồng hồ giờ trong ngày {#sec_distributed_monotonic_timeofday}

Máy tính hiện đại có ít nhất hai loại đồng hồ khác nhau: *đồng hồ giờ trong ngày* (*time-of-day clock*) và *đồng hồ đơn điệu* (*monotonic clock*). Dù cả hai đều đo thời gian, chúng phục vụ những mục đích khác nhau nên cần phân biệt.

#### Đồng hồ giờ trong ngày {#time-of-day-clocks}

Đồng hồ giờ trong ngày làm đúng điều ta trực giác nghĩ đồng hồ phải làm: trả về ngày và giờ hiện tại theo một loại lịch nào đó (còn gọi là *wall-clock time*). Ví dụ, `clock_gettime(CLOCK_REALTIME)` của Linux và `System.currentTimeMillis()` của Java trả về số giây (hoặc mili giây) kể từ *epoch*: ở đây epoch là nửa đêm UTC ngày 1 tháng 1 năm 1970 theo lịch Gregory, không tính giây nhuận. Một số hệ thống dùng mốc tham chiếu khác. (Linux gọi đồng hồ này là đồng hồ *real-time*, nhưng nó chẳng liên quan gì đến hệ điều hành thời gian thực, xem ["Bảo đảm thời gian phản hồi"](/ch9#sec_distributed_clocks_realtime).)

Đồng hồ giờ trong ngày thường được đồng bộ với NTP, nên lý tưởng là một timestamp trên máy này biểu diễn cùng thời điểm với timestamp đó trên máy khác. Tuy nhiên, đồng hồ giờ trong ngày cũng có nhiều điểm kỳ quặc, sẽ nói thêm ở phần sau. Đặc biệt, nếu đồng hồ cục bộ chạy nhanh hơn server NTP quá nhiều, nó có thể bị ép đặt lại và trông như nhảy ngược về quá khứ. Những bước nhảy như vậy, cùng những bước nhảy tương tự do giây nhuận gây ra, khiến đồng hồ giờ trong ngày không phù hợp để đo thời gian đã trôi qua [^40].

Đồng hồ giờ trong ngày cũng có thể nhảy khi giờ mùa hè (DST) bắt đầu hoặc kết thúc, nhưng có thể tránh loại bước nhảy này bằng cách luôn dùng múi giờ UTC, vốn không có DST. Trước đây, độ phân giải của đồng hồ giờ trong ngày cũng khá thô, chẳng hạn đồng hồ của các phiên bản Windows cũ mỗi lần nhảy tới 10 mili giây [^41]. Trên các hệ thống mới hơn, đây không còn là vấn đề lớn.

#### Đồng hồ đơn điệu {#monotonic-clocks}

Đồng hồ đơn điệu phù hợp để đo khoảng thời gian (khoảng cách giữa hai thời điểm), chẳng hạn timeout hay thời gian phản hồi của dịch vụ. `clock_gettime(CLOCK_MONOTONIC)`, `clock_gettime(CLOCK_BOOTTIME)` [^42] của Linux và `System.nanoTime()` của Java đều là đồng hồ đơn điệu. Gọi là "đơn điệu" vì nó bảo đảm chỉ chạy tiến (còn đồng hồ giờ trong ngày có thể đột ngột nhảy ngược về quá khứ).

Bạn có thể đọc đồng hồ đơn điệu một lần, làm vài việc, rồi vài giây sau đọc lại. *Hiệu* của hai giá trị cho biết thời gian đã trôi qua — nó giống đồng hồ bấm giờ hơn là đồng hồ treo tường. Tuy nhiên, giá trị *tuyệt đối* của đồng hồ đơn điệu không có ý nghĩa gì: nó có thể là số nano giây kể từ lúc máy khởi động, hoặc tính từ một mốc tùy ý nào đó. Đặc biệt, không thể so sánh giá trị đồng hồ đơn điệu của hai máy khác nhau vì chúng không biểu diễn cùng một thứ.

Trên server có nhiều socket CPU, mỗi CPU có thể có bộ đếm thời gian riêng và chưa chắc đồng bộ với các CPU khác [^43]. Hệ điều hành bù trừ sự chênh lệch này và cố làm cho thread ứng dụng thấy đồng hồ tăng đơn điệu, kể cả khi thread được lập lịch qua lại giữa các CPU. Tuy vậy, nên dè dặt với bảo đảm tính đơn điệu này [^44].

Nếu NTP phát hiện đồng hồ thạch anh cục bộ chạy nhanh hơn hoặc chậm hơn server NTP, nó có thể điều chỉnh tốc độ đồng hồ đơn điệu tiến lên, việc này gọi là *slewing* (hiệu chỉnh dần) đồng hồ. Mặc định, NTP được phép tăng hoặc giảm tốc độ đồng hồ tối đa 0,05%, nhưng không thể làm đồng hồ đơn điệu nhảy tiến hay lùi đột ngột. Đồng hồ đơn điệu thường có độ phân giải tốt: trên hầu hết hệ thống, nó đo được khoảng thời gian cỡ micro giây hoặc nhỏ hơn.

Trong hệ thống phân tán, dùng đồng hồ đơn điệu để đo thời gian trôi qua (ví dụ timeout) thường không sao, vì nó không đòi hỏi đồng hồ của các node phải đồng bộ với nhau, và sai số nhỏ trong phép đo cũng không gây ảnh hưởng lớn.

### Đồng bộ đồng hồ và độ chính xác {#sec_distributed_clock_accuracy}

Đồng hồ đơn điệu không cần đồng bộ, nhưng đồng hồ giờ trong ngày chỉ hữu ích khi được hiệu chỉnh theo server NTP hoặc nguồn thời gian bên ngoài khác. Tiếc là cách làm cho đồng hồ chạy đúng giờ kém tin cậy và kém chính xác hơn nhiều so với tưởng tượng — cả đồng hồ phần cứng lẫn NTP đều có thể thất thường. Dưới đây là vài ví dụ:

* Đồng hồ thạch anh trong máy tính không thật chính xác: nó có *độ trôi* (*drift*), tức là chạy nhanh hơn hoặc chậm hơn so với tốc độ đúng, và độ trôi này thay đổi theo nhiệt độ của máy. Google giả định đồng hồ server của họ trôi tối đa 200 ppm (hai trăm phần triệu) [^45]. Tức là đồng hồ đồng bộ lại với server mỗi 30 giây sẽ trôi 6 mili giây, còn đồng hồ chỉ đồng bộ mỗi ngày một lần sẽ trôi 17 giây. Dù mọi thứ khác đều ổn, độ trôi của đồng hồ vẫn giới hạn độ chính xác tốt nhất có thể đạt được.
* Nếu đồng hồ máy tính chênh với server NTP quá nhiều, nó có thể từ chối đồng bộ, hoặc đồng hồ cục bộ bị ép đặt lại [^39]. Ứng dụng quan sát thời gian trước và sau khi đặt lại có thể thấy thời gian lùi về sau hoặc nhảy vọt về phía trước.
* Nếu tường lửa vô tình chặn liên lạc giữa node và server NTP, lỗi cấu hình này có thể không ai phát hiện trong thời gian dài. Trong lúc đó, độ trôi có thể tích tụ thành độ lệch lớn giữa đồng hồ của các node. Có bằng chứng giai thoại cho thấy chuyện này thật sự đã xảy ra.
* Độ chính xác của đồng bộ NTP không thể tốt hơn độ trễ mạng, nên độ chính xác bị hạn chế trên mạng tắc nghẽn có độ trễ gói tin dao động. Một thí nghiệm cho thấy sai số tối thiểu khi đồng bộ qua Internet là 35 mili giây [^46], và khi độ trễ mạng thỉnh thoảng tăng vọt, sai số đạt khoảng một giây. Tùy cấu hình, độ trễ mạng quá lớn thậm chí có thể khiến client NTP bỏ hẳn việc đồng bộ.
* Một số server NTP sai hoặc cấu hình sai, báo cáo thời gian lệch hàng giờ [^47] [^48]. Client NTP truy vấn nhiều server và loại bỏ giá trị ngoại lai để giảm nhẹ các lỗi này. Dù vậy, đặt tính đúng đắn của hệ thống vào thời gian do một người lạ trên Internet báo cho bạn vẫn khiến người ta có phần bất an.
* Giây nhuận làm một phút dài 59 hoặc 61 giây, phá vỡ các giả định về thời gian của những hệ thống không được thiết kế với giây nhuận [^49]. Giây nhuận đã làm sập nhiều hệ thống lớn [^40] [^50], cho thấy giả định sai về đồng hồ dễ lọt vào hệ thống một cách âm thầm đến mức nào. Cách tốt nhất để xử lý giây nhuận có lẽ là bắt server NTP "nói dối": dàn đều việc điều chỉnh giây nhuận ra cả ngày, gọi là *smearing* [^51] [^52], tuy trong thực tế hành vi của các server NTP không nhất quán [^53]. May là từ năm 2035 sẽ không còn dùng giây nhuận nữa, và vấn đề này sẽ biến mất.
* Trong máy ảo, đồng hồ phần cứng cũng được ảo hóa, gây thêm thách thức cho ứng dụng cần đo thời gian chính xác [^54]. Khi nhiều máy ảo dùng chung lõi CPU, một máy ảo chạy thì các máy ảo khác có thể bị tạm dừng hàng chục mili giây. Từ góc nhìn ứng dụng, khoảng dừng này biểu hiện như đồng hồ đột ngột nhảy tiến [^29]. Nếu máy ảo bị dừng vài giây, đồng hồ của nó sau đó có thể chậm vài giây so với thời gian thực, nhưng NTP vẫn có thể báo đồng hồ gần như đồng bộ hoàn toàn [^55].
* Nếu phần mềm chạy trên thiết bị mà bạn không kiểm soát hoàn toàn (như thiết bị di động hay thiết bị nhúng), đồng hồ phần cứng của thiết bị có thể hoàn toàn không đáng tin. Một số người dùng cố tình đặt đồng hồ phần cứng sai ngày giờ, chẳng hạn để gian lận trong game [^56]. Do đó, đồng hồ có thể bị đặt về quá khứ hoặc tương lai vô lý.

Nếu đủ coi trọng độ chính xác của đồng hồ và sẵn sàng đầu tư nhiều tài nguyên, hoàn toàn có thể đạt độ chính xác rất cao. Chẳng hạn, quy định MiFID II của châu Âu dành cho các tổ chức tài chính yêu cầu mọi quỹ giao dịch tần suất cao giữ đồng hồ sai lệch với UTC trong vòng 100 micro giây, để phục vụ việc điều tra các bất thường thị trường như "flash crash" và giúp phát hiện thao túng thị trường [^57].

Mức chính xác này đạt được nhờ phần cứng chuyên dụng (bộ thu GPS và/hoặc đồng hồ nguyên tử), Precision Time Protocol (PTP), cùng việc triển khai và giám sát cẩn thận [^58] [^59]. Tuy nhiên, chỉ dựa vào GPS cũng có rủi ro, vì tín hiệu GPS rất dễ bị gây nhiễu. Ở một số nơi (chẳng hạn gần các cơ sở quân sự) việc này xảy ra thường xuyên [^60]. Một số nhà cung cấp đám mây đã bắt đầu cung cấp đồng bộ đồng hồ độ chính xác cao cho máy ảo [^61]. Dù vậy, đồng bộ đồng hồ vẫn cần hết sức cẩn thận. Nếu daemon NTP cấu hình sai, hoặc tường lửa chặn lưu lượng NTP, sai số đồng hồ do độ trôi sẽ nhanh chóng trở nên lớn.

### Phụ thuộc vào đồng hồ đồng bộ {#sec_distributed_clocks_relying}

Vấn đề của đồng hồ là nó trông đơn giản, dễ dùng, nhưng lại đầy cạm bẫy đến bất ngờ: một ngày không nhất thiết có đúng 86.400 giây, đồng hồ giờ trong ngày có thể chạy ngược, và thời gian trên node này có thể chênh rất xa so với node kia.

Trước đó trong chương này ta đã bàn về việc mạng mất gói và gói tin bị trễ tùy ý. Dù mạng phần lớn thời gian chạy tốt, khi thiết kế phần mềm vẫn phải giả định mạng thỉnh thoảng sẽ hỏng và xử lý các lỗi đó cho ổn thỏa. Đồng hồ cũng vậy: phần lớn thời gian chúng chạy tốt, nhưng phần mềm vững chắc phải sẵn sàng đối phó với đồng hồ sai.

Một phần vấn đề là đồng hồ sai rất dễ không ai nhận ra. Nếu CPU của máy bị lỗi hoặc mạng cấu hình sai, nhiều khả năng nó sẽ hoàn toàn không chạy được, nên vấn đề nhanh chóng bị phát hiện và sửa. Ngược lại, nếu đồng hồ thạch anh bị lỗi hoặc client NTP cấu hình sai, thì dù đồng hồ ngày càng lệch xa thực tế, hầu hết mọi thứ vẫn trông như đang chạy bình thường. Nếu một đoạn phần mềm dựa vào đồng hồ đồng bộ chính xác, kết quả nhiều khả năng là mất dữ liệu âm thầm và tinh vi, chứ không phải một vụ sập ầm ĩ [^62] [^63].

Vì vậy, nếu dùng phần mềm đòi hỏi đồng hồ đồng bộ, bạn phải giám sát cẩn thận độ lệch đồng hồ giữa tất cả các máy. Node nào có đồng hồ lệch quá xa các node khác nên bị tuyên bố đã chết và loại khỏi cluster. Giám sát như vậy bảo đảm phát hiện được đồng hồ hỏng trước khi nó gây quá nhiều thiệt hại.

#### Timestamp để sắp xếp sự kiện {#sec_distributed_lww}

Hãy xét một tình huống rất dễ khiến người ta muốn dựa vào đồng hồ nhưng lại rất nguy hiểm: sắp xếp thứ tự các sự kiện trên nhiều node [^64]. Ví dụ, nếu hai client cùng ghi vào một cơ sở dữ liệu phân tán, ai đến trước? Lần ghi nào mới hơn?

{{< xref fig="9-3" page="/ch9" anchor="fig_distributed_timestamps" >}}Hình 9-3{{< /xref >}} minh họa một cách dùng nguy hiểm đồng hồ giờ trong ngày trong cơ sở dữ liệu dùng sao chép đa leader (*multi-leader replication*) (ví dụ này tương tự {{< xref fig="6-8" page="/ch6" anchor="fig_replication_causality" >}}Hình 6-8{{< /xref >}}). Client A ghi *x* = 1 trên node 1, lần ghi này được truyền sang node 3, client B tăng *x* trên node 3 (lúc này *x* = 2), và cuối cùng cả hai lần ghi đều được truyền sang node 2.

{{< fig num="9-3" id="fig_distributed_timestamps" src="/fig/ddia_0903.png" caption="Lần ghi của client B xảy ra sau lần ghi của client A theo quan hệ nhân quả, nhưng lại có timestamp sớm hơn." class="ddia-figure ddia-figure--wide" width="2880" height="1325" />}}


Trong {{< xref fig="9-3" page="/ch9" anchor="fig_distributed_timestamps" >}}Hình 9-3{{< /xref >}}, khi một lần ghi được truyền sang các node khác, nó được gắn timestamp theo đồng hồ giờ trong ngày của node nơi lần ghi khởi nguồn. Trong ví dụ này, đồng bộ đồng hồ rất tốt: độ lệch giữa node 1 và node 3 dưới 3 mili giây, mức mà trong thực tế có lẽ khó đạt được.

Thao tác tăng được xây dựng trên lần ghi *x* = 1 trước đó, nên ta tự nhiên nghĩ lần ghi *x* = 2 phải có timestamp lớn hơn. Tiếc là {{< xref fig="9-3" page="/ch9" anchor="fig_distributed_timestamps" >}}Hình 9-3{{< /xref >}} không như vậy: lần ghi *x* = 1 có timestamp 42,004 giây, còn lần ghi *x* = 2 lại có timestamp 42,003 giây.

Như đã bàn ở ["Last write wins (loại bỏ các lần ghi đồng thời)"](/ch6#sec_replication_lww), một cách giải quyết xung đột giữa các giá trị được ghi đồng thời trên nhiều node khác nhau là *last write wins* (LWW): với mỗi khóa, chỉ giữ lần ghi có timestamp lớn nhất và loại bỏ mọi lần ghi có timestamp nhỏ hơn. Trong ví dụ ở {{< xref fig="9-3" page="/ch9" anchor="fig_distributed_timestamps" >}}Hình 9-3{{< /xref >}}, khi node 2 nhận hai sự kiện này, nó sẽ kết luận sai rằng *x* = 1 là giá trị mới hơn và loại bỏ lần ghi *x* = 2, nên thao tác tăng bị mất.

Để tránh vấn đề này, có thể bảo đảm rằng mỗi khi ghi đè một giá trị, timestamp của giá trị mới luôn cao hơn giá trị bị ghi đè, kể cả khi timestamp đó đã vượt trước đồng hồ cục bộ của bên ghi. Nhưng cách này phải trả giá bằng một lần đọc thêm để tìm timestamp lớn nhất hiện tại. Các hệ thống như Cassandra và ScyllaDB muốn ghi vào mọi bản sao chỉ trong một vòng round-trip, nên chúng dùng thẳng timestamp sinh từ đồng hồ của client, kết hợp chiến lược last write wins [^62]. Cách làm này có một số vấn đề nghiêm trọng:

* Dữ liệu ghi vào cơ sở dữ liệu có thể biến mất một cách khó hiểu: node có đồng hồ chậm chỉ có thể ghi đè giá trị mà node có đồng hồ nhanh đã ghi trước đó sau khi khoảng thời gian tương ứng với độ lệch đồng hồ giữa hai node trôi qua [^63] [^65]. Điều này có thể âm thầm loại bỏ một lượng dữ liệu tùy ý mà không báo lỗi nào cho ứng dụng.
* LWW không phân biệt được các lần ghi tuần tự xảy ra liên tiếp trong thời gian ngắn (trong {{< xref fig="9-3" page="/ch9" anchor="fig_distributed_timestamps" >}}Hình 9-3{{< /xref >}}, thao tác tăng của client B rõ ràng xảy ra *sau* lần ghi của client A) với các lần ghi thật sự đồng thời (không bên ghi nào biết về lần ghi của bên kia). Để tránh vi phạm quan hệ nhân quả, cần thêm cơ chế theo dõi quan hệ nhân quả như version vector (xem ["Phát hiện ghi đồng thời"](/ch6#sec_replication_concurrent)).
* Hai node có thể độc lập sinh ra các lần ghi có cùng timestamp, nhất là khi độ phân giải đồng hồ chỉ là mili giây. Giải quyết xung đột kiểu này cần thêm một giá trị phân định (đơn giản là chọn một số ngẫu nhiên lớn), nhưng cách này cũng có thể vi phạm quan hệ nhân quả [^62].

Do đó, dù giữ lại giá trị "mới nhất" và loại bỏ các giá trị khác nghe có vẻ là cách giải quyết xung đột hấp dẫn, ta phải nhận ra rằng định nghĩa của "mới" phụ thuộc vào đồng hồ giờ trong ngày cục bộ, mà đồng hồ này rất có thể không chính xác. Ngay cả khi đồng hồ được đồng bộ NTP chặt chẽ, vẫn có thể xảy ra chuyện gói tin được gửi lúc timestamp 100 mili giây (theo đồng hồ bên gửi) nhưng đến nơi lúc timestamp 99 mili giây (theo đồng hồ bên nhận) — trông như gói tin đến trước khi được gửi, điều này tất nhiên là không thể.

Có thể làm đồng bộ NTP chính xác đến mức tránh hẳn kiểu sắp xếp sai thứ tự này không? Có lẽ không. Ngoài các nguồn sai số khác như độ trôi của đồng hồ thạch anh, độ chính xác đồng bộ của NTP bản thân đã bị giới hạn bởi thời gian round-trip của mạng. Muốn bảo đảm đúng thứ tự, sai số đồng hồ phải nhỏ hơn đáng kể so với độ trễ mạng, mà điều đó không thể thực hiện được.

Cái gọi là *đồng hồ logic* (*logical clock*) [^66], dựa trên bộ đếm tăng dần thay vì tinh thể thạch anh dao động, là lựa chọn an toàn hơn để sắp xếp sự kiện (xem ["Phát hiện ghi đồng thời"](/ch6#sec_replication_concurrent)). Đồng hồ logic không đo giờ trong ngày hay số giây đã trôi qua, mà chỉ ghi lại thứ tự tương đối giữa các sự kiện (sự kiện này xảy ra trước hay sau sự kiện kia). Ngược lại, đồng hồ giờ trong ngày và đồng hồ đơn điệu đo thời gian thực tế trôi qua, nên còn gọi là *đồng hồ vật lý* (*physical clock*). Ta sẽ bàn chi tiết hơn về đồng hồ logic ở ["ID generator và logical clock"](/ch10#sec_consistency_logical).

#### Giá trị đọc đồng hồ kèm khoảng tin cậy {#clock-readings-with-a-confidence-interval}

Đồng hồ giờ trong ngày của máy có thể cho giá trị đọc với độ phân giải micro giây hoặc thậm chí nano giây, nhưng đo chi tiết đến vậy không có nghĩa là giá trị thật sự chính xác đến mức đó. Thực tế, nhiều khả năng nó không chính xác như vậy. Như đã nói, ngay cả khi đồng bộ mỗi phút một lần với server NTP trong mạng cục bộ, đồng hồ thạch anh không chính xác cũng dễ dàng trôi vài mili giây. Với server NTP trên Internet công cộng, độ chính xác tốt nhất có lẽ chỉ cỡ vài chục mili giây, và khi mạng tắc nghẽn, sai số dễ vọt lên hơn 100 mili giây.

Vì vậy, không nên hiểu một giá trị đọc đồng hồ là một thời điểm chính xác, mà nên xem nó là một khoảng thời gian nằm trong một *khoảng tin cậy* (*confidence interval*). Chẳng hạn, hệ thống có thể chắc chắn 95% rằng thời điểm hiện tại nằm trong khoảng từ giây thứ 10,3 đến giây thứ 10,5 của phút này, nhưng không thể biết chính xác hơn [^67]. Nếu chỉ xác định được thời gian với sai số ±100 mili giây, thì những chữ số micro giây trong timestamp về cơ bản vô nghĩa.

Biên của độ bất định có thể tính từ nguồn thời gian. Nếu máy nối trực tiếp với bộ thu GPS hoặc đồng hồ nguyên tử, khoảng sai số kỳ vọng do chính thiết bị quyết định, và với GPS còn phụ thuộc chất lượng tín hiệu vệ tinh. Nếu lấy thời gian từ server, độ bất định xấp xỉ bằng: độ trôi kỳ vọng của đồng hồ thạch anh kể từ lần đồng bộ cuối với server, cộng với độ bất định của chính server NTP, cộng với thời gian round-trip mạng đến server — đây là phép xấp xỉ bậc một, và giả định server đáng tin.

Tiếc là hầu hết hệ thống không để lộ độ bất định này. Ví dụ, khi gọi `clock_gettime()`, giá trị trả về không cho biết sai số kỳ vọng của timestamp, nên bạn không biết khoảng tin cậy của nó là năm mili giây hay năm năm.

Cũng có ngoại lệ: API *TrueTime* của Google Spanner [^45] và ClockBound của Amazon đều báo cáo tường minh khoảng tin cậy của đồng hồ cục bộ. Khi hỏi thời gian hiện tại, bạn nhận hai giá trị: `[earliest, latest]`, lần lượt là timestamp *sớm nhất có thể* và *muộn nhất có thể*. Dựa trên tính toán độ bất định, đồng hồ biết chắc rằng thời gian thực tế hiện tại nằm trong khoảng này. Độ rộng của khoảng phụ thuộc vào nhiều yếu tố, trong đó có việc đồng hồ thạch anh cục bộ đã lâu bao nhiêu kể từ lần đồng bộ với nguồn thời gian chính xác hơn.

#### Đồng hồ đồng bộ cho snapshot toàn cục {#sec_distributed_spanner}

Ở ["Snapshot isolation và repeatable read"](/ch8#sec_transactions_snapshot_isolation), ta đã bàn về *kiểm soát đồng thời đa phiên bản* (MVCC). Với cơ sở dữ liệu cần hỗ trợ cả giao dịch đọc-ghi ngắn, nhanh lẫn giao dịch chỉ đọc lớn, chạy lâu (như sao lưu hay phân tích), MVCC là tính năng rất hữu ích. Nó cho phép giao dịch chỉ đọc thấy trạng thái nhất quán của cơ sở dữ liệu tại một thời điểm cụ thể, tức là một *snapshot*, mà không phải khóa hay cản trở các giao dịch đọc-ghi.

Nói chung, MVCC cần ID giao dịch tăng đơn điệu. Nếu một lần ghi xảy ra sau snapshot (tức ID giao dịch của lần ghi lớn hơn ID giao dịch của snapshot), giao dịch snapshot sẽ không thấy lần ghi đó. Trên cơ sở dữ liệu một node, một bộ đếm đơn giản là đủ để sinh ID giao dịch.

Tuy nhiên, khi cơ sở dữ liệu phân tán trên nhiều máy, thậm chí trải qua nhiều trung tâm dữ liệu, việc sinh ID giao dịch tăng đơn điệu toàn cục (trên mọi shard) trở nên khó vì cần phối hợp. ID giao dịch còn phải phản ánh quan hệ nhân quả: nếu giao dịch B đọc hoặc ghi đè giá trị mà giao dịch A ghi trước đó, ID của B phải lớn hơn A, nếu không snapshot sẽ không nhất quán. Với lượng lớn giao dịch ngắn, nhanh, việc sinh ID giao dịch trong hệ thống phân tán sẽ thành nút thắt cổ chai khó chịu nổi. (Ta sẽ bàn về các ID generator như vậy ở ["ID generator và logical clock"](/ch10#sec_consistency_logical).)

Có thể dùng thẳng timestamp từ đồng hồ giờ trong ngày đã đồng bộ làm ID giao dịch không? Nếu đồng bộ đồng hồ làm đủ tốt, timestamp thật sự có thuộc tính cần thiết: giao dịch càng muộn thì timestamp càng lớn. Dĩ nhiên vấn đề vẫn là độ bất định về độ chính xác của đồng hồ.

Spanner hiện thực snapshot isolation xuyên trung tâm dữ liệu theo cách này [^68] [^69]. Nó dùng khoảng tin cậy của đồng hồ do API TrueTime báo cáo, dựa trên quan sát sau: giả sử có hai khoảng tin cậy, mỗi khoảng gồm timestamp sớm nhất và muộn nhất có thể (*A* = [*A*<sub>earliest</sub>, *A*<sub>latest</sub>], *B* = [*B*<sub>earliest</sub>, *B*<sub>latest</sub>]), và hai khoảng này không chồng lên nhau (tức *A*<sub>earliest</sub> < *A*<sub>latest</sub> < *B*<sub>earliest</sub> < *B*<sub>latest</sub>) thì B chắc chắn xảy ra sau A, không còn nghi ngờ gì. Chỉ khi hai khoảng chồng lên nhau ta mới không chắc thứ tự của A và B.

Để bảo đảm timestamp giao dịch phản ánh quan hệ nhân quả, Spanner cố ý chờ một khoảng thời gian bằng độ dài khoảng tin cậy trước khi commit giao dịch đọc-ghi. Nhờ vậy, mọi giao dịch có thể đọc dữ liệu này sẽ xảy ra đủ muộn để khoảng tin cậy của chúng không còn chồng lên nhau. Để rút ngắn thời gian chờ, Spanner cần giữ độ bất định của đồng hồ càng nhỏ càng tốt. Vì vậy, Google triển khai bộ thu GPS hoặc đồng hồ nguyên tử ở mỗi trung tâm dữ liệu, giữ sai số đồng bộ đồng hồ trong khoảng 7 mili giây [^45].

Nói chính xác, Spanner không nhất thiết phải dùng đồng hồ nguyên tử và bộ thu GPS: điều thực sự quan trọng là có khoảng tin cậy, còn nguồn thời gian chính xác chỉ giúp thu hẹp khoảng đó. Các hệ thống khác cũng bắt đầu dùng cách tương tự. Ví dụ, YugabyteDB khi chạy trên AWS có thể tận dụng ClockBound [^70], và một số hệ thống khác cũng bắt đầu dựa vào đồng bộ đồng hồ ở nhiều mức độ [^71] [^72].

### Tạm dừng tiến trình {#sec_distributed_clocks_pauses}

Xét thêm một ví dụ về việc dùng đồng hồ nguy hiểm trong hệ thống phân tán. Giả sử mỗi shard của một cơ sở dữ liệu chỉ có một leader, và chỉ leader được nhận lần ghi. Làm sao một node biết mình vẫn là leader (chưa bị node khác tuyên bố đã chết) để an toàn nhận lần ghi?

Một cách là leader lấy từ các node khác một *lease* (hợp đồng thuê), tương tự khóa có timeout [^73]. Tại mỗi thời điểm chỉ một node được giữ lease. Do đó, khi một node có được lease, nó biết mình là leader cho đến khi lease hết hạn. Để giữ vai trò leader, node phải định kỳ gia hạn lease trước khi hết hạn. Nếu node hỏng, nó ngừng gia hạn, nên khi lease hết hạn, node khác có thể tiếp quản.

Có thể hình dung vòng lặp xử lý yêu cầu trông đại khái như sau:

```js
while (true) {
    request = getIncomingRequest();

    // Bảo đảm lease còn ít nhất 10 giây
    if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000) {
        lease = lease.renew();
    }

    if (lease.isValid()) {
        process(request);
    }
}
```

Đoạn mã này có vấn đề gì? Thứ nhất, nó dựa vào đồng hồ đồng bộ: thời điểm hết hạn của lease do một máy khác đặt (ví dụ tính bằng thời gian hiện tại cộng 30 giây), nhưng lại được so sánh với đồng hồ hệ thống cục bộ. Nếu hai đồng hồ chênh nhau quá vài giây, đoạn mã sẽ chạy kỳ quặc.

Thứ hai, ngay cả khi sửa giao thức để chỉ dùng đồng hồ đơn điệu cục bộ, vẫn còn một vấn đề khác: mã giả định rằng giữa lúc đọc thời gian (`System.currentTimeMillis()`) và lúc xử lý yêu cầu (`process(request)`) chỉ trôi qua rất ít thời gian. Thường thì đoạn mã này quả thật chạy rất nhanh, nên chừa 10 giây là đủ bảo đảm lease không hết hạn giữa chừng khi đang xử lý yêu cầu.

Nhưng nếu chương trình bất ngờ bị tạm dừng khi đang chạy thì sao? Chẳng hạn, giả sử thread dừng 15 giây ở gần dòng `lease.isValid()` rồi mới chạy tiếp. Khi đó, lúc xử lý yêu cầu, lease rất có thể đã hết hạn từ lâu và node khác đã thành leader. Tuy nhiên, không có gì báo cho thread biết nó vừa dừng lâu như vậy, nên chỉ đến vòng lặp tiếp theo nó mới nhận ra lease đã hết hạn — và đến lúc đó nó có thể đã làm điều không an toàn khi xử lý yêu cầu.

Có hợp lý khi nghĩ thread có thể bị dừng lâu như vậy không? Đáng tiếc là hoàn toàn hợp lý. Có nhiều nguyên nhân gây ra khoảng dừng dài:

* Khi nhiều thread tranh chấp tài nguyên chung như khóa, hàng đợi, thread có thể mất rất nhiều thời gian để chờ. Chuyển sang máy có nhiều lõi CPU hơn thậm chí có thể làm vấn đề này tệ hơn, và vấn đề tranh chấp thường rất khó chẩn đoán [^74].
* Nhiều runtime của ngôn ngữ lập trình (như Java Virtual Machine) có *bộ thu gom rác* (GC, *garbage collection*), thỉnh thoảng cần dừng mọi thread đang chạy. Trước đây, những *khoảng dừng GC stop-the-world* như vậy đôi khi khiến chương trình đứng hàng phút [^75]! Các thuật toán GC hiện đại đã giảm nhẹ vấn đề này rất nhiều, nhưng khoảng dừng GC vẫn có thể khá rõ rệt (xem ["Hạn chế tác động của garbage collection"](/ch9#sec_distributed_gc_impact)).
* Trong môi trường ảo hóa, máy ảo có thể bị *tạm dừng* (*suspend*): dừng mọi tiến trình và lưu nội dung bộ nhớ ra đĩa, rồi sau đó *tiếp tục* (*resume*): khôi phục nội dung bộ nhớ và chạy tiếp từ chỗ đã dừng. Khoảng dừng này có thể xảy ra bất cứ lúc nào trong quá trình thực thi và kéo dài bao lâu cũng được. Tính năng này đôi khi dùng cho *live migration*: chuyển máy ảo từ host này sang host khác mà không cần khởi động lại, và khi đó thời gian dừng phụ thuộc tốc độ tiến trình ghi vào bộ nhớ [^76].
* Trên thiết bị người dùng cuối như laptop và điện thoại, quá trình thực thi cũng có thể bị tạm dừng và tiếp tục bất cứ lúc nào, chẳng hạn khi người dùng gập màn hình laptop.
* Khi hệ điều hành chuyển sang thread khác, hoặc hypervisor chuyển sang máy ảo khác, thread hiện tại có thể dừng ở vị trí bất kỳ trong mã. Với máy ảo, thời gian CPU bị máy ảo khác chiếm gọi là *steal time*. Nếu máy tải nặng — tức là có hàng đợi thread dài đang chờ chạy — thread bị dừng có thể phải một lúc sau mới được chạy lại.
* Nếu ứng dụng truy cập đĩa đồng bộ, thread có thể bị dừng để chờ thao tác I/O đĩa chậm hoàn tất [^77]. Trong nhiều ngôn ngữ, truy cập đĩa có thể xảy ra bất ngờ ngay cả khi mã không đọc ghi file tường minh. Ví dụ, class loader của Java chỉ nạp file class khi class được dùng lần đầu, và việc này có thể xảy ra tại bất kỳ thời điểm nào của chương trình. Khoảng dừng I/O và khoảng dừng GC thậm chí có thể cộng dồn lên nhau [^78]. Nếu "đĩa" thực chất là hệ thống file mạng hay block device qua mạng (như Amazon EBS), độ trễ I/O còn chịu ảnh hưởng của độ dao động độ trễ mạng [^31].
* Nếu hệ điều hành cho phép *hoán đổi ra đĩa* (*swapping to disk*) hay còn gọi là *phân trang* (*paging*), một lần truy cập bộ nhớ đơn giản cũng có thể gây page fault, buộc phải nạp một trang từ đĩa vào bộ nhớ. Thread bị dừng cho đến khi thao tác I/O chậm này hoàn tất. Nếu áp lực bộ nhớ cao, có thể còn phải hoán đổi một trang khác ra đĩa trước. Trong trường hợp cực đoan, hệ điều hành dành phần lớn thời gian để hoán đổi trang vào ra mà gần như không làm việc gì thật sự, hiện tượng này gọi là *thrashing*. Để tránh vấn đề này, server thường tắt phân trang (nếu bạn thà kill một tiến trình để giải phóng bộ nhớ còn hơn mạo hiểm bị thrashing).
* Gửi tín hiệu `SIGSTOP` cho một tiến trình Unix cũng làm nó dừng, ví dụ khi bấm Ctrl-Z trong shell. Tín hiệu này lập tức ngừng cấp chu kỳ CPU cho tiến trình cho đến khi `SIGCONT` cho nó chạy lại, và lúc đó nó tiếp tục chạy từ vị trí đã dừng. Ngay cả khi môi trường của bạn thường không dùng `SIGSTOP`, người vận hành vẫn có thể vô tình gửi tín hiệu này.

Bất kỳ sự kiện nào ở trên cũng có thể *preempt* (chiếm quyền) một thread đang chạy tại vị trí bất kỳ rồi cho nó chạy lại sau một lúc, mà thread không hề hay biết. Vấn đề này tương tự việc bảo đảm an toàn thread trong mã đa thread trên một máy: không thể giả định gì về thời gian, vì chuyển ngữ cảnh và thực thi song song có thể xảy ra bất cứ lúc nào.

Khi viết mã đa thread trên một máy, ta có nhiều công cụ quen thuộc để bảo đảm an toàn thread: mutex, semaphore, bộ đếm nguyên tử, cấu trúc dữ liệu lock-free, hàng đợi chặn, v.v. Tiếc là các công cụ này không áp dụng thẳng được cho hệ thống phân tán, vì hệ thống phân tán không có bộ nhớ chung, chỉ có thông điệp truyền qua mạng không đáng tin cậy.

Node trong hệ thống phân tán phải giả định rằng quá trình thực thi của nó có thể bị dừng lâu bất cứ lúc nào, kể cả khi đang chạy giữa chừng một hàm. Trong khoảng dừng, thế giới bên ngoài vẫn tiếp tục chạy, thậm chí có thể tuyên bố node đã chết vì nó mãi không phản hồi. Cuối cùng khi node chạy lại, nó thậm chí không nhận ra mình đã "ngủ", cho đến khi đọc đồng hồ lần nữa sau đó.

#### Bảo đảm thời gian phản hồi {#sec_distributed_clocks_realtime}

Như đã nói, trong nhiều ngôn ngữ lập trình và hệ điều hành, thread và tiến trình có thể bị dừng lâu tùy ý. Tuy nhiên, nếu bỏ đủ công sức thì các nguyên nhân gây dừng này *thật sự có thể* bị loại bỏ.

Một số phần mềm chạy trong môi trường mà không phản hồi kịp thời có thể gây thiệt hại nghiêm trọng. Máy tính điều khiển máy bay, tên lửa, robot, ô tô và các thiết bị vật lý khác phải phản hồi nhanh và có thể dự đoán trước đối với dữ liệu cảm biến. Trong các hệ thống này, phần mềm phải phản hồi trước một *hạn chót* (*deadline*) được quy định rõ, và lỡ hạn chót có thể khiến toàn hệ thống hỏng. Những hệ thống như vậy gọi là hệ thống *thời gian thực cứng* (*hard real-time*).


> [!NOTE]
> Trong hệ thống nhúng, *real-time* nghĩa là hệ thống được thiết kế và kiểm thử cẩn thận để đáp ứng các bảo đảm về thời gian đã quy định trong mọi tình huống. Điều này khác với cách dùng từ *real-time* lỏng lẻo hơn trong lĩnh vực web, thường chỉ việc server đẩy dữ liệu cho client hoặc xử lý luồng mà không có ràng buộc nghiêm ngặt về thời gian phản hồi (xem [Chương 12](/ch12#ch_stream)).


Chẳng hạn, khi cảm biến trên xe phát hiện va chạm đang xảy ra, bạn chắc chắn không muốn túi khí bung chậm vì hệ thống điều khiển vừa lúc gặp khoảng dừng GC.

Để cung cấp bảo đảm thời gian thực trong một hệ thống, mọi tầng của ngăn xếp phần mềm đều phải hỗ trợ. Cần một *hệ điều hành thời gian thực* (RTOS) cho phép lập lịch tiến trình với bảo đảm được cấp thời gian CPU trong những khoảng đã quy định. Các hàm thư viện phải ghi rõ thời gian thực thi trong trường hợp xấu nhất. Việc cấp phát bộ nhớ động có thể bị hạn chế hoặc cấm hoàn toàn (tuy có garbage collector thời gian thực, ứng dụng vẫn phải bảo đảm không giao cho GC quá nhiều việc). Ngoài ra còn phải kiểm thử và đo đạc rất nhiều để xác nhận hệ thống thật sự đáp ứng các bảo đảm.

Những yêu cầu này không chỉ kéo theo nhiều công việc phụ trội mà còn hạn chế nghiêm trọng ngôn ngữ lập trình, thư viện và công cụ có thể dùng, vì phần lớn ngôn ngữ và công cụ không cung cấp bảo đảm thời gian thực. Chính vì vậy, phát triển hệ thống thời gian thực cực kỳ tốn kém và thường dùng nhất cho các thiết bị nhúng liên quan đến an toàn. Hơn nữa, "thời gian thực" không đồng nghĩa với "hiệu năng cao" — thực tế, hệ thống thời gian thực có thể có thông lượng thấp hơn, vì phản hồi kịp thời phải được ưu tiên trên hết (xem thêm ["Độ trễ và mức sử dụng tài nguyên"](/ch9#sidebar_distributed_latency_utilization)).

Với đa số hệ thống xử lý dữ liệu phía server, bảo đảm thời gian thực vừa không kinh tế vừa không phù hợp. Vì vậy các hệ thống này đành chịu khoảng dừng tiến trình và đồng hồ không ổn định của môi trường phi thời gian thực.

#### Hạn chế tác động của garbage collection {#sec_distributed_gc_impact}

Garbage collection từng là một trong những nguyên nhân lớn nhất gây ra khoảng dừng tiến trình [^79]. May là các thuật toán GC đã tiến bộ đáng kể: ngày nay, bộ thu gom được tinh chỉnh phù hợp thường chỉ dừng vài mili giây. Runtime Java cung cấp các bộ thu gom như Concurrent Mark Sweep (CMS), Garbage-First (G1), Z Garbage Collector (ZGC), Epsilon và Shenandoah, mỗi loại tối ưu cho đặc điểm sử dụng bộ nhớ khác nhau như tạo đối tượng với tần suất cao hay heap lớn. Ngược lại, Go cung cấp một bộ thu gom mark-sweep đồng thời đơn giản hơn, cố gắng tự tối ưu.

Nếu cần tránh hoàn toàn khoảng dừng GC, có thể chọn ngôn ngữ hoàn toàn không có garbage collector. Chẳng hạn, Swift dùng đếm tham chiếu tự động để biết khi nào giải phóng được bộ nhớ, còn Rust và Mojo theo dõi vòng đời đối tượng qua hệ thống kiểu, để trình biên dịch quyết định cần giữ bộ nhớ bao lâu.

Cũng có thể tiếp tục dùng ngôn ngữ có garbage collection nhưng giảm nhẹ tác động của khoảng dừng. Một cách là coi khoảng dừng GC như thời gian ngừng hoạt động có kế hoạch ngắn của node: khi một node đang thu gom rác, các node khác xử lý yêu cầu của client. Nếu runtime báo trước cho ứng dụng rằng node sắp có khoảng dừng GC, ứng dụng có thể ngừng gửi yêu cầu mới cho node đó, chờ nó xử lý xong các yêu cầu đang dở, rồi chạy GC lúc không có yêu cầu nào. Mẹo này giấu khoảng dừng GC khỏi client và giảm các phân vị cao của thời gian phản hồi [^80] [^81].

Có một biến thể của ý tưởng này: chỉ để bộ thu gom xử lý các đối tượng sống ngắn dễ thu hồi nhanh, và khởi động lại tiến trình định kỳ, trước khi các đối tượng sống lâu kịp tích lũy đến mức cần chạy một lần GC đầy đủ [^79] [^82]. Có thể mỗi lần chỉ khởi động lại một node, và chuyển lưu lượng khỏi node đó trước lần khởi động lại đã lên kế hoạch, giống như rolling upgrade (xem [Chương 5](/ch5#ch_encoding)).

Các biện pháp này không loại bỏ hoàn toàn khoảng dừng garbage collection, nhưng có thể giảm đáng kể tác động của chúng lên ứng dụng.



## Tri thức, sự thật và lời nói dối {#sec_distributed_truth}

Đến đây, chương này đã xem xét những điểm khác nhau giữa hệ thống phân tán và chương trình trên một máy: hệ thống không có bộ nhớ chung, chỉ truyền thông điệp qua mạng không đáng tin cậy với độ trễ không cố định, và còn có thể gặp hỏng hóc từng phần, đồng hồ không đáng tin cậy và tiến trình bị tạm dừng.

Nếu chưa quen với hệ thống phân tán, hệ quả của những vấn đề này khiến người ta cực kỳ mất phương hướng. Một node trong mạng không thể *biết chắc* bất cứ điều gì về các node khác, mà chỉ có thể đoán dựa trên những thông điệp nhận được (hoặc không nhận được). Một node chỉ biết node kia đang ở trạng thái nào — đã lưu dữ liệu gì, có đang chạy bình thường không — khi trao đổi thông điệp với nó. Nếu node ở xa không phản hồi thì không có cách nào biết trạng thái của nó, vì không thể phân biệt đáng tin cậy giữa sự cố mạng và sự cố của chính node.

Cuộc thảo luận về những hệ thống này gần như mang tính triết học: ta thực sự biết điều gì là đúng, điều gì là sai trong hệ thống? Nếu cơ chế cảm nhận và đo lường đều không đáng tin cậy, ta có thể tin vào hiểu biết của mình đến mức nào [^83]? Hệ thống phần mềm có nên tuân theo những quy luật mà ta cho rằng thế giới vật lý nhất thiết phải tuân theo, như nhân quả?

May là ta không cần truy vấn đến tận ý nghĩa của cuộc sống. Trong hệ thống phân tán, ta có thể viết rõ các giả định về hành vi của hệ thống (gọi là *mô hình hệ thống*, *system model*), rồi thiết kế hệ thống thực tế sao cho thỏa các giả định đó. Ta cũng có thể chứng minh một thuật toán chạy đúng trong một mô hình hệ thống cụ thể. Nghĩa là ta vẫn có thể hiện thực hành vi đáng tin cậy ngay cả khi mô hình hệ thống bên dưới chỉ cung cấp rất ít bảo đảm.

Tuy nhiên, làm cho phần mềm chạy tốt trong mô hình hệ thống không đáng tin cậy hoàn toàn không dễ. Phần còn lại của chương sẽ đi sâu hơn vào tri thức và sự thật trong hệ thống phân tán, giúp ta suy nghĩ về những giả định có thể đưa ra và những bảo đảm ta muốn cung cấp. Ở [Chương 10](/ch10#ch_consistency), ta sẽ tiếp tục xem xét một số thuật toán phân tán: chúng cung cấp những bảo đảm cụ thể dưới những giả định cụ thể.

### Nguyên tắc đa số {#sec_distributed_majority}

Hãy hình dung một mạng có lỗi bất đối xứng: một node nhận được mọi thông điệp gửi cho nó, nhưng mọi thông điệp nó gửi đi đều bị mất hoặc bị trễ [^22]. Node này chạy hoàn toàn bình thường và vẫn nhận yêu cầu từ các node khác, nhưng các node khác không nghe được phản hồi của nó. Sau khi hết timeout, vì mãi không nhận được hồi đáp, các node khác tuyên bố nó đã chết. Cảnh tượng sau đó như một cơn ác mộng: node nửa mất liên lạc bị lôi ra nghĩa địa, vừa vùng vẫy vừa kêu "Tôi chưa chết!" — nhưng chẳng ai nghe thấy tiếng kêu, và đoàn đưa tang vẫn kiên quyết tiến bước.

Trong một kịch bản đỡ ác mộng hơn, node nửa mất liên lạc có thể nhận ra các thông điệp nó gửi không được node khác xác nhận, và hiểu rằng mạng hẳn đã có sự cố. Tuy vậy, các node khác vẫn tuyên bố sai rằng nó đã chết, và nó chẳng làm gì được.

Kịch bản thứ ba: giả sử một node bị tạm dừng một phút. Trong thời gian đó, nó không xử lý yêu cầu cũng không gửi phản hồi. Các node khác vừa chờ vừa thử lại, cuối cùng hết kiên nhẫn, tuyên bố nó đã chết và khiêng nó lên xe tang. Rồi khoảng dừng kết thúc, thread của node tiếp tục chạy như không có chuyện gì xảy ra. Các node khác kinh ngạc thấy node lẽ ra đã chết bỗng nhô đầu ra khỏi quan tài, khỏe mạnh và vui vẻ trò chuyện với mọi người. Lúc mới hồi phục, node này thậm chí không biết cả một phút đã trôi qua, cũng không biết mình đã bị tuyên bố chết — với nó, lần trò chuyện cuối với các node khác dường như chỉ mới vừa xảy ra.

Những câu chuyện này cho thấy node không nhất thiết có thể tin vào phán đoán của chính mình về tình hình. Hệ thống phân tán không thể chỉ dựa vào một node duy nhất, vì node có thể hỏng bất cứ lúc nào và đẩy hệ thống vào bế tắc không phục hồi được. Vì vậy, nhiều thuật toán phân tán dựa vào *quorum* — cho nhiều node bỏ phiếu (xem ["Đọc và ghi quorum"](/ch6#sec_replication_quorum_condition)): một quyết định phải nhận được số phiếu tối thiểu từ một số node, nhờ đó giảm sự phụ thuộc vào bất kỳ node đơn lẻ nào.

Quyết định tuyên bố một node đã chết cũng vậy. Nếu quorum các node tuyên bố một node khác đã chết thì nó phải được coi là đã chết, dù nó tự thấy mình vẫn sống khỏe. Node đơn lẻ phải tuân theo quyết định của quorum và rút lui.

Quorum phổ biến nhất là đa số tuyệt đối, tức hơn một nửa tổng số node, dù cũng có thể dùng các dạng khác. Quorum đa số cho phép hệ thống tiếp tục chạy khi một số ít node hỏng: ba node chịu được một node hỏng, năm node chịu được hai. Đồng thời nó vẫn an toàn, vì trong hệ thống chỉ có thể hình thành một đa số, không thể có hai đa số cùng đưa ra quyết định mâu thuẫn. Ta sẽ bàn chi tiết hơn về cách dùng quorum khi nói về *thuật toán đồng thuận* ở [Chương 10](/ch10#ch_consistency).

### Khóa phân tán và lease {#sec_distributed_lock_fencing}

Khóa và lease trong ứng dụng phân tán rất dễ bị dùng sai, và là nguồn lỗi chương trình thường gặp [^84]. Hãy xem một cách cụ thể mà chúng có thể sai.

Ở ["Tạm dừng tiến trình"](/ch9#sec_distributed_clocks_pauses), ta đã thấy lease là một loại khóa có timeout: nếu người giữ ban đầu ngừng phản hồi (có thể vì sập, bị dừng quá lâu hoặc mất kết nối mạng), lease có thể được giao cho người giữ mới. Khi hệ thống yêu cầu chỉ có một thứ duy nhất, ta có thể dùng lease. Ví dụ:

* Chỉ cho một node làm leader của một shard cơ sở dữ liệu, để tránh split brain (xem ["Xử lý sự cố node"](/ch6#sec_replication_failover)).
* Chỉ cho một giao dịch hoặc client cập nhật một tài nguyên hay đối tượng cụ thể, để ghi đồng thời không làm nó hỏng.
* Mỗi file đầu vào của một job xử lý lớn chỉ nên do một node xử lý, tránh nhiều node làm trùng một việc và lãng phí tài nguyên tính toán.

Cần suy nghĩ kỹ: nếu nhiều node cùng tin rằng mình đang giữ lease — có thể do tiến trình bị tạm dừng — thì chuyện gì xảy ra? Với ví dụ thứ ba, hậu quả chỉ là lãng phí chút tài nguyên tính toán, không nghiêm trọng. Nhưng ở hai ví dụ đầu, dữ liệu có thể bị mất hoặc hỏng, nghiêm trọng hơn nhiều.

Chẳng hạn, {{< xref fig="9-4" page="/ch9" anchor="fig_distributed_lease_pause" >}}Hình 9-4{{< /xref >}} cho thấy dữ liệu bị hỏng do hiện thực khóa sai. (Đây không chỉ là vấn đề lý thuyết: HBase từng có lỗi này [^85] [^86].) Giả sử bạn muốn bảo đảm file trong một dịch vụ lưu trữ mỗi lúc chỉ được một client truy cập, vì nhiều client ghi cùng lúc sẽ làm hỏng file. Bạn thử hiện thực điều này bằng cách yêu cầu client phải lấy lease từ dịch vụ khóa trước khi truy cập file. Dịch vụ khóa kiểu này thường hiện thực bằng thuật toán đồng thuận, ta sẽ bàn thêm ở [Chương 10](/ch10#ch_consistency).

{{< fig num="9-4" id="fig_distributed_lease_pause" src="/fig/ddia_0904.png" caption="Hiện thực sai khóa phân tán: lease đã hết hạn nhưng client 1 vẫn tin rằng nó còn hiệu lực, và làm hỏng file trong kho lưu trữ." class="ddia-figure ddia-figure--wide" width="2658" height="975" />}}


Vấn đề chính là tình huống đã bàn ở ["Tạm dừng tiến trình"](/ch9#sec_distributed_clocks_pauses): nếu client giữ lease bị dừng quá lâu, lease sẽ hết hạn. Lúc này client khác có thể lấy lease cho cùng file và bắt đầu ghi. Khi client bị dừng hồi phục, nó lầm tưởng lease của mình vẫn còn hiệu lực và cũng tiếp tục ghi vào file. Kết quả là split brain: lần ghi của hai client xung đột với nhau và cuối cùng làm hỏng file.

{{< xref fig="9-5" page="/ch9" anchor="fig_distributed_lease_delay" >}}Hình 9-5{{< /xref >}} cho thấy một vấn đề khác có hậu quả tương tự. Trong ví dụ này không có tiến trình nào bị dừng, chỉ có client 1 bị sập. Ngay trước khi sập, client 1 gửi yêu cầu ghi đến dịch vụ lưu trữ, nhưng yêu cầu bị trễ rất lâu trong mạng. (Hãy nhớ lại ["Lỗi mạng trong thực tế"](/ch9#sec_distributed_network_faults): gói tin đôi khi bị trễ hơn một phút.) Khi yêu cầu ghi đến dịch vụ lưu trữ, lease đã timeout từ lâu, client 2 đã lấy lease và đã gửi lần ghi của mình. Kết quả là dữ liệu bị hỏng tương tự {{< xref fig="9-4" page="/ch9" anchor="fig_distributed_lease_pause" >}}Hình 9-4{{< /xref >}}.

{{< fig num="9-5" id="fig_distributed_lease_delay" src="/fig/ddia_0905.png" caption="Thông điệp do người giữ lease cũ gửi có thể bị trễ rất lâu, đến tận khi node khác đã tiếp quản lease mới tới nơi." class="ddia-figure ddia-figure--wide" width="2658" height="975" />}}


#### Dùng fencing để chặn zombie và yêu cầu bị trễ {#sec_distributed_fencing_tokens}

Từ *zombie* đôi khi dùng để chỉ người giữ lease cũ chưa biết mình đã mất lease và vẫn hành xử như người giữ hiện tại. Vì không thể loại bỏ hoàn toàn zombie, ta phải bảo đảm chúng không thể gây hại dưới dạng split brain. Việc này gọi là dùng *fencing* để chặn zombie.

Một số hệ thống cố chặn zombie bằng cách tắt nó, ví dụ ngắt kết nối mạng của nó [^9], tắt máy ảo qua giao diện quản lý của nhà cung cấp đám mây, hoặc thậm chí cắt nguồn điện của máy [^87]. Cách này gọi là STONITH, viết tắt của "Shoot The Other Node In The Head" (bắn vào đầu node kia). Tiếc là nó có mấy vấn đề: không chống được độ trễ mạng rất dài như ở {{< xref fig="9-5" page="/ch9" anchor="fig_distributed_lease_delay" >}}Hình 9-5{{< /xref >}}, tất cả các node có thể tắt lẫn nhau [^19], và khi zombie được phát hiện rồi tắt thì có lẽ đã quá muộn, dữ liệu đã hỏng.

{{< xref fig="9-6" page="/ch9" anchor="fig_distributed_fencing" >}}Hình 9-6{{< /xref >}} cho thấy một cơ chế fencing vững chắc hơn, chống được cả zombie lẫn yêu cầu bị trễ.

{{< fig num="9-6" id="fig_distributed_fencing" src="/fig/ddia_0906.png" caption="Chỉ cho phép thực hiện lần ghi theo thứ tự fencing token tăng dần để bảo đảm truy cập kho lưu trữ an toàn." class="ddia-figure ddia-figure--wide" width="2658" height="975" />}}


Giả sử mỗi lần cấp khóa hay lease, dịch vụ khóa còn trả về một *fencing token*. Đó là một con số tăng lên sau mỗi lần cấp khóa, ví dụ do dịch vụ khóa tự tăng. Sau đó có thể yêu cầu client mỗi lần gửi yêu cầu ghi đến dịch vụ lưu trữ đều phải kèm fencing token hiện tại của mình.


> [!NOTE]
> Fencing token còn có vài tên gọi khác. Trong Chubby, dịch vụ khóa của Google, nó gọi là *sequencer* [^88], còn trong Kafka, nó gọi là *số epoch* (*epoch number*). Trong các thuật toán đồng thuận mà ta sẽ bàn ở [Chương 10](/ch10#ch_consistency), *số ballot* (*ballot number*) của Paxos hay *số term* (*term number*) của Raft cũng đóng vai trò tương tự.


Trong {{< xref fig="9-6" page="/ch9" anchor="fig_distributed_fencing" >}}Hình 9-6{{< /xref >}}, client 1 lấy lease cùng token 33, rồi bị dừng rất lâu khiến lease hết hạn. Client 2 sau đó lấy lease cùng token 34 (token luôn tăng), và gửi yêu cầu ghi kèm token 34 đến dịch vụ lưu trữ. Sau đó client 1 chạy lại, cũng gửi yêu cầu ghi đến dịch vụ lưu trữ, mang token 33 của nó. Tuy nhiên, dịch vụ lưu trữ nhớ rằng nó đã xử lý lần ghi có token cao hơn (34), nên từ chối yêu cầu với token 33. Client vừa lấy lease phải thực hiện ngay một lần ghi vào dịch vụ lưu trữ, và khi lần ghi này hoàn tất, mọi zombie đều bị chặn ở ngoài.

Nếu dùng ZooKeeper làm dịch vụ khóa, có thể dùng ID giao dịch `zxid` hoặc phiên bản node `cversion` làm fencing token [^85]. Trong etcd, số revision cùng ID lease đóng vai trò tương tự [^89]. API FencedLock của Hazelcast thì sinh fencing token một cách tường minh [^90].

Cơ chế này đòi hỏi dịch vụ lưu trữ có thể kiểm tra token đi kèm lần ghi có bị lỗi thời hay không. Một cách khác là để dịch vụ hỗ trợ lần ghi kiểu compare-and-set (CAS) nguyên tử: lần ghi chỉ thành công nếu từ lần client hiện tại đọc đối tượng lần cuối, chưa có client nào khác ghi vào đối tượng đó. Dịch vụ lưu trữ đối tượng hỗ trợ kiểu kiểm tra này: Amazon S3 gọi là *ghi có điều kiện* (*conditional write*), Azure Blob Storage gọi là *header có điều kiện* (*conditional header*), còn Google Cloud Storage gọi là *điều kiện tiên quyết của yêu cầu* (*request precondition*).

#### Fencing với nhiều bản sao {#fencing-with-multiple-replicas}

Nếu client chỉ cần ghi vào một dịch vụ lưu trữ hỗ trợ ghi có điều kiện như vậy, dịch vụ khóa phần nào thừa [^91] [^92], vì hoàn toàn có thể dựa thẳng vào dịch vụ lưu trữ đó để cấp lease [^93]. Tuy nhiên, khi đã có fencing token, ta cũng có thể dùng nó cho nhiều dịch vụ hoặc nhiều bản sao, bảo đảm người giữ lease cũ bị chặn trên tất cả các dịch vụ đó.

Chẳng hạn, giả sử dịch vụ lưu trữ là một kho khóa-giá trị dùng sao chép không leader (*leaderless replication*) và dùng last write wins để giải quyết xung đột (xem ["Sao chép không leader (*leaderless replication*)"](/ch6#sec_replication_leaderless)). Trong hệ thống như vậy, client gửi lần ghi trực tiếp đến từng bản sao, và mỗi bản sao tự quyết định có nhận lần ghi hay không dựa trên timestamp do client gán.

Như {{< xref fig="9-7" page="/ch9" anchor="fig_distributed_fencing_leaderless" >}}Hình 9-7{{< /xref >}} cho thấy, có thể đặt fencing token của bên ghi vào các bit hoặc chữ số có trọng số cao nhất của timestamp. Như vậy, mọi timestamp do người giữ lease mới sinh ra đều lớn hơn mọi timestamp do người giữ lease cũ sinh ra, dù lần ghi của người giữ cũ thực tế diễn ra muộn hơn.

{{< fig num="9-7" id="fig_distributed_fencing_leaderless" src="/fig/ddia_0907.png" caption="Dùng fencing token để bảo vệ lần ghi vào cơ sở dữ liệu dùng sao chép không leader (*leaderless replication*)." class="ddia-figure ddia-figure--wide" width="2658" height="1466" />}}


Trong {{< xref fig="9-7" page="/ch9" anchor="fig_distributed_fencing_leaderless" >}}Hình 9-7{{< /xref >}}, fencing token của client 2 là 34, nên mọi timestamp nó sinh ra bắt đầu bằng 34… đều lớn hơn bất kỳ timestamp nào bắt đầu bằng 33… do client 1 sinh ra. Client 2 ghi thành công vào một tập bản sao đạt quorum nhưng không kết nối được bản sao 3. Vì vậy, khi client zombie 1 sau đó thử ghi, lần ghi này tuy bị bản sao 1 và 2 bỏ qua, nhưng vẫn có thể thành công trên bản sao 3. Điều này không sao, vì các lần đọc quorum sau đó sẽ ưu tiên lần ghi của client 2 có timestamp lớn hơn, còn read repair hoặc quy trình anti-entropy cuối cùng sẽ ghi đè giá trị do client 1 ghi.

Từ các ví dụ này có thể thấy, giả định rằng tại mỗi thời điểm chỉ có một node giữ lease là không an toàn. May là chỉ cần chú ý một chút, ta có thể dùng fencing token để ngăn zombie và yêu cầu bị trễ gây hại.

### Lỗi Byzantine {#sec_distributed_byzantine}

Fencing token có thể phát hiện và chặn node *vô tình* mắc lỗi, chẳng hạn node chưa biết lease của mình đã hết hạn. Nhưng nếu node cố ý phá vỡ các bảo đảm của hệ thống, nó có thể dễ dàng vượt qua lớp bảo vệ này bằng cách gửi thông điệp kèm fencing token giả.

Cuốn sách này giả định node tuy không đáng tin cậy nhưng trung thực: chúng có thể phản hồi chậm hoặc không bao giờ phản hồi vì lỗi, có thể giữ trạng thái lỗi thời do GC tạm dừng hay độ trễ mạng, nhưng ta giả định rằng khi node *có* phản hồi thì nó nói "sự thật" — ít nhất theo hiểu biết của nó, nó đang tuân theo quy tắc của giao thức.

Nếu node có khả năng "nói dối", tức gửi phản hồi sai hoặc hỏng tùy ý, bài toán hệ thống phân tán sẽ khó hơn nhiều. Chẳng hạn, một node có thể bỏ nhiều phiếu mâu thuẫn nhau trong cùng một vòng bầu cử. Hành vi này gọi là *lỗi Byzantine* (*Byzantine fault*), còn bài toán đạt đồng thuận trong môi trường không ai tin ai như vậy gọi là *bài toán các vị tướng Byzantine* (*Byzantine Generals Problem*) [^94].

> [!TIP] Bài toán các vị tướng Byzantine
>
> Bài toán các vị tướng Byzantine là sự tổng quát hóa của *bài toán hai vị tướng* (*Two Generals Problem*) [^95]. Bài toán hai vị tướng hình dung hai tướng lĩnh phải thống nhất kế hoạch tác chiến nhưng đóng quân ở hai nơi khác nhau, chỉ có thể liên lạc bằng cách cử người đưa tin, mà người đưa tin đôi khi đến muộn hoặc mất tích (như gói tin mạng). Ta sẽ bàn bài toán *đồng thuận* (*consensus*) này ở [Chương 10](/ch10#ch_consistency).
>
> Trong phiên bản Byzantine của bài toán, có *n* vị tướng cần thống nhất, nhưng trong số đó lẫn một số kẻ phản bội. Phần lớn các tướng trung thành và gửi thông điệp thật, còn kẻ phản bội có thể gửi thông điệp giả để lừa dối và gây rối những người khác. Ai là kẻ phản bội thì không thể biết trước.
>
> Byzantium vốn là một thành phố Hy Lạp cổ, sau này thành Constantinople, nay là Istanbul của Thổ Nhĩ Kỳ. Không có bằng chứng lịch sử nào cho thấy tướng lĩnh Byzantium thích mưu mô hơn tướng lĩnh nơi khác. Cái tên thực ra bắt nguồn từ nghĩa của từ *Byzantine* trong ngữ cảnh chính trị: *phức tạp quá mức, quan liêu và xảo quyệt*. Cách dùng này có từ rất lâu trước khi có máy tính [^96]. Lamport muốn chọn một quốc tịch không làm mếch lòng độc giả, và người ta khuyên ông đừng gọi bài toán là *bài toán các vị tướng Albania* [^97].


Một hệ thống được gọi là có khả năng *chịu lỗi Byzantine* (*Byzantine fault tolerance*, BFT) nếu nó vẫn chạy đúng ngay cả khi một số node gặp lỗi, không tuân theo giao thức, hoặc có kẻ tấn công độc hại can thiệp vào mạng. Loại vấn đề này thực sự quan trọng trong một số tình huống đặc thù. Ví dụ:

* Trong môi trường hàng không vũ trụ, bức xạ có thể làm hỏng dữ liệu trong bộ nhớ hoặc thanh ghi CPU, khiến node phản hồi các node khác theo cách tùy ý và không thể đoán trước. Hệ thống hỏng sẽ có cái giá cực kỳ đắt — ví dụ máy bay rơi khiến mọi người trên khoang thiệt mạng, hoặc tên lửa đâm vào Trạm vũ trụ quốc tế — nên hệ thống điều khiển bay phải chịu được lỗi Byzantine [^98] [^99].
* Trong hệ thống có nhiều bên tham gia, một số bên có thể cố lừa dối hoặc gian lận những bên khác. Khi đó node không thể tin ngay thông điệp từ node khác, vì thông điệp có thể có ác ý. Tiền mã hóa như Bitcoin và các blockchain khác có thể xem là một cách để các bên không tin nhau đồng thuận về việc một giao dịch có xảy ra hay không mà không cần dựa vào cơ quan trung tâm [^100].

Tuy nhiên, với các hệ thống được bàn trong cuốn sách này, thường có thể yên tâm giả định không có lỗi Byzantine. Mọi node trong trung tâm dữ liệu đều do tổ chức của bạn kiểm soát nên hy vọng đáng tin, và mức bức xạ đủ thấp để hỏng bộ nhớ không phải vấn đề chính — dù người ta đã bắt đầu tính chuyện đưa trung tâm dữ liệu lên quỹ đạo [^101]. Các tenant trong hệ thống đa người thuê không tin nhau, nhưng hệ thống dựa vào tường lửa, ảo hóa và chính sách kiểm soát truy cập để cô lập tenant với nhau, chứ không dùng chịu lỗi Byzantine. Giao thức giúp hệ thống chịu lỗi Byzantine có chi phí cao [^102], còn hệ thống nhúng có khả năng chịu lỗi lại dựa vào hỗ trợ ở tầng phần cứng [^98]. Với đa số hệ thống dữ liệu phía server, chi phí triển khai giải pháp chịu lỗi Byzantine cao đến mức không thực tế.

Ứng dụng web quả thật phải lường trước rằng client do người dùng cuối kiểm soát, như trình duyệt web, có thể hành xử tùy ý, thậm chí độc hại. Đó là lý do kiểm tra đầu vào, làm sạch dữ liệu và escape đầu ra quan trọng đến vậy, chẳng hạn để chống SQL injection và cross-site scripting. Tuy nhiên, thường ta không dùng giao thức chịu lỗi Byzantine cho việc này, mà chỉ cần để server đóng vai thẩm quyền quyết định hành vi nào của client được chấp nhận, hành vi nào không. Trong mạng ngang hàng không có thẩm quyền trung tâm như vậy, chịu lỗi Byzantine mới quan trọng hơn [^103] [^104].

Lỗi phần mềm cũng có thể coi là lỗi Byzantine, nhưng nếu mọi node triển khai cùng một phần mềm thì thuật toán chịu lỗi Byzantine cũng không cứu được bạn. Hầu hết thuật toán chịu lỗi Byzantine yêu cầu hơn hai phần ba số node chạy bình thường (ví dụ trong bốn node chỉ được có tối đa một node lỗi). Muốn dùng cách này để chống lỗi phần mềm, bạn phải có bốn bản hiện thực độc lập của cùng một phần mềm và cầu mong lỗi chỉ xuất hiện ở một bản.

Tương tự, nếu có giao thức bảo vệ ta khỏi lỗ hổng, xâm phạm an ninh và tấn công độc hại thì tất nhiên rất hấp dẫn. Tiếc là điều này cũng không thực tế: trong đa số hệ thống, kẻ tấn công đã xâm nhập được một node thì nhiều khả năng cũng xâm nhập được mọi node, vì các node nhiều khả năng đang chạy cùng một phần mềm. Vì vậy các cơ chế truyền thống như xác thực, kiểm soát truy cập, mã hóa và tường lửa vẫn là phương tiện chính để chống kẻ tấn công.

<a id="sec_distributed_weak_lying"></a>

#### Các dạng nói dối yếu {#weak-forms-of-lying}

Dù thường giả định node trung thực, vẫn nên đưa vào phần mềm một số cơ chế chống các dạng "nói dối" yếu hơn, chẳng hạn thông điệp không hợp lệ do vấn đề phần cứng, lỗi phần mềm hoặc cấu hình sai. Các cơ chế này chưa phải chịu lỗi Byzantine đầy đủ, vì không chặn được kẻ tấn công quyết tâm, nhưng chúng vẫn là những bước đơn giản và thực dụng để tăng độ tin cậy. Ví dụ:

* Vấn đề phần cứng, hoặc lỗi trong hệ điều hành, driver, router và các thành phần khác, thật sự có thể làm hỏng gói tin mạng. Checksum tích hợp sẵn trong TCP và UDP thường phát hiện được gói tin hỏng, nhưng thỉnh thoảng cũng bỏ sót [^105] [^106] [^107]. Thường chỉ cần vài biện pháp đơn giản là đủ chống loại hỏng này, chẳng hạn thêm checksum vào giao thức tầng ứng dụng. Kết nối mã hóa TLS cũng chống được dữ liệu hỏng.
* Ứng dụng truy cập công khai phải làm sạch cẩn thận mọi đầu vào của người dùng, ví dụ kiểm tra giá trị số nằm trong khoảng hợp lý và giới hạn độ dài chuỗi để kẻ tấn công không thể cấp phát lượng bộ nhớ khổng lồ gây từ chối dịch vụ. Dịch vụ nội bộ sau tường lửa có thể nới lỏng kiểm tra đầu vào, nhưng parser của giao thức vẫn nên giữ các kiểm tra cơ bản [^105].
* Client NTP có thể cấu hình nhiều địa chỉ server. Khi đồng bộ, client liên hệ mọi server, ước lượng sai số của từng server và kiểm tra xem đa số server có thống nhất về một khoảng thời gian không. Chỉ cần phần lớn server bình thường, server NTP cấu hình sai báo thời gian sai sẽ bị nhận diện là giá trị ngoại lai và bị loại khỏi quá trình đồng bộ [^39]. Dùng nhiều server giúp NTP vững chắc hơn so với chỉ dựa vào một server.

### Mô hình hệ thống và thực tế {#sec_distributed_system_model}

Người ta đã thiết kế nhiều thuật toán để giải quyết các vấn đề của hệ thống phân tán. Ví dụ, ta sẽ xem xét các giải pháp cho bài toán đồng thuận ở [Chương 10](/ch10#ch_consistency). Để thật sự hữu ích, các thuật toán này phải chịu được các loại lỗi của hệ thống phân tán đã bàn trong chương này.

Cách viết thuật toán không nên phụ thuộc quá nhiều vào cấu hình phần cứng và phần mềm cụ thể của môi trường chạy. Điều này đòi hỏi ta phải bằng cách nào đó hình thức hóa các loại lỗi dự kiến xảy ra trong hệ thống. Vì vậy ta định nghĩa *mô hình hệ thống* (*system model*): một sự trừu tượng hóa mô tả những giả định mà thuật toán được phép đưa ra.

Về giả định thời gian, thường dùng ba mô hình hệ thống sau:

*mô hình đồng bộ* (*synchronous model*)
: Mô hình đồng bộ giả định độ trễ mạng, khoảng dừng tiến trình và sai số đồng hồ đều có cận trên. Điều này không có nghĩa đồng hồ hoàn toàn đồng bộ, cũng không có nghĩa độ trễ mạng bằng không. Nó chỉ có nghĩa là ta biết độ trễ mạng, khoảng dừng và độ trôi đồng hồ sẽ không bao giờ vượt quá một giới hạn cố định [^108]. Với đa số hệ thống thực tế, mô hình đồng bộ không thực tế, vì như chương này đã bàn, độ trễ và khoảng dừng không giới hạn là có thật.

*mô hình đồng bộ một phần* (*partially synchronous model*)
: Đồng bộ một phần nghĩa là hệ thống *phần lớn thời gian* chạy như hệ thống đồng bộ, nhưng thỉnh thoảng vượt quá cận của độ trễ mạng, khoảng dừng tiến trình và độ trôi đồng hồ [^108]. Với nhiều hệ thống, đây là mô hình thực tế: phần lớn thời gian mạng và tiến trình hành xử khá quy củ, nếu không ta chẳng làm được việc gì, nhưng ta cũng phải đối mặt với thực tế là mọi giả định về thời gian đều có lúc thất bại. Khi đó, độ trễ mạng, khoảng dừng tiến trình và sai số đồng hồ có thể lớn tùy ý.

*mô hình bất đồng bộ* (*asynchronous model*)
: Trong mô hình này, thuật toán không được đưa ra bất kỳ giả định thời gian nào — thực tế nó thậm chí không có đồng hồ để dùng (nên cũng không dùng được timeout). Một số thuật toán có thể thiết kế cho mô hình bất đồng bộ, nhưng mô hình này hạn chế rất nhiều.

Ngoài vấn đề thời gian, còn phải xét đến sự cố node. Các mô hình hệ thống node thường gặp gồm:

Lỗi crash-stop (*crash-stop fault*)
: Trong mô hình *crash-stop* (hay *fail-stop*), thuật toán có thể giả định node chỉ hỏng theo một cách: sập [^109]. Node có thể đột ngột ngừng phản hồi bất cứ lúc nào, sau đó biến mất vĩnh viễn, không bao giờ quay lại.

Lỗi crash-recovery (*crash-recovery fault*)
: Ta giả định node có thể sập bất cứ lúc nào và có thể bắt đầu phản hồi lại sau một khoảng thời gian không xác định. Trong mô hình crash-recovery, node có bộ lưu trữ ổn định (tức lưu trữ đĩa không mất dữ liệu khi mất điện) giữ được dữ liệu sau khi sập, nhưng trạng thái trong bộ nhớ bị mất.

Suy giảm hiệu năng và chức năng không đầy đủ
: Ngoài sập và khởi động lại, node còn có thể chậm: chúng vẫn có thể trả lời health check nhưng chậm đến mức không làm được việc thực sự nào. Ví dụ, card mạng gigabit có thể đột ngột tụt thông lượng xuống 1 Kb/s do lỗi driver [^110], tiến trình chịu áp lực bộ nhớ có thể dành phần lớn thời gian cho garbage collection [^111], SSD bị mòn có thể hoạt động cực kỳ thất thường, còn phần cứng có thể bị ảnh hưởng bởi nhiệt độ cao, đầu nối lỏng, rung cơ học, vấn đề nguồn điện, lỗi firmware và nhiều yếu tố khác [^112]. Những tình huống này gọi là *limping node*, *gray failure* hay *fail-slow* [^113], thậm chí có thể khó xử lý hơn node hỏng hẳn. Còn một vấn đề liên quan: tiến trình không còn thực hiện một số việc lẽ ra phải làm, trong khi các chức năng khác vẫn chạy, chẳng hạn khi thread nền bị sập hoặc deadlock [^114].

Lỗi Byzantine (tùy ý)
: Node có thể làm bất cứ điều gì, kể cả lừa dối các node khác như đã mô tả ở phần trước.

Khi mô hình hóa hệ thống thực tế, mô hình đồng bộ một phần kèm lỗi crash-recovery thường hữu ích nhất. Nó cho phép độ trễ mạng không giới hạn, khoảng dừng tiến trình và node chậm. Vậy thuật toán phân tán đối phó với mô hình này thế nào?

#### Định nghĩa tính đúng đắn của thuật toán {#defining-the-correctness-of-an-algorithm}

Để định nghĩa một thuật toán thế nào là *đúng*, có thể mô tả các *thuộc tính* nó phải có. Ví dụ, kết quả của thuật toán sắp xếp có thuộc tính sau: với hai phần tử khác nhau bất kỳ trong danh sách kết quả, phần tử bên trái nhỏ hơn phần tử bên phải. Đây chỉ là cách định nghĩa hình thức thế nào là danh sách "đã sắp xếp".

Tương tự, ta có thể liệt kê các thuộc tính mà thuật toán phân tán cần có để định nghĩa thế nào là đúng. Ví dụ, nếu thuật toán sinh fencing token cho một khóa (xem ["Dùng fencing để chặn zombie và yêu cầu bị trễ"](/ch9#sec_distributed_fencing_tokens)), ta có thể yêu cầu nó thỏa các thuộc tính sau:

Tính duy nhất
: Không có hai yêu cầu fencing token nào trả về cùng một giá trị.

Dãy đơn điệu
: Nếu yêu cầu *x* trả về token *t*<sub>*x*</sub>, yêu cầu *y* trả về token *t*<sub>*y*</sub>, và *x* đã hoàn tất trước khi *y* bắt đầu, thì *t*<sub>*x*</sub> < *t*<sub>*y*</sub>.

Tính sẵn sàng
: Node yêu cầu fencing token và không sập thì cuối cùng sẽ nhận được phản hồi.

Thuật toán được coi là đúng trong một mô hình hệ thống nếu nó luôn thỏa các thuộc tính này trong mọi tình huống mà mô hình hệ thống cho phép xảy ra. Tuy nhiên, nếu mọi node đều sập, hoặc mọi độ trễ mạng đột nhiên trở nên vô hạn, thì không thuật toán nào làm được việc. Với một mô hình cho phép hệ thống hỏng hoàn toàn, làm sao ta vẫn có thể đưa ra bảo đảm hữu ích?

#### Safety và liveness {#sec_distributed_safety_liveness}

Để làm rõ vấn đề này, cần phân biệt hai loại thuộc tính: *safety* và *liveness*. Trong ví dụ vừa rồi, *tính duy nhất* và *dãy đơn điệu* là thuộc tính safety, còn *tính sẵn sàng* là thuộc tính liveness.

Hai loại này khác nhau ở đâu? Một manh mối rõ ràng là định nghĩa của thuộc tính liveness thường chứa từ "cuối cùng". (Đúng vậy, bạn đoán đúng rồi: *nhất quán cuối cùng* (*eventual consistency*) là một thuộc tính liveness [^115].)

Safety thường được định nghĩa không chính thức là *điều xấu không xảy ra*, còn liveness là *điều tốt cuối cùng sẽ xảy ra*. Tuy nhiên, không nên hiểu cách nói bình dân này quá sâu, vì "tốt" và "xấu" là phán đoán giá trị, không thật hợp để mô tả thuật toán. Định nghĩa chặt chẽ của safety và liveness chính xác hơn nhiều: [^116]

* Nếu thuộc tính safety bị vi phạm, ta có thể chỉ ra thời điểm cụ thể nó bị phá vỡ. Ví dụ, nếu tính duy nhất bị vi phạm, ta có thể tìm ra thao tác cụ thể đã trả về fencing token trùng lặp. Một khi thuộc tính safety đã bị vi phạm thì không thể hoàn tác — thiệt hại đã xảy ra.
* Thuộc tính liveness thì ngược lại: tại một thời điểm nào đó nó có thể chưa đúng (ví dụ node đã gửi yêu cầu nhưng chưa nhận được phản hồi), nhưng trong tương lai vẫn luôn còn hy vọng nó được thỏa (tức là cuối cùng nhận được phản hồi).

Lợi ích của việc phân biệt thuộc tính safety và liveness là giúp ta xử lý các mô hình hệ thống khó. Với thuật toán phân tán, thường yêu cầu thuộc tính safety *luôn* đúng trong mọi tình huống mà mô hình hệ thống cho phép [^108]. Nghĩa là dù mọi node sập, hay cả mạng hỏng, thuật toán vẫn phải bảo đảm không bao giờ trả về kết quả sai, và do đó thuộc tính safety vẫn được thỏa.

Còn với thuộc tính liveness, ta có thể đặt điều kiện. Ví dụ, có thể quy định chỉ khi đa số node chưa sập và mạng cuối cùng phục hồi sau gián đoạn thì yêu cầu mới phải nhận được phản hồi. Định nghĩa của mô hình đồng bộ một phần yêu cầu hệ thống cuối cùng quay về trạng thái đồng bộ — nghĩa là mọi gián đoạn mạng chỉ kéo dài trong thời gian hữu hạn rồi được khắc phục.

#### Ánh xạ mô hình hệ thống vào thế giới thực {#mapping-system-models-to-the-real-world}

Thuộc tính safety, thuộc tính liveness và mô hình hệ thống rất hữu ích khi suy luận về tính đúng đắn của thuật toán phân tán. Nhưng khi hiện thực thuật toán trong thực tế, mặt lộn xộn của thế giới thực lại quay lại gây rắc rối, và lúc đó ta thấy rõ mô hình hệ thống chỉ là sự trừu tượng hóa đơn giản của thực tế.

Ví dụ, thuật toán trong mô hình crash-recovery thường giả định dữ liệu trong bộ lưu trữ ổn định sống sót qua lần sập. Nhưng nếu dữ liệu đĩa bị hỏng, hoặc bị xóa do lỗi phần cứng hay cấu hình sai thì sao [^117]? Nếu server có lỗi firmware, khi khởi động lại không nhận ra ổ cứng dù ổ cứng đã được nối đúng thì sao [^118]?

Các thuật toán quorum (xem ["Đọc và ghi quorum"](/ch6#sec_replication_quorum_condition)) dựa vào việc node nhớ dữ liệu mà nó tuyên bố đã lưu. Nếu node mắc "chứng mất trí nhớ", quên dữ liệu đã lưu trước đó, điều kiện quorum bị phá vỡ, kéo theo tính đúng đắn của thuật toán cũng bị phá vỡ. Có lẽ cần định nghĩa mô hình hệ thống mới: giả định bộ lưu trữ ổn định thường vẫn còn sau khi sập nhưng đôi khi cũng bị mất. Nhưng mô hình như vậy lại càng khó suy luận.

Mô tả lý thuyết của thuật toán có thể tuyên bố thẳng là giả định một số chuyện không bao giờ xảy ra — trong hệ thống không Byzantine, ta thật sự phải đưa ra giả định về loại lỗi nào có thể xảy ra, loại nào không. Tuy nhiên, bản hiện thực thực tế vẫn có thể phải có mã xử lý những chuyện "không thể" xảy ra về lý thuyết, dù cách xử lý chỉ gói gọn trong `printf("Sucks to be you")` và `exit(666)`, tức là để người vận hành dọn dẹp tàn cuộc [^119]. (Đây chính là một điểm khác nhau giữa khoa học máy tính và kỹ nghệ phần mềm.)

Điều này không có nghĩa mô hình hệ thống lý thuyết, trừu tượng là vô giá trị — hoàn toàn ngược lại. Chúng cực kỳ hữu ích để chắt lọc sự phức tạp của hệ thống thực thành một tập lỗi có thể quản lý và suy luận được, giúp ta hiểu vấn đề và cố giải quyết nó một cách có hệ thống.

### Phương pháp hình thức và kiểm thử ngẫu nhiên {#sec_distributed_formal}

Làm sao biết một thuật toán thật sự thỏa các thuộc tính cần thiết? Tính đồng thời, hỏng hóc từng phần và độ trễ mạng tạo ra số trạng thái tiềm tàng khổng lồ. Ta cần bảo đảm các thuộc tính này đúng ở mọi trạng thái có thể, và không bỏ sót trường hợp biên nào.

Một cách là *xác minh hình thức* (*formal verification*) thuật toán: mô tả thuật toán bằng ngôn ngữ toán học rồi dùng kỹ thuật chứng minh để chứng minh nó thỏa các thuộc tính cần thiết trong mọi tình huống mà mô hình hệ thống cho phép. Chứng minh một thuật toán đúng không có nghĩa là *bản hiện thực* của nó trong hệ thống thực chắc chắn luôn chạy đúng. Dù vậy, đây là một bước đầu tiên rất tốt, vì phân tích lý thuyết có thể phát hiện những vấn đề của thuật toán vốn có thể nằm im rất lâu trong hệ thống thực cho đến khi một tình huống bất thường phá vỡ giả định của bạn (ví dụ giả định về thời gian) và chúng bùng phát.

Kết hợp phân tích lý thuyết với kiểm thử thực nghiệm để xác minh hành vi của bản hiện thực có đúng như kỳ vọng là cách làm vững vàng. Các kỹ thuật như *kiểm thử dựa trên thuộc tính* (*property-based testing*), *fuzz testing* và *kiểm thử mô phỏng tất định* (*deterministic simulation testing*, DST) đều dùng ngẫu nhiên hóa để kiểm thử hệ thống trong nhiều tình huống khác nhau. Các công ty như Amazon Web Services đã kết hợp thành công những kỹ thuật này trong nhiều sản phẩm [^120] [^121].

#### Model checking và ngôn ngữ đặc tả {#model-checking-and-specification-languages}

*Model checker* là công cụ giúp xác minh hành vi của thuật toán hay hệ thống có đúng kỳ vọng không. Đặc tả thuật toán phải viết bằng ngôn ngữ chuyên dụng như TLA+, Gallina hay FizzBee. Nhờ các ngôn ngữ này, ta có thể tập trung vào hành vi của thuật toán mà không bị sa vào chi tiết hiện thực. Model checker sau đó thử một cách có hệ thống các tình huống có thể xảy ra, dùng mô hình để xác minh các bất biến có đúng ở mọi trạng thái của thuật toán không.

Nói chính xác, model checking không chứng minh được bất biến của thuật toán đúng ở mọi trạng thái có thể, vì không gian trạng thái của đa số thuật toán thực tế là vô hạn. Muốn xác minh mọi trạng thái thật sự, cần chứng minh hình thức, việc này khả thi nhưng thường khó hơn nhiều so với chạy model checker. Vì vậy, khi dùng model checker, thường phải thu nhỏ mô hình thuật toán thành một phiên bản xấp xỉ có thể xác minh trọn vẹn, hoặc đặt một giới hạn nào đó cho quá trình thực thi (như giới hạn số thông điệp tối đa được gửi). Như vậy, những lỗi chỉ xuất hiện trong các lần thực thi dài hơn sẽ không bị phát hiện.

Dù vậy, model checker vẫn đạt được sự cân bằng tốt giữa độ dễ dùng và khả năng tìm ra lỗi tinh vi. CockroachDB, TiDB, Kafka và nhiều hệ thống phân tán khác đều dùng đặc tả mô hình để tìm và sửa lỗi [^122] [^123] [^124]. Chẳng hạn, các nhà nghiên cứu đã dùng TLA+ để chỉ ra khả năng mất dữ liệu ở Viewstamped Replication (VR) do mô tả bằng lời của thuật toán có chỗ mơ hồ [^125].

Theo thiết kế, model checker không chạy mã thực mà chạy một mô hình đơn giản hóa chỉ mô tả ý tưởng cốt lõi của giao thức. Điều này giúp khám phá không gian trạng thái một cách có hệ thống dễ hơn, nhưng cũng kéo theo rủi ro đặc tả và bản hiện thực dần lệch nhau [^126]. Có thể kiểm tra hành vi của mô hình và bản hiện thực thực tế có tương đương không, nhưng việc này đòi hỏi gắn thêm công cụ đo vào bản hiện thực thực tế [^127].

#### Chèn lỗi {#sec_fault_injection}

Nhiều lỗi chỉ phát sinh khi máy hoặc mạng gặp sự cố. *Chèn lỗi* (*fault injection*) là một kỹ thuật hiệu quả (đôi khi khá đáng sợ) để xác minh bản hiện thực hệ thống có chạy đúng kỳ vọng khi gặp sự cố không. Ý tưởng rất đơn giản: đưa lỗi vào môi trường của hệ thống đang chạy, rồi quan sát hệ thống phản ứng thế nào. Lỗi chèn vào có thể là sự cố mạng, máy sập, đĩa hỏng, tiến trình bị dừng — mọi cách máy tính có thể hỏng mà bạn nghĩ ra được đều có thể thử.

Kiểm thử chèn lỗi thường chạy trong môi trường rất giống môi trường sản xuất thật của hệ thống. Một số nhóm thậm chí chèn lỗi thẳng vào môi trường sản xuất. Netflix đã phổ biến cách làm này qua công cụ Chaos Monkey [^128]. Chèn lỗi trong môi trường sản xuất thường gọi là *chaos engineering*, ta đã bàn ở ["Độ tin cậy và khả năng chịu lỗi"](/ch2#sec_introduction_reliability).

Khi chạy kiểm thử chèn lỗi, trước hết phải triển khai hệ thống cần kiểm thử, cùng bộ điều phối chèn lỗi và các script. Bộ điều phối quyết định chèn lỗi nào, khi nào, còn các script cục bộ hoặc từ xa thì làm cho từng node hay tiến trình hỏng. Script chèn lỗi dùng nhiều công cụ khác nhau để kích hoạt lỗi: tiến trình Linux có thể dừng hoặc kết thúc bằng lệnh `kill`, đĩa có thể gỡ bằng `umount`, kết nối mạng có thể bị cắt bằng quy tắc tường lửa. Kiểm tra hành vi hệ thống trong và sau khi chèn lỗi sẽ xác nhận hệ thống có chạy đúng kỳ vọng không.

Các công cụ cần để kích hoạt từng loại lỗi rất đa dạng, nên viết kiểm thử chèn lỗi khá rườm rà. Cách làm phổ biến là dùng framework chèn lỗi như Jepsen để đơn giản hóa quy trình. Các framework này tích hợp nhiều hệ điều hành và có sẵn nhiều bộ chèn lỗi [^129]. Jepsen đã thành công rực rỡ trong việc tìm ra các lỗi nghiêm trọng ở nhiều hệ thống được dùng rộng rãi [^130] [^131].

#### Kiểm thử mô phỏng tất định {#deterministic-simulation-testing}

Kiểm thử mô phỏng tất định (DST) cũng đã trở thành một bổ sung phổ biến cho model checking và chèn lỗi. Nó khám phá không gian trạng thái theo cách giống model checker, nhưng kiểm thử mã thực chứ không phải mô hình.

Trong DST, bộ mô phỏng tự động chạy rất nhiều lần thực thi ngẫu nhiên hóa của hệ thống. Giao tiếp mạng, I/O và đo thời gian của đồng hồ trong lúc mô phỏng đều được thay bằng các thành phần mô phỏng, để bộ mô phỏng kiểm soát chính xác thứ tự mọi việc xảy ra trong các tình huống thời gian và lỗi khác nhau. Nhờ vậy bộ mô phỏng khám phá được nhiều tình huống hơn hẳn kiểm thử viết tay hay chèn lỗi. Nếu kiểm thử thất bại, có thể chạy lại, vì bộ mô phỏng biết chính xác chuỗi thao tác gây ra thất bại, điều mà chèn lỗi không làm được vì không kiểm soát hệ thống chi tiết đến vậy.

DST đòi hỏi bộ mô phỏng kiểm soát được mọi nguồn phi tất định như độ trễ mạng. Để mã trở nên tất định, thường dùng một trong ba chiến lược sau:

Tầng ứng dụng
: Một số hệ thống được xây dựng ngay từ đầu với mục tiêu dễ thực thi mã một cách tất định. Ví dụ, FoundationDB là một trong những hệ thống tiên phong về DST, được xây dựng trên thư viện giao tiếp bất đồng bộ tên Flow. Flow cho nhà phát triển một điểm chèn để đưa mô phỏng mạng tất định vào hệ thống [^132]. Tương tự, TigerBeetle là cơ sở dữ liệu xử lý giao dịch trực tuyến (OLTP) hỗ trợ DST từ gốc. Nó mô hình hóa trạng thái hệ thống thành một máy trạng thái, mọi thay đổi trạng thái đều diễn ra trong một event loop duy nhất. Kết hợp với các primitive mô phỏng tất định như đồng hồ, kiến trúc này có thể chạy một cách tất định [^133].

Tầng runtime
: Các ngôn ngữ lập trình có runtime bất đồng bộ và thư viện phổ biến cung cấp điểm chèn để đưa tính tất định vào. Có thể dùng runtime đơn luồng để ép mọi mã bất đồng bộ chạy tuần tự. Ví dụ, FrostDB sửa runtime của Go để các goroutine chạy lần lượt [^134]. Thư viện madsim của Rust cũng dùng cách tương tự. Madsim cung cấp bản hiện thực tất định cho API của runtime bất đồng bộ Tokio, thư viện AWS S3, thư viện Kafka dành cho Rust và nhiều thành phần khác. Ứng dụng chỉ cần đổi sang thư viện và runtime tất định, không phải sửa mã của chính nó, là có thể chạy kiểm thử tất định.

Tầng máy
: Thay vì sửa mã ở runtime, cũng có thể làm cho cả máy trở nên tất định. Quá trình này rất tinh vi: máy phải trả lời một cách tất định cho mọi lời gọi vốn có tính phi tất định. Các công cụ như Antithesis làm được việc này bằng cách xây dựng một hypervisor tùy biến, thay các thao tác vốn phi tất định bằng thao tác tất định. Từ đồng hồ đến mạng đến lưu trữ, mọi thứ đều phải được tính đến. Xong rồi, nhà phát triển có thể chạy toàn bộ hệ thống phân tán trong một nhóm container bên trong hypervisor và có được một hệ thống phân tán hoàn toàn tất định.

Lợi thế của DST không chỉ là khả năng chạy lại. Khi các công cụ như Antithesis phát hiện hành vi tương đối hiếm gặp, chúng tách một lần thực thi kiểm thử thành nhiều lần thực thi con, nhằm khám phá thêm nhiều đường đi mã trong ứng dụng. Kiểm thử tất định thường dùng đồng hồ và lời gọi mạng mô phỏng, nên chạy nhanh hơn thời gian trôi qua ngoài đời. Ví dụ, phần trừu tượng hóa thời gian của TigerBeetle có thể mô phỏng độ trễ mạng và timeout mà không cần thật sự chờ đủ lâu để timeout kích hoạt. Các kỹ thuật này cho phép bộ mô phỏng khám phá nhiều đường đi mã hơn với tốc độ nhanh hơn.

<a id="sidebar_distributed_determinism"></a>

> [!TIP] Sức mạnh của tính tất định
>
> Phi tất định chính là cốt lõi của mọi khó khăn trong hệ thống phân tán mà chương này đã bàn: tính đồng thời, độ trễ mạng, tiến trình bị dừng, đồng hồ nhảy và sự cố sập đều xảy ra theo cách không thể đoán trước, và mỗi lần chạy hệ thống có thể khác nhau. Ngược lại, nếu làm cho hệ thống tất định được thì nhiều thứ sẽ đơn giản hơn hẳn.
>
> Thật ra, làm cho mọi thứ tất định là một ý tưởng đơn giản mà mạnh mẽ, lặp đi lặp lại trong thiết kế hệ thống phân tán. Ngoài kiểm thử mô phỏng tất định, các chương trước đã giới thiệu nhiều cách khác tận dụng tính tất định:
>
> * Một lợi thế then chốt của event sourcing (xem ["Event sourcing và CQRS"](/ch3#sec_datamodels_events)) là có thể phát lại log sự kiện một cách tất định để dựng lại các khung nhìn vật chất hóa dẫn xuất.
> * Workflow engine (xem ["Durable execution và workflow"](/ch5#sec_encoding_dataflow_workflows)) dựa vào định nghĩa workflow tất định để cung cấp ngữ nghĩa durable execution.
> * *Sao chép máy trạng thái* (*state machine replication*), mà ta sẽ bàn ở ["Sử dụng shared log"](/ch10#sec_consistency_smr), đồng bộ dữ liệu giữa các bản sao bằng cách thực thi độc lập cùng một chuỗi giao dịch tất định trên mỗi bản sao. Ta đã gặp hai biến thể của ý tưởng này: sao chép (*replication*) dựa trên câu lệnh (xem ["Triển khai replication log"](/ch6#sec_replication_implementation)) và thực thi tuần tự giao dịch qua thủ tục lưu trữ (xem ["Ưu và nhược điểm của thủ tục lưu trữ"](/ch8#sec_transactions_stored_proc_tradeoffs)).
>
> Tuy nhiên, để mã hoàn toàn tất định vẫn phải hết sức cẩn thận. Kể cả đã loại bỏ mọi tính đồng thời, và đã thay I/O, giao tiếp mạng, đồng hồ và bộ sinh số ngẫu nhiên bằng mô phỏng tất định, trong hệ thống vẫn có thể còn sót phi tất định. Ví dụ, trong một số ngôn ngữ lập trình, thứ tự duyệt các phần tử của bảng băm có thể không tất định. Việc có chạm giới hạn tài nguyên hay không (cấp phát bộ nhớ thất bại, tràn stack) cũng mang tính phi tất định.

## Tóm tắt {#summary}

Chương này đã bàn về nhiều vấn đề có thể xảy ra trong hệ thống phân tán, trong đó có:

* Mỗi khi cố gửi gói tin qua mạng, gói tin có thể bị mất hoặc bị trễ tùy ý. Phản hồi cũng có thể bị mất hoặc bị trễ, nên chừng nào chưa nhận được phản hồi, bạn không có cách nào biết thông điệp đã được chuyển đến hay chưa.
* Dù đã cố hết sức cấu hình NTP, đồng hồ của một node vẫn có thể lệch nghiêm trọng so với các node khác, và có thể đột ngột nhảy tiến hoặc lùi. Dựa vào đồng hồ như vậy là rất nguy hiểm, vì nhiều khả năng bạn không có cách đo tốt khoảng tin cậy của đồng hồ.
* Tiến trình có thể bị tạm dừng rất lâu ở một điểm bất kỳ trong quá trình thực thi, bị các node khác tuyên bố đã chết, rồi sau đó nó chạy lại mà hoàn toàn không nhận ra mình từng bị dừng.

Việc có thể xảy ra *hỏng hóc từng phần* như vậy chính là đặc trưng của hệ thống phân tán. Mỗi khi phần mềm cố làm bất cứ việc gì liên quan đến node khác, nó có thể thỉnh thoảng thất bại, chậm đi một cách ngẫu nhiên, hoặc không phản hồi gì cả (và cuối cùng bị timeout). Trong hệ thống phân tán, ta cố gắng xây khả năng chịu hỏng hóc từng phần vào phần mềm, để cả hệ thống vẫn chạy được ngay cả khi một số thành phần bị hỏng.

Bước đầu tiên để chịu lỗi là *phát hiện* lỗi, nhưng ngay cả bước này cũng khó. Hầu hết hệ thống không có cơ chế chính xác để biết một node đã hỏng hay chưa, nên đa số thuật toán phân tán chỉ có thể dựa vào timeout để xác định node ở xa còn dùng được không. Tuy nhiên, timeout không phân biệt được sự cố mạng với sự cố node, và độ trễ mạng dao động đôi khi khiến hệ thống nghi nhầm một node đã sập. Node què còn khó xử lý hơn: nó vẫn phản hồi, nhưng chậm đến mức không làm được việc gì hữu ích.

Ngay cả khi phát hiện được lỗi, bắt hệ thống chịu được lỗi cũng không dễ: giữa các máy không có biến toàn cục, không có bộ nhớ chung, không có tri thức chung, cũng không có hình thức trạng thái chia sẻ nào khác [^83]. Các node thậm chí không thể thống nhất với nhau bây giờ là mấy giờ, chứ đừng nói đến những vấn đề sâu xa hơn. Cách duy nhất để thông tin truyền từ node này sang node khác là gửi nó qua mạng không đáng tin cậy. Những quyết định quan trọng không thể giao an toàn cho một node đơn lẻ, nên ta cần giao thức để huy động các node khác tham gia và cố gắng đạt được đồng thuận giữa các node trong quorum.

Nếu bạn quen viết phần mềm trong môi trường lý tưởng hoàn hảo như toán học của một máy tính duy nhất — nơi cùng một thao tác luôn cho cùng một kết quả một cách tất định — thì chuyển sang thực tế vật lý lộn xộn của hệ thống phân tán khó tránh khỏi cú sốc. Ngược lại, các kỹ sư hệ thống phân tán thường cho rằng một vấn đề mà giải quyết được trên một máy thì là tầm thường [^4]. Thực tế, ngày nay một máy đơn làm được rất nhiều việc. Nếu tránh được việc mở chiếc hộp Pandora, cứ để công việc trên một máy, ví dụ dùng storage engine nhúng (xem ["Storage engine nhúng"](/ch4#sidebar_embedded)), thì thường đáng làm như vậy.

Tuy nhiên, như đã bàn ở ["Hệ thống phân tán và hệ thống một node"](/ch1#sec_introduction_distributed), khả năng mở rộng không phải lý do duy nhất để dùng hệ thống phân tán. Khả năng chịu lỗi và độ trễ thấp (đặt dữ liệu gần người dùng hơn về mặt địa lý) cũng quan trọng không kém, và những mục tiêu này không đạt được bằng một node đơn lẻ. Sức mạnh của hệ thống phân tán là về nguyên tắc nó có thể không bao giờ ngừng hoạt động ở mức dịch vụ, vì mọi sự cố và bảo trì đều xử lý được ở mức node. (Dĩ nhiên trong thực tế, nếu triển khai một cấu hình sai lên mọi node thì hệ thống phân tán vẫn bị hạ gục hoàn toàn.)

Chương này cũng đi lạc đề một chút để xem sự không đáng tin cậy của mạng, đồng hồ và tiến trình có phải là quy luật tự nhiên không thể tránh hay không. Câu trả lời là không: mạng có thể cung cấp bảo đảm phản hồi thời gian thực cứng và độ trễ có giới hạn, chỉ có điều cái giá cực kỳ đắt, và kéo theo mức sử dụng tài nguyên phần cứng thấp đi. Đa số hệ thống không liên quan đến an toàn sẽ chọn rẻ mà không đáng tin cậy thay vì đắt mà đáng tin cậy.

Suốt chương này ta toàn bàn về vấn đề, vẽ nên một bức tranh khá ảm đạm. Chương tiếp theo sẽ chuyển sang các giải pháp, bàn về một số thuật toán được thiết kế để đối phó với các vấn đề của hệ thống phân tán.



### Tài liệu tham khảo


[^1]: Mark Cavage. [There’s Just No Getting Around It: You’re Building a Distributed System](https://queue.acm.org/detail.cfm?id=2482856). *ACM Queue*, volume 11, issue 4, pages 80-89, April 2013. [doi:10.1145/2466486.2482856](https://doi.org/10.1145/2466486.2482856) 
[^2]: Jay Kreps. [Getting Real About Distributed System Reliability](https://blog.empathybox.com/post/19574936361/getting-real-about-distributed-system-reliability). *blog.empathybox.com*, March 2012. Archived at [perma.cc/9B5Q-AEBW](https://perma.cc/9B5Q-AEBW) 
[^3]: Coda Hale. [You Can’t Sacrifice Partition Tolerance](https://perma.cc/6GJU-X4G5). *codahale.com*, October 2010. <https://perma.cc/6GJU-X4G5>
[^4]: Jeff Hodges. [Notes on Distributed Systems for Young Bloods](https://www.somethingsimilar.com/2013/01/14/notes-on-distributed-systems-for-young-bloods/). *somethingsimilar.com*, January 2013. Archived at [perma.cc/B636-62CE](https://perma.cc/B636-62CE) 
[^5]: Van Jacobson. [Congestion Avoidance and Control](https://www.cs.usask.ca/ftp/pub/discus/seminars2002-2003/p314-jacobson.pdf). At *ACM Symposium on Communications Architectures and Protocols* (SIGCOMM), August 1988. [doi:10.1145/52324.52356](https://doi.org/10.1145/52324.52356) 
[^6]: Bert Hubert. [The Ultimate SO\_LINGER Page, or: Why Is My TCP Not Reliable](https://blog.netherlabs.nl/articles/2009/01/18/the-ultimate-so_linger-page-or-why-is-my-tcp-not-reliable). *blog.netherlabs.nl*, January 2009. Archived at [perma.cc/6HDX-L2RR](https://perma.cc/6HDX-L2RR) 
[^7]: Jerome H. Saltzer, David P. Reed, and David D. Clark. [End-To-End Arguments in System Design](https://groups.csail.mit.edu/ana/Publications/PubPDFs/End-to-End%20Arguments%20in%20System%20Design.pdf). *ACM Transactions on Computer Systems*, volume 2, issue 4, pages 277–288, November 1984. [doi:10.1145/357401.357402](https://doi.org/10.1145/357401.357402) 
[^8]: Peter Bailis and Kyle Kingsbury. [The Network Is Reliable](https://queue.acm.org/detail.cfm?id=2655736). *ACM Queue*, volume 12, issue 7, pages 48-55, July 2014. [doi:10.1145/2639988.2655736](https://doi.org/10.1145/2639988.2655736)
[^9]: Joshua B. Leners, Trinabh Gupta, Marcos K. Aguilera, and Michael Walfish. [Taming Uncertainty in Distributed Systems with Help from the Network](https://cs.nyu.edu/~mwalfish/papers/albatross-eurosys15.pdf). At *10th European Conference on Computer Systems* (EuroSys), April 2015. [doi:10.1145/2741948.2741976](https://doi.org/10.1145/2741948.2741976) 
[^10]: Phillipa Gill, Navendu Jain, and Nachiappan Nagappan. [Understanding Network Failures in Data Centers: Measurement, Analysis, and Implications](https://conferences.sigcomm.org/sigcomm/2011/papers/sigcomm/p350.pdf). At *ACM SIGCOMM Conference*, August 2011. [doi:10.1145/2018436.2018477](https://doi.org/10.1145/2018436.2018477) 
[^11]: Urs Hölzle. [But recently a farmer had started grazing a herd of cows nearby. And whenever they stepped on the fiber link, they bent it enough to cause a blip](https://x.com/uhoelzle/status/1263333283107991558). *x.com*, May 2020. Archived at [perma.cc/WX8X-ZZA5](https://perma.cc/WX8X-ZZA5) 
[^12]: CBC News. [Hundreds lose internet service in northern B.C. after beaver chews through cable](https://www.cbc.ca/news/canada/british-columbia/beaver-internet-down-tumbler-ridge-1.6001594). *cbc.ca*, April 2021. Archived at [perma.cc/UW8C-H2MY](https://perma.cc/UW8C-H2MY) 
[^13]: Will Oremus. [The Global Internet Is Being Attacked by Sharks, Google Confirms](https://slate.com/technology/2014/08/shark-attacks-threaten-google-s-undersea-internet-cables-video.html). *slate.com*, August 2014. Archived at [perma.cc/P6F3-C6YG](https://perma.cc/P6F3-C6YG) 
[^14]: Jess Auerbach Jahajeeah. [Down to the wire: The ship fixing our internet](https://continent.substack.com/p/down-to-the-wire-the-ship-fixing). *continent.substack.com*, November 2023. Archived at [perma.cc/DP7B-EQ7S](https://perma.cc/DP7B-EQ7S) 
[^15]: Santosh Janardhan. [More details about the October 4 outage](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/). *engineering.fb.com*, October 2021. Archived at [perma.cc/WW89-VSXH](https://perma.cc/WW89-VSXH) 
[^16]: Tom Parfitt. [Georgian woman cuts off web access to whole of Armenia](https://www.theguardian.com/world/2011/apr/06/georgian-woman-cuts-web-access). *theguardian.com*, April 2011. Archived at [perma.cc/KMC3-N3NZ](https://perma.cc/KMC3-N3NZ) 
[^17]: Antonio Voce, Tural Ahmedzade and Ashley Kirk. [‘Shadow fleets’ and subaquatic sabotage: are Europe’s undersea internet cables under attack?](https://www.theguardian.com/world/ng-interactive/2025/mar/05/shadow-fleets-subaquatic-sabotage-europe-undersea-internet-cables-under-attack) *theguardian.com*, March 2025. Archived at [perma.cc/HA7S-ZDBV](https://perma.cc/HA7S-ZDBV) 
[^18]: Shengyun Liu, Paolo Viotti, Christian Cachin, Vivien Quéma, and Marko Vukolić. [XFT: Practical Fault Tolerance beyond Crashes](https://www.usenix.org/system/files/conference/osdi16/osdi16-liu.pdf). At *12th USENIX Symposium on Operating Systems Design and Implementation* (OSDI), November 2016. 
[^19]: Mark Imbriaco. [Downtime last Saturday](https://github.blog/news-insights/the-library/downtime-last-saturday/). *github.blog*, December 2012. Archived at [perma.cc/M7X5-E8SQ](https://perma.cc/M7X5-E8SQ) 
[^20]: Tom Lianza and Chris Snook. [A Byzantine failure in the real world](https://blog.cloudflare.com/a-byzantine-failure-in-the-real-world/). *blog.cloudflare.com*, November 2020. Archived at [perma.cc/83EZ-ALCY](https://perma.cc/83EZ-ALCY) 
[^21]: Mohammed Alfatafta, Basil Alkhatib, Ahmed Alquraan, and Samer Al-Kiswany. [Toward a Generic Fault Tolerance Technique for Partial Network Partitioning](https://www.usenix.org/conference/osdi20/presentation/alfatafta). At *14th USENIX Symposium on Operating Systems Design and Implementation* (OSDI), November 2020. 
[^22]: Marc A. Donges. [Re: bnx2 cards Intermittantly Going Offline](https://www.spinics.net/lists/netdev/msg210485.html). Message to Linux *netdev* mailing list, *spinics.net*, September 2012. Archived at [perma.cc/TXP6-H8R3](https://perma.cc/TXP6-H8R3) 
[^23]: Troy Toman. [Inside a CODE RED: Network Edition](https://signalvnoise.com/svn3/inside-a-code-red-network-edition/). *signalvnoise.com*, September 2020. Archived at [perma.cc/BET6-FY25](https://perma.cc/BET6-FY25) 
[^24]: Kyle Kingsbury. [Call Me Maybe: Elasticsearch](https://aphyr.com/posts/317-call-me-maybe-elasticsearch). *aphyr.com*, June 2014. [perma.cc/JK47-S89J](https://perma.cc/JK47-S89J) 
[^25]: Salvatore Sanfilippo. [A Few Arguments About Redis Sentinel Properties and Fail Scenarios](https://antirez.com/news/80). *antirez.com*, October 2014. [perma.cc/8XEU-CLM8](https://perma.cc/8XEU-CLM8) 
[^26]: Nicolas Liochon. [CAP: If All You Have Is a Timeout, Everything Looks Like a Partition](http://blog.thislongrun.com/2015/05/CAP-theorem-partition-timeout-zookeeper.html). *blog.thislongrun.com*, May 2015. Archived at [perma.cc/FS57-V2PZ](https://perma.cc/FS57-V2PZ) 
[^27]: Matthew P. Grosvenor, Malte Schwarzkopf, Ionel Gog, Robert N. M. Watson, Andrew W. Moore, Steven Hand, and Jon Crowcroft. [Queues Don’t Matter When You Can JUMP Them!](https://www.usenix.org/system/files/conference/nsdi15/nsdi15-paper-grosvenor_update.pdf) At *12th USENIX Symposium on Networked Systems Design and Implementation* (NSDI), May 2015. 
[^28]: Theo Julienne. [Debugging network stalls on Kubernetes](https://github.blog/engineering/debugging-network-stalls-on-kubernetes/). *github.blog*, November 2019. Archived at [perma.cc/K9M8-XVGL](https://perma.cc/K9M8-XVGL) 
[^29]: Guohui Wang and T. S. Eugene Ng. [The Impact of Virtualization on Network Performance of Amazon EC2 Data Center](https://www.cs.rice.edu/~eugeneng/papers/INFOCOM10-ec2.pdf). At *29th IEEE International Conference on Computer Communications* (INFOCOM), March 2010. [doi:10.1109/INFCOM.2010.5461931](https://doi.org/10.1109/INFCOM.2010.5461931) 
[^30]: Brandon Philips. [etcd: Distributed Locking and Service Discovery](https://www.youtube.com/watch?v=HJIjTTHWYnE). At *Strange Loop*, September 2014. 
[^31]: Steve Newman. [A Systematic Look at EC2 I/O](https://www.sentinelone.com/blog/a-systematic-look-at-ec2-i-o/). *blog.scalyr.com*, October 2012. Archived at [perma.cc/FL4R-H2VE](https://perma.cc/FL4R-H2VE) 
[^32]: Naohiro Hayashibara, Xavier Défago, Rami Yared, and Takuya Katayama. [The ϕ Accrual Failure Detector](https://hdl.handle.net/10119/4784). Japan Advanced Institute of Science and Technology, School of Information Science, Technical Report IS-RR-2004-010, May 2004. Archived at [perma.cc/NSM2-TRYA](https://perma.cc/NSM2-TRYA) 
[^33]: Jeffrey Wang. [Phi Accrual Failure Detector](https://ternarysearch.blogspot.com/2013/08/phi-accrual-failure-detector.html). *ternarysearch.blogspot.co.uk*, August 2013. [perma.cc/L452-AMLV](https://perma.cc/L452-AMLV) 
[^34]: Srinivasan Keshav. *An Engineering Approach to Computer Networking: ATM Networks, the Internet, and the Telephone Network*. Addison-Wesley Professional, May 1997. ISBN: 978-0-201-63442-6 
[^35]: Othmar Kyas. *ATM Networks*. International Thomson Publishing, 1995. ISBN: 978-1-850-32128-6 
[^36]: Mellanox Technologies. [InfiniBand FAQ, Rev 1.3](https://network.nvidia.com/related-docs/whitepapers/InfiniBandFAQ_FQ_100.pdf). *network.nvidia.com*, December 2014. Archived at [perma.cc/LQJ4-QZVK](https://perma.cc/LQJ4-QZVK) 
[^37]: Jose Renato Santos, Yoshio Turner, and G. (John) Janakiraman. [End-to-End Congestion Control for InfiniBand](https://infocom2003.ieee-infocom.org/papers/28_01.PDF). At *22nd Annual Joint Conference of the IEEE Computer and Communications Societies* (INFOCOM), April 2003. Also published by HP Laboratories Palo Alto, Tech Report HPL-2002-359. [doi:10.1109/INFCOM.2003.1208949](https://doi.org/10.1109/INFCOM.2003.1208949) 
[^38]: Jialin Li, Naveen Kr. Sharma, Dan R. K. Ports, and Steven D. Gribble. [Tales of the Tail: Hardware, OS, and Application-level Sources of Tail Latency](https://syslab.cs.washington.edu/papers/latency-socc14.pdf). At *ACM Symposium on Cloud Computing* (SOCC), November 2014. [doi:10.1145/2670979.2670988](https://doi.org/10.1145/2670979.2670988) 
[^39]: Ulrich Windl, David Dalton, Marc Martinec, and Dale R. Worley. [The NTP FAQ and HOWTO](https://www.ntp.org/ntpfaq/). *ntp.org*, November 2006. 
[^40]: John Graham-Cumming. [How and why the leap second affected Cloudflare DNS](https://blog.cloudflare.com/how-and-why-the-leap-second-affected-cloudflare-dns/). *blog.cloudflare.com*, January 2017. Archived at [archive.org](https://web.archive.org/web/20250202041444/https%3A//blog.cloudflare.com/how-and-why-the-leap-second-affected-cloudflare-dns/) 
[^41]: David Holmes. [Inside the Hotspot VM: Clocks, Timers and Scheduling Events – Part I – Windows](https://web.archive.org/web/20160308031939/https%3A//blogs.oracle.com/dholmes/entry/inside_the_hotspot_vm_clocks). *blogs.oracle.com*, October 2006. Archived at [archive.org](https://web.archive.org/web/20160308031939/https%3A//blogs.oracle.com/dholmes/entry/inside_the_hotspot_vm_clocks) 
[^42]: Joran Dirk Greef. [Three Clocks are Better than One](https://tigerbeetle.com/blog/2021-08-30-three-clocks-are-better-than-one/). *tigerbeetle.com*, August 2021. Archived at [perma.cc/5RXG-EU6B](https://perma.cc/5RXG-EU6B) 
[^43]: Oliver Yang. [Pitfalls of TSC usage](https://oliveryang.net/2015/09/pitfalls-of-TSC-usage/). *oliveryang.net*, September 2015. Archived at [perma.cc/Z2QY-5FRA](https://perma.cc/Z2QY-5FRA) 
[^44]: Steve Loughran. [Time on Multi-Core, Multi-Socket Servers](https://steveloughran.blogspot.com/2015/09/time-on-multi-core-multi-socket-servers.html). *steveloughran.blogspot.co.uk*, September 2015. Archived at [perma.cc/7M4S-D4U6](https://perma.cc/7M4S-D4U6) 
[^45]: James C. Corbett, Jeffrey Dean, Michael Epstein, Andrew Fikes, Christopher Frost, JJ Furman, Sanjay Ghemawat, Andrey Gubarev, Christopher Heiser, Peter Hochschild, Wilson Hsieh, Sebastian Kanthak, Eugene Kogan, Hongyi Li, Alexander Lloyd, Sergey Melnik, David Mwaura, David Nagle, Sean Quinlan, Rajesh Rao, Lindsay Rolig, Dale Woodford, Yasushi Saito, Christopher Taylor, Michal Szymaniak, and Ruth Wang. [Spanner: Google’s Globally-Distributed Database](https://research.google/pubs/pub39966/). At *10th USENIX Symposium on Operating System Design and Implementation* (OSDI), October 2012. 
[^46]: M. Caporaloni and R. Ambrosini. [How Closely Can a Personal Computer Clock Track the UTC Timescale Via the Internet?](https://iopscience.iop.org/0143-0807/23/4/103/) *European Journal of Physics*, volume 23, issue 4, pages L17–L21, June 2012. [doi:10.1088/0143-0807/23/4/103](https://doi.org/10.1088/0143-0807/23/4/103) 
[^47]: Nelson Minar. [A Survey of the NTP Network](https://alumni.media.mit.edu/~nelson/research/ntp-survey99/). *alumni.media.mit.edu*, December 1999. Archived at [perma.cc/EV76-7ZV3](https://perma.cc/EV76-7ZV3) 
[^48]: Viliam Holub. [Synchronizing Clocks in a Cassandra Cluster Pt. 1 – The Problem](https://perma.cc/N3RV-5LNL). *blog.rapid7.com*, March 2014. Archived at [perma.cc/N3RV-5LNL](https://perma.cc/N3RV-5LNL)
[^49]: Poul-Henning Kamp. [The One-Second War (What Time Will You Die?)](https://queue.acm.org/detail.cfm?id=1967009) *ACM Queue*, volume 9, issue 4, pages 44–48, April 2011. [doi:10.1145/1966989.1967009](https://doi.org/10.1145/1966989.1967009) 
[^50]: Nelson Minar. [Leap Second Crashes Half the Internet](https://www.somebits.com/weblog/tech/bad/leap-second-2012.html). *somebits.com*, July 2012. Archived at [perma.cc/2WB8-D6EU](https://perma.cc/2WB8-D6EU) 
[^51]: Christopher Pascoe. [Time, Technology and Leaping Seconds](https://googleblog.blogspot.com/2011/09/time-technology-and-leaping-seconds.html). *googleblog.blogspot.co.uk*, September 2011. Archived at [perma.cc/U2JL-7E74](https://perma.cc/U2JL-7E74) 
[^52]: Mingxue Zhao and Jeff Barr. [Look Before You Leap – The Coming Leap Second and AWS](https://aws.amazon.com/blogs/aws/look-before-you-leap-the-coming-leap-second-and-aws/). *aws.amazon.com*, May 2015. Archived at [perma.cc/KPE9-XMFM](https://perma.cc/KPE9-XMFM) 
[^53]: Darryl Veitch and Kanthaiah Vijayalayan. [Network Timing and the 2015 Leap Second](https://opus.lib.uts.edu.au/bitstream/10453/43923/1/LeapSecond_camera.pdf). At *17th International Conference on Passive and Active Measurement* (PAM), April 2016. [doi:10.1007/978-3-319-30505-9\_29](https://doi.org/10.1007/978-3-319-30505-9_29) 
[^54]: VMware, Inc. [Timekeeping in VMware Virtual Machines](https://www.vmware.com/docs/vmware_timekeeping). *vmware.com*, October 2008. Archived at [perma.cc/HM5R-T5NF](https://perma.cc/HM5R-T5NF) 
[^55]: Victor Yodaiken. [Clock Synchronization in Finance and Beyond](https://www.yodaiken.com/wp-content/uploads/2018/05/financeandbeyond.pdf). *yodaiken.com*, November 2017. Archived at [perma.cc/9XZD-8ZZN](https://perma.cc/9XZD-8ZZN) 
[^56]: Mustafa Emre Acer, Emily Stark, Adrienne Porter Felt, Sascha Fahl, Radhika Bhargava, Bhanu Dev, Matt Braithwaite, Ryan Sleevi, and Parisa Tabriz. [Where the Wild Warnings Are: Root Causes of Chrome HTTPS Certificate Errors](https://acmccs.github.io/papers/p1407-acerA.pdf). At *ACM SIGSAC Conference on Computer and Communications Security* (CCS), pages 1407–1420, October 2017. [doi:10.1145/3133956.3134007](https://doi.org/10.1145/3133956.3134007) 
[^57]: European Securities and Markets Authority. [MiFID II / MiFIR: Regulatory Technical and Implementing Standards – Annex I](https://www.esma.europa.eu/sites/default/files/library/2015/11/2015-esma-1464_annex_i_-_draft_rts_and_its_on_mifid_ii_and_mifir.pdf). *esma.europa.eu*, Report ESMA/2015/1464, September 2015. Archived at [perma.cc/ZLX9-FGQ3](https://perma.cc/ZLX9-FGQ3) 
[^58]: Luke Bigum. [Solving MiFID II Clock Synchronisation With Minimum Spend (Part 1)](https://catach.blogspot.com/2015/11/solving-mifid-ii-clock-synchronisation.html). *catach.blogspot.com*, November 2015. Archived at [perma.cc/4J5W-FNM4](https://perma.cc/4J5W-FNM4) 
[^59]: Oleg Obleukhov and Ahmad Byagowi. [How Precision Time Protocol is being deployed at Meta](https://engineering.fb.com/2022/11/21/production-engineering/precision-time-protocol-at-meta/). *engineering.fb.com*, November 2022. Archived at [perma.cc/29G6-UJNW](https://perma.cc/29G6-UJNW) 
[^60]: John Wiseman. [gpsjam.org](https://gpsjam.org/), July 2022. 
[^61]: Josh Levinson, Julien Ridoux, and Chris Munns. [It’s About Time: Microsecond-Accurate Clocks on Amazon EC2 Instances](https://aws.amazon.com/blogs/compute/its-about-time-microsecond-accurate-clocks-on-amazon-ec2-instances/). *aws.amazon.com*, November 2023. Archived at [perma.cc/56M6-5VMZ](https://perma.cc/56M6-5VMZ) 
[^62]: Kyle Kingsbury. [Call Me Maybe: Cassandra](https://aphyr.com/posts/294-call-me-maybe-cassandra/). *aphyr.com*, September 2013. Archived at [perma.cc/4MBR-J96V](https://perma.cc/4MBR-J96V) 
[^63]: John Daily. [Clocks Are Bad, or, Welcome to the Wonderful World of Distributed Systems](https://riak.com/clocks-are-bad-or-welcome-to-distributed-systems/). *riak.com*, November 2013. Archived at [perma.cc/4XB5-UCXY](https://perma.cc/4XB5-UCXY) 
[^64]: Marc Brooker. [It’s About Time!](https://brooker.co.za/blog/2023/11/27/about-time.html) *brooker.co.za*, November 2023. Archived at [perma.cc/N6YK-DRPA](https://perma.cc/N6YK-DRPA) 
[^65]: Kyle Kingsbury. [The Trouble with Timestamps](https://aphyr.com/posts/299-the-trouble-with-timestamps). *aphyr.com*, October 2013. Archived at [perma.cc/W3AM-5VAV](https://perma.cc/W3AM-5VAV) 
[^66]: Leslie Lamport. [Time, Clocks, and the Ordering of Events in a Distributed System](https://www.microsoft.com/en-us/research/publication/time-clocks-ordering-events-distributed-system/). *Communications of the ACM*, volume 21, issue 7, pages 558–565, July 1978. [doi:10.1145/359545.359563](https://doi.org/10.1145/359545.359563) 
[^67]: Justin Sheehy. [There Is No Now: Problems With Simultaneity in Distributed Systems](https://queue.acm.org/detail.cfm?id=2745385). *ACM Queue*, volume 13, issue 3, pages 36–41, March 2015. [doi:10.1145/2733108](https://doi.org/10.1145/2733108) 
[^68]: Murat Demirbas. [Spanner: Google’s Globally-Distributed Database](https://muratbuffalo.blogspot.com/2013/07/spanner-googles-globally-distributed_4.html). *muratbuffalo.blogspot.co.uk*, July 2013. Archived at [perma.cc/6VWR-C9WB](https://perma.cc/6VWR-C9WB) 
[^69]: Dahlia Malkhi and Jean-Philippe Martin. [Spanner’s Concurrency Control](https://www.cs.cornell.edu/~ie53/publications/DC-col51-Sep13.pdf). *ACM SIGACT News*, volume 44, issue 3, pages 73–77, September 2013. [doi:10.1145/2527748.2527767](https://doi.org/10.1145/2527748.2527767) 
[^70]: Franck Pachot. [Achieving Precise Clock Synchronization on AWS](https://www.yugabyte.com/blog/aws-clock-synchronization/). *yugabyte.com*, December 2024. Archived at [perma.cc/UYM6-RNBS](https://perma.cc/UYM6-RNBS) 
[^71]: Spencer Kimball. [Living Without Atomic Clocks: Where CockroachDB and Spanner diverge](https://www.cockroachlabs.com/blog/living-without-atomic-clocks/). *cockroachlabs.com*, January 2022. Archived at [perma.cc/AWZ7-RXFT](https://perma.cc/AWZ7-RXFT) 
[^72]: Murat Demirbas. [Use of Time in Distributed Databases (part 4): Synchronized clocks in production databases](https://muratbuffalo.blogspot.com/2025/01/use-of-time-in-distributed-databases.html). *muratbuffalo.blogspot.com*, January 2025. Archived at [perma.cc/9WNX-Q9U3](https://perma.cc/9WNX-Q9U3) 
[^73]: Cary G. Gray and David R. Cheriton. [Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency](https://courses.cs.duke.edu/spring11/cps210/papers/p202-gray.pdf). At *12th ACM Symposium on Operating Systems Principles* (SOSP), December 1989. [doi:10.1145/74850.74870](https://doi.org/10.1145/74850.74870) 
[^74]: Daniel Sturman, Scott Delap, Max Ross, et al. [Roblox Return to Service](https://corp.roblox.com/newsroom/2022/01/roblox-return-to-service-10-28-10-31-2021). *corp.roblox.com*, January 2022. Archived at [perma.cc/8ALT-WAS4](https://perma.cc/8ALT-WAS4) 
[^75]: Todd Lipcon. [Avoiding Full GCs with MemStore-Local Allocation Buffers](https://www.slideshare.net/slideshow/hbase-hug-presentation/7038178). *slideshare.net*, February 2011. Archived at <https://perma.cc/CH62-2EWJ>
[^76]: Christopher Clark, Keir Fraser, Steven Hand, Jacob Gorm Hansen, Eric Jul, Christian Limpach, Ian Pratt, and Andrew Warfield. [Live Migration of Virtual Machines](https://www.usenix.org/legacy/publications/library/proceedings/nsdi05/tech/full_papers/clark/clark.pdf). At *2nd USENIX Symposium on Symposium on Networked Systems Design & Implementation* (NSDI), May 2005. 
[^77]: Mike Shaver. [fsyncers and Curveballs](https://web.archive.org/web/20220107141023/http%3A//shaver.off.net/diary/2008/05/25/fsyncers-and-curveballs/). *shaver.off.net*, May 2008. Archived at [archive.org](https://web.archive.org/web/20220107141023/http%3A//shaver.off.net/diary/2008/05/25/fsyncers-and-curveballs/) 
[^78]: Zhenyun Zhuang and Cuong Tran. [Eliminating Large JVM GC Pauses Caused by Background IO Traffic](https://engineering.linkedin.com/blog/2016/02/eliminating-large-jvm-gc-pauses-caused-by-background-io-traffic). *engineering.linkedin.com*, February 2016. Archived at [perma.cc/ML2M-X9XT](https://perma.cc/ML2M-X9XT) 
[^79]: Martin Thompson. [Java Garbage Collection Distilled](https://mechanical-sympathy.blogspot.com/2013/07/java-garbage-collection-distilled.html). *mechanical-sympathy.blogspot.co.uk*, July 2013. Archived at [perma.cc/DJT3-NQLQ](https://perma.cc/DJT3-NQLQ) 
[^80]: David Terei and Amit Levy. [Blade: A Data Center Garbage Collector](https://arxiv.org/pdf/1504.02578). arXiv:1504.02578, April 2015. 
[^81]: Martin Maas, Tim Harris, Krste Asanović, and John Kubiatowicz. [Trash Day: Coordinating Garbage Collection in Distributed Systems](https://timharris.uk/papers/2015-hotos.pdf). At *15th USENIX Workshop on Hot Topics in Operating Systems* (HotOS), May 2015. 
[^82]: Martin Fowler. [The LMAX Architecture](https://martinfowler.com/articles/lmax.html). *martinfowler.com*, July 2011. Archived at [perma.cc/5AV4-N6RJ](https://perma.cc/5AV4-N6RJ) 
[^83]: Joseph Y. Halpern and Yoram Moses. [Knowledge and common knowledge in a distributed environment](https://groups.csail.mit.edu/tds/papers/Halpern/JACM90.pdf). *Journal of the ACM* (JACM), volume 37, issue 3, pages 549–587, July 1990. [doi:10.1145/79147.79161](https://doi.org/10.1145/79147.79161) 
[^84]: Chuzhe Tang, Zhaoguo Wang, Xiaodong Zhang, Qianmian Yu, Binyu Zang, Haibing Guan, and Haibo Chen. [Ad Hoc Transactions in Web Applications: The Good, the Bad, and the Ugly](https://ipads.se.sjtu.edu.cn/_media/publications/concerto-sigmod22.pdf). At *ACM International Conference on Management of Data* (SIGMOD), June 2022. [doi:10.1145/3514221.3526120](https://doi.org/10.1145/3514221.3526120) 
[^85]: Flavio P. Junqueira and Benjamin Reed. [*ZooKeeper: Distributed Process Coordination*](https://www.oreilly.com/library/view/zookeeper/9781449361297/). O’Reilly Media, 2013. ISBN: 978-1-449-36130-3 
[^86]: Enis Söztutar. [HBase and HDFS: Understanding Filesystem Usage in HBase](https://www.slideshare.net/slideshow/hbase-and-hdfs-understanding-filesystem-usage/22990858). At *HBaseCon*, June 2013. Archived at [perma.cc/4DXR-9P88](https://perma.cc/4DXR-9P88) 
[^87]: SUSE LLC. [SUSE Linux Enterprise High Availability 15 SP6 Administration Guide, Section 12: Fencing and STONITH](https://documentation.suse.com/sle-ha/15-SP6/html/SLE-HA-all/cha-ha-fencing.html). *documentation.suse.com*, March 2025. Archived at [perma.cc/8LAR-EL9D](https://perma.cc/8LAR-EL9D) 
[^88]: Mike Burrows. [The Chubby Lock Service for Loosely-Coupled Distributed Systems](https://research.google/pubs/pub27897/). At *7th USENIX Symposium on Operating System Design and Implementation* (OSDI), November 2006. 
[^89]: Kyle Kingsbury. [etcd 3.4.3](https://jepsen.io/analyses/etcd-3.4.3). *jepsen.io*, January 2020. Archived at [perma.cc/2P3Y-MPWU](https://perma.cc/2P3Y-MPWU) 
[^90]: Ensar Basri Kahveci. [Distributed Locks are Dead; Long Live Distributed Locks!](https://hazelcast.com/blog/long-live-distributed-locks/) *hazelcast.com*, April 2019. Archived at [perma.cc/7FS5-LDXE](https://perma.cc/7FS5-LDXE) 
[^91]: Martin Kleppmann. [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html). *martin.kleppmann.com*, February 2016. Archived at [perma.cc/Y24W-YQ5L](https://perma.cc/Y24W-YQ5L) 
[^92]: Salvatore Sanfilippo. [Is Redlock safe?](https://antirez.com/news/101) *antirez.com*, February 2016. Archived at [perma.cc/B6GA-9Q6A](https://perma.cc/B6GA-9Q6A) 
[^93]: Gunnar Morling. [Leader Election With S3 Conditional Writes](https://www.morling.dev/blog/leader-election-with-s3-conditional-writes/). *www.morling.dev*, August 2024. Archived at [perma.cc/7V2N-J78Y](https://perma.cc/7V2N-J78Y) 
[^94]: Leslie Lamport, Robert Shostak, and Marshall Pease. [The Byzantine Generals Problem](https://www.microsoft.com/en-us/research/publication/byzantine-generals-problem/). *ACM Transactions on Programming Languages and Systems* (TOPLAS), volume 4, issue 3, pages 382–401, July 1982. [doi:10.1145/357172.357176](https://doi.org/10.1145/357172.357176) 
[^95]: Jim N. Gray. [Notes on Data Base Operating Systems](https://jimgray.azurewebsites.net/papers/dbos.pdf). in *Operating Systems: An Advanced Course*, Lecture Notes in Computer Science, volume 60, edited by R. Bayer, R. M. Graham, and G. Seegmüller, pages 393–481, Springer-Verlag, 1978. ISBN: 978-3-540-08755-7. Archived at [perma.cc/7S9M-2LZU](https://perma.cc/7S9M-2LZU) 
[^96]: Brian Palmer. [How Complicated Was the Byzantine Empire?](https://slate.com/news-and-politics/2011/10/the-byzantine-tax-code-how-complicated-was-byzantium-anyway.html) *slate.com*, October 2011. Archived at [perma.cc/AN7X-FL3N](https://perma.cc/AN7X-FL3N) 
[^97]: Leslie Lamport. [My Writings](https://lamport.azurewebsites.net/pubs/pubs.html). *lamport.azurewebsites.net*, December 2014. Archived at [perma.cc/5NNM-SQGR](https://perma.cc/5NNM-SQGR) 
[^98]: John Rushby. [Bus Architectures for Safety-Critical Embedded Systems](https://www.csl.sri.com/papers/emsoft01/emsoft01.pdf). At *1st International Workshop on Embedded Software* (EMSOFT), October 2001. [doi:10.1007/3-540-45449-7\_22](https://doi.org/10.1007/3-540-45449-7_22) 
[^99]: Jake Edge. [ELC: SpaceX Lessons Learned](https://lwn.net/Articles/540368/). *lwn.net*, March 2013. Archived at [perma.cc/AYX8-QP5X](https://perma.cc/AYX8-QP5X) 
[^100]: Shehar Bano, Alberto Sonnino, Mustafa Al-Bassam, Sarah Azouvi, Patrick McCorry, Sarah Meiklejohn, and George Danezis. [SoK: Consensus in the Age of Blockchains](https://doi.org/10.1145/3318041.3355458). At *1st ACM Conference on Advances in Financial Technologies* (AFT), October 2019. [doi:10.1145/3318041.3355458](https://doi.org/10.1145/3318041.3355458)
[^101]: Ezra Feilden, Adi Oltean, and Philip Johnston. [Why we should train AI in space](https://www.starcloud.com/wp). White Paper, *starcloud.com*, September 2024. Archived at [perma.cc/7Y3S-8UB6](https://perma.cc/7Y3S-8UB6) 
[^102]: James Mickens. [The Saddest Moment](https://www.usenix.org/system/files/login-logout_1305_mickens.pdf). *USENIX ;login*, May 2013. Archived at [perma.cc/T7BZ-XCFR](https://perma.cc/T7BZ-XCFR) 
[^103]: Martin Kleppmann and Heidi Howard. [Byzantine Eventual Consistency and the Fundamental Limits of Peer-to-Peer Databases](https://arxiv.org/abs/2012.00472). *arxiv.org*, December 2020. [doi:10.48550/arXiv.2012.00472](https://doi.org/10.48550/arXiv.2012.00472) 
[^104]: Martin Kleppmann. [Making CRDTs Byzantine Fault Tolerant](https://martin.kleppmann.com/papers/bft-crdt-papoc22.pdf). At *9th Workshop on Principles and Practice of Consistency for Distributed Data* (PaPoC), April 2022. [doi:10.1145/3517209.3524042](https://doi.org/10.1145/3517209.3524042) 
[^105]: Evan Gilman. [The Discovery of Apache ZooKeeper’s Poison Packet](https://perma.cc/RV6L-Y5CQ). *pagerduty.com*, May 2015. Archived at [perma.cc/RV6L-Y5CQ](https://perma.cc/RV6L-Y5CQ)
[^106]: Jonathan Stone and Craig Partridge. [When the CRC and TCP Checksum Disagree](https://conferences2.sigcomm.org/sigcomm/2000/conf/paper/sigcomm2000-9-1.pdf). At *ACM Conference on Applications, Technologies, Architectures, and Protocols for Computer Communication* (SIGCOMM), August 2000. [doi:10.1145/347059.347561](https://doi.org/10.1145/347059.347561) 
[^107]: Evan Jones. [How Both TCP and Ethernet Checksums Fail](https://www.evanjones.ca/tcp-and-ethernet-checksums-fail.html). *evanjones.ca*, October 2015. Archived at [perma.cc/9T5V-B8X5](https://perma.cc/9T5V-B8X5) 
[^108]: Cynthia Dwork, Nancy Lynch, and Larry Stockmeyer. [Consensus in the Presence of Partial Synchrony](https://groups.csail.mit.edu/tds/papers/Lynch/jacm88.pdf). *Journal of the ACM*, volume 35, issue 2, pages 288–323, April 1988. [doi:10.1145/42282.42283](https://doi.org/10.1145/42282.42283) 
[^109]: Richard D. Schlichting and Fred B. Schneider. [Fail-stop processors: an approach to designing fault-tolerant computing systems](https://www.cs.cornell.edu/fbs/publications/Fail_Stop.pdf). *ACM Transactions on Computer Systems* (TOCS), volume 1, issue 3, pages 222–238, August 1983. [doi:10.1145/357369.357371](https://doi.org/10.1145/357369.357371) 
[^110]: Thanh Do, Mingzhe Hao, Tanakorn Leesatapornwongsa, Tiratat Patana-anake, and Haryadi S. Gunawi. [Limplock: Understanding the Impact of Limpware on Scale-out Cloud Systems](https://ucare.cs.uchicago.edu/pdf/socc13-limplock.pdf). At *4th ACM Symposium on Cloud Computing* (SoCC), October 2013. [doi:10.1145/2523616.2523627](https://doi.org/10.1145/2523616.2523627) 
[^111]: Josh Snyder and Joseph Lynch. [Garbage collecting unhealthy JVMs, a proactive approach](https://netflixtechblog.medium.com/introducing-jvmquake-ec944c60ba70). Netflix Technology Blog, *netflixtechblog.medium.com*, November 2019. Archived at [perma.cc/8BTA-N3YB](https://perma.cc/8BTA-N3YB) 
[^112]: Haryadi S. Gunawi, Riza O. Suminto, Russell Sears, Casey Golliher, Swaminathan Sundararaman, Xing Lin, Tim Emami, Weiguang Sheng, Nematollah Bidokhti, Caitie McCaffrey, Gary Grider, Parks M. Fields, Kevin Harms, Robert B. Ross, Andree Jacobson, Robert Ricci, Kirk Webb, Peter Alvaro, H. Birali Runesha, Mingzhe Hao, and Huaicheng Li. [Fail-Slow at Scale: Evidence of Hardware Performance Faults in Large Production Systems](https://www.usenix.org/system/files/conference/fast18/fast18-gunawi.pdf). At *16th USENIX Conference on File and Storage Technologies*, February 2018. 
[^113]: Peng Huang, Chuanxiong Guo, Lidong Zhou, Jacob R. Lorch, Yingnong Dang, Murali Chintalapati, and Randolph Yao. [Gray Failure: The Achilles’ Heel of Cloud-Scale Systems](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/06/paper-1.pdf). At *16th Workshop on Hot Topics in Operating Systems* (HotOS), May 2017. [doi:10.1145/3102980.3103005](https://doi.org/10.1145/3102980.3103005) 
[^114]: Chang Lou, Peng Huang, and Scott Smith. [Understanding, Detecting and Localizing Partial Failures in Large System Software](https://www.usenix.org/conference/nsdi20/presentation/lou). At *17th USENIX Symposium on Networked Systems Design and Implementation* (NSDI), February 2020. 
[^115]: Peter Bailis and Ali Ghodsi. [Eventual Consistency Today: Limitations, Extensions, and Beyond](https://queue.acm.org/detail.cfm?id=2462076). *ACM Queue*, volume 11, issue 3, pages 55-63, March 2013. [doi:10.1145/2460276.2462076](https://doi.org/10.1145/2460276.2462076) 
[^116]: Bowen Alpern and Fred B. Schneider. [Defining Liveness](https://www.cs.cornell.edu/fbs/publications/DefLiveness.pdf). *Information Processing Letters*, volume 21, issue 4, pages 181–185, October 1985. [doi:10.1016/0020-0190(85)90056-0](https://doi.org/10.1016/0020-0190%2885%2990056-0) 
[^117]: Flavio P. Junqueira. [Dude, Where’s My Metadata?](https://fpj.me/2015/05/28/dude-wheres-my-metadata/) *fpj.me*, May 2015. Archived at [perma.cc/D2EU-Y9S5](https://perma.cc/D2EU-Y9S5) 
[^118]: Scott Sanders. [January 28th Incident Report](https://github.com/blog/2106-january-28th-incident-report). *github.com*, February 2016. Archived at [perma.cc/5GZR-88TV](https://perma.cc/5GZR-88TV) 
[^119]: Jay Kreps. [A Few Notes on Kafka and Jepsen](https://blog.empathybox.com/post/62279088548/a-few-notes-on-kafka-and-jepsen). *blog.empathybox.com*, September 2013. [perma.cc/XJ5C-F583](https://perma.cc/XJ5C-F583) 
[^120]: Marc Brooker and Ankush Desai. [Systems Correctness Practices at AWS](https://dl.acm.org/doi/pdf/10.1145/3712057). *Queue, Volume 22, Issue 6*, November/December 2024. [doi:10.1145/3712057](https://doi.org/10.1145/3712057) 
[^121]: Andrey Satarin. [Testing Distributed Systems: Curated list of resources on testing distributed systems](https://asatarin.github.io/testing-distributed-systems/). *asatarin.github.io*. Archived at [perma.cc/U5V8-XP24](https://perma.cc/U5V8-XP24) 
[^122]: Jack Vanlightly. [Verifying Kafka transactions - Diary entry 2 - Writing an initial TLA+ spec](https://jack-vanlightly.com/analyses/2024/12/3/verifying-kafka-transactions-diary-entry-2-writing-an-initial-tla-spec). *jack-vanlightly.com*, December 2024. Archived at [perma.cc/NSQ8-MQ5N](https://perma.cc/NSQ8-MQ5N) 
[^123]: Siddon Tang. [From Chaos to Order — Tools and Techniques for Testing TiDB, A Distributed NewSQL Database](https://www.pingcap.com/blog/chaos-practice-in-tidb/). *pingcap.com*, April 2018. Archived at [perma.cc/5EJB-R29F](https://perma.cc/5EJB-R29F) 
[^124]: Nathan VanBenschoten. [Parallel Commits: An atomic commit protocol for globally distributed transactions](https://www.cockroachlabs.com/blog/parallel-commits/). *cockroachlabs.com*, November 2019. Archived at [perma.cc/5FZ7-QK6J](https://perma.cc/5FZ7-QK6J%20) 
[^125]: Jack Vanlightly. [Paper: VR Revisited - State Transfer (part 3)](https://jack-vanlightly.com/analyses/2022/12/28/paper-vr-revisited-state-transfer-part-3). *jack-vanlightly.com*, December 2022. Archived at [perma.cc/KNK3-K6WS](https://perma.cc/KNK3-K6WS) 
[^126]: Hillel Wayne. [What if the spec doesn’t match the code?](https://buttondown.com/hillelwayne/archive/what-if-the-spec-doesnt-match-the-code/) *buttondown.com*, March 2024. Archived at [perma.cc/8HEZ-KHER](https://perma.cc/8HEZ-KHER) 
[^127]: Lingzhi Ouyang, Xudong Sun, Ruize Tang, Yu Huang, Madhav Jivrajani, Xiaoxing Ma, Tianyin Xu. [Multi-Grained Specifications for Distributed System Model Checking and Verification](https://arxiv.org/abs/2409.14301). At *20th European Conference on Computer Systems* (EuroSys), March 2025. [doi:10.1145/3689031.3696069](https://doi.org/10.1145/3689031.3696069) 
[^128]: Yury Izrailevsky and Ariel Tseitlin. [The Netflix Simian Army](https://netflixtechblog.com/the-netflix-simian-army-16e57fbab116). *netflixtechblog.com*, July, 2011. Archived at [perma.cc/M3NY-FJW6](https://perma.cc/M3NY-FJW6) 
[^129]: Kyle Kingsbury. [Jepsen: On the perils of network partitions](https://aphyr.com/posts/281-jepsen-on-the-perils-of-network-partitions). *aphyr.com*, May, 2013. Archived at [perma.cc/W98G-6HQP](https://perma.cc/W98G-6HQP) 
[^130]: Kyle Kingsbury. [Jepsen Analyses](https://jepsen.io/analyses). *jepsen.io*, 2024. Archived at [perma.cc/8LDN-D2T8](https://perma.cc/8LDN-D2T8) 
[^131]: Rupak Majumdar and Filip Niksic. [Why is random testing effective for partition tolerance bugs?](https://dl.acm.org/doi/pdf/10.1145/3158134) *Proceedings of the ACM on Programming Languages* (PACMPL), volume 2, issue POPL, article no. 46, December 2017. [doi:10.1145/3158134](https://doi.org/10.1145/3158134) 
[^132]: FoundationDB project authors. [Simulation and Testing](https://apple.github.io/foundationdb/testing.html). *apple.github.io*. Archived at [perma.cc/NQ3L-PM4C](https://perma.cc/NQ3L-PM4C) 
[^133]: Alex Kladov. [Simulation Testing For Liveness](https://tigerbeetle.com/blog/2023-07-06-simulation-testing-for-liveness/). *tigerbeetle.com*, July 2023. Archived at [perma.cc/RKD4-HGCR](https://perma.cc/RKD4-HGCR) 
[^134]: Alfonso Subiotto Marqués. [(Mostly) Deterministic Simulation Testing in Go](https://www.polarsignals.com/blog/posts/2024/05/28/mostly-dst-in-go). *polarsignals.com*, May 2024. Archived at [perma.cc/ULD6-TSA4](https://perma.cc/ULD6-TSA4) 
