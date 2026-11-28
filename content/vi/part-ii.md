---
translationKey: ddia-v2-part-ii
title: Dữ liệu phân tán
book_kind: part
book_number: II
search_exclude: true
weight: 200
breadcrumbs: false
---

> *Một công nghệ thành công thì thực tế phải được đặt trên quan hệ công chúng, vì có thể lừa được người khác chứ không thể lừa được tự nhiên.*
>
> — Báo cáo của Ủy ban Rogers (1986)
>

-------

Trong [Phần I](/part-i) của cuốn sách, ta đã bàn về những khái niệm nền tảng của hệ thống dữ liệu.
Giờ đến [Phần II](/part-ii), ta bước lên một tầng cao hơn và đặt ra câu hỏi: điều gì xảy ra nếu *nhiều máy* cùng tham gia lưu trữ và truy xuất dữ liệu?

Bạn có thể muốn phân tán cơ sở dữ liệu lên nhiều máy vì đủ loại lý do:

Khả năng mở rộng
: Nếu khối lượng dữ liệu, tải đọc hoặc tải ghi vượt quá khả năng xử lý của một máy, bạn có thể dàn tải ra nhiều máy tính.

Chịu lỗi / tính sẵn sàng cao
: Nếu ứng dụng cần tiếp tục hoạt động khi một máy (hoặc nhiều máy, mạng, thậm chí cả một trung tâm dữ liệu) gặp sự cố, bạn có thể dùng nhiều máy để tạo ra sự dư thừa. Khi một máy hỏng, máy khác có thể tiếp quản.

Độ trễ
: Nếu người dùng ở khắp nơi trên thế giới, bạn có thể cân nhắc triển khai nhiều server trên toàn cầu để mỗi người dùng được phục vụ từ trung tâm dữ liệu gần nhất về mặt địa lý, tránh phải chờ gói tin mạng băng qua nửa vòng trái đất.

## Mở rộng để chịu tải cao hơn

Nếu thứ bạn cần chỉ là mở rộng để chịu *tải* (*load*) cao hơn, cách đơn giản nhất là mua máy mạnh hơn (đôi khi gọi là *mở rộng theo chiều dọc* (*vertical scaling*), hay *scale up*). Nhiều bộ xử lý, bộ nhớ và ổ đĩa có thể được nối với nhau dưới cùng một hệ điều hành, và kết nối tốc độ cao cho phép bất kỳ bộ xử lý nào truy cập bất kỳ phần nào của bộ nhớ hay ổ đĩa. Trong *kiến trúc chia sẻ bộ nhớ* (*shared-memory architecture*) này, mọi thành phần có thể coi là một máy duy nhất.

> [!NOTE]
> Trong các máy lớn (mainframe), dù bất kỳ bộ xử lý nào cũng có thể truy cập bất kỳ phần nào của bộ nhớ, vẫn có những vùng nhớ gần một số bộ xử lý hơn (gọi là *truy cập bộ nhớ không đồng nhất* (*nonuniform memory access*, NUMA) [^1]). Để tận dụng hiệu quả đặc điểm kiến trúc này, cần chia nhỏ việc xử lý sao cho mỗi bộ xử lý chủ yếu truy cập vùng nhớ lân cận, nghĩa là *phân vùng* (*partitioning*) vẫn cần thiết ngay cả khi bề ngoài chỉ có một máy đang chạy.

Vấn đề của cách tiếp cận chia sẻ bộ nhớ là chi phí tăng nhanh hơn tuyến tính: một máy có gấp đôi số bộ xử lý, gấp đôi bộ nhớ, gấp đôi dung lượng đĩa thường đắt hơn gấp đôi rất nhiều. Và do tồn tại các nút thắt cổ chai, nó cũng có thể không đủ sức xử lý gấp đôi tải.

Kiến trúc chia sẻ bộ nhớ có thể cung cấp khả năng chịu lỗi ở mức hạn chế: máy cao cấp có thể dùng linh kiện cắm nóng (thay đĩa, module nhớ, thậm chí bộ xử lý mà không cần tắt máy) — nhưng nó chắc chắn bị trói vào một vị trí địa lý duy nhất.

Một cách tiếp cận khác là *kiến trúc chia sẻ đĩa* (*shared-disk architecture*): dùng nhiều máy có bộ xử lý và bộ nhớ riêng, nhưng lưu dữ liệu trên một mảng đĩa dùng chung giữa các máy, các đĩa này được nối qua mạng tốc độ cao. Kiến trúc này được dùng cho một số kho dữ liệu, nhưng chi phí tranh chấp và khóa giới hạn khả năng mở rộng của hướng chia sẻ đĩa [^2].

> [!NOTE]
> Lưu trữ gắn mạng (*Network Attached Storage*, NAS) hoặc *mạng lưu trữ* (*Storage Area Network*, SAN)

### Kiến trúc shared-nothing

Ngược lại, *kiến trúc shared-nothing* [^3] (đôi khi còn gọi là *mở rộng theo chiều ngang* (*horizontal scaling*), hay *scaling out*) đã khá phổ biến.
Trong kiến trúc này, mỗi máy / máy ảo chạy phần mềm cơ sở dữ liệu được gọi là một *node*. Mỗi node chỉ dùng bộ xử lý, bộ nhớ và đĩa của riêng mình. Mọi sự phối hợp giữa các node đều được thực hiện ở tầng phần mềm, qua mạng thông thường.

Hệ thống shared-nothing không cần phần cứng đặc biệt, nên bạn có thể dùng bất kỳ máy nào — chẳng hạn máy có tỷ lệ giá/hiệu năng tốt nhất. Bạn có thể phân tán dữ liệu qua nhiều vùng địa lý để giảm độ trễ cho người dùng, hoặc vẫn trụ vững khi mất cả một trung tâm dữ liệu.
Với sự xuất hiện của máy ảo trên đám mây, ngay cả công ty nhỏ giờ cũng có thể xây kiến trúc phân tán đa vùng mà không cần vận hành ở quy mô Google.

Trong phần này, ta sẽ tập trung vào kiến trúc shared-nothing. Nó không nhất thiết là lựa chọn tốt nhất cho mọi tình huống, nhưng là kiến trúc đòi hỏi bạn thận trọng nhất.
Nếu dữ liệu của bạn nằm trên nhiều node, bạn cần nhận ra những ràng buộc và đánh đổi của một hệ thống phân tán như vậy — cơ sở dữ liệu không thể giấu những thứ này đi một cách thần kỳ.

Dù kiến trúc shared-nothing phân tán có nhiều ưu điểm, nó thường kéo theo độ phức tạp bổ sung cho ứng dụng, đôi khi còn hạn chế sức biểu đạt của mô hình dữ liệu mà bạn có thể dùng.
Trong một số trường hợp, một chương trình đơn luồng đơn giản có thể chạy tốt hơn một cụm có hơn 100 lõi CPU [^4]. Mặt khác, hệ thống shared-nothing có thể cực kỳ mạnh mẽ. Vài chương tiếp theo sẽ bàn chi tiết về các vấn đề mà dữ liệu phân tán đem lại.


### Sao chép và phân vùng

Có hai cách phổ biến để phân tán dữ liệu qua nhiều node:

*Sao chép* (*replication*)
: Giữ các bản sao giống hệt nhau của dữ liệu trên vài node khác nhau, có thể đặt ở các vị trí khác nhau. Sao chép (*replication*) tạo ra sự dư thừa: nếu một số node không khả dụng, các node còn lại vẫn có thể phục vụ dữ liệu. Sao chép (*replication*) cũng giúp cải thiện hiệu năng. [Chương 6](/ch6) sẽ bàn về sao chép.

*Phân vùng* (*partitioning*, còn gọi là *sharding*)
: Chia một cơ sở dữ liệu lớn thành các tập con nhỏ hơn (gọi là *partition*), để các partition khác nhau có thể được giao cho các *node* khác nhau. [Chương 7](/ch7) sẽ bàn về sharding.

Sao chép và phân vùng là hai cơ chế khác nhau, nhưng chúng thường được dùng cùng nhau, như minh họa ở [Hình II-1](#fig_replication_partitioning).

{{< fig num="II-1" id="fig_replication_partitioning" src="/v1/ddia_part-ii_01.png" caption="Một cơ sở dữ liệu được chia thành hai partition, mỗi partition có hai replica" class="ddia-figure ddia-figure--wide" width="2880" height="1606" />}}


Khi đã hiểu các khái niệm này, ta có thể bắt đầu bàn về những lựa chọn khó khăn cần đưa ra trong hệ thống phân tán. [Chương 8](/ch8) sẽ bàn về *giao dịch* (*transaction*), rất hữu ích để hiểu các loại sự cố có thể xảy ra trong hệ thống dữ liệu và ta có thể làm gì với chúng.
[Chương 9](/ch9) và [Chương 10](/ch10) sẽ bàn về những giới hạn căn bản của hệ thống phân tán.

Trong [Phần III](/part-iii) của sách, ta sẽ bàn cách tích hợp nhiều kho dữ liệu (có thể là phân tán) thành một hệ thống lớn hơn để đáp ứng các nhu cầu ứng dụng phức tạp. Nhưng trước hết, ta hãy nói về dữ liệu phân tán.


## [6. Sao chép](/ch6)
- [Sao chép đơn leader](/ch6#sec_replication_leader)
- [Vấn đề của replication lag](/ch6#sec_replication_lag)
- [Sao chép đa leader](/ch6#sec_replication_multi_leader)
- [Sao chép không leader](/ch6#sec_replication_leaderless)
- [Tóm tắt](/ch6#summary)

## [7. Phân mảnh](/ch7)
- [Ưu và nhược điểm của sharding](/ch7#sec_sharding_reasons)
- [Sharding dữ liệu khóa–giá trị](/ch7#sec_sharding_key_value)
- [Định tuyến yêu cầu](/ch7#sec_sharding_routing)
- [Sharding và chỉ mục phụ](/ch7#sec_sharding_secondary_indexes)
- [Tóm tắt](/ch7#summary)

## [8. Giao dịch](/ch8)
- [Giao dịch thực chất là gì?](/ch8#sec_transactions_overview)
- [Các mức cô lập yếu](/ch8#sec_transactions_isolation_levels)
- [Serializability](/ch8#sec_transactions_serializability)
- [Giao dịch phân tán](/ch8#sec_transactions_distributed)
- [Tóm tắt](/ch8#summary)
- [Tài liệu tham khảo](/ch8#references)

## [9. Những rắc rối của hệ phân tán](/ch9)
- [Lỗi và hỏng hóc từng phần](/ch9#sec_distributed_partial_failure)
- [Mạng không đáng tin cậy](/ch9#sec_distributed_networks)
- [Đồng hồ không đáng tin cậy](/ch9#sec_distributed_clocks)
- [Tri thức, sự thật và lời nói dối](/ch9#sec_distributed_truth)
- [Tóm tắt](/ch9#summary)

## [10. Nhất quán và đồng thuận](/ch10)
- [Linearizability](/ch10#sec_consistency_linearizability)
- [Bộ sinh ID và đồng hồ logic](/ch10#sec_consistency_logical)
- [Đồng thuận](/ch10#sec_consistency_consensus)
- [Tóm tắt](/ch10#summary)


### Tham khảo

[^1]: Ulrich Drepper: “[What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf),” akka‐dia.org, November 21, 2007.
[^2]: Ben Stopford: “[Shared Nothing vs. Shared Disk Architectures: An Independent View](http://www.benstopford.com/2009/11/24/understanding-the-shared-nothing-architecture/),” benstopford.com, November 24, 2009.
[^3]: Michael Stonebraker: “[The Case for Shared Nothing](http://db.cs.berkeley.edu/papers/hpts85-nothing.pdf),” IEEE Database EngineeringBulletin, volume 9, number 1, pages 4–9, March 1986.
[^4]: Frank McSherry, Michael Isard, and Derek G. Murray: “[Scalability! But at What COST?](http://www.frankmcsherry.org/assets/COST.pdf),” at 15th USENIX Workshop on Hot Topics in Operating Systems (HotOS),May 2015.
