---
translationKey: ddia-v2-ch6
title: Sao chép
book_kind: chapter
book_number: "6"
book_part: II
weight: 206
breadcrumbs: false
---

<a id="ch_replication"></a>

![](/map/ch05.png)

> *Điểm khác biệt nổi bật nhất giữa thứ có thể hỏng và thứ "không thể" hỏng là: khi thứ "không thể" hỏng mà lại hỏng thật, thì thường nó hỏng sạch sành sanh, không sửa nổi.*
>
> Douglas Adams, *Mostly Harmless* (1992)

*Sao chép* (*replication*) nghĩa là giữ bản sao của cùng một dữ liệu trên nhiều máy được nối với nhau qua mạng. Như đã bàn ở ["Hệ thống phân tán và hệ thống một node"](/ch1#sec_introduction_distributed), ta có thể muốn sao chép (*replication*) dữ liệu vì một số lý do sau:

* Để đưa dữ liệu về gần người dùng hơn về mặt địa lý (nhờ đó giảm độ trễ truy cập)
* Để hệ thống vẫn tiếp tục hoạt động ngay cả khi một phần của nó gặp sự cố (nhờ đó tăng tính sẵn sàng)
* Để tăng số lượng máy có thể phục vụ truy vấn đọc (nhờ đó tăng thông lượng đọc)

Chương này giả định tập dữ liệu đủ nhỏ để mỗi máy giữ được một bản sao của toàn bộ tập dữ liệu. Ở [Chương 7](/ch7#ch_sharding), ta sẽ nới lỏng giả định này và bàn về việc *sharding* (*sharding*, còn gọi là *phân region*, *partitioning*) những tập dữ liệu quá lớn so với một máy. Các chương sau đó sẽ bàn về nhiều loại lỗi có thể xảy ra trong hệ thống dữ liệu được sao chép (*replication*), và cách đối phó với chúng.

Nếu dữ liệu cần sao chép (*replication*) không thay đổi theo thời gian thì việc sao chép (*replication*) rất đơn giản: chỉ cần chép dữ liệu đến mọi node một lần là xong. Toàn bộ độ khó của sao chép (*replication*) nằm ở việc xử lý các *thay đổi* của dữ liệu được sao chép (*replication*), và đó chính là chủ đề của chương này. Ta sẽ bàn về ba loại thuật toán để sao chép (*replication*) thay đổi giữa các node: *sao chép đơn leader* (*single-leader replication*), *sao chép đa leader* (*multi-leader replication*) và *sao chép không leader* (*leaderless replication*). Hầu như mọi cơ sở dữ liệu phân tán đều dùng một trong ba cách này. Mỗi cách có ưu nhược điểm riêng, và ta sẽ xem xét chi tiết từng cách.

Sao chép (*replication*) có nhiều đánh đổi cần cân nhắc, chẳng hạn dùng *sao chép đồng bộ* (*synchronous replication*) hay *sao chép bất đồng bộ* (*asynchronous replication*), và xử lý các replica hỏng ra sao. Đây thường là các tùy chọn cấu hình của cơ sở dữ liệu. Chi tiết khác nhau giữa các cơ sở dữ liệu, nhưng nguyên lý cơ bản đằng sau các cách triển khai khác nhau nhìn chung giống nhau. Chương này sẽ bàn về hệ quả của những lựa chọn đó.

Sao chép (*replication*) cơ sở dữ liệu là chủ đề đã quá quen thuộc: các nguyên lý cơ bản gần như không đổi kể từ khi người ta bắt đầu nghiên cứu nó vào thập niên 1970 [^1], vì các ràng buộc căn bản của mạng cũng không thay đổi. Dù vậy, những khái niệm như *nhất quán cuối cùng* (*eventual consistency*) vẫn thường gây nhầm lẫn. Trong ["Vấn đề của replication lag"](/ch6#sec_replication_lag), ta sẽ nói chính xác hơn về nhất quán cuối cùng và bàn về các đảm bảo như *read-your-writes* (*read-your-writes*) và *monotonic reads* (*monotonic reads*).


> [!TIP] Sao lưu và sao chép (*replication*)
>
> Bạn có thể hỏi: đã có sao chép (*replication*) thì còn cần sao lưu không? Câu trả lời là có, vì hai thứ phục vụ mục đích khác nhau. Replica phản ánh rất nhanh các lần ghi trên một node sang các node khác, còn bản sao lưu lưu ảnh chụp (snapshot) của dữ liệu tại một thời điểm trong quá khứ để có thể khôi phục về trạng thái trước đó. Nếu bạn lỡ xóa dữ liệu, sao chép (*replication*) không giúp được gì, vì thao tác xóa cũng được lan truyền đến mọi replica. Để khôi phục dữ liệu đó, bạn vẫn cần bản sao lưu.
>
> Thực ra, sao chép (*replication*) và sao lưu thường bổ trợ cho nhau. Như sẽ thấy trong ["Thiết lập replica mới"](/ch6#sec_replication_new_replica), sao lưu đôi khi là một bước để thiết lập sao chép (*replication*), ngược lại, việc lưu trữ (archive) log sao chép (*replication*) cũng có thể là một phần của quy trình sao lưu.
>
> Một số cơ sở dữ liệu tự duy trì các snapshot bất biến của các trạng thái quá khứ, tương đương một dạng sao lưu tích hợp sẵn. Tuy nhiên, điều đó có nghĩa là phiên bản cũ và trạng thái hiện tại phải nằm trên cùng loại phương tiện lưu trữ. Khi dữ liệu lớn, đặt bản sao lưu của dữ liệu cũ vào lưu trữ đối tượng, vốn được tối ưu cho truy cập không thường xuyên, thường rẻ hơn so với để trong kho lưu trữ chính, nơi chỉ cần giữ trạng thái hiện tại của cơ sở dữ liệu.


## Sao chép đơn leader (*single-leader replication*) {#sec_replication_leader}

Mỗi node lưu một bản sao của cơ sở dữ liệu được gọi là một *replica* (*replica*). Khi có nhiều replica, một câu hỏi không thể tránh khỏi nảy sinh: làm sao đảm bảo mọi dữ liệu cuối cùng đều xuất hiện trên tất cả các replica?

Mỗi lần ghi vào cơ sở dữ liệu đều phải được mọi replica xử lý. Nếu không, các replica sẽ chứa dữ liệu khác nhau. Giải pháp phổ biến nhất gọi là *sao chép dựa trên leader* (*leader-based replication*), còn gọi là *primary-backup* (*primary-backup*) hay *active/passive* (*active/passive*). Cách hoạt động như sau (xem {{< xref fig="6-1" page="/ch6" anchor="fig_replication_leader_follower" >}}Hình 6-1{{< /xref >}}):

1. Một trong các replica được chỉ định làm *leader* (*leader*, còn gọi là *primary*, hay *source* [^2]). Khi client muốn ghi vào cơ sở dữ liệu, nó phải gửi yêu cầu đến leader. Leader trước hết ghi dữ liệu mới vào bộ nhớ lưu trữ cục bộ.
2. Các replica khác được gọi là *follower* (*follower*, còn gọi là *read replica*, *secondary* hay *hot standby*). Mỗi khi leader ghi dữ liệu mới vào bộ nhớ lưu trữ cục bộ, nó cũng gửi thay đổi dữ liệu đó cho tất cả follower dưới dạng *replication log* (*replication log*) hoặc *change stream* (*change stream*). Mỗi follower lấy log từ leader và áp dụng mọi lần ghi theo đúng thứ tự mà leader đã xử lý, qua đó cập nhật bản sao cơ sở dữ liệu cục bộ của mình.
3. Khi client muốn đọc từ cơ sở dữ liệu, nó có thể truy vấn leader hoặc bất kỳ follower nào, nhưng chỉ leader mới nhận ghi (từ góc nhìn của client, follower là chỉ đọc).

{{< fig num="6-1" id="fig_replication_leader_follower" src="/fig/ddia_0601.png" caption="Sao chép đơn leader (*single-leader replication*) gửi mọi lần ghi đến leader được chỉ định, rồi leader gửi luồng thay đổi đến các replica follower." class="ddia-figure ddia-figure--panorama" width="2880" height="925" />}}

Nếu cơ sở dữ liệu được sharding (xem [Chương 7](/ch7#ch_sharding)), mỗi shard có một leader. Leader của các shard khác nhau có thể nằm trên các node khác nhau, nhưng mỗi shard vẫn phải có đúng một leader. Trong ["Sao chép đa leader (*multi-leader replication*)"](/ch6#sec_replication_multi_leader), ta sẽ bàn một mô hình khác, trong đó cùng một shard có thể có nhiều leader cùng lúc.

Sao chép đơn leader (*single-leader replication*) được dùng cực kỳ rộng rãi. Nó là tính năng tích hợp sẵn của nhiều cơ sở dữ liệu quan hệ như PostgreSQL, MySQL, Oracle Data Guard [^3] và SQL Server Always On Availability Groups [^4], cũng được dùng bởi các cơ sở dữ liệu tài liệu như MongoDB, DynamoDB [^5], các message broker như Kafka, các thiết bị khối được sao chép (*replication*) như DRBD, và một số hệ thống tệp mạng. Nhiều thuật toán đồng thuận như Raft cũng dựa trên một leader duy nhất. Các hệ thống như CockroachDB [^6], TiDB [^7], etcd và hàng đợi quorum của RabbitMQ dùng nó để sao chép (*replication*), và tự động bầu leader mới khi leader cũ hỏng ([Chương 10](/ch10#ch_consistency) sẽ bàn kỹ hơn về đồng thuận).


> [!NOTE]
> Trong các tài liệu cũ hơn, bạn có thể gặp thuật ngữ *sao chép master–slave* (*master–slave replication*). Nó có nghĩa giống sao chép dựa trên leader (*leader-based replication*), nhưng ngày nay cách gọi này bị coi là xúc phạm và nên tránh [^8].


### Sao chép đồng bộ (*synchronous replication*) và bất đồng bộ {#sec_replication_sync_async}

Một chi tiết quan trọng của hệ thống sao chép (*replication*) là việc sao chép (*replication*) diễn ra *đồng bộ* (*synchronous*) hay *bất đồng bộ* (*asynchronous*). (Trong cơ sở dữ liệu quan hệ, đây thường là một tùy chọn cấu hình. Các hệ thống khác thường cố định một trong hai.)

Hãy hình dung tình huống ở {{< xref fig="6-1" page="/ch6" anchor="fig_replication_leader_follower" >}}Hình 6-1{{< /xref >}}: người dùng một website cập nhật ảnh đại diện. Tại một thời điểm nào đó, client gửi yêu cầu cập nhật đến leader. Ngay sau đó leader nhận được. Rồi tại một thời điểm nào đó, leader chuyển tiếp thay đổi dữ liệu cho các follower, và cuối cùng thông báo cho client rằng cập nhật đã thành công. {{< xref fig="6-2" page="/ch6" anchor="fig_replication_sync_replication" >}}Hình 6-2{{< /xref >}} cho thấy một trình tự thời gian khả dĩ.

{{< fig num="6-2" id="fig_replication_sync_replication" src="/fig/ddia_0602.png" caption="Sao chép dựa trên leader (*leader-based replication*), trong đó một follower sao chép đồng bộ (*synchronous replication*) và một follower sao chép bất đồng bộ (*asynchronous replication*)." class="ddia-figure ddia-figure--wide" width="2880" height="1128" />}}

Trong ví dụ ở {{< xref fig="6-2" page="/ch6" anchor="fig_replication_sync_replication" >}}Hình 6-2{{< /xref >}}, việc sao chép (*replication*) tới follower 1 là *đồng bộ*: leader phải đợi follower 1 xác nhận đã nhận được lần ghi thì mới báo thành công cho người dùng, và mới cho các client khác thấy lần ghi này. Việc sao chép (*replication*) tới follower 2 là *bất đồng bộ*: leader gửi thông điệp đi nhưng không chờ follower phản hồi.

Trong hình, follower 2 có một độ trễ đáng kể trước khi xử lý thông điệp. Thông thường việc sao chép (*replication*) khá nhanh: hầu hết hệ thống cơ sở dữ liệu áp dụng thay đổi lên follower trong chưa đến một giây, nhưng chúng không đảm bảo việc sao chép (*replication*) sẽ hoàn tất trong bao lâu. Đôi khi follower có thể tụt sau leader vài phút hoặc lâu hơn, chẳng hạn khi follower đang phục hồi sau sự cố, khi hệ thống đang chạy gần giới hạn dung lượng, hoặc khi mạng giữa các node có vấn đề.

Ưu điểm của sao chép đồng bộ (*synchronous replication*) là follower được đảm bảo có bản sao dữ liệu mới nhất và nhất quán với leader. Nếu leader đột ngột hỏng, ta có thể yên tâm dữ liệu vẫn còn trên follower. Nhược điểm là nếu follower đồng bộ không phản hồi — dù do sập, do lỗi mạng hay lý do nào khác — thì lần ghi không thể tiếp tục. Leader phải chặn mọi lần ghi cho đến khi replica đồng bộ khả dụng trở lại.

Vì vậy, đặt tất cả follower ở chế độ đồng bộ là không thực tế: chỉ cần một node ngừng hoạt động là cả hệ thống tê liệt. Trên thực tế, khi cơ sở dữ liệu nói về sao chép đồng bộ (*synchronous replication*), thường là *một* follower đồng bộ và các follower còn lại bất đồng bộ. Nếu follower đồng bộ không khả dụng hoặc quá chậm, một follower bất đồng bộ sẽ được chuyển thành đồng bộ. Cách này đảm bảo có ít nhất hai node giữ dữ liệu mới nhất: leader và một follower đồng bộ. Cấu hình này đôi khi còn gọi là *bán đồng bộ* (*semi-synchronous*).

Một số hệ thống cập nhật đồng bộ cho *đa số* replica (ví dụ 3 trong 5 replica, tính cả leader), còn thiểu số còn lại được cập nhật bất đồng bộ. Đây là một ví dụ về *quorum* (*quorum*), mà ta sẽ bàn thêm ở ["Đọc và ghi quorum"](/ch6#sec_replication_quorum_condition). Các hệ thống dùng giao thức đồng thuận để tự động bầu leader thường dùng quorum đa số. [Chương 10](/ch10#ch_consistency) sẽ quay lại vấn đề này.

Đôi khi sao chép dựa trên leader (*leader-based replication*) được cấu hình hoàn toàn bất đồng bộ. Khi đó, nếu leader hỏng và không thể phục hồi, mọi lần ghi chưa kịp sao chép (*replication*) sang follower sẽ bị mất. Nghĩa là lần ghi không được đảm bảo bền vững, dù đã xác nhận thành công cho client. Tuy vậy, cấu hình bất đồng bộ hoàn toàn cũng có ưu điểm: leader vẫn có thể tiếp tục xử lý ghi ngay cả khi mọi follower đều đã tụt lại.

Làm yếu tính bền vững nghe không giống một đánh đổi đáng giá, nhưng sao chép bất đồng bộ (*asynchronous replication*) vẫn được dùng rộng rãi, đặc biệt khi có rất nhiều follower hoặc chúng nằm rải rác ở các vị trí địa lý khác nhau [^9]. Ta sẽ quay lại điều này trong ["Vấn đề của replication lag"](/ch6#sec_replication_lag).

### Thiết lập replica mới {#sec_replication_new_replica}

Đôi khi cần thiết lập một follower mới, có thể để tăng số lượng replica, hoặc để thay thế node đã hỏng. Làm sao đảm bảo follower mới có bản sao chính xác của dữ liệu trên leader?

Chỉ chép tệp dữ liệu từ node này sang node khác thường là không đủ: client liên tục ghi vào cơ sở dữ liệu, dữ liệu luôn thay đổi, nên việc chép tệp thông thường sẽ đọc các phần khác nhau của cơ sở dữ liệu tại các thời điểm khác nhau, và kết quả có thể vô nghĩa.

Có thể khóa cơ sở dữ liệu để các tệp trên đĩa nhất quán, nhưng khi đó cơ sở dữ liệu không nhận ghi được, trái với mục tiêu sẵn sàng cao. May là việc thiết lập follower thường không cần ngừng hoạt động. Về mặt khái niệm, quy trình như sau:

1. Lấy một snapshot nhất quán của cơ sở dữ liệu leader tại một thời điểm. Nếu có thể, không khóa toàn bộ cơ sở dữ liệu. Hầu hết cơ sở dữ liệu đều có tính năng này vì sao lưu cũng cần đến nó. Trong một số trường hợp cần công cụ bên thứ ba, chẳng hạn Percona XtraBackup cho MySQL.
2. Sao chép (*replication*) snapshot sang node follower mới.
3. Follower kết nối đến leader và yêu cầu mọi thay đổi dữ liệu xảy ra sau khi snapshot được tạo. Điều này đòi hỏi snapshot gắn với một vị trí chính xác trong replication log của leader. Mỗi hệ thống gọi vị trí này một tên: PostgreSQL gọi là *log sequence number* (*log sequence number*, LSN). MySQL có hai cơ chế là *binlog coordinates* (*binlog coordinates*) và *global transaction identifier* (*global transaction identifiers*, GTID).
4. Khi follower xử lý xong phần thay đổi dữ liệu tồn đọng sau snapshot, ta nói nó đã *bắt kịp* (caught up). Từ đó nó có thể tiếp tục xử lý các thay đổi dữ liệu do leader tạo ra theo thời gian thực.

Các bước thiết lập follower trên thực tế khác nhau tùy cơ sở dữ liệu. Một số hệ thống tự động hóa hoàn toàn quy trình này, số khác đòi hỏi quản trị viên thực hiện thủ công một quy trình nhiều bước khá khó hiểu.

Bạn cũng có thể lưu trữ replication log vào lưu trữ đối tượng. Rồi định kỳ lưu snapshot của toàn bộ cơ sở dữ liệu vào lưu trữ đối tượng, tạo nên một giải pháp sao lưu và khôi phục sau thảm họa rất tốt cho cơ sở dữ liệu. Khi dựng follower mới, bước 1 và 2 cũng có thể thực hiện bằng cách tải các tệp này từ lưu trữ đối tượng. Chẳng hạn WAL-G cung cấp chức năng này cho PostgreSQL, MySQL và SQL Server, còn Litestream cung cấp chức năng tương tự cho SQLite.


<a id="sec_replication_object_storage"></a>

> [!TIP] Cơ sở dữ liệu dựa trên lưu trữ đối tượng
>
> Lưu trữ đối tượng không chỉ dùng để lưu trữ lâu dài. Nhiều cơ sở dữ liệu đã bắt đầu dùng các lưu trữ đối tượng như Amazon Web Services S3, Google Cloud Storage và Azure Blob Storage để phục vụ dữ liệu cho truy vấn trực tuyến. Đặt dữ liệu cơ sở dữ liệu vào lưu trữ đối tượng có nhiều lợi ích:
>
> * Lưu trữ đối tượng rẻ hơn các giải pháp lưu trữ đám mây khác. Nhờ đó cơ sở dữ liệu đám mây có thể đặt dữ liệu ít được truy vấn vào kho lưu trữ rẻ hơn, độ trễ cao hơn, đồng thời dùng bộ nhớ, SSD và NVMe để chứa tập dữ liệu đang làm việc (working set).
> * Lưu trữ đối tượng còn cung cấp sao chép (*replication*) đa availability zone, hai region hoặc đa region, kèm cam kết độ bền vững rất cao, nhờ vậy cơ sở dữ liệu cũng tránh được phí mạng giữa các availability zone.
> * Cơ sở dữ liệu có thể tận dụng tính năng *ghi có điều kiện* (*conditional write*) của lưu trữ đối tượng — về bản chất là một thao tác *compare-and-set* (CAS) — để triển khai giao dịch và bầu leader [^10] [^11].
> * Đặt dữ liệu của nhiều cơ sở dữ liệu trong cùng một lưu trữ đối tượng có thể đơn giản hóa việc tích hợp dữ liệu, nhất là khi dùng các định dạng mở như Apache Parquet và Apache Iceberg.
>
> Những lợi ích này chuyển trách nhiệm về giao dịch, bầu leader và sao chép (*replication*) sang lưu trữ đối tượng, nhờ đó đơn giản hóa đáng kể kiến trúc cơ sở dữ liệu.
>
> Hệ thống triển khai sao chép (*replication*) bằng lưu trữ đối tượng cũng phải đối mặt với một số đánh đổi. Đáng chú ý, độ trễ đọc và ghi của lưu trữ đối tượng cao hơn nhiều so với đĩa cục bộ hay thiết bị khối ảo như EBS. Nhiều nhà cung cấp đám mây còn tính phí theo số lần gọi API, buộc hệ thống phải gộp đọc và ghi thành lô để giảm chi phí, mà xử lý theo lô lại làm tăng thêm độ trễ. Ngoài ra, nhiều lưu trữ đối tượng không có giao diện hệ tệp chuẩn, nên những hệ thống chưa tích hợp lưu trữ đối tượng không thể tận dụng trực tiếp. Các giao diện như *hệ tệp trong không gian người dùng* (*filesystem in userspace*, FUSE) cho phép người vận hành mount bucket lưu trữ đối tượng thành hệ tệp, để ứng dụng không cần biết dữ liệu thực ra nằm trong lưu trữ đối tượng. Tuy nhiên, nhiều giao diện FUSE cho lưu trữ đối tượng không hỗ trợ các tính năng POSIX như ghi không tuần tự hay symbolic link, trong khi hệ thống có thể phụ thuộc vào chúng.
>
> Mỗi hệ thống xử lý các đánh đổi này một cách khác nhau. Một số dùng kiến trúc *lưu trữ phân tầng* (*tiered storage*), đặt dữ liệu ít truy cập vào lưu trữ đối tượng, còn dữ liệu mới hoặc thường dùng nằm trên phương tiện nhanh hơn như SSD, NVMe, thậm chí bộ nhớ. Một số khác dùng lưu trữ đối tượng làm tầng lưu trữ chính nhưng lưu WAL trên hệ thống lưu trữ độ trễ thấp như Amazon EBS hay Neon Safekeepers [^12]. Gần đây, một số hệ thống đi xa hơn với *kiến trúc không đĩa* (*zero-disk architecture*, ZDA): mọi dữ liệu đều được lưu bền vững vào lưu trữ đối tượng, còn đĩa và bộ nhớ chỉ đóng vai trò cache. Nhờ vậy node không cần giữ trạng thái bền vững, và việc vận hành được đơn giản hóa rất nhiều. WarpStream, Confluent Freight, Bufstream của Buf và Redpanda Serverless đều là các hệ thống tương thích Kafka xây dựng theo kiến trúc không đĩa. Gần như mọi kho dữ liệu đám mây hiện đại cũng dùng kiến trúc tương tự, cũng như công cụ tìm kiếm vector Turbopuffer và storage engine LSM cloud-native SlateDB.

### Xử lý sự cố node {#sec_replication_failover}

Bất kỳ node nào trong hệ thống cũng có thể ngừng hoạt động, do sự cố bất ngờ hoặc do bảo trì có kế hoạch, chẳng hạn khởi động lại máy để cài bản vá bảo mật kernel. Việc có thể khởi động lại từng node mà không làm gián đoạn dịch vụ rất có lợi cho vận hành và bảo trì. Vì vậy mục tiêu của ta là: toàn hệ thống vẫn tiếp tục chạy dù từng node riêng lẻ bị hỏng, và giảm tối đa ảnh hưởng của việc node ngừng hoạt động.

Làm thế nào để đạt tính sẵn sàng cao với sao chép dựa trên leader (*leader-based replication*)?

#### Follower hỏng: phục hồi bằng cách bắt kịp {#follower-failure-catch-up-recovery}

Mỗi follower ghi lại trên đĩa cục bộ các thay đổi dữ liệu nhận từ leader. Nếu follower sập rồi khởi động lại, hoặc mạng giữa leader và follower tạm thời gián đoạn, việc phục hồi khá dễ: từ log, follower biết giao dịch cuối cùng nó đã xử lý trước khi lỗi xảy ra. Sau đó nó kết nối đến leader và yêu cầu mọi thay đổi dữ liệu xảy ra trong lúc nó ngắt kết nối. Áp dụng xong các thay đổi này, nó đã bắt kịp leader và có thể tiếp tục nhận luồng thay đổi dữ liệu như trước.

Về mặt khái niệm, phục hồi follower rất đơn giản, nhưng về hiệu năng có thể rắc rối. Nếu thông lượng ghi của cơ sở dữ liệu rất cao, hoặc follower ngoại tuyến lâu, số lần ghi cần bắt kịp có thể rất lớn. Trong lúc bắt kịp, cả follower đang phục hồi lẫn leader đều chịu tải cao — leader còn phải gửi các lần ghi tồn đọng cho follower.

Khi mọi follower đã xác nhận xử lý xong một đoạn log ghi, leader có thể xóa đoạn đó. Nhưng nếu một follower không khả dụng lâu, leader phải chọn: hoặc giữ log mãi để đợi follower phục hồi và bắt kịp, với rủi ro cạn dung lượng đĩa của leader, hoặc xóa phần log chưa được follower đó xác nhận, và khi follower trở lại, nó không thể dùng log để bắt kịp nữa mà phải khôi phục từ bản sao lưu.

#### Leader hỏng: failover {#leader-failure-failover}

Xử lý leader hỏng khó hơn: phải đưa một follower lên làm leader mới, cấu hình lại client để gửi ghi đến leader mới, và để các follower còn lại bắt đầu nhận thay đổi dữ liệu từ leader mới. Quá trình này gọi là *failover* (*failover*).

Failover có thể thực hiện thủ công — báo cho quản trị viên biết leader đã hỏng, rồi họ thực hiện các bước cần thiết để chọn leader mới — hoặc tự động. Failover tự động thường gồm các bước sau:

1. *Xác định leader đã hỏng.* Có rất nhiều thứ có thể trục trặc: sập, mất điện, lỗi mạng, v.v. Không có cách nào chắc chắn để biết chính xác chuyện gì đã xảy ra, nên hầu hết hệ thống chỉ dùng *timeout* (*timeout*): các node thường xuyên trao đổi thông điệp qua lại, và nếu một node không phản hồi trong một khoảng thời gian (ví dụ 30 giây) thì coi là đã hỏng. (Quy tắc này không áp dụng khi chủ động đưa leader ngoại tuyến để bảo trì có kế hoạch.)
2. *Chọn leader mới.* Có thể thực hiện qua một cuộc bầu cử (đa số các replica còn lại chọn leader), hoặc do một *node điều khiển* (*controller node*) được chỉ định từ trước bổ nhiệm [^13]. Ứng viên phù hợp nhất thường là replica có dữ liệu mới nhất, tức đã nhận nhiều thay đổi dữ liệu nhất từ leader cũ, để giảm thiểu mất mát dữ liệu. Làm cho mọi node thống nhất về leader mới là một bài toán đồng thuận, sẽ được bàn chi tiết ở [Chương 10](/ch10#ch_consistency).
3. *Cấu hình lại hệ thống để dùng leader mới.* Client giờ phải gửi yêu cầu ghi đến leader mới (xem ["Định tuyến yêu cầu"](/ch7#sec_sharding_routing)). Nếu leader cũ trở lại, nó có thể vẫn tin mình là leader, không biết rằng các replica khác đã buộc nó thôi vai trò. Hệ thống phải đảm bảo leader cũ trở thành follower và thừa nhận leader mới.

Failover có rất nhiều chỗ có thể xảy ra sai sót:

* Nếu dùng sao chép bất đồng bộ (*asynchronous replication*), leader mới có thể chưa nhận được toàn bộ các lần ghi trước khi leader cũ hỏng. Sau khi chọn leader mới, nếu leader cũ gia nhập lại cụm thì những lần ghi chưa được sao chép (*replication*) đó xử lý ra sao? Trong lúc đó, leader mới có thể đã nhận những lần ghi xung đột với chúng. Cách phổ biến nhất là đơn giản loại bỏ các lần ghi chưa sao chép (*replication*) của leader cũ, nghĩa là những lần ghi mà bạn tưởng đã commit thực ra không được lưu bền vững.
* Việc loại bỏ các lần ghi đặc biệt nguy hiểm nếu nội dung cơ sở dữ liệu còn cần phối hợp với một hệ thống lưu trữ bên ngoài cơ sở dữ liệu. Chẳng hạn, GitHub từng gặp một sự cố [^14] khi một follower MySQL có dữ liệu lỗi thời được đưa lên làm leader. Cơ sở dữ liệu dùng bộ đếm tự tăng để gán khóa chính cho các dòng mới. Bộ đếm của leader mới tụt sau leader cũ, nên nó dùng lại một số khóa chính mà leader cũ đã gán. Các khóa chính này cũng được dùng trong kho lưu trữ Redis. Việc dùng lại khóa chính khiến dữ liệu MySQL và Redis không nhất quán, cuối cùng làm lộ một số dữ liệu riêng tư cho nhầm người dùng.
* Trong một số tình huống lỗi (xem [Chương 9](/ch9#ch_distributed)), có thể xảy ra việc hai node đều tin mình là leader. Tình huống này gọi là *split brain* (*split brain*) và rất nguy hiểm: nếu cả hai leader đều nhận ghi mà hệ thống không có quy trình giải quyết xung đột (xem ["Sao chép đa leader (*multi-leader replication*)"](/ch6#sec_replication_multi_leader)), dữ liệu rất dễ bị mất hoặc hỏng. Một số hệ thống có cơ chế an toàn: khi phát hiện hai leader thì tắt một trong hai node, nhưng nếu thiết kế không tốt, cơ chế này có thể tắt cả hai node [^15]. Hơn nữa, đến lúc hệ thống phát hiện split brain và tắt node cũ, có thể đã quá muộn, dữ liệu đã hỏng.
* Nên đặt timeout bao lâu trước khi tuyên bố leader đã hỏng? Timeout càng dài thì thời gian phục hồi khi leader thực sự hỏng càng lâu. Timeout quá ngắn lại dễ kích hoạt failover không cần thiết. Chẳng hạn, một đợt tăng tải đột biến ngắn có thể khiến thời gian phản hồi của node vượt timeout, hoặc mạng chập chờn có thể làm trễ gói tin. Nếu hệ thống vốn đã chịu tải cao hoặc gặp vấn đề mạng, failover không cần thiết chỉ làm tình hình tệ hơn.


> [!NOTE]
> Ngăn split brain bằng cách giới hạn hoặc tắt leader cũ được gọi là *fencing* (*fencing*), cách gọi hình ảnh hơn là *Shoot The Other Node In The Head* (STONITH). ["Khóa phân tán và lease"](/ch9#sec_distributed_lock_fencing) sẽ bàn chi tiết hơn về fencing.


Những vấn đề này không có lời giải đơn giản. Vì vậy, dù phần mềm hỗ trợ failover tự động, một số đội vận hành vẫn thích thực hiện thủ công.

Điều quan trọng nhất của failover là chọn follower có dữ liệu mới nhất làm leader mới. Với sao chép đồng bộ (*synchronous replication*) hoặc bán đồng bộ, nên chọn follower mà leader cũ đã chờ trước khi xác nhận ghi, với sao chép bất đồng bộ (*asynchronous replication*), có thể chọn follower có log sequence number lớn nhất. Cách này giảm thiểu mất dữ liệu khi failover: mất các lần ghi trong vài phần giây thì có thể chấp nhận được, còn chọn phải follower tụt hậu nhiều ngày có thể là thảm họa.

Sự cố node, mạng không đáng tin cậy, và các đánh đổi xoay quanh tính nhất quán, độ bền vững, tính sẵn sàng và độ trễ của replica đều là những vấn đề căn bản của hệ thống phân tán. [Chương 9](/ch9#ch_distributed) và [Chương 10](/ch10#ch_consistency) sẽ đi sâu hơn.

### Triển khai replication log {#sec_replication_implementation}

Sao chép dựa trên leader (*leader-based replication*) thực sự hoạt động ra sao ở tầng dưới? Thực tế có vài cách sao chép (*replication*) khác nhau, ta lần lượt điểm qua dưới đây.

#### Sao chép (*replication*) dựa trên câu lệnh {#statement-based-replication}

Trong trường hợp đơn giản nhất, leader ghi lại mọi yêu cầu ghi nó thực thi (tức *câu lệnh*) và gửi log câu lệnh này cho follower. Với cơ sở dữ liệu quan hệ, điều đó nghĩa là mọi câu lệnh `INSERT`, `UPDATE` hay `DELETE` đều được chuyển cho follower, mỗi follower phân tích cú pháp và thực thi câu lệnh SQL này như thể nó đến trực tiếp từ client.

Nghe có vẻ hợp lý, nhưng cách sao chép (*replication*) này có nhiều chỗ dễ sai:

* Bất kỳ câu lệnh nào gọi hàm *phi tất định* (*nondeterministic*) đều có thể cho giá trị khác nhau trên mỗi replica, chẳng hạn `NOW()` để lấy ngày giờ hiện tại hoặc `RAND()` để lấy số ngẫu nhiên.
* Nếu câu lệnh dùng cột tự tăng, hoặc phụ thuộc vào dữ liệu hiện có trong cơ sở dữ liệu (ví dụ `UPDATE …​ WHERE <some condition>`), thì nó phải được thực thi theo đúng cùng thứ tự trên mọi replica, nếu không có thể cho kết quả khác nhau. Khi có nhiều giao dịch thực thi đồng thời, điều này trở thành một hạn chế.
* Các câu lệnh có *tác dụng phụ* (*side effect*) (ví dụ trigger, *thủ tục lưu trữ* (*stored procedure*), hàm do người dùng định nghĩa) có thể gây ra tác dụng phụ khác nhau trên từng replica, trừ khi các tác dụng phụ đó hoàn toàn tất định.

Những vấn đề này có thể được né tránh. Chẳng hạn, khi ghi lại câu lệnh, leader có thể thay các lời gọi hàm phi tất định bằng giá trị trả về cố định, để mọi follower nhận cùng một giá trị. Việc thực thi các câu lệnh tất định theo thứ tự cố định rất giống mô hình event sourcing giới thiệu ở ["Event sourcing và CQRS"](/ch3#sec_datamodels_events). Cách này còn được gọi là *sao chép máy trạng thái* (*state machine replication*). ["Dùng shared log"](/ch10#sec_consistency_smr) sẽ bàn về nền tảng lý thuyết của nó.

MySQL trước phiên bản 5.1 dùng sao chép (*replication*) dựa trên câu lệnh. Vì log khá gọn nhẹ, đôi khi ngày nay vẫn còn dùng cách này. Tuy nhiên, theo mặc định, hễ trong câu lệnh có bất kỳ yếu tố phi tất định nào, MySQL sẽ chuyển sang sao chép (*replication*) dựa trên dòng (sẽ nói ở phần sau). VoltDB cũng dùng sao chép (*replication*) dựa trên câu lệnh, và yêu cầu giao dịch phải tất định để đảm bảo an toàn [^16]. Tuy nhiên, trên thực tế rất khó đảm bảo tính tất định, nên nhiều cơ sở dữ liệu ưa chuộng các cách sao chép (*replication*) khác.

#### Chuyển write-ahead log (WAL) {#write-ahead-log-wal-shipping}

Ở [Chương 4](/ch4#ch_storage), ta đã thấy storage engine B-tree cần write-ahead log để hoạt động đáng tin cậy: mỗi thay đổi đều được ghi vào WAL trước, để sau khi sập có thể khôi phục cây về trạng thái nhất quán. WAL chứa mọi thông tin cần thiết để đưa chỉ mục (*index*) và heap về trạng thái nhất quán, nên chính log đó cũng có thể dùng để dựng replica trên node khác: ngoài việc ghi log xuống đĩa, leader còn gửi nó qua mạng cho follower. Follower xử lý log sẽ dựng nên một bản sao tệp giống hệt trên leader.

Các cơ sở dữ liệu như PostgreSQL và Oracle dùng cách sao chép (*replication*) này [^17] [^18]. Nhược điểm chính là log mô tả dữ liệu ở tầng rất thấp: WAL ghi lại những byte nào ở khối đĩa nào đã thay đổi. Do đó sao chép (*replication*) gắn chặt với storage engine. Khi cơ sở dữ liệu nâng cấp từ phiên bản này sang phiên bản khác và định dạng lưu trữ thay đổi, thường không thể chạy các phiên bản phần mềm cơ sở dữ liệu khác nhau trên leader và follower.

Điều này nghe như một chi tiết triển khai vụn vặt, nhưng có thể gây ảnh hưởng lớn đến vận hành. Nếu giao thức sao chép (*replication*) cho phép follower chạy phiên bản phần mềm mới hơn leader, ta có thể nâng cấp follower trước, rồi thực hiện failover để một node đã nâng cấp trở thành leader mới, qua đó nâng cấp phần mềm cơ sở dữ liệu không cần ngừng hoạt động. Nếu giao thức sao chép (*replication*) không cho phép lệch phiên bản — như thường thấy với chuyển WAL — thì việc nâng cấp như vậy phải ngừng hoạt động.

<a id="sec_replication_logical"></a>

#### Sao chép (*replication*) log logic (dựa trên dòng) {#logical-row-based-log-replication}

Một cách khác là dùng các định dạng log khác nhau cho sao chép (*replication*) và storage engine, nhờ đó tách replication log khỏi chi tiết nội bộ của storage engine. Loại replication log này gọi là *logical log* (*logical log*), để phân biệt với cách biểu diễn dữ liệu (*vật lý*, *physical*) của storage engine.

Logical log của cơ sở dữ liệu quan hệ thường là một chuỗi bản ghi mô tả các lần ghi vào bảng ở mức độ chi tiết từng dòng:

* Khi chèn một dòng, log chứa giá trị mới của mọi cột.
* Khi xóa một dòng, log chứa đủ thông tin để xác định duy nhất dòng bị xóa, thường là khóa chính. Nếu bảng không có khóa chính thì cần ghi giá trị cũ của mọi cột.
* Khi cập nhật một dòng, log chứa đủ thông tin để xác định duy nhất dòng bị cập nhật, cùng giá trị mới của mọi cột (hoặc ít nhất là giá trị mới của mọi cột có thay đổi).

Giao dịch sửa nhiều dòng sinh ra nhiều bản ghi log như vậy, theo sau là một bản ghi cho biết giao dịch đã commit. Khi MySQL được cấu hình sao chép (*replication*) dựa trên dòng, ngoài WAL nó còn duy trì một logical replication log riêng gọi là *binlog*. PostgreSQL thì giải mã WAL vật lý thành các sự kiện chèn, cập nhật, xóa dòng để thực hiện sao chép (*replication*) logic [^19].

Vì logical log tách khỏi chi tiết nội bộ của storage engine nên dễ giữ tương thích ngược hơn, nhờ đó leader và follower có thể chạy các phiên bản phần mềm cơ sở dữ liệu khác nhau, và có thể nâng cấp lên phiên bản mới với rất ít thời gian ngừng hoạt động [^20].

Định dạng logical log cũng dễ để ứng dụng bên ngoài phân tích hơn. Điều này hữu ích khi muốn gửi nội dung cơ sở dữ liệu sang hệ thống bên ngoài, chẳng hạn đưa vào kho dữ liệu để phân tích ngoại tuyến, hoặc xây chỉ mục (*index*) và cache tùy biến [^21]. Kỹ thuật này gọi là *change data capture* (*change data capture*, CDC), và ta sẽ quay lại ở ["Change data capture"](/ch12#sec_stream_cdc).

## Vấn đề của replication lag {#sec_replication_lag}

Chịu được sự cố node chỉ là một lý do để dùng sao chép (*replication*). Như đã nhắc ở ["Hệ thống phân tán và hệ thống một node"](/ch1#sec_introduction_distributed), các lý do khác gồm khả năng mở rộng (xử lý lượng yêu cầu vượt quá sức một máy) và độ trễ (đặt replica gần người dùng hơn về mặt địa lý).

Sao chép dựa trên leader (*leader-based replication*) yêu cầu mọi lần ghi đi qua một node, nhưng truy vấn chỉ đọc có thể gửi đến bất kỳ replica nào. Với khối lượng công việc đọc nhiều, ghi ít — thường thấy ở các dịch vụ trực tuyến — có một phương án hấp dẫn: tạo nhiều follower và dàn yêu cầu đọc ra các follower đó. Cách này vừa giảm tải cho leader, vừa cho phép các replica ở gần xử lý yêu cầu đọc.

Trong kiến trúc *mở rộng đọc* (*read-scaling*) này, chỉ cần thêm follower là tăng được khả năng xử lý yêu cầu chỉ đọc. Tuy nhiên, cách này thực tế chỉ phù hợp với sao chép bất đồng bộ (*asynchronous replication*). Nếu cố sao chép đồng bộ (*synchronous replication*) đến mọi follower, chỉ cần một node hỏng hoặc mạng gián đoạn là cả hệ thống không ghi được. Node càng nhiều thì càng dễ có node ngừng hoạt động, nên cấu hình đồng bộ hoàn toàn sẽ cực kỳ thiếu tin cậy.

Đáng tiếc là khi ứng dụng đọc từ follower *bất đồng bộ*, nó có thể thấy thông tin lỗi thời nếu follower tụt hậu. Cơ sở dữ liệu vì thế có vẻ không nhất quán: chạy cùng một truy vấn đồng thời trên leader và follower có thể cho kết quả khác nhau, vì follower chưa phản ánh hết các lần ghi. Sự không nhất quán này chỉ là tạm thời — nếu ngừng ghi và chờ một lúc, follower cuối cùng sẽ bắt kịp leader và trở lại nhất quán. Vì vậy hiện tượng này được gọi là *nhất quán cuối cùng* (*eventual consistency*) [^22].


> [!NOTE]
> Thuật ngữ *nhất quán cuối cùng* do Douglas Terry và cộng sự đưa ra [^23], được Werner Vogels phổ biến [^24], và về sau trở thành khẩu hiệu của nhiều dự án NoSQL. Tuy nhiên, nhất quán cuối cùng không phải đặc quyền của cơ sở dữ liệu NoSQL: cơ sở dữ liệu quan hệ dùng sao chép bất đồng bộ (*asynchronous replication*) thì follower của nó cũng có đặc tính tương tự.


Từ "cuối cùng" được cố ý để mơ hồ: nhìn chung, replica có thể tụt hậu bao lâu thì không có giới hạn trên. Khi hoạt động bình thường, độ trễ giữa lúc ghi diễn ra trên leader và lúc thay đổi phản ánh trên follower — tức *replication lag* (*replication lag*) — có thể chỉ vài phần giây, thực tế khó nhận ra. Nhưng khi hệ thống chạy gần giới hạn dung lượng hoặc mạng gặp sự cố, độ trễ dễ dàng tăng lên vài giây, thậm chí vài phút.

Khi độ trễ lớn đến mức đó, sự không nhất quán do nó gây ra không còn là vấn đề lý thuyết mà trở thành vấn đề thực sự của ứng dụng. Phần này sẽ tập trung vào ba loại vấn đề dễ phát sinh do replication lag, và điểm qua một số giải pháp.

### Read-your-writes {#sec_replication_ryw}

Nhiều ứng dụng cho phép người dùng gửi dữ liệu rồi xem lại nội dung mình đã gửi. Đó có thể là một bản ghi trong cơ sở dữ liệu khách hàng, một bình luận dưới chủ đề thảo luận, hoặc thứ gì đó tương tự. Dữ liệu mới phải được ghi vào leader, nhưng khi người dùng xem, dữ liệu có thể được đọc từ follower. Cách này đặc biệt phù hợp nếu dữ liệu được đọc thường xuyên nhưng ít khi ghi.

Sao chép bất đồng bộ (*asynchronous replication*) gây ra vấn đề ở đây, như minh họa ở {{< xref fig="6-3" page="/ch6" anchor="fig_replication_read_your_writes" >}}Hình 6-3{{< /xref >}}: khi người dùng xem dữ liệu ngay sau khi ghi, dữ liệu mới có thể chưa đến replica đó. Với người dùng, dữ liệu vừa gửi trông như bị mất, và họ đương nhiên sẽ không hài lòng.

{{< fig num="6-3" id="fig_replication_read_your_writes" src="/fig/ddia_0603.png" caption="Người dùng ghi dữ liệu rồi đọc từ một replica cũ. Để ngăn dị thường này, cần nhất quán read-after-write." class="ddia-figure ddia-figure--wide" width="2880" height="1128" />}}

Trường hợp này cần *nhất quán read-after-write* (*read-after-write consistency*), còn gọi là *nhất quán read-your-writes* (*read-your-writes consistency*) [^23]. Nó đảm bảo rằng khi người dùng tải lại trang, họ luôn thấy các cập nhật mình đã gửi. Còn với người dùng khác thì không đảm bảo gì: cập nhật của họ có thể phải một lúc sau mới xuất hiện. Nhưng ít nhất người dùng có thể yên tâm dữ liệu mình nhập đã được lưu đúng.

Làm sao triển khai nhất quán read-after-write trong hệ thống sao chép dựa trên leader (*leader-based replication*)? Có nhiều cách, ví dụ:

* Khi đọc nội dung mà người dùng có thể đã sửa, đọc từ leader hoặc từ follower được cập nhật đồng bộ. Nội dung khác thì đọc từ follower cập nhật bất đồng bộ. Điều này đòi hỏi hệ thống biết được, mà không cần truy vấn thực tế, liệu một mục nội dung có thể đã bị sửa hay không. Ví dụ, hồ sơ cá nhân trên mạng xã hội thường chỉ do chính chủ chỉnh sửa. Do đó có thể đặt một quy tắc đơn giản: hồ sơ của chính người dùng luôn đọc từ leader, hồ sơ của người dùng khác đọc từ follower.
* Nếu phần lớn nội dung trong ứng dụng có thể do người dùng chỉnh sửa, cách trên sẽ không hiệu quả, vì hầu như mọi thứ đều phải đọc từ leader, làm mất ý nghĩa của mở rộng đọc. Khi đó có thể dùng tiêu chí khác để quyết định có đọc từ leader hay không. Ví dụ, ghi lại thời điểm cập nhật gần nhất, và mọi lần đọc trong một phút sau đó đều gửi đến leader [^25]. Cũng có thể theo dõi replication lag của follower và không gửi truy vấn đến những follower tụt sau leader hơn một phút.
* Client có thể nhớ timestamp của lần ghi gần nhất, và hệ thống đảm bảo replica phục vụ lần đọc của người dùng đó đã phản ánh ít nhất đến timestamp này. Nếu replica chưa đủ mới, có thể để replica khác xử lý lần đọc, hoặc để truy vấn chờ replica bắt kịp [^26]. Timestamp có thể là *timestamp logic* (*logical timestamp*, ví dụ log sequence number thể hiện thứ tự ghi), hoặc lấy từ đồng hồ hệ thống thực, ở trường hợp sau, đồng hồ phải được đồng bộ chính xác, xem ["Đồng hồ không đáng tin cậy"](/ch9#sec_distributed_clocks).
* Nếu các replica phân bố ở nhiều region (để gần người dùng hơn hoặc tăng tính sẵn sàng), có thêm một tầng phức tạp: mọi yêu cầu phải do leader xử lý đều phải được định tuyến đến region chứa leader.

Một tình huống phức tạp khác xuất hiện khi cùng một người dùng truy cập dịch vụ từ nhiều thiết bị, chẳng hạn vừa dùng trình duyệt máy tính vừa dùng ứng dụng di động. Khi đó có thể cần nhất quán read-after-write *xuyên thiết bị*: người dùng nhập thông tin trên một thiết bị, rồi xem trên thiết bị khác, thì phải thấy thông tin vừa nhập.

Cần cân nhắc thêm các vấn đề sau:

* Cách dựa vào việc "nhớ timestamp lần cập nhật gần nhất của người dùng" trở nên khó hơn, vì mã chạy trên một thiết bị không biết thiết bị kia đã cập nhật gì. Siêu dữ liệu này cần được lưu tập trung.
* Nếu replica phân bố ở nhiều region, các kết nối từ những thiết bị khác nhau không được đảm bảo sẽ được định tuyến đến cùng một region. Ví dụ, máy tính để bàn của người dùng dùng băng thông rộng tại nhà, còn điện thoại dùng mạng di động, đường đi mạng của hai thiết bị có thể hoàn toàn khác nhau. Nếu giải pháp yêu cầu đọc từ leader, trước hết có thể phải định tuyến yêu cầu từ mọi thiết bị của người dùng đó đến cùng một region.


> [!TIP] Region và availability zone
>
> Cuốn sách này dùng *region* (*region*) để chỉ một hoặc nhiều trung tâm dữ liệu ở cùng một vị trí địa lý. Nhà cung cấp đám mây thường đặt nhiều trung tâm dữ liệu trong cùng một khu vực địa lý, mỗi trung tâm dữ liệu gọi là một *availability zone* (*availability zone*), gọi tắt là *zone*. Như vậy, một region đám mây gồm nhiều availability zone. Mỗi availability zone là một trung tâm dữ liệu nằm trong cơ sở vật chất riêng biệt, có hệ thống điện, làm mát và hạ tầng riêng.
>
> Các availability zone trong cùng một region được nối với nhau bằng mạng tốc độ cao, với độ trễ đủ thấp để hầu hết hệ thống phân tán có thể rải node trên nhiều availability zone, như thể chúng nằm cùng một zone. Cấu hình đa availability zone chịu được việc một availability zone ngừng hoạt động hoàn toàn, nhưng không chịu được sự cố cấp region khi mọi availability zone trong region đều không khả dụng. Để tiếp tục hoạt động khi xảy ra sự cố cấp region, hệ thống phân tán phải được triển khai trên nhiều region, và điều này có thể kéo theo độ trễ cao hơn, thông lượng thấp hơn và phí mạng đám mây cao hơn. Ta sẽ bàn thêm về các đánh đổi này ở ["Topology sao chép (*replication topology*) đa leader"](/ch6#sec_replication_topologies). Tạm thời, hãy nhớ rằng region trong sách này là một nhóm availability zone hoặc trung tâm dữ liệu ở cùng một vị trí địa lý.


### Monotonic reads {#sec_replication_monotonic_reads}

Dị thường thứ hai có thể xảy ra khi đọc từ follower bất đồng bộ là người dùng có thể thấy *thời gian đi ngược*.

Điều này có thể xảy ra khi người dùng đọc nhiều lần liên tiếp từ các replica khác nhau. Ví dụ, ở {{< xref fig="6-4" page="/ch6" anchor="fig_replication_monotonic_reads" >}}Hình 6-4{{< /xref >}}, người dùng 2345 thực hiện hai lần cùng một truy vấn: lần đầu truy vấn một follower có replication lag nhỏ, lần sau truy vấn một follower có độ trễ lớn hơn. (Tình huống này rất dễ xảy ra khi người dùng làm mới trang web, còn mỗi yêu cầu được định tuyến ngẫu nhiên đến một server khác nhau.) Truy vấn đầu trả về bình luận do người dùng 1234 vừa thêm, nhưng truy vấn thứ hai không trả về gì, vì follower tụt hậu chưa nhận được lần ghi đó. Thực chất, truy vấn thứ hai quan sát trạng thái hệ thống cũ hơn truy vấn đầu. Nếu truy vấn đầu vốn không có kết quả thì cũng không sao, vì người dùng 2345 có lẽ không biết người dùng 1234 vừa thêm bình luận, nhưng nếu bình luận hiện ra rồi lại biến mất thì rất khó hiểu.

{{< fig num="6-4" id="fig_replication_monotonic_reads" src="/fig/ddia_0604.png" caption="Người dùng đọc từ replica mới trước, rồi đọc từ replica cũ. Thời gian như bị quay ngược. Để ngăn dị thường này, cần monotonic reads." class="ddia-figure ddia-figure--wide" width="2880" height="1474" />}}

*Monotonic reads* [^22] đảm bảo dị thường này không xảy ra. Nó yếu hơn nhất quán mạnh, nhưng mạnh hơn nhất quán cuối cùng. Khi đọc dữ liệu, vẫn có thể thấy giá trị cũ. Monotonic reads chỉ đảm bảo rằng khi cùng một người dùng thực hiện nhiều lần đọc tuần tự thì không thấy thời gian đi ngược — một khi đã đọc được dữ liệu mới hơn, về sau sẽ không đọc lại dữ liệu cũ hơn.

Một cách triển khai monotonic reads là đảm bảo mỗi người dùng luôn đọc từ cùng một replica (những người dùng khác nhau có thể dùng replica khác nhau). Chẳng hạn, có thể chọn replica dựa trên hash của ID người dùng thay vì chọn ngẫu nhiên. Nếu replica đó hỏng, cần định tuyến lại truy vấn của người dùng sang replica khác.

### Consistent prefix reads {#sec_replication_consistent_prefix}

Dị thường thứ ba do replication lag vi phạm quan hệ nhân quả. Hãy hình dung đoạn hội thoại ngắn giữa ông Poons và bà Cake:

Ông Poons
:   Bà Cake, bà nhìn thấy tương lai xa đến đâu?

Bà Cake
:   Thường khoảng mười giây, ông Poons.

Hai câu này có quan hệ nhân quả: bà Cake nghe câu hỏi của ông Poons rồi mới trả lời.

Giờ hãy hình dung người thứ ba nghe lén cuộc trò chuyện này qua các follower. Lời của bà Cake đi qua follower có replication lag nhỏ, còn lời của ông Poons đi qua follower có độ trễ lớn hơn (xem {{< xref fig="6-5" page="/ch6" anchor="fig_replication_consistent_prefix" >}}Hình 6-5{{< /xref >}}). Khi đó người nghe lén sẽ nghe thấy:

Bà Cake
:   Thường khoảng mười giây, ông Poons.

Ông Poons
:   Bà Cake, bà nhìn thấy tương lai xa đến đâu?

Với người nghe lén, bà Cake đã trả lời câu hỏi trước khi ông Poons hỏi. Khả năng ngoại cảm này thật ấn tượng, nhưng cũng rất khó hiểu [^27].

{{< fig num="6-5" id="fig_replication_consistent_prefix" src="/fig/ddia_0605.png" caption="Nếu một số shard sao chép (*replication*) chậm hơn các shard khác, người quan sát có thể thấy câu trả lời trước câu hỏi." class="ddia-figure ddia-figure--standard" width="2880" height="1843" />}}

Để ngăn dị thường này cần một đảm bảo khác: *consistent prefix reads* (*consistent prefix reads*) [^22]. Nó đảm bảo rằng nếu một chuỗi lần ghi xảy ra theo một thứ tự nhất định, thì bất kỳ ai đọc các lần ghi đó cũng sẽ thấy chúng xuất hiện theo đúng thứ tự ấy.

Đây đặc biệt là vấn đề trong cơ sở dữ liệu được sharding (phân region), mà ta sẽ bàn ở [Chương 7](/ch7#ch_sharding). Nếu cơ sở dữ liệu luôn áp dụng các lần ghi theo cùng một thứ tự, lần đọc sẽ luôn thấy một tiền tố nhất quán, và dị thường này không xảy ra. Tuy nhiên, ở nhiều cơ sở dữ liệu phân tán, các shard hoạt động độc lập với nhau, không có thứ tự ghi toàn cục. Khi người dùng đọc cơ sở dữ liệu, họ có thể thấy một phần ở trạng thái cũ hơn, còn một phần khác ở trạng thái mới hơn.

Một giải pháp là đảm bảo các lần ghi có quan hệ nhân quả với nhau được ghi vào cùng một shard, nhưng một số ứng dụng không thể làm điều này một cách hiệu quả. Cũng có những thuật toán theo dõi tường minh các phụ thuộc nhân quả, và ta sẽ quay lại ở ["Quan hệ “xảy ra trước” và tính đồng thời"](/ch6#sec_replication_happens_before).

### Giải pháp cho replication lag {#id131}

Khi làm việc với hệ thống nhất quán cuối cùng, đáng suy nghĩ nghiêm túc: nếu replication lag tăng lên vài phút hoặc vài giờ thì ứng dụng sẽ hành xử ra sao? Nếu câu trả lời là "không ảnh hưởng gì" thì tuyệt, nhưng nếu nó mang lại trải nghiệm tồi cho người dùng, bạn phải thiết kế hệ thống để cung cấp đảm bảo mạnh hơn, chẳng hạn read-after-write. Sao chép (*replication*) vốn bất đồng bộ mà lại giả vờ nó đồng bộ thì sớm muộn cũng gây rắc rối.

Như đã nói ở trên, ứng dụng có thể cung cấp đảm bảo mạnh hơn cơ sở dữ liệu bên dưới, ví dụ gửi một số lần đọc đến leader hoặc follower được cập nhật đồng bộ. Tuy nhiên, xử lý những vấn đề này trong mã ứng dụng vừa phức tạp vừa dễ sai.

Với lập trình viên ứng dụng, mô hình lập trình đơn giản nhất là chọn cơ sở dữ liệu cung cấp đảm bảo nhất quán mạnh cho các replica (như linearizability, xem [Chương 10](/ch10#ch_consistency)) và giao dịch ACID (xem [Chương 8](/ch8#ch_transactions)). Khi đó có thể gần như bỏ qua thách thức do sao chép (*replication*) gây ra và coi cơ sở dữ liệu như chỉ có một node. Phong trào *NoSQL* nổi lên đầu thập niên 2010 từng cổ xúy quan điểm rằng những tính năng này hạn chế khả năng mở rộng, và hệ thống quy mô lớn buộc phải chấp nhận nhất quán cuối cùng.

Kể từ đó, nhiều cơ sở dữ liệu đã cung cấp nhất quán mạnh và giao dịch mà vẫn giữ lợi thế của cơ sở dữ liệu phân tán về khả năng chịu lỗi, tính sẵn sàng cao và khả năng mở rộng. Như đã nhắc ở ["Mô hình quan hệ và mô hình tài liệu"](/ch3#sec_datamodels_history), để phân biệt với NoSQL, xu hướng này được gọi là *NewSQL* (dù trọng tâm không nằm ở bản thân SQL mà ở các phương pháp mới để quản lý giao dịch có khả năng mở rộng).

Dù ngày nay đã có cơ sở dữ liệu phân tán nhất quán mạnh và mở rộng được, một số ứng dụng vẫn có lý do chính đáng để chọn các cách sao chép (*replication*) khác với đảm bảo nhất quán yếu hơn: chúng có thể bền bỉ hơn trước gián đoạn mạng, và chi phí thấp hơn các hệ thống giao dịch. Phần còn lại của chương này sẽ tiếp tục khám phá các cách tiếp cận đó.

## Sao chép đa leader (*multi-leader replication*) {#sec_replication_multi_leader}

Cho đến giờ, chương này mới chỉ bàn về kiến trúc sao chép (*replication*) dùng một leader duy nhất. Dù đây là cách làm phổ biến, vẫn còn một số lựa chọn đáng chú ý.

Sao chép đơn leader (*single-leader replication*) có một nhược điểm lớn: mọi lần ghi đều phải đi qua leader duy nhất. Dù vì lý do gì, nếu không kết nối được với leader — chẳng hạn mạng giữa client và leader bị gián đoạn — thì không thể ghi vào cơ sở dữ liệu.

Mô hình đơn leader có thể mở rộng một cách tự nhiên để cho phép nhiều node nhận ghi. Việc sao chép (*replication*) vẫn diễn ra theo cách cũ: mỗi node xử lý một lần ghi đều phải chuyển tiếp thay đổi dữ liệu cho mọi node khác. Ta gọi cấu hình này là *sao chép đa leader* (*multi-leader replication*), còn gọi là mô hình *active/active* (*active/active*) hay *sao chép hai chiều* (*bidirectional replication*). Trong cấu hình này, mỗi leader đồng thời đóng vai follower của các leader khác.

Cũng như sao chép đơn leader (*single-leader replication*), sao chép đa leader (*multi-leader replication*) có thể là đồng bộ hoặc bất đồng bộ. Giả sử có hai leader *A* và *B*, và cần ghi vào *A*. Nếu lần ghi phải được sao chép đồng bộ (*synchronous replication*) từ *A* sang *B*, thì khi mạng giữa hai bên gián đoạn, không thể ghi vào *A* cho đến khi mạng phục hồi. Sao chép đa leader (*multi-leader replication*) đồng bộ như vậy cung cấp một mô hình rất giống sao chép đơn leader (*single-leader replication*): tức là tương đương việc đặt *B* làm leader, còn *A* chuyển tiếp mọi yêu cầu ghi sang *B* để thực thi.

Vì vậy, phần này sẽ không đi sâu vào sao chép đa leader (*multi-leader replication*) đồng bộ mà coi nó tương đương sao chép đơn leader (*single-leader replication*). Phần thảo luận dưới đây tập trung vào sao chép đa leader (*multi-leader replication*) bất đồng bộ: dù kết nối giữa một leader với các leader khác bị gián đoạn, nó vẫn có thể xử lý ghi.

### Vận hành xuyên nhiều region {#sec_replication_multi_dc}

Dùng cấu hình đa leader trong một region đơn lẻ thường ít ý nghĩa, vì lợi ích hiếm khi bù được độ phức tạp tăng thêm. Tuy nhiên, có một số tình huống mà cấu hình này thực sự hợp lý.

Hãy hình dung một cơ sở dữ liệu có replica ở nhiều region, có thể để vẫn chạy được khi cả một region ngừng hoạt động, hoặc để gần người dùng hơn về mặt địa lý. Kiểu triển khai này gọi là *phân tán theo địa lý* (*geographically distributed*), *geo-distributed* (*geo-distributed*) hoặc *geo-replicated* (*geo-replicated*). Với sao chép đơn leader (*single-leader replication*), leader phải nằm ở *một* region, và mọi lần ghi đều phải đi qua region đó.

Trong cấu hình đa leader, *mỗi* region có thể có một leader. {{< xref fig="6-6" page="/ch6" anchor="fig_replication_multi_dc" >}}Hình 6-6{{< /xref >}} minh họa kiến trúc này: trong từng region dùng sao chép (*replication*) leader–follower thông thường (follower có thể nằm ở availability zone khác với leader), còn giữa các region, leader của mỗi region sao chép (*replication*) thay đổi sang leader của các region khác.

{{< fig num="6-6" id="fig_replication_multi_dc" src="/fig/ddia_0606.png" caption="Sao chép đa leader (*multi-leader replication*) xuyên nhiều region." class="ddia-figure ddia-figure--wide" width="2658" height="1293" />}}

Dưới đây so sánh cấu hình đơn leader và đa leader khi triển khai đa region:

Hiệu năng
:   Trong cấu hình đơn leader, mọi lần ghi phải đi qua internet đến region của leader. Điều này có thể làm tăng đáng kể độ trễ ghi, thậm chí đi ngược mục đích triển khai đa region. Trong cấu hình đa leader, mỗi lần ghi được xử lý ngay tại region cục bộ rồi sao chép bất đồng bộ (*asynchronous replication*) sang các region khác. Do đó độ trễ mạng giữa các region được che khỏi người dùng, và hiệu năng cảm nhận được có thể tốt hơn.

Chịu được sự cố ngừng hoạt động của region
:   Trong cấu hình đơn leader, nếu region chứa leader không khả dụng, có thể failover để đưa một follower ở region khác lên làm leader. Trong cấu hình đa leader, mỗi region có thể tiếp tục hoạt động độc lập với các region khác. Khi region ngoại tuyến hoạt động trở lại, việc sao chép (*replication*) sẽ bắt kịp.

Chịu được sự cố mạng
:   Ngay cả khi dùng kết nối chuyên dụng giữa các region, lưu lượng này vẫn có thể kém tin cậy hơn lưu lượng giữa các availability zone trong cùng một region, hay bên trong một availability zone. Cấu hình đơn leader rất nhạy với sự cố của đường truyền giữa các region: client ở một region muốn ghi vào leader ở region khác thì phải gửi yêu cầu qua đường truyền này và đợi phản hồi mới hoàn tất thao tác.

    Cấu hình đa leader dùng sao chép bất đồng bộ (*asynchronous replication*) chịu được sự cố mạng tốt hơn: khi mạng tạm thời gián đoạn, leader của mỗi region vẫn có thể xử lý ghi độc lập.

Tính nhất quán
:   Hệ thống đơn leader có thể cung cấp đảm bảo nhất quán mạnh, chẳng hạn giao dịch serializable, mà ta sẽ bàn ở [Chương 8](/ch8#ch_transactions). Nhược điểm lớn nhất của hệ thống đa leader là mức nhất quán nó cung cấp yếu hơn nhiều. Chẳng hạn, không thể đảm bảo số dư tài khoản ngân hàng không âm, hay tên người dùng là duy nhất: hoàn toàn có thể các leader khác nhau xử lý riêng lẻ những lần ghi hợp lệ khi xét một mình (chi một khoản từ tài khoản, đăng ký một tên người dùng), nhưng khi gộp chúng với các lần ghi trên leader khác thì vi phạm ràng buộc.

    Đây là một giới hạn căn bản của hệ thống phân tán [^28]. Nếu bắt buộc phải thực thi những ràng buộc như vậy, tốt nhất nên dùng hệ thống đơn leader. Tuy nhiên, như ta sẽ thấy ở ["Xử lý xung đột ghi"](/ch6#sec_replication_write_conflicts), với nhiều ứng dụng không cần các ràng buộc đó, hệ thống đa leader vẫn cung cấp được những thuộc tính nhất quán hữu ích.

Sao chép đa leader (*multi-leader replication*) ít phổ biến hơn đơn leader, nhưng nhiều cơ sở dữ liệu vẫn hỗ trợ, như MySQL, Oracle, SQL Server và YugabyteDB. Đôi khi nó đến dưới dạng thành phần bổ sung bên ngoài, như Redis Enterprise, EDB Postgres Distributed và pglogical [^29].

Ở nhiều cơ sở dữ liệu, sao chép đa leader (*multi-leader replication*) là tính năng được gắn thêm về sau, nên thường có những cạm bẫy cấu hình tinh vi và tương tác bất ngờ với các tính năng khác của cơ sở dữ liệu. Chẳng hạn, khóa tự tăng, trigger và ràng buộc toàn vẹn đều có thể gây vấn đề. Vì thế, sao chép đa leader (*multi-leader replication*) thường bị coi là vùng nguy hiểm nên tránh nếu có thể [^30].

#### Topology sao chép (*replication topology*) đa leader {#sec_replication_topologies}

*Topology sao chép* (*replication topology*) mô tả các đường truyền thông mà các lần ghi đi qua khi lan truyền từ node này sang node khác. Nếu chỉ có hai leader, như ở {{< xref fig="6-9" page="/ch6" anchor="fig_replication_write_conflict" >}}Hình 6-9{{< /xref >}}, thì chỉ có một topology hợp lý: leader 1 phải gửi mọi lần ghi cho leader 2, và ngược lại. Khi có hơn hai leader, có nhiều topology để chọn. {{< xref fig="6-7" page="/ch6" anchor="fig_replication_topologies" >}}Hình 6-7{{< /xref >}} đưa ra ba ví dụ.

{{< fig num="6-7" id="fig_replication_topologies" src="/fig/ddia_0607.png" caption="Ba ví dụ topology có thể dùng cho sao chép đa leader (*multi-leader replication*)." class="ddia-figure ddia-figure--panorama" width="2880" height="800" />}}

Topology tổng quát nhất là *all-to-all* (*all-to-all*) ở {{< xref fig="6-7" page="/ch6" anchor="fig_replication_topologies" >}}Hình 6-7{{< /xref >}}(c), trong đó mỗi leader gửi lần ghi của mình cho mọi leader khác. Tuy nhiên, hệ thống thực tế cũng dùng các topology hạn chế hơn. Chẳng hạn trong *topology vòng* (*circular topology*), mỗi node nhận lần ghi từ một node, rồi chuyển tiếp các lần ghi đó cùng với lần ghi của chính mình sang một node khác. Một topology phổ biến khác là *topology hình sao* (*star topology*): một node gốc được chỉ định để chuyển tiếp lần ghi đến mọi node còn lại. Topology hình sao còn có thể tổng quát hóa thành hình cây.


> [!NOTE]
> Đừng nhầm topology mạng hình sao với *star schema*, thứ mô tả cấu trúc của mô hình dữ liệu, xem ["Star schema và snowflake: lược đồ cho phân tích"](/ch3#sec_datamodels_analytics).


Trong topology vòng và hình sao, một lần ghi có thể phải đi qua nhiều node mới tới được mọi replica. Do đó các node phải chuyển tiếp những thay đổi dữ liệu nhận từ node khác. Để tránh vòng lặp sao chép (*replication*) vô hạn, mỗi node có một định danh duy nhất, mỗi lần ghi trong replication log được gắn định danh của mọi node mà nó đã đi qua [^31]. Khi một node nhận được thay đổi dữ liệu có gắn định danh của chính nó, nó bỏ qua thay đổi đó, vì điều này cho thấy nó đã xử lý rồi.

#### Vấn đề của các topology khác nhau {#problems-with-different-topologies}

Topology vòng và hình sao có một vấn đề: chỉ cần một node hỏng là có thể làm gián đoạn dòng thông điệp sao chép (*replication*) giữa các node khác. Trước khi node đó được sửa, các node khác không thể liên lạc với nhau. Dù có thể cấu hình lại topology để đi vòng qua node hỏng, hầu hết triển khai đều phải làm việc này thủ công. Các topology kết nối dày hơn (như all-to-all) chịu lỗi tốt hơn, vì thông điệp có thể truyền theo nhiều đường khác nhau, tránh được điểm hỏng đơn lẻ.

Mặt khác, topology all-to-all cũng có vấn đề. Cụ thể, tốc độ các đường truyền mạng có thể khác nhau (chẳng hạn do tắc nghẽn mạng), khiến một số thông điệp sao chép (*replication*) "vượt mặt" các thông điệp khác, như ở {{< xref fig="6-8" page="/ch6" anchor="fig_replication_causality" >}}Hình 6-8{{< /xref >}}.

{{< fig num="6-8" id="fig_replication_causality" src="/fig/ddia_0608.png" caption="Với sao chép đa leader (*multi-leader replication*), các lần ghi có thể đến một số replica sai thứ tự." class="ddia-figure ddia-figure--wide" width="2880" height="1602" />}}

Ở {{< xref fig="6-8" page="/ch6" anchor="fig_replication_causality" >}}Hình 6-8{{< /xref >}}, client A chèn một dòng vào bảng trên leader 1, rồi client B cập nhật dòng đó trên leader 3. Nhưng leader 2 có thể nhận hai lần ghi này theo thứ tự ngược lại: nó nhận bản cập nhật trước (theo nó, đây là cập nhật một dòng chưa tồn tại trong cơ sở dữ liệu), rồi một lúc sau mới nhận lần chèn lẽ ra phải đến trước.

Đây cũng là vấn đề về quan hệ nhân quả, tương tự trường hợp đã thấy ở ["Consistent prefix reads"](/ch6#sec_replication_consistent_prefix). Bản cập nhật phụ thuộc vào lần chèn trước đó, nên phải đảm bảo mọi node xử lý lần chèn trước rồi mới xử lý bản cập nhật. Chỉ gắn timestamp cho mỗi lần ghi là chưa đủ, vì không thể tin đồng hồ của các node được đồng bộ đủ chính xác để leader 2 sắp xếp đúng các sự kiện này (xem [Chương 9](/ch9#ch_distributed)).

Để sắp xếp đúng các sự kiện này, có thể dùng *version vector* (*version vector*, xem ["Phát hiện ghi đồng thời"](/ch6#sec_replication_concurrent)) sẽ được giới thiệu ở phần sau của chương. Tuy nhiên, nhiều hệ thống sao chép đa leader (*multi-leader replication*) không dùng kỹ thuật sắp xếp bản cập nhật đáng tin cậy nên dễ gặp vấn đề ở {{< xref fig="6-8" page="/ch6" anchor="fig_replication_causality" >}}Hình 6-8{{< /xref >}}. Nếu dùng sao chép đa leader (*multi-leader replication*), bạn nên nắm các rủi ro này, đọc kỹ tài liệu và kiểm thử cơ sở dữ liệu kỹ lưỡng để chắc chắn nó thực sự cung cấp các đảm bảo mà bạn tưởng.

### Sync engine và phần mềm local-first {#sec_replication_offline_clients}

Ứng dụng cần tiếp tục hoạt động khi mất kết nối mạng là một tình huống khác phù hợp với sao chép đa leader (*multi-leader replication*).

Hãy lấy ví dụ ứng dụng lịch trên điện thoại, laptop và các thiết bị khác. Dù thiết bị có kết nối mạng hay không, bạn đều cần xem cuộc họp (gửi yêu cầu đọc) và thêm cuộc họp (gửi yêu cầu ghi) bất cứ lúc nào. Các thay đổi thực hiện khi ngoại tuyến phải được đồng bộ với server và các thiết bị khác vào lần tiếp theo thiết bị trực tuyến.

Trong trường hợp này, mỗi thiết bị có một bản sao cơ sở dữ liệu cục bộ đóng vai leader, có thể nhận ghi, còn các bản sao lịch trên các thiết bị đồng bộ với nhau qua một quy trình sao chép đa leader (*multi-leader replication*) bất đồng bộ. Replication lag có thể lên đến hàng giờ hoặc hàng ngày, tùy lúc thiết bị kết nối mạng lại.

Về kiến trúc, cấu hình này tương đương việc đẩy sao chép đa leader (*multi-leader replication*) giữa các region đến cực điểm: mỗi thiết bị là một "region", và kết nối mạng giữa chúng cực kỳ không đáng tin cậy.

#### Cộng tác thời gian thực, offline-first và ứng dụng local-first {#real-time-collaboration-offline-first-and-local-first-apps}

Ngoài ra, nhiều ứng dụng Web hiện đại còn cung cấp tính năng *cộng tác thời gian thực*, ví dụ Google Docs và Sheets cho tài liệu và bảng tính, Figma cho thiết kế đồ họa, Linear cho quản lý dự án. Các ứng dụng này phản hồi nhanh vì thao tác nhập của người dùng được phản ánh ngay lên giao diện, không cần chờ một vòng đi về qua mạng đến server. Các chỉnh sửa của một người cũng hiển thị cho người cộng tác với độ trễ rất thấp [^32] [^33] [^34].

Điều này cũng tạo thành kiến trúc đa leader: mỗi tab trình duyệt đang mở tệp chia sẻ là một replica, và các cập nhật lên tệp được sao chép bất đồng bộ (*asynchronous replication*) sang thiết bị của những người dùng khác đang mở tệp đó. Ngay cả khi ứng dụng không hỗ trợ chỉnh sửa ngoại tuyến, chỉ cần nhiều người dùng có thể cùng chỉnh sửa mà không đợi server phản hồi thì nó đã là hệ thống đa leader.

Chỉnh sửa ngoại tuyến và cộng tác thời gian thực cần hạ tầng sao chép (*replication*) tương tự: ứng dụng phải ghi lại mọi thay đổi người dùng thực hiện trên tệp, gửi ngay cho người cộng tác khi trực tuyến, còn khi ngoại tuyến thì lưu cục bộ rồi gửi sau. Đồng thời, ứng dụng còn phải nhận thay đổi của người cộng tác, hợp nhất vào bản sao cục bộ của người dùng và cập nhật giao diện để hiển thị phiên bản mới nhất. Khi nhiều người dùng sửa tệp đồng thời, có thể cần logic giải quyết xung đột để hợp nhất các thay đổi.

Thư viện phần mềm hỗ trợ quá trình này gọi là *sync engine* (*sync engine*). Ý tưởng này đã có từ lâu, nhưng thuật ngữ "sync engine" chỉ mới được chú ý gần đây [^35] [^36] [^37]. Ứng dụng cho phép người dùng tiếp tục sửa tệp khi ngoại tuyến gọi là ứng dụng *offline-first* (*offline-first*) [^38], có thể xây dựng bằng sync engine. *Phần mềm local-first* (*local-first software*) đòi hỏi nhiều hơn: ngoài hỗ trợ offline-first, nó còn phải đảm bảo ứng dụng cộng tác tiếp tục hoạt động ngay cả khi nhà phát triển phần mềm tắt hết dịch vụ trực tuyến [^39]. Một cách thực hiện là dùng giao thức đồng bộ theo chuẩn mở, để nhiều nhà cung cấp dịch vụ cùng hỗ trợ giao thức đó [^40]. Ví dụ, Git là hệ thống cộng tác local-first (dù không hỗ trợ cộng tác thời gian thực), vì có thể đồng bộ qua GitHub, GitLab hay bất kỳ dịch vụ lưu trữ kho mã nào khác.

#### Ưu và nhược điểm của sync engine {#pros-and-cons-of-sync-engines}

Cách phổ biến hiện nay để xây ứng dụng Web là để client chỉ giữ rất ít trạng thái bền vững, mỗi khi cần hiển thị dữ liệu mới hoặc cập nhật dữ liệu, nó gửi yêu cầu đến server. Với sync engine thì ngược lại: client nắm trạng thái bền vững, còn việc liên lạc với server chuyển xuống chạy nền. Cách này có nhiều ưu điểm:

* Dữ liệu nằm cục bộ nên giao diện người dùng có thể phản hồi nhanh hơn nhiều so với việc đợi lời gọi dịch vụ trả dữ liệu về. Một số ứng dụng đặt mục tiêu phản hồi thao tác nhập của người dùng ở *khung hình kế tiếp* của hệ thống đồ họa: với màn hình tần số làm tươi 60 Hz, nghĩa là phải render xong trong 16 mili giây.
* Cho phép người dùng làm việc ngoại tuyến rất có giá trị, nhất là trên thiết bị di động với kết nối chập chờn. Với sync engine, ứng dụng không cần chế độ ngoại tuyến riêng: ngoại tuyến chỉ là trường hợp độ trễ mạng rất lớn.
* So với việc gọi dịch vụ tường minh trong mã ứng dụng, sync engine đơn giản hóa mô hình lập trình của ứng dụng frontend. Như đã nói ở ["Vấn đề của lời gọi thủ tục từ xa (RPC)"](/ch5#sec_problems_with_rpc), mỗi lời gọi dịch vụ đều phải xử lý lỗi. Ví dụ, khi yêu cầu cập nhật dữ liệu trên server thất bại, giao diện phải phản ánh lỗi bằng cách nào đó. Sync engine để ứng dụng đọc ghi trực tiếp lên dữ liệu cục bộ, vốn hầu như không thất bại, tạo ra phong cách lập trình mang tính khai báo hơn [^41].
* Để hiển thị thời gian thực các chỉnh sửa của người dùng khác, cần nhận thông báo thay đổi và cập nhật giao diện một cách hiệu quả. Kết hợp sync engine với mô hình *reactive programming* (*reactive programming*) là cách tốt để làm điều này [^42].

Sync engine hiệu quả nhất khi có thể tải trước toàn bộ dữ liệu người dùng có thể cần và lưu bền vững ở client. Khi đó có thể truy cập ngoại tuyến lúc cần, nhưng cũng có nghĩa là nếu lượng dữ liệu người dùng được phép truy cập rất lớn thì sync engine không phù hợp. Ví dụ, tải toàn bộ tệp do chính người dùng tạo ra thường không vấn đề (một người dùng thường không tạo ra nhiều dữ liệu đến thế), còn tải toàn bộ danh mục sản phẩm của một trang thương mại điện tử thì phần lớn là không hợp lý.

Lotus Notes tiên phong áp dụng ý tưởng sync engine vào thập niên 1980 [^43], dù khi đó chưa dùng tên này. Chức năng đồng bộ cho các ứng dụng cụ thể như lịch cũng đã tồn tại nhiều năm. Ngày nay có khá nhiều sync engine đa dụng, trong đó một số dựa vào dịch vụ backend độc quyền như Google Firestore, Realm hay Ditto, số khác có backend mã nguồn mở, phù hợp để xây phần mềm local-first, như PouchDB/CouchDB, Automerge hay Yjs.

Trò chơi điện tử nhiều người chơi cũng cần phản hồi ngay các thao tác cục bộ của người chơi, rồi điều hòa với thao tác của người chơi khác nhận được bất đồng bộ qua mạng. Trong thuật ngữ phát triển game, phần tương ứng với sync engine gọi là *netcode*. Kỹ thuật dùng trong netcode được tùy biến rất sâu cho nhu cầu của game [^44], không thể chuyển sang phần mềm khác một cách trực tiếp, nên cuốn sách này không đi sâu.


### Xử lý xung đột ghi {#sec_replication_write_conflicts}

Khó khăn lớn nhất của sao chép đa leader (*multi-leader replication*) — dù là cơ sở dữ liệu server phân tán theo địa lý hay sync engine local-first trên thiết bị người dùng cuối — là các lần ghi đồng thời trên những leader khác nhau có thể xung đột với nhau và cần được giải quyết.

Chẳng hạn, {{< xref fig="6-9" page="/ch6" anchor="fig_replication_write_conflict" >}}Hình 6-9{{< /xref >}} cho thấy hai người dùng cùng lúc chỉnh sửa một trang wiki. Người dùng 1 đổi tiêu đề trang từ A thành B, còn người dùng 2 độc lập đổi tiêu đề từ A thành C. Thay đổi của cả hai đều được áp dụng thành công lên leader cục bộ của mỗi bên, nhưng khi sao chép bất đồng bộ (*asynchronous replication*) các thay đổi này, hệ thống phát hiện xung đột. Cơ sở dữ liệu đơn leader không gặp vấn đề này.

{{< fig num="6-9" id="fig_replication_write_conflict" src="/fig/ddia_0609.png" caption="Hai leader cập nhật đồng thời cùng một bản ghi, gây ra xung đột ghi." class="ddia-figure ddia-figure--wide" width="2880" height="1488" />}}

> [!NOTE]
> Ta gọi hai lần ghi ở {{< xref fig="6-9" page="/ch6" anchor="fig_replication_write_conflict" >}}Hình 6-9{{< /xref >}} là *ghi đồng thời* (*concurrent writes*), vì khi chúng được thực hiện lần đầu, bên nào cũng không biết về bên kia. Việc chúng có thực sự xảy ra cùng lúc theo thời gian vật lý hay không không quan trọng. Nếu các lần ghi xảy ra khi ngoại tuyến, chúng thậm chí có thể cách nhau rất lâu. Điều thực sự quan trọng là khi một lần ghi xảy ra, lần ghi kia đã có hiệu lực hay chưa.

Ta sẽ bàn cách cơ sở dữ liệu xác định hai lần ghi có đồng thời hay không ở ["Phát hiện ghi đồng thời"](/ch6#sec_replication_concurrent). Tạm thời cứ giả định xung đột đã phát hiện được, và xét cách giải quyết nào là tốt nhất.

#### Tránh xung đột {#conflict-avoidance}

Một chiến lược là tránh xung đột ngay từ đầu. Chẳng hạn, nếu ứng dụng đảm bảo mọi lần ghi vào một bản ghi nhất định đều đi qua cùng một leader, thì dù toàn bộ cơ sở dữ liệu dùng sao chép đa leader (*multi-leader replication*) cũng không phát sinh xung đột. Client của sync engine khi cập nhật ngoại tuyến không dùng được cách này, nhưng với hệ thống phía server sao chép (*replication*) xuyên region thì đôi khi khả thi [^30].

Ví dụ, trong ứng dụng mà người dùng chỉ được sửa dữ liệu của chính mình, có thể đảm bảo yêu cầu của một người dùng luôn được định tuyến đến cùng một region, và đọc ghi bằng leader của region đó. Những người dùng khác nhau có thể có region "chính" khác nhau (có thể chọn theo khoảng cách địa lý đến người dùng), nhưng từ góc nhìn của mỗi người dùng thì về bản chất vẫn là cấu hình đơn leader.

Tuy nhiên, đôi khi cần đổi leader được chỉ định cho một bản ghi: có thể một region không khả dụng và phải chuyển lưu lượng sang region khác, hoặc người dùng chuyển chỗ ở và giờ gần region khác hơn. Nếu người dùng đúng lúc ghi dữ liệu trong khi leader được chỉ định đang chuyển đổi, xung đột có thể xảy ra và phải giải quyết bằng một trong các cách bên dưới. Do đó, nếu cho phép đổi leader thì việc tránh xung đột có thể không còn hiệu lực.

Một ví dụ khác về tránh xung đột. Giả sử cần chèn bản ghi mới và dùng bộ đếm tự tăng để sinh ID duy nhất. Khi hệ thống có hai leader, có thể cho một leader chỉ sinh số lẻ và leader kia chỉ sinh số chẵn. Như vậy hai leader sẽ không đồng thời gán cùng một ID cho các bản ghi khác nhau. ["Bộ sinh ID và đồng hồ logic"](/ch10#sec_consistency_logical) sẽ bàn các cách cấp phát ID khác.


#### Last write wins (loại bỏ các lần ghi đồng thời) {#sec_replication_lww}

Nếu không thể tránh xung đột, cách giải quyết đơn giản nhất là gắn timestamp cho mỗi lần ghi và luôn lấy giá trị có timestamp lớn nhất. Ví dụ ở {{< xref fig="6-9" page="/ch6" anchor="fig_replication_write_conflict" >}}Hình 6-9{{< /xref >}}, giả sử lần ghi của người dùng 1 có timestamp lớn hơn người dùng 2. Cả hai leader sẽ xác định tiêu đề mới của trang là B và loại bỏ lần ghi đặt tiêu đề thành C. Nếu hai lần ghi tình cờ có cùng timestamp, có thể so sánh giá trị để chọn bên thắng (ví dụ với chuỗi, chọn giá trị đứng trước theo thứ tự bảng chữ cái).

Phương pháp này gọi là *last write wins* (LWW) (*last write wins*), vì lần ghi có timestamp lớn nhất được coi là lần ghi "cuối cùng". Tuy nhiên, tên gọi này dễ gây hiểu lầm: khi hai lần ghi đồng thời như ở {{< xref fig="6-9" page="/ch6" anchor="fig_replication_write_conflict" >}}Hình 6-9{{< /xref >}}, hoàn toàn không thể xác định lần nào trước, lần nào sau, nên thứ tự timestamp của các lần ghi đồng thời thực chất là ngẫu nhiên.

Vì vậy, ý nghĩa thực sự của LWW là: khi cùng một bản ghi được ghi đồng thời trên các leader khác nhau, chọn ngẫu nhiên một lần ghi làm bên thắng, còn các lần ghi khác bị âm thầm loại bỏ, dù chúng đã được xử lý thành công trên leader của mình. Cách này tuy giúp mọi replica cuối cùng hội tụ về cùng một trạng thái nhất quán, nhưng cái giá phải trả là mất dữ liệu.

Nếu tránh được xung đột — ví dụ chỉ chèn các bản ghi được định danh bằng khóa duy nhất như UUID và không bao giờ cập nhật — thì LWW không có vấn đề gì. Nhưng nếu cần cập nhật bản ghi hiện có, hoặc các leader khác nhau có thể chèn bản ghi cùng khóa, thì phải xét xem mất bản cập nhật có chấp nhận được với ứng dụng hay không. Nếu không, nên dùng các cách giải quyết xung đột khác dưới đây.

LWW còn một vấn đề nữa: nếu timestamp của lần ghi lấy từ đồng hồ thực (ví dụ Unix timestamp), hệ thống sẽ cực kỳ nhạy với việc đồng bộ đồng hồ. Giả sử đồng hồ của một node chạy nhanh hơn các node khác. Khi sau đó cố ghi đè giá trị mà node này đã ghi, timestamp của lần ghi mới có thể lại nhỏ hơn và bị bỏ qua, dù rõ ràng nó xảy ra muộn hơn. Dùng *đồng hồ logic* (*logical clock*) có thể giải quyết vấn đề này, xem ["Bộ sinh ID và đồng hồ logic"](/ch10#sec_consistency_logical).

#### Giải quyết xung đột thủ công {#manual-conflict-resolution}

Nếu không muốn loại bỏ ngẫu nhiên một số lần ghi, lựa chọn tiếp theo là giải quyết xung đột thủ công. Bạn có thể đã quen với cách làm ở các hệ thống quản lý phiên bản như Git: khi commit trên hai nhánh cùng sửa một dòng của một tệp, việc hợp nhất nhánh sẽ sinh ra xung đột merge, và phải giải quyết xung đột thì mới hoàn tất merge.

Trong cơ sở dữ liệu, để một xung đột chặn toàn bộ quá trình sao chép (*replication*) cho đến khi có người giải quyết rõ ràng là không thực tế. Cơ sở dữ liệu thường lưu mọi giá trị được ghi đồng thời của một bản ghi — ví dụ B và C ở {{< xref fig="6-9" page="/ch6" anchor="fig_replication_write_conflict" >}}Hình 6-9{{< /xref >}}. Các giá trị này đôi khi gọi là *siblings* (*siblings*). Lần truy vấn bản ghi kế tiếp, cơ sở dữ liệu trả về *tất cả* giá trị chứ không chỉ giá trị mới nhất. Sau đó có thể tùy ý chọn cách giải quyết: xử lý tự động trong mã ứng dụng (ví dụ ghép B và C thành "B/C"), hoặc hỏi người dùng, cuối cùng ghi lại một giá trị mới vào cơ sở dữ liệu để giải quyết xung đột.

CouchDB và một số hệ thống khác dùng cách giải quyết xung đột này, nhưng nó có khá nhiều vấn đề:

* API của cơ sở dữ liệu thay đổi. Ví dụ, tiêu đề trang wiki vốn chỉ là một chuỗi, nay lại thành một tập chuỗi. Thường chỉ có một phần tử, nhưng khi có xung đột thì có thể có nhiều. Mã ứng dụng xử lý loại dữ liệu này khá vướng víu.
* Bắt người dùng tự hợp nhất các sibling là gánh nặng lớn cho cả nhà phát triển lẫn người dùng: nhà phát triển phải làm giao diện giải quyết xung đột, còn người dùng có thể không hiểu vì sao mình phải làm việc này và phải làm gì. Trong nhiều trường hợp, hợp nhất tự động tốt hơn là làm phiền người dùng.
* Hợp nhất sibling tự động nếu không cẩn thận cũng có thể cho kết quả bất ngờ. Ví dụ, giỏ hàng của Amazon trước đây cho phép cập nhật đồng thời, rồi giữ lại mọi mặt hàng xuất hiện trong bất kỳ sibling nào, tức lấy hợp của các giỏ hàng. Nếu khách xóa một mặt hàng ở một sibling trong khi sibling kia vẫn còn mặt hàng đó, mặt hàng đã xóa sẽ bất ngờ xuất hiện lại [^45]. {{< xref fig="6-10" page="/ch6" anchor="fig_replication_amazon_anomaly" >}}Hình 6-10{{< /xref >}} minh họa tình huống này: thiết bị 1 xóa Book, thiết bị 2 đồng thời xóa DVD, sau khi giải quyết xung đột thì cả hai mặt hàng đều quay lại.
* Nếu nhiều node cùng lúc quan sát thấy xung đột và mỗi node tự giải quyết, thì bản thân quá trình giải quyết có thể sinh thêm xung đột, và kết quả các node đưa ra có thể không nhất quán. Ví dụ, nếu không có thứ tự sắp xếp cố định, một node có thể hợp nhất B và C thành "B/C", node khác lại thành "C/B". Khi hợp nhất tiếp "B/C" với "C/B", kết quả có thể thành thứ kỳ quặc như "B/C/C/B".

{{< fig num="6-10" id="fig_replication_amazon_anomaly" src="/fig/ddia_0610.png" caption="Ví dụ dị thường giỏ hàng Amazon: khi hợp nhất xung đột giỏ hàng bằng phép hợp, mặt hàng đã xóa có thể xuất hiện lại." class="ddia-figure ddia-figure--wide" width="2658" height="1065" />}}


<a id="sec_replication_automatic_resolution"></a>

#### Giải quyết xung đột tự động {#automatic-conflict-resolution}

Với nhiều ứng dụng, cách xử lý xung đột tốt nhất là dùng thuật toán tự động hợp nhất các lần ghi đồng thời thành một trạng thái nhất quán. Giải quyết xung đột tự động có thể đảm bảo mọi replica *hội tụ* (*converge*) về cùng một trạng thái: miễn là đã xử lý cùng một tập lần ghi, trạng thái các replica sẽ giống nhau, bất kể các lần ghi đến theo thứ tự nào.

LWW là một ví dụ đơn giản về thuật toán giải quyết xung đột. Với các kiểu dữ liệu khác nhau, người ta đã phát triển những thuật toán hợp nhất phức tạp hơn, nhằm giữ lại tối đa tác dụng dự kiến của mọi cập nhật, nhờ đó tránh mất dữ liệu:

* Nếu dữ liệu là văn bản (ví dụ tiêu đề hay nội dung trang wiki), có thể phát hiện những ký tự nào được chèn hoặc xóa giữa các phiên bản liền kề. Kết quả hợp nhất giữ lại mọi lần chèn và xóa ở bất kỳ sibling nào. Nếu người dùng đồng thời chèn văn bản tại cùng một vị trí, có thể sắp xếp theo thứ tự xác định để mọi node cho cùng kết quả.
* Nếu dữ liệu là tập hợp phần tử, dù có thứ tự như danh sách việc cần làm hay không có thứ tự như giỏ hàng, đều có thể theo dõi chèn và xóa giống như hợp nhất văn bản. Để tránh dị thường giỏ hàng ở {{< xref fig="6-10" page="/ch6" anchor="fig_replication_amazon_anomaly" >}}Hình 6-10{{< /xref >}}, thuật toán ghi nhớ rằng Book và DVD đã bị xóa, nên kết quả hợp nhất là Cart = {Soap}.
* Nếu dữ liệu là bộ đếm số nguyên có thể tăng hoặc giảm (ví dụ số lượt thích của một bài đăng mạng xã hội), thuật toán hợp nhất có thể tính xem trên từng sibling đã xảy ra bao nhiêu lần tăng và giảm, rồi cộng lại cho đúng, không đếm trùng cũng không mất cập nhật.
* Nếu dữ liệu là map khóa–giá trị, có thể áp dụng một thuật toán giải quyết xung đột nào đó cho các giá trị dưới cùng một khóa. Các cập nhật trên khóa khác nhau thì xử lý độc lập.

Giải quyết xung đột không phải vạn năng. Ví dụ, nếu quy định một danh sách tối đa năm phần tử mà nhiều người dùng đồng thời thêm phần tử khiến tổng vượt năm, thì lựa chọn duy nhất là bỏ bớt một số. Dù vậy, giải quyết xung đột tự động vẫn đủ để xây nhiều ứng dụng hữu ích. Một khi đã quyết định xây ứng dụng offline-first hay local-first có thể cộng tác, giải quyết xung đột là không thể tránh, và tự động hóa thường là lựa chọn tốt nhất.

### CRDT và operational transformation {#sec_replication_crdts}

Khi triển khai giải quyết xung đột tự động, thường dùng hai loại thuật toán: *conflict-free replicated data type* (CRDT) [^46] và *operational transformation* (OT) [^47]. Hai loại có triết lý thiết kế và đặc tính hiệu năng khác nhau, nhưng đều có thể tự động hợp nhất các loại dữ liệu đã nêu ở trên.

{{< xref fig="6-11" page="/ch6" anchor="fig_replication_ot_crdt" >}}Hình 6-11{{< /xref >}} cho thấy OT và CRDT lần lượt hợp nhất các cập nhật đồng thời của văn bản ra sao. Giả sử hai replica ban đầu cùng chứa văn bản "ice". Một replica chèn chữ "n" vào đầu, thành "nice". Cùng lúc đó, replica kia chèn dấu chấm than vào cuối, thành "ice!".

{{< fig num="6-11" id="fig_replication_ot_crdt" src="/fig/ddia_0611.png" caption="OT và CRDT lần lượt hợp nhất hai lần chèn đồng thời vào một chuỗi như thế nào." class="ddia-figure ddia-figure--wide" width="2658" height="981" />}}

Hai loại thuật toán dùng hai cách khác nhau để cho ra kết quả hợp nhất "nice!":

OT
:   Ghi lại chỉ số vị trí chèn hoặc xóa ký tự: "n" được chèn ở chỉ số 0, "!" được chèn ở chỉ số 3. Sau đó hai replica trao đổi các thao tác. Việc chèn "n" ở chỉ số 0 có thể áp dụng nguyên trạng, nhưng nếu chèn "!" trực tiếp ở chỉ số 3 của trạng thái "nice" thì kết quả sẽ sai thành "nic!e". Do đó phải biến đổi chỉ số của mỗi thao tác theo các thao tác đồng thời đã được áp dụng. Ở đây, để tính đến "n" đã chèn ở chỉ số nhỏ hơn, cần biến đổi vị trí chèn "!" thành chỉ số 4.

CRDT
:   Hầu hết CRDT không dùng chỉ số mà gán cho mỗi ký tự một ID duy nhất và bất biến, rồi dựa vào đó xác định vị trí chèn và xóa. Ví dụ ở {{< xref fig="6-11" page="/ch6" anchor="fig_replication_ot_crdt" >}}Hình 6-11{{< /xref >}}, ID của "i" là 1A, của "c" là 2A, v.v. Khi chèn dấu chấm than, thao tác sinh ra chứa cả ID của ký tự mới (4B) lẫn ID của ký tự hiện có đứng trước vị trí chèn (3A). Muốn chèn vào đầu chuỗi thì đặt ID ký tự đứng trước là "nil". Các lần chèn đồng thời tại cùng một vị trí được sắp xếp theo ID ký tự. Nhờ đó không cần biến đổi thao tác mà vẫn đảm bảo các replica hội tụ.

Nhiều thuật toán được xây dựng trên các biến thể khác nhau của những ý tưởng này. Danh sách và mảng có thể dùng cách tương tự, thay ký tự bằng phần tử danh sách. Các kiểu dữ liệu khác như map khóa–giá trị cũng dễ bổ sung. OT và CRDT có đánh đổi riêng về hiệu năng và tính năng, nhưng cũng có thể kết hợp ưu điểm của cả hai trong cùng một thuật toán [^48].

OT được dùng nhiều nhất cho soạn thảo cộng tác văn bản theo thời gian thực, ví dụ Google Docs [^32], còn CRDT được dùng trong các cơ sở dữ liệu phân tán như Redis Enterprise, Riak, Azure Cosmos DB [^49]. Sync engine cho dữ liệu JSON có thể cài đặt bằng CRDT (như Automerge, Yjs) hoặc OT (như ShareDB).

#### Xung đột là gì? {#what-is-a-conflict}

Một số xung đột rất rõ ràng. Trong ví dụ ở {{< xref fig="6-9" page="/ch6" anchor="fig_replication_write_conflict" >}}Hình 6-9{{< /xref >}}, hai lần ghi đồng thời sửa cùng một trường của cùng một bản ghi, đặt nó thành hai giá trị khác nhau. Không nghi ngờ gì, đó là xung đột.

Những xung đột khác tinh vi hơn, khó nhận ra. Lấy hệ thống đặt phòng họp làm ví dụ: nó ghi lại phòng nào được nhóm nào đặt vào thời gian nào. Ứng dụng phải đảm bảo tại mỗi thời điểm một phòng chỉ được giao cho một nhóm, nghĩa là các lượt đặt cùng phòng không được chồng lấn. Nếu hai lượt đặt khác nhau cùng chiếm một phòng vào cùng thời điểm thì sinh ra xung đột. Ngay cả khi ứng dụng kiểm tra phòng trống trước khi cho đặt, xung đột vẫn có thể xảy ra nếu hai lượt đặt được thực hiện trên hai leader khác nhau.

Vấn đề này không có câu trả lời ngắn gọn có sẵn, nhưng ở các phần tiếp theo ta sẽ dần hiểu sâu hơn. [Chương 8](/ch8#ch_transactions) sẽ đưa thêm ví dụ về xung đột, còn ["Sắp xếp sự kiện để nắm bắt quan hệ nhân quả"](/ch13#sec_future_capture_causality) sẽ bàn các cách phát hiện và giải quyết xung đột trong hệ thống sao chép (*replication*) một cách có khả năng mở rộng.

## Sao chép không leader (*leaderless replication*) {#sec_replication_leaderless}

Sao chép đơn leader (*single-leader replication*) và đa leader đã bàn ở trên đều dựa trên cùng một ý tưởng: client gửi yêu cầu ghi đến một node (leader), rồi hệ thống cơ sở dữ liệu lo việc sao chép (*replication*) lần ghi đó sang các replica khác. Leader quyết định thứ tự xử lý các lần ghi, còn follower áp dụng các lần ghi của leader theo cùng thứ tự đó.

Một số hệ thống lưu trữ dữ liệu chọn cách khác: bỏ khái niệm leader, cho phép bất kỳ replica nào nhận ghi trực tiếp từ client. Một số hệ thống dữ liệu sao chép (*replication*) đầu tiên dùng chính mô hình không leader [^1] [^50], nhưng trong thời kỳ cơ sở dữ liệu quan hệ thống trị, ý tưởng này gần như bị lãng quên. Năm 2007, Amazon dùng nó cho hệ thống *Dynamo* nội bộ [^45], và kiến trúc không leader nhờ đó lại phổ biến. Riak, Cassandra và ScyllaDB đều là các kho dữ liệu mã nguồn mở lấy cảm hứng từ Dynamo và dùng mô hình sao chép không leader (*leaderless replication*), nên loại cơ sở dữ liệu này còn gọi là cơ sở dữ liệu *kiểu Dynamo* (*Dynamo-style*).


> [!NOTE]
> Hệ thống *Dynamo* gốc chỉ được mô tả trong một bài báo [^45] và chưa từng được phát hành bên ngoài Amazon. *DynamoDB*, tên gần giống, là cơ sở dữ liệu đám mây do AWS ra mắt sau này, nhưng kiến trúc hoàn toàn khác: nó dùng sao chép đơn leader (*single-leader replication*) dựa trên thuật toán đồng thuận Multi-Paxos [^5].


Ở một số cách triển khai không leader, client gửi lần ghi trực tiếp đến nhiều replica, ở số khác, một node *coordinator* (*coordinator*) làm việc đó thay client. Tuy nhiên, khác với cơ sở dữ liệu có leader, coordinator không áp đặt thứ tự ghi. Ta sẽ thấy sự khác biệt thiết kế này ảnh hưởng sâu sắc đến cách dùng cơ sở dữ liệu.

### Ghi dữ liệu khi node gặp sự cố {#id287}

Giả sử một cơ sở dữ liệu có ba replica, trong đó một replica tạm thời không khả dụng, có thể đang khởi động lại để cài bản cập nhật hệ thống. Trong cấu hình đơn leader, để tiếp tục xử lý ghi có thể phải thực hiện failover (xem ["Xử lý sự cố node"](/ch6#sec_replication_failover)).

Cấu hình không leader hoàn toàn không có failover. {{< xref fig="6-12" page="/ch6" anchor="fig_replication_quorum_node_outage" >}}Hình 6-12{{< /xref >}} cho thấy tình huống: client (người dùng 1234) gửi lần ghi song song đến cả ba replica. Hai replica khả dụng nhận lần ghi, còn replica không khả dụng bỏ lỡ nó. Giả sử chỉ cần hai trong ba replica xác nhận là đủ coi lần ghi thành công: sau khi người dùng 1234 nhận hai phản hồi *ok*, hệ thống coi lần ghi là thành công, và client bỏ qua việc một replica đã bỏ lỡ lần ghi.

{{< fig num="6-12" id="fig_replication_quorum_node_outage" src="/fig/ddia_0612.png" caption="Ghi quorum, đọc quorum và read repair sau khi một node ngừng hoạt động." class="ddia-figure ddia-figure--wide" width="2880" height="1572" />}}


Giờ hãy hình dung node không khả dụng phục hồi và client bắt đầu đọc từ nó. Mọi lần ghi xảy ra trong lúc nó ngừng hoạt động đều không được lưu ở đây, nên khi đọc từ nó, phản hồi có thể chứa giá trị *cũ* (lỗi thời).

Để giải quyết vấn đề này, khi đọc cơ sở dữ liệu, client không thể chỉ gửi yêu cầu đến một replica: *yêu cầu đọc cũng được gửi song song đến nhiều node*. Các node khác nhau có thể trả phản hồi khác nhau, chẳng hạn một node trả giá trị mới nhất, node khác trả giá trị cũ.

Để phân biệt phản hồi nào mới nhất, phản hồi nào đã lỗi thời, mỗi giá trị được ghi đều phải mang số phiên bản hoặc timestamp, tương tự cách đã giới thiệu ở ["Last write wins (loại bỏ các lần ghi đồng thời)"](/ch6#sec_replication_lww). Khi client nhận nhiều kết quả đọc, nó lấy giá trị có timestamp lớn nhất — dù chỉ một replica trả giá trị đó còn mấy replica kia đều trả giá trị cũ. Chi tiết hơn xem ["Phát hiện ghi đồng thời"](/ch6#sec_replication_concurrent).

#### Bắt kịp các lần ghi bị bỏ lỡ {#sec_replication_read_repair}

Hệ thống sao chép (*replication*) phải đảm bảo mọi dữ liệu cuối cùng đều được sao chép (*replication*) đến từng replica. Khi node không khả dụng phục hồi, làm sao nó bổ sung các lần ghi đã bỏ lỡ lúc ngừng hoạt động? Kho dữ liệu kiểu Dynamo dùng một số cơ chế sau:

*Read repair* (*read repair*)
:   Khi client đọc song song nhiều node, nó có thể phát hiện phản hồi cũ. Ví dụ ở {{< xref fig="6-12" page="/ch6" anchor="fig_replication_quorum_node_outage" >}}Hình 6-12{{< /xref >}}, người dùng 2345 nhận giá trị phiên bản 6 từ replica 3, và giá trị phiên bản 7 từ replica 1 và 2. Client thấy giá trị của replica 3 đã lỗi thời nên ghi giá trị mới hơn ngược lại replica này. Cách này hiệu quả với những giá trị được đọc thường xuyên.

*Hinted handoff* (*hinted handoff*)
:   Khi một replica không khả dụng, replica khác có thể lưu các lần ghi thay nó, và ghi chúng lại dưới dạng *gợi ý* (*hint*). Khi replica lẽ ra phải nhận các lần ghi này phục hồi, replica đang giữ gợi ý sẽ gửi chúng đi rồi xóa gợi ý. Quá trình *chuyển giao* này giúp replica bắt kịp, ngay cả với những giá trị chưa từng được đọc nên không thể cập nhật bằng read repair.

*Anti-entropy* (*anti-entropy*)
:   Ngoài ra, có một tiến trình nền định kỳ tìm điểm khác biệt dữ liệu giữa các replica và sao chép (*replication*) dữ liệu thiếu từ replica này sang replica khác. Khác với replication log của sao chép dựa trên leader (*leader-based replication*), *quá trình anti-entropy* (*anti-entropy process*) này không sao chép (*replication*) các lần ghi theo thứ tự cụ thể, và có thể mất khá lâu dữ liệu mới được sao chép (*replication*).

#### Đọc và ghi quorum {#sec_replication_quorum_condition}

Trong ví dụ ở {{< xref fig="6-12" page="/ch6" anchor="fig_replication_quorum_node_outage" >}}Hình 6-12{{< /xref >}}, lần ghi chỉ hoàn tất trên hai trong ba replica mà ta vẫn coi là thành công. Nếu chỉ một replica nhận ghi thì sao? Ngưỡng dưới này có thể hạ thấp đến đâu?

Nếu đảm bảo mỗi lần ghi thành công được lưu ít nhất trên hai trong ba replica, thì tối đa chỉ một replica là cũ. Do đó, chỉ cần đọc ít nhất hai replica là có thể chắc chắn ít nhất một replica là mới nhất. Ngay cả khi replica thứ ba ngừng hoạt động hoặc phản hồi chậm, lần đọc vẫn trả về giá trị mới nhất.

Tổng quát hơn, giả sử có *n* replica, mỗi lần ghi phải được *w* node xác nhận mới thành công, và mỗi lần đọc truy vấn ít nhất *r* node. (Trong ví dụ trên, *n* = 3, *w* = 2, *r* = 2.) Miễn là *w* + *r* > *n*, ta kỳ vọng khi đọc sẽ nhận được giá trị mới nhất, vì trong *r* node được truy vấn, ít nhất một node chắc chắn là mới nhất. Các thao tác tuân theo giá trị *r*, *w* này gọi là *đọc quorum* (*quorum read*) và *ghi quorum* (*quorum write*) [^50]. Có thể coi *r* và *w* là số phiếu tối thiểu cần có để một lần đọc hoặc ghi hợp lệ.

Trong cơ sở dữ liệu kiểu Dynamo, các tham số *n*, *w*, *r* thường đều cấu hình được. Cách làm phổ biến là chọn *n* lẻ (thường là 3 hoặc 5) và đặt *w* = *r* = (*n* + 1) / 2 (làm tròn lên), nhưng có thể điều chỉnh tùy nhu cầu. Chẳng hạn, khối lượng công việc ít ghi nhiều đọc có thể hợp với *w* = *n*, *r* = 1. Khi đó đọc nhanh hơn, nhưng nhược điểm là chỉ cần một node hỏng là mọi lần ghi cơ sở dữ liệu đều thất bại.


> [!NOTE]
> Số node trong cụm có thể nhiều hơn *n*, nhưng một giá trị bất kỳ chỉ được lưu trên *n* node. Nhờ vậy có thể sharding tập dữ liệu, hỗ trợ tập dữ liệu mà một node không chứa nổi. Ta sẽ tiếp tục bàn về sharding ở [Chương 7](/ch7#ch_sharding).


Điều kiện quorum *w* + *r* > *n* cho phép hệ thống chịu được node không khả dụng theo cách sau:

* Nếu *w* < *n*, vẫn xử lý được ghi khi một node không khả dụng.
* Nếu *r* < *n*, vẫn xử lý được đọc khi một node không khả dụng.
* Với *n* = 3, *w* = 2, *r* = 2, có thể chịu được một node không khả dụng, như ở {{< xref fig="6-12" page="/ch6" anchor="fig_replication_quorum_node_outage" >}}Hình 6-12{{< /xref >}}.
* Với *n* = 5, *w* = 3, *r* = 3, có thể chịu được hai node không khả dụng, như ở {{< xref fig="6-13" page="/ch6" anchor="fig_replication_quorum_overlap" >}}Hình 6-13{{< /xref >}}.

Thông thường, yêu cầu đọc và ghi luôn được gửi song song đến cả *n* replica. Các tham số *w* và *r* quyết định phải đợi bao nhiêu node, tức là trong *n* node phải có bao nhiêu node báo thành công thì mới coi lần đọc hoặc ghi là thành công.

{{< fig num="6-13" id="fig_replication_quorum_overlap" src="/fig/ddia_0613.png" caption="Nếu w + r > n, trong r replica được đọc, ít nhất một replica chắc chắn đã thấy lần ghi thành công gần nhất." class="ddia-figure ddia-figure--wide" width="2880" height="1226" />}}


Nếu số node khả dụng ít hơn *w* hoặc *r* cần thiết, lần ghi hoặc đọc sẽ trả về lỗi. Node không khả dụng có thể do nhiều nguyên nhân: node ngừng hoạt động (sập hoặc mất điện), lỗi khi thực hiện thao tác (đĩa đầy, không ghi được), mạng giữa client và node gián đoạn, v.v. Ta chỉ quan tâm node có trả về phản hồi thành công hay không, không cần phân biệt loại sự cố cụ thể.

### Giới hạn của nhất quán quorum {#sec_replication_quorum_limitations}

Nếu có *n* replica và chọn *w*, *r* thỏa *w* + *r* > *n*, thường có thể kỳ vọng mỗi lần đọc trả về giá trị được ghi gần nhất của một khóa. Đó là vì tập node tham gia lần ghi và tập node tham gia lần đọc chắc chắn giao nhau. Nghĩa là trong các node được đọc, ít nhất một node giữ giá trị mới nhất, như ở {{< xref fig="6-13" page="/ch6" anchor="fig_replication_quorum_overlap" >}}Hình 6-13{{< /xref >}}.

*r* và *w* thường được chọn là đa số node (nhiều hơn *n*/2), vì như vậy vừa đảm bảo *w* + *r* > *n*, vừa chịu được tối đa *n*/2 (làm tròn xuống) node hỏng. Tuy nhiên, quorum không nhất thiết phải là đa số. Điều thực sự quan trọng là tập node dùng cho thao tác đọc và tập node dùng cho thao tác ghi có ít nhất một node chung. Quorum còn có thể bố trí theo cách khác, mang lại sự linh hoạt nhất định cho thiết kế thuật toán phân tán [^51].

Cũng có thể đặt *w* và *r* nhỏ hơn, sao cho *w* + *r* ≤ *n*, tức không thỏa điều kiện quorum. Khi đó yêu cầu đọc và ghi vẫn được gửi đến *n* node, nhưng thao tác cần ít phản hồi thành công hơn để được coi là thành công.

*w* và *r* càng nhỏ thì càng dễ đọc phải giá trị cũ, vì lần đọc nhiều khả năng không phủ tới node giữ giá trị mới nhất. Bù lại là độ trễ thấp hơn và tính sẵn sàng cao hơn: khi mạng gián đoạn khiến nhiều replica không với tới được, hệ thống có nhiều cơ hội hơn để tiếp tục xử lý đọc ghi. Chỉ khi số replica với tới được thấp hơn *w* hoặc *r* thì cơ sở dữ liệu mới lần lượt không ghi được hoặc không đọc được.

Tuy nhiên, ngay cả khi *w* + *r* > *n*, vẫn có một số trường hợp biên khiến các thuộc tính nhất quán khó hiểu, ví dụ:

* Nếu node giữ giá trị mới bị hỏng, rồi khôi phục dữ liệu từ replica giữ giá trị cũ, số replica giữ giá trị mới có thể tụt xuống dưới *w*, phá vỡ điều kiện quorum.
* Trong lúc cân bằng lại, một phần dữ liệu được chuyển từ node này sang node khác (xem [Chương 7](/ch7#ch_sharding)). Các node có thể bất đồng về việc *n* bản sao của một giá trị nên do những node nào giữ, khiến quorum đọc và quorum ghi không còn giao nhau.
* Nếu thao tác đọc diễn ra đồng thời với thao tác ghi, lần đọc có thể thấy hoặc không thấy giá trị của lần ghi đồng thời. Đặc biệt, một lần đọc có thể thấy giá trị mới, trong khi lần đọc sau lại thấy giá trị cũ, xem ["Linearizability và quorum"](/ch10#sec_consistency_quorum_linearizable).
* Nếu một lần ghi thành công trên một số replica nhưng thất bại trên các replica còn lại (ví dụ đĩa của một số node đã đầy), tổng số lần thành công ít hơn *w*, thì lần ghi tổng thể bị coi là thất bại, nhưng lần ghi trên các replica thành công không được rollback. Nghĩa là dù hệ thống báo ghi thất bại, các lần đọc sau đó vẫn có thể trả về giá trị của lần ghi này [^52].
* Nếu cơ sở dữ liệu dùng timestamp của đồng hồ thực để xác định lần ghi nào mới hơn (như Cassandra và ScyllaDB), một node khác có đồng hồ nhanh hơn, hễ đã ghi cùng một khóa thì các lần ghi sau đó có thể bị âm thầm loại bỏ. Ta đã gặp vấn đề này ở ["Last write wins (loại bỏ các lần ghi đồng thời)"](/ch6#sec_replication_lww), và sẽ bàn thêm ở ["Phụ thuộc vào đồng hồ đồng bộ"](/ch9#sec_distributed_clocks_relying).
* Nếu hai lần ghi xảy ra đồng thời, một replica có thể xử lý lần này trước, replica khác lại xử lý lần kia trước, sinh ra xung đột tương tự sao chép đa leader (*multi-leader replication*) (xem ["Xử lý xung đột ghi"](/ch6#sec_replication_write_conflicts)). Ta sẽ quay lại vấn đề này ở ["Phát hiện ghi đồng thời"](/ch6#sec_replication_concurrent).

Vì vậy, quorum thoạt trông như đảm bảo lần đọc trả về giá trị ghi gần nhất, nhưng thực tế không đơn giản như thế. Cơ sở dữ liệu kiểu Dynamo thường được tối ưu cho các trường hợp chấp nhận được nhất quán cuối cùng. Các tham số *w* và *r* cho phép điều chỉnh xác suất đọc phải giá trị cũ [^53], nhưng không nên coi chúng là đảm bảo tuyệt đối.

#### Giám sát độ cũ {#monitoring-staleness}

Từ góc độ vận hành, giám sát xem kết quả cơ sở dữ liệu trả về có mới hay không là rất quan trọng. Ngay cả khi ứng dụng chịu được việc đọc dữ liệu cũ, bạn vẫn cần biết việc sao chép (*replication*) có khỏe mạnh hay không. Nếu sao chép (*replication*) tụt hậu nghiêm trọng, hệ thống nên phát cảnh báo để điều tra các nguyên nhân như lỗi mạng hay node quá tải.

Với sao chép dựa trên leader (*leader-based replication*), cơ sở dữ liệu thường cung cấp các chỉ số replication lag để hệ thống giám sát thu thập. Đó là vì các lần ghi được áp dụng trên leader và follower theo cùng thứ tự, và mỗi node có vị trí riêng trong replication log, tức số lần ghi đã được áp dụng cục bộ. Lấy vị trí hiện tại của leader trừ đi vị trí hiện tại của follower là đo được replication lag.

Hệ thống sao chép không leader (*leaderless replication*) không có thứ tự áp dụng ghi cố định, nên việc giám sát khó hơn. Số lượng gợi ý mà replica lưu để chuyển giao có thể dùng làm chỉ số sức khỏe, nhưng rất khó diễn giải một cách có ý nghĩa [^54]. Nhất quán cuối cùng cố ý đưa ra một đảm bảo mơ hồ, nhưng để vận hành được, ta phải định lượng được "cuối cùng" là bao xa.


### Hiệu năng của sao chép đơn leader (*single-leader replication*) và không leader {#sec_replication_leaderless_perf}

Hệ thống sao chép (*replication*) dựa trên một leader có thể cung cấp đảm bảo nhất quán mạnh, còn hệ thống không leader rất khó, thậm chí không thể làm được. Tuy nhiên, như đã thấy ở ["Vấn đề của replication lag"](/ch6#sec_replication_lag), trong hệ thống sao chép dựa trên leader (*leader-based replication*), nếu đọc từ follower cập nhật bất đồng bộ thì vẫn có thể nhận giá trị cũ.

Đọc từ leader đảm bảo phản hồi mới nhất, nhưng có vấn đề về hiệu năng:

* Thông lượng đọc bị giới hạn bởi khả năng xử lý của leader, ngược lại, mở rộng đọc có thể dàn đọc ra các replica cập nhật bất đồng bộ, nhưng các replica này có thể trả giá trị cũ.
* Khi leader hỏng, phải đợi hệ thống phát hiện sự cố và hoàn tất failover mới tiếp tục xử lý yêu cầu. Dù failover nhanh, sự tăng vọt ngắn của thời gian phản hồi cũng bị người dùng nhận ra. Nếu mất nhiều thời gian, hệ thống sẽ không khả dụng trong khoảng đó.
* Hệ thống cực kỳ nhạy với vấn đề hiệu năng của leader. Nếu leader phản hồi chậm vì quá tải hoặc tranh chấp tài nguyên, thời gian phản hồi của người dùng cũng tăng ngay.

Ưu điểm lớn của kiến trúc không leader là bền bỉ hơn trước những vấn đề này. Hệ thống không cần failover, và vì yêu cầu vốn đã được gửi song song đến nhiều replica nên một replica chậm hoặc không khả dụng ảnh hưởng rất ít đến thời gian phản hồi: client chỉ cần dùng kết quả của các replica khác phản hồi nhanh hơn. Cách dùng phản hồi nhanh nhất này gọi là *request hedging* (*request hedging*), có thể giảm đáng kể độ trễ đuôi [^55].

Nguyên nhân gốc rễ khiến hệ thống không leader bền bỉ như vậy là nó không phân biệt trường hợp bình thường với trường hợp sự cố. Điều này đặc biệt có lợi khi xử lý cái gọi là *gray failure* (*gray failure*): node không hỏng hẳn mà ở trạng thái suy giảm, xử lý yêu cầu chậm bất thường [^56]. Node đơn thuần quá tải cũng vậy (ví dụ sau khi node ngoại tuyến một thời gian, phục hồi bằng hinted handoff có thể sinh rất nhiều tải phụ). Hệ thống dựa trên leader phải phán đoán tình hình có đủ nghiêm trọng để failover hay không, mà bản thân failover lại có thể gây thêm gián đoạn. Hệ thống không leader hoàn toàn không cần phán đoán này.

Tất nhiên, hệ thống không leader cũng có thể gặp vấn đề hiệu năng:

* Dù không cần failover, vẫn phải có một replica phát hiện replica khác không khả dụng thì mới lưu được gợi ý về các lần ghi bị bỏ lỡ. Khi replica không khả dụng phục hồi, quá trình chuyển giao còn phải gửi các gợi ý này cho nó. Khi hệ thống vốn đã chịu áp lực, điều này đặt thêm tải lên replica [^54].
* Replica càng nhiều, quorum càng lớn, số phản hồi phải đợi trước khi yêu cầu hoàn tất càng nhiều. Ngay cả khi chỉ đợi *r* hoặc *w* replica nhanh nhất, ngay cả khi mọi yêu cầu được gửi song song, *r* hay *w* lớn hơn vẫn làm tăng xác suất gặp replica chậm, từ đó tăng thời gian phản hồi tổng thể (xem ["Áp dụng các chỉ số thời gian phản hồi"](/ch2#sec_introduction_slo_sla)).
* Khi sự cố mạng diện rộng khiến client mất kết nối với nhiều replica, có thể hoàn toàn không lập được quorum. Một số cơ sở dữ liệu không leader cho phép bất kỳ replica nào với tới được nhận ghi, dù nó không thuộc tập replica thông thường của khóa đó (Riak và Dynamo gọi là *sloppy quorum* (*sloppy quorum*) [^45]. Cassandra và ScyllaDB gọi là *mức nhất quán ANY*). Các lần đọc sau đó không đảm bảo thấy lần ghi này, nhưng với một số ứng dụng, vẫn tốt hơn là ghi thất bại hẳn.

Sao chép đa leader (*multi-leader replication*) thậm chí có thể chịu được gián đoạn mạng tốt hơn sao chép không leader (*leaderless replication*), vì đọc ghi chỉ cần liên lạc với một leader, và leader đó có thể nằm cùng region với client. Tuy nhiên, các lần ghi trên một leader được lan truyền bất đồng bộ sang các leader khác nên kết quả đọc có thể cũ tùy ý. Đọc và ghi quorum đưa ra một sự dung hòa: vừa chịu lỗi tốt, vừa có xác suất cao đọc được dữ liệu mới nhất.

#### Vận hành đa region {#multi-region-operation}

Trước đây ta coi sao chép (*replication*) xuyên region là một trường hợp dùng của sao chép đa leader (*multi-leader replication*) (xem ["Sao chép đa leader (*multi-leader replication*)"](/ch6#sec_replication_multi_leader)). Sao chép không leader (*leaderless replication*) cũng phù hợp để vận hành đa region, vì nó vốn được thiết kế để chịu được các lần ghi đồng thời xung đột, gián đoạn mạng và độ trễ tăng vọt.

Cassandra và ScyllaDB hỗ trợ đa region trong mô hình không leader thông thường: client gửi ghi trực tiếp đến các replica ở mọi region, và có thể chọn nhiều mức nhất quán để quy định yêu cầu cần ít nhất bao nhiêu phản hồi mới coi là thành công. Chẳng hạn, có thể yêu cầu toàn bộ replica ở mọi region cùng tạo thành một quorum, hoặc yêu cầu mỗi region tự tạo quorum riêng, hoặc chỉ yêu cầu quorum ở region của client. Quorum cục bộ không phải đợi các yêu cầu chậm từ region khác, nhưng cũng dễ trả kết quả cũ hơn.

Riak thì giới hạn mọi liên lạc giữa client và node cơ sở dữ liệu trong region cục bộ, nên *n* biểu thị số replica trong một region. Việc sao chép (*replication*) xuyên region giữa các cụm cơ sở dữ liệu diễn ra bất đồng bộ ở nền, theo cách tương tự sao chép đa leader (*multi-leader replication*).


### Phát hiện ghi đồng thời {#sec_replication_concurrent}

Cũng như sao chép đa leader (*multi-leader replication*), cơ sở dữ liệu không leader cho phép ghi đồng thời vào cùng một khóa, dẫn đến xung đột cần giải quyết. Xung đột có thể xuất hiện ngay khi ghi, nhưng không phải lúc nào cũng vậy. Nó cũng có thể chỉ bị phát hiện ở giai đoạn read repair, hinted handoff hoặc anti-entropy.

Vấn đề là độ trễ mạng thay đổi và hệ thống có thể hỏng từng phần, nên các sự kiện có thể đến các node theo thứ tự khác nhau. Ví dụ, {{< xref fig="6-14" page="/ch6" anchor="fig_replication_concurrency" >}}Hình 6-14{{< /xref >}} cho thấy client A và B đồng thời ghi khóa *X* trong kho dữ liệu ba node:

* Node 1 nhận lần ghi của A, nhưng do gián đoạn ngắn nên không bao giờ nhận được lần ghi của B.
* Node 2 nhận lần ghi của A trước, rồi đến lần ghi của B.
* Node 3 nhận lần ghi của B trước, rồi đến lần ghi của A.

{{< fig num="6-14" id="fig_replication_concurrency" src="/fig/ddia_0614.png" caption="Các lần ghi đồng thời trong kho dữ liệu kiểu Dynamo không có thứ tự được xác định rõ." class="ddia-figure ddia-figure--wide" width="2880" height="1406" />}}


Nếu mỗi node chỉ cần nhận được yêu cầu ghi của client là ghi đè giá trị của khóa, các node sẽ không nhất quán vĩnh viễn, như yêu cầu *get* cuối cùng ở {{< xref fig="6-14" page="/ch6" anchor="fig_replication_concurrency" >}}Hình 6-14{{< /xref >}} cho thấy: node 2 cho rằng giá trị cuối cùng của *X* là B, còn các node khác cho rằng là A.

Để đạt nhất quán cuối cùng, các replica phải hội tụ về cùng một giá trị. Có thể dùng bất kỳ cơ chế giải quyết xung đột nào đã bàn ở ["Xử lý xung đột ghi"](/ch6#sec_replication_write_conflicts), ví dụ last write wins như Cassandra và ScyllaDB dùng, giải quyết thủ công, hoặc CRDT như giới thiệu ở ["CRDT và operational transformation"](/ch6#sec_replication_crdts) và Riak dùng.

Last write wins rất dễ triển khai: gắn timestamp cho mỗi lần ghi, giá trị có timestamp lớn hơn luôn ghi đè giá trị nhỏ hơn. Nhưng timestamp không cho bạn biết hai giá trị có thực sự xung đột hay không: chúng có thể được ghi đồng thời, hoặc ghi lần lượt. Nếu muốn giải quyết xung đột tường minh, hệ thống phải phát hiện ghi đồng thời kỹ hơn.

#### Quan hệ “xảy ra trước” và tính đồng thời {#sec_replication_happens_before}

Làm sao biết hai thao tác có đồng thời hay không? Hãy xem vài ví dụ để có trực giác:

* Ở {{< xref fig="6-8" page="/ch6" anchor="fig_replication_causality" >}}Hình 6-8{{< /xref >}}, hai lần ghi không đồng thời: lần chèn của A *xảy ra trước* lần tăng của B, vì giá trị B tăng chính là giá trị A đã chèn. Nói cách khác, thao tác của B dựa trên thao tác của A, nên B chắc chắn xảy ra muộn hơn. Cũng có thể nói B *phụ thuộc nhân quả* vào A.
* Ngược lại, hai lần ghi ở {{< xref fig="6-14" page="/ch6" anchor="fig_replication_concurrency" >}}Hình 6-14{{< /xref >}} là đồng thời: khi mỗi client bắt đầu thao tác, nó không biết client kia cũng đang thao tác trên cùng khóa. Do đó giữa hai thao tác không có phụ thuộc nhân quả.

Nếu thao tác B biết về A, phụ thuộc vào A, hoặc bằng cách nào đó dựa trên A, ta nói thao tác A *xảy ra trước* (*happens before*) thao tác B. Một thao tác có xảy ra trước thao tác kia hay không là chìa khóa để định nghĩa tính đồng thời. Thực ra, chỉ cần hai thao tác không thao tác nào xảy ra trước thao tác kia — tức là không bên nào biết về bên kia — thì có thể gọi chúng là *đồng thời* (*concurrent*) [^57].

Vì vậy, với hai thao tác A và B bất kỳ, chỉ có ba khả năng: A xảy ra trước B, B xảy ra trước A, hoặc A và B đồng thời. Ta cần một thuật toán để xác định hai thao tác có đồng thời hay không. Nếu một thao tác xảy ra trước thao tác kia, thao tác sau phải ghi đè thao tác trước. Nếu chúng đồng thời, có một xung đột cần giải quyết.


> [!TIP] Tính đồng thời, thời gian và thuyết tương đối
>
> Có vẻ hai thao tác chỉ nên gọi là đồng thời nếu chúng xảy ra "cùng một lúc", nhưng thực ra việc chúng có chồng lấn về thời gian vật lý hay không không quan trọng. Do vấn đề đồng hồ trong hệ thống phân tán, việc xác định hai sự kiện có xảy ra đúng cùng lúc hay không khá khó. Ta sẽ bàn thêm ở [Chương 9](/ch9#ch_distributed).
>
> Khi định nghĩa tính đồng thời, thời gian chính xác không quan trọng: chỉ cần hai thao tác không biết về nhau thì ta gọi chúng là đồng thời, bất kể thực tế chúng xảy ra vào thời điểm vật lý nào. Đôi khi người ta liên hệ nguyên lý này với thuyết tương đối hẹp trong vật lý [^57]. Thuyết tương đối hẹp cho rằng thông tin không thể truyền nhanh hơn tốc độ ánh sáng. Do đó, nếu hai sự kiện cách nhau một khoảng, và độ chênh lệch thời gian giữa chúng ngắn hơn thời gian ánh sáng truyền qua khoảng cách đó, thì chúng không thể ảnh hưởng lẫn nhau.
>
> Trong hệ thống máy tính, ngay cả khi xét theo tốc độ ánh sáng một thao tác về nguyên tắc vẫn kịp ảnh hưởng đến thao tác kia, hai thao tác vẫn có thể đồng thời. Chẳng hạn, khi mạng chậm hoặc đã gián đoạn, hai thao tác dù cách nhau một khoảng thời gian vẫn không thể biết về nhau vì sự cố mạng.


#### Nắm bắt quan hệ xảy ra trước {#capturing-the-happens-before-relationship}

Dưới đây là một thuật toán có thể xác định hai thao tác là đồng thời hay một thao tác xảy ra trước thao tác kia. Để đơn giản, ta bắt đầu với cơ sở dữ liệu chỉ có một replica. Khi đã hiểu cách làm với một replica, ta mở rộng sang cơ sở dữ liệu không leader có nhiều replica.

{{< xref fig="6-15" page="/ch6" anchor="fig_replication_causality_single" >}}Hình 6-15{{< /xref >}} cho thấy hai client đồng thời thêm mặt hàng vào cùng một giỏ hàng. (Nếu ví dụ này quá nhàm chán, hãy tưởng tượng hai kiểm soát viên không lưu đồng thời thêm máy bay vào vùng trời họ đang giám sát.) Giỏ hàng ban đầu trống, hai client lần lượt gửi tổng cộng năm lần ghi đến cơ sở dữ liệu:

1. Client 1 thêm `milk` vào giỏ hàng. Đây là lần ghi đầu tiên cho khóa này, server lưu thành công và gán phiên bản 1. Sau đó trả giá trị cùng số phiên bản cho client.
2. Client 2 thêm `eggs` vào giỏ hàng, mà không biết client 1 đồng thời thêm `milk` (nó tưởng `eggs` là mặt hàng duy nhất trong giỏ). Server gán phiên bản 2 cho lần ghi này, lưu `eggs` và `milk` thành hai giá trị riêng biệt (sibling), rồi trả *cả hai* giá trị cùng số phiên bản 2 cho client.
3. Client 1 không biết lần ghi của client 2, lại muốn thêm `flour`, nên cho rằng nội dung giỏ hàng phải là `[milk, flour]`. Nó gửi giá trị này cùng số phiên bản 1 mà server đã đưa trước đó. Server dựa vào số phiên bản có thể xác định: `[milk, flour]` thay thế `[milk]` trước đó, nhưng đồng thời với `[eggs]`. Vì vậy server gán phiên bản 3 cho `[milk, flour]`, ghi đè `[milk]` phiên bản 1, giữ `[eggs]` phiên bản 2, và trả cả hai giá trị còn lại cho client.
4. Cùng lúc đó, client 2 muốn thêm `ham`, mà không biết client 1 vừa thêm `flour`. Client 2 đã nhận `[milk]` và `[eggs]` trong phản hồi trước, nên hợp nhất hai giá trị này rồi thêm `ham`, thành giá trị mới `[eggs, milk, ham]`. Nó gửi giá trị này cùng số phiên bản 2 trước đó cho server. Server xác định phiên bản 2 có thể ghi đè `[eggs]`, nhưng đồng thời với `[milk, flour]`. Hai giá trị còn lại là `[milk, flour]` phiên bản 3 và `[eggs, milk, ham]` phiên bản 4.
5. Cuối cùng, client 1 muốn thêm `bacon`. Trước đó, ở phản hồi phiên bản 3, nó nhận `[milk, flour]` và `[eggs]`, nên hợp nhất hai giá trị, thêm `bacon`, rồi gửi giá trị cuối `[milk, flour, eggs, bacon]` cùng số phiên bản 3 cho server. Giá trị này ghi đè `[milk, flour]` (`[eggs]` đã bị ghi đè ở bước trước), nhưng đồng thời với `[eggs, milk, ham]`, nên server giữ cả hai giá trị đồng thời này.

{{< fig num="6-15" id="fig_replication_causality_single" src="/fig/ddia_0615.png" caption="Nắm bắt các phụ thuộc nhân quả khi hai client đồng thời chỉnh sửa giỏ hàng." class="ddia-figure ddia-figure--wide" width="2880" height="1633" />}}


Luồng dữ liệu giữa các thao tác ở {{< xref fig="6-15" page="/ch6" anchor="fig_replication_causality_single" >}}Hình 6-15{{< /xref >}} được biểu diễn bằng đồ thị ở {{< xref fig="6-16" page="/ch6" anchor="fig_replication_causal_dependencies" >}}Hình 6-16{{< /xref >}}. Mũi tên cho biết thao tác nào *xảy ra trước* thao tác khác, tức là thao tác sau *biết* hoặc *phụ thuộc* vào thao tác trước. Trong ví dụ này, client không bao giờ nắm đầy đủ dữ liệu mới nhất trên server, vì luôn có thao tác khác diễn ra đồng thời. Nhưng các phiên bản giá trị cũ cuối cùng sẽ bị ghi đè, và không lần ghi nào bị mất.

{{< fig num="6-16" id="fig_replication_causal_dependencies" src="/fig/ddia_0616.png" caption="Biểu diễn đồ thị các quan hệ phụ thuộc nhân quả ở Hình 6-15." link="#fig_replication_causality_single" class="ddia-figure ddia-figure--panorama" width="2880" height="718" />}}


Lưu ý rằng server chỉ dựa vào số phiên bản là xác định được hai thao tác có đồng thời hay không, không cần diễn giải bản thân giá trị, nên giá trị có thể là cấu trúc dữ liệu bất kỳ. Thuật toán như sau:

* Server duy trì một số phiên bản cho mỗi khóa, mỗi lần ghi khóa đó thì tăng số phiên bản, và lưu số phiên bản mới cùng giá trị được ghi.
* Khi client đọc một khóa, server trả về mọi giá trị sibling (tức mọi giá trị chưa bị ghi đè) cùng số phiên bản mới nhất. Client phải đọc trước khi ghi.
* Khi client ghi một khóa, nó phải kèm số phiên bản từ lần đọc trước, và phải hợp nhất mọi giá trị nhận được ở lần đọc trước đó, chẳng hạn dùng CRDT hoặc hỏi người dùng. Phản hồi của yêu cầu ghi giống phản hồi đọc, cũng trả về mọi sibling, nên có thể thực hiện nhiều lần ghi liên tiếp như ví dụ giỏ hàng.
* Khi server nhận lần ghi với một số phiên bản cụ thể, nó có thể ghi đè mọi giá trị có số phiên bản không cao hơn, vì các giá trị đó đã được hợp nhất vào giá trị mới. Giá trị có số phiên bản cao hơn phải được giữ lại, vì chúng đồng thời với lần ghi đến.

Lần ghi kèm số phiên bản của lần đọc trước cho biết lần ghi này dựa trên trạng thái trước đó nào. Nếu lần ghi không có số phiên bản, nó đồng thời với mọi lần ghi khác, nên không ghi đè gì cả mà chỉ trở thành một trong các giá trị được trả về ở những lần đọc sau.

#### Version vector {#version-vectors}

Ví dụ ở {{< xref fig="6-15" page="/ch6" anchor="fig_replication_causality_single" >}}Hình 6-15{{< /xref >}} chỉ có một replica. Nếu không có leader mà nhiều replica đều nhận được ghi thì thuật toán cần thay đổi thế nào?

{{< xref fig="6-15" page="/ch6" anchor="fig_replication_causality_single" >}}Hình 6-15{{< /xref >}} dùng một số phiên bản để nắm bắt quan hệ phụ thuộc giữa các thao tác, nhưng khi nhiều replica đồng thời nhận ghi thì một số phiên bản là không đủ. Lúc này phải duy trì số phiên bản riêng cho *từng replica*, theo từng khóa. Khi replica xử lý một lần ghi, nó tăng số phiên bản của chính mình, đồng thời ghi lại số phiên bản của các replica khác mà nó đã thấy. Thông tin này cho biết giá trị nào cần ghi đè, giá trị nào cần giữ lại làm sibling.

Tập hợp số phiên bản của mọi replica gọi là *version vector* (*version vector*) [^58]. Ý tưởng này có một số biến thể, trong đó đáng chú ý nhất có lẽ là *dotted version vector* (*dotted version vector*) [^59] [^60], được Riak 2.0 sử dụng [^61] [^62]. Ở đây ta không đi vào chi tiết. Cách hoạt động của nó rất giống ví dụ giỏ hàng.

Giống số phiên bản ở {{< xref fig="6-15" page="/ch6" anchor="fig_replication_causality_single" >}}Hình 6-15{{< /xref >}}, khi đọc, replica cơ sở dữ liệu gửi version vector cho client, và ở lần ghi sau đó client phải gửi lại nó cho cơ sở dữ liệu. (Riak mã hóa version vector thành một chuỗi gọi là *ngữ cảnh nhân quả*, *causal context*.) Version vector cho phép cơ sở dữ liệu phân biệt lần ghi đè với lần ghi đồng thời.

Version vector cũng đảm bảo việc đọc từ một replica rồi gửi lần ghi sang replica khác là an toàn. Làm vậy có thể sinh sibling, nhưng miễn là hợp nhất sibling đúng cách thì không mất dữ liệu.


> [!TIP] Version vector và vector clock
>
> *Version vector* đôi khi còn gọi là *vector clock* (*vector clock*), dù hai thứ không hoàn toàn giống nhau. Sự khác biệt khá tinh tế, chi tiết xem tài liệu liên quan [^60] [^63] [^64]. Nói ngắn gọn, khi so sánh trạng thái của các replica, nên dùng version vector.

## Tóm tắt {#summary}

Chương này đã xem xét vấn đề sao chép (*replication*). Sao chép (*replication*) phục vụ nhiều mục đích:

*Tính sẵn sàng cao*
:   Hệ thống tiếp tục chạy ngay cả khi một hoặc nhiều máy, một availability zone, thậm chí cả một region ngừng hoạt động

*Hoạt động ngoại tuyến*
:   Ứng dụng vẫn tiếp tục làm việc khi mạng bị gián đoạn

*Độ trễ*
:   Đặt dữ liệu gần người dùng về mặt địa lý để họ tương tác với nó nhanh hơn

*Khả năng mở rộng*
:   Dàn tải đọc ra nhiều replica để xử lý lượng đọc vượt quá khả năng của một máy

Mục tiêu của sao chép (*replication*) có vẻ đơn giản — giữ bản sao của cùng một dữ liệu trên nhiều máy — nhưng thực tế cực kỳ hóc búa. Nó đòi hỏi ta cân nhắc kỹ về tính đồng thời, về mọi thứ có thể trục trặc, và cách đối phó với hậu quả của sự cố. Ít nhất phải xử lý node không khả dụng và gián đoạn mạng, và đó còn chưa kể các sự cố tinh vi hơn như hỏng dữ liệu âm thầm do lỗi phần mềm hoặc lỗi phần cứng.

Ta đã bàn ba cách sao chép (*replication*) chính:

*Sao chép đơn leader* (*single-leader replication*)
:   Client gửi mọi lần ghi đến một node (leader), leader gửi luồng sự kiện thay đổi dữ liệu cho các replica khác (follower). Có thể đọc trên bất kỳ replica nào, nhưng kết quả từ follower có thể đã cũ.

*Sao chép đa leader* (*multi-leader replication*)
:   Client gửi mỗi lần ghi đến một trong nhiều leader, leader nào cũng nhận được ghi. Các leader gửi luồng sự kiện thay đổi dữ liệu cho nhau và cho các follower.

*Sao chép không leader* (*leaderless replication*)
:   Client gửi mỗi lần ghi đến nhiều node, và đọc song song từ nhiều node để phát hiện và sửa các node giữ dữ liệu cũ.

Mỗi cách có ưu nhược điểm riêng. Sao chép đơn leader (*single-leader replication*) phổ biến vì tương đối dễ hiểu và cho nhất quán mạnh. Sao chép đa leader (*multi-leader replication*) và không leader bền bỉ hơn trước sự cố node, gián đoạn mạng và độ trễ tăng vọt, nhưng cái giá là phải giải quyết xung đột và chỉ cung cấp đảm bảo nhất quán yếu hơn.

Sao chép (*replication*) có thể đồng bộ hoặc bất đồng bộ. Lựa chọn này ảnh hưởng sâu sắc đến hành vi của hệ thống khi có sự cố. Khi hệ thống chạy trơn tru, sao chép bất đồng bộ (*asynchronous replication*) có thể rất nhanh, nhưng vẫn phải làm rõ điều gì xảy ra khi replication lag tăng lên và khi server hỏng. Nếu leader hỏng và bạn đưa một follower cập nhật bất đồng bộ lên làm leader mới, dữ liệu vừa commit gần đây có thể bị mất.

Ta đã xem xét một số dị thường do replication lag gây ra, và bàn vài mô hình nhất quán giúp quyết định ứng dụng nên hành xử ra sao khi có replication lag:

*Nhất quán read-after-write*
:   Người dùng luôn phải thấy dữ liệu mình đã gửi.

*Monotonic reads*
:   Sau khi người dùng đã thấy dữ liệu tại một thời điểm, về sau họ không được thấy dữ liệu ở thời điểm cũ hơn.

*Consistent prefix reads*
:   Người dùng phải thấy dữ liệu ở trạng thái phù hợp với quan hệ nhân quả, chẳng hạn thấy câu hỏi và câu trả lời theo đúng thứ tự.

Cuối cùng, ta đã bàn cách sao chép đa leader (*multi-leader replication*) và không leader giúp mọi replica cuối cùng hội tụ về một trạng thái nhất quán: dùng version vector hoặc thuật toán tương tự để phát hiện những lần ghi nào là đồng thời, rồi dùng thuật toán giải quyết xung đột như CRDT để hợp nhất các giá trị được ghi đồng thời. Last write wins và giải quyết xung đột thủ công cũng là các phương án khả dĩ.

Chương này luôn giả định mỗi replica giữ một bản sao đầy đủ của toàn bộ cơ sở dữ liệu, nhưng với tập dữ liệu lớn điều này không thực tế. Chương tiếp theo sẽ giới thiệu *sharding*, giúp mỗi máy chỉ cần lưu một phần dữ liệu.




### Tài liệu tham khảo


[^1]: B. G. Lindsay, P. G. Selinger, C. Galtieri, J. N. Gray, R. A. Lorie, T. G. Price, F. Putzolu, I. L. Traiger, and B. W. Wade. [Notes on Distributed Databases](https://dominoweb.draco.res.ibm.com/reports/RJ2571.pdf). IBM Research, Research Report RJ2571(33471), July 1979. Archived at [perma.cc/EPZ3-MHDD](https://perma.cc/EPZ3-MHDD)
[^2]: Kenny Gryp. [MySQL Terminology Updates](https://dev.mysql.com/blog-archive/mysql-terminology-updates/). *dev.mysql.com*, July 2020. Archived at [perma.cc/S62G-6RJ2](https://perma.cc/S62G-6RJ2)
[^3]: Oracle Corporation. [Oracle (Active) Data Guard 19c: Real-Time Data Protection and Availability](https://www.oracle.com/technetwork/database/availability/dg-adg-technical-overview-wp-5347548.pdf). White Paper, *oracle.com*, March 2019. Archived at [perma.cc/P5ST-RPKE](https://perma.cc/P5ST-RPKE)
[^4]: Microsoft. [What is an Always On availability group?](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server) *learn.microsoft.com*, September 2024. Archived at [perma.cc/ABH6-3MXF](https://perma.cc/ABH6-3MXF)
[^5]: Mostafa Elhemali, Niall Gallagher, Nicholas Gordon, Joseph Idziorek, Richard Krog, Colin Lazier, Erben Mo, Akhilesh Mritunjai, Somu Perianayagam, Tim Rath, Swami Sivasubramanian, James Christopher Sorenson III, Sroaj Sosothikul, Doug Terry, and Akshat Vig. [Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service](https://www.usenix.org/conference/atc22/presentation/elhemali). At *USENIX Annual Technical Conference* (ATC), July 2022.
[^6]: Rebecca Taft, Irfan Sharif, Andrei Matei, Nathan VanBenschoten, Jordan Lewis, Tobias Grieger, Kai Niemi, Andy Woods, Anne Birzin, Raphael Poss, Paul Bardea, Amruta Ranade, Ben Darnell, Bram Gruneir, Justin Jaffray, Lucy Zhang, and Peter Mattis. [CockroachDB: The Resilient Geo-Distributed SQL Database](https://dl.acm.org/doi/abs/10.1145/3318464.3386134). At *ACM SIGMOD International Conference on Management of Data* (SIGMOD), pages 1493–1509, June 2020. [doi:10.1145/3318464.3386134](https://doi.org/10.1145/3318464.3386134)
[^7]: Dongxu Huang, Qi Liu, Qiu Cui, Zhuhe Fang, Xiaoyu Ma, Fei Xu, Li Shen, Liu Tang, Yuxing Zhou, Menglong Huang, Wan Wei, Cong Liu, Jian Zhang, Jianjun Li, Xuelian Wu, Lingyu Song, Ruoxi Sun, Shuaipeng Yu, Lei Zhao, Nicholas Cameron, Liquan Pei, and Xin Tang. [TiDB: a Raft-based HTAP database](https://www.vldb.org/pvldb/vol13/p3072-huang.pdf). *Proceedings of the VLDB Endowment*, volume 13, issue 12, pages 3072–3084. [doi:10.14778/3415478.3415535](https://doi.org/10.14778/3415478.3415535)
[^8]: Mallory Knodel and Niels ten Oever. [Terminology, Power, and Inclusive Language in Internet-Drafts and RFCs](https://www.ietf.org/archive/id/draft-knodel-terminology-14.html). *IETF Internet-Draft*, August 2023. Archived at [perma.cc/5ZY9-725E](https://perma.cc/5ZY9-725E)
[^9]: Buck Hodges. [Postmortem: VSTS 4 September 2018](https://devblogs.microsoft.com/devopsservice/?p=17485). *devblogs.microsoft.com*, September 2018. Archived at [perma.cc/ZF5R-DYZS](https://perma.cc/ZF5R-DYZS)
[^10]: Gunnar Morling. [Leader Election With S3 Conditional Writes](https://www.morling.dev/blog/leader-election-with-s3-conditional-writes/). *www.morling.dev*, August 2024. Archived at [perma.cc/7V2N-J78Y](https://perma.cc/7V2N-J78Y)
[^11]: Vignesh Chandramohan, Rohan Desai, and Chris Riccomini. [SlateDB Manifest Design](https://github.com/slatedb/slatedb/blob/main/rfcs/0001-manifest.md). *github.com*, May 2024. Archived at [perma.cc/8EUY-P32Z](https://perma.cc/8EUY-P32Z)
[^12]: Stas Kelvich. [Why does Neon use Paxos instead of Raft, and what’s the difference?](https://neon.tech/blog/paxos) *neon.tech*, August 2022. Archived at [perma.cc/SEZ4-2GXU](https://perma.cc/SEZ4-2GXU)
[^13]: Dimitri Fontaine. [An introduction to the pg\_auto\_failover project](https://tapoueh.org/blog/2021/11/an-introduction-to-the-pg_auto_failover-project/). *tapoueh.org*, November 2021. Archived at [perma.cc/3WH5-6BAF](https://perma.cc/3WH5-6BAF)
[^14]: Jesse Newland. [GitHub availability this week](https://github.blog/news-insights/the-library/github-availability-this-week/). *github.blog*, September 2012. Archived at [perma.cc/3YRF-FTFJ](https://perma.cc/3YRF-FTFJ)
[^15]: Mark Imbriaco. [Downtime last Saturday](https://github.blog/news-insights/the-library/downtime-last-saturday/). *github.blog*, December 2012. Archived at [perma.cc/M7X5-E8SQ](https://perma.cc/M7X5-E8SQ)
[^16]: John Hugg. [‘All In’ with Determinism for Performance and Testing in Distributed Systems](https://www.youtube.com/watch?v=gJRj3vJL4wE). At *Strange Loop*, September 2015.
[^17]: Hironobu Suzuki. [The Internals of PostgreSQL](https://www.interdb.jp/pg/). *interdb.jp*, 2017.
[^18]: Amit Kapila. [WAL Internals of PostgreSQL](https://www.pgcon.org/2012/schedule/attachments/258_212_Internals%20Of%20PostgreSQL%20Wal.pdf). At *PostgreSQL Conference* (PGCon), May 2012. Archived at [perma.cc/6225-3SUX](https://perma.cc/6225-3SUX)
[^19]: Amit Kapila. [Evolution of Logical Replication](https://amitkapila16.blogspot.com/2023/09/evolution-of-logical-replication.html). *amitkapila16.blogspot.com*, September 2023. Archived at [perma.cc/F9VX-JLER](https://perma.cc/F9VX-JLER)
[^20]: Aru Petchimuthu. [Upgrade your Amazon RDS for PostgreSQL or Amazon Aurora PostgreSQL database, Part 2: Using the pglogical extension](https://aws.amazon.com/blogs/database/part-2-upgrade-your-amazon-rds-for-postgresql-database-using-the-pglogical-extension/). *aws.amazon.com*, August 2021. Archived at [perma.cc/RXT8-FS2T](https://perma.cc/RXT8-FS2T)
[^21]: Yogeshwer Sharma, Philippe Ajoux, Petchean Ang, David Callies, Abhishek Choudhary, Laurent Demailly, Thomas Fersch, Liat Atsmon Guz, Andrzej Kotulski, Sachin Kulkarni, Sanjeev Kumar, Harry Li, Jun Li, Evgeniy Makeev, Kowshik Prakasam, Robbert van Renesse, Sabyasachi Roy, Pratyush Seth, Yee Jiun Song, Benjamin Wester, Kaushik Veeraraghavan, and Peter Xie. [Wormhole: Reliable Pub-Sub to Support Geo-Replicated Internet Services](https://www.usenix.org/system/files/conference/nsdi15/nsdi15-paper-sharma.pdf). At *12th USENIX Symposium on Networked Systems Design and Implementation* (NSDI), May 2015.
[^22]: Douglas B. Terry. [Replicated Data Consistency Explained Through Baseball](https://www.microsoft.com/en-us/research/publication/replicated-data-consistency-explained-through-baseball/). Microsoft Research, Technical Report MSR-TR-2011-137, October 2011. Archived at [perma.cc/F4KZ-AR38](https://perma.cc/F4KZ-AR38)
[^23]: Douglas B. Terry, Alan J. Demers, Karin Petersen, Mike J. Spreitzer, Marvin M. Theher, and Brent B. Welch. [Session Guarantees for Weakly Consistent Replicated Data](https://csis.pace.edu/~marchese/CS865/Papers/SessionGuaranteesPDIS.pdf). At *3rd International Conference on Parallel and Distributed Information Systems* (PDIS), September 1994. [doi:10.1109/PDIS.1994.331722](https://doi.org/10.1109/PDIS.1994.331722)
[^24]: Werner Vogels. [Eventually Consistent](https://queue.acm.org/detail.cfm?id=1466448). *ACM Queue*, volume 6, issue 6, pages 14–19, October 2008. [doi:10.1145/1466443.1466448](https://doi.org/10.1145/1466443.1466448)
[^25]: Simon Willison. [Reply to: “My thoughts about Fly.io (so far) and other newish technology I’m getting into”](https://news.ycombinator.com/item?id=31434055). *news.ycombinator.com*, May 2022. Archived at [perma.cc/ZRV4-WWV8](https://perma.cc/ZRV4-WWV8)
[^26]: Nithin Tharakan. [Scaling Bitbucket’s Database](https://www.atlassian.com/blog/bitbucket/scaling-bitbuckets-database). *atlassian.com*, October 2020. Archived at [perma.cc/JAB7-9FGX](https://perma.cc/JAB7-9FGX)
[^27]: Terry Pratchett. *Reaper Man: A Discworld Novel*. Victor Gollancz, 1991. ISBN: 978-0-575-04979-6
[^28]: Peter Bailis, Alan Fekete, Michael J. Franklin, Ali Ghodsi, Joseph M. Hellerstein, and Ion Stoica. [Coordination Avoidance in Database Systems](https://arxiv.org/abs/1402.2237). *Proceedings of the VLDB Endowment*, volume 8, issue 3, pages 185–196, November 2014. [doi:10.14778/2735508.2735509](https://doi.org/10.14778/2735508.2735509)
[^29]: Yaser Raja and Peter Celentano. [PostgreSQL bi-directional replication using pglogical](https://aws.amazon.com/blogs/database/postgresql-bi-directional-replication-using-pglogical/). *aws.amazon.com*, January 2022. Archived at <https://perma.cc/BUQ2-5QWN>
[^30]: Robert Hodges. [If You \*Must\* Deploy Multi-Master Replication, Read This First](https://scale-out-blog.blogspot.com/2012/04/if-you-must-deploy-multi-master.html). *scale-out-blog.blogspot.com*, April 2012. Archived at [perma.cc/C2JN-F6Y8](https://perma.cc/C2JN-F6Y8)
[^31]: Lars Hofhansl. [HBASE-7709: Infinite Loop Possible in Master/Master Replication](https://issues.apache.org/jira/browse/HBASE-7709). *issues.apache.org*, January 2013. Archived at [perma.cc/24G2-8NLC](https://perma.cc/24G2-8NLC)
[^32]: John Day-Richter. [What’s Different About the New Google Docs: Making Collaboration Fast](https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs.html). *drive.googleblog.com*, September 2010. Archived at [perma.cc/5TL8-TSJ2](https://perma.cc/5TL8-TSJ2)
[^33]: Evan Wallace. [How Figma’s multiplayer technology works](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/). *figma.com*, October 2019. Archived at [perma.cc/L49H-LY4D](https://perma.cc/L49H-LY4D)
[^34]: Tuomas Artman. [Scaling the Linear Sync Engine](https://linear.app/blog/scaling-the-linear-sync-engine). *linear.app*, June 2023.
[^35]: Amr Saafan. [Why Sync Engines Might Be the Future of Web Applications](https://perma.cc/5N73-5M3V). *nilebits.com*, September 2024. Archived at [perma.cc/5N73-5M3V](https://perma.cc/5N73-5M3V)
[^36]: Isaac Hagoel. [Are Sync Engines The Future of Web Applications?](https://dev.to/isaachagoel/are-sync-engines-the-future-of-web-applications-1bbi) *dev.to*, July 2024. Archived at [perma.cc/R9HF-BKKL](https://perma.cc/R9HF-BKKL)
[^37]: Sujay Jayakar. [A Map of Sync](https://stack.convex.dev/a-map-of-sync). *stack.convex.dev*, October 2024. Archived at [perma.cc/82R3-H42A](https://perma.cc/82R3-H42A)
[^38]: Alex Feyerke. [Designing Offline-First Web Apps](https://alistapart.com/article/offline-first/). *alistapart.com*, December 2013. Archived at [perma.cc/WH7R-S2DS](https://perma.cc/WH7R-S2DS)
[^39]: Martin Kleppmann, Adam Wiggins, Peter van Hardenberg, and Mark McGranaghan. [Local-first software: You own your data, in spite of the cloud](https://www.inkandswitch.com/local-first/). At *ACM SIGPLAN International Symposium on New Ideas, New Paradigms, and Reflections on Programming and Software* (Onward!), October 2019, pages 154–178. [doi:10.1145/3359591.3359737](https://doi.org/10.1145/3359591.3359737)
[^40]: Martin Kleppmann. [The past, present, and future of local-first](https://martin.kleppmann.com/2024/05/30/local-first-conference.html). At *Local-First Conference*, May 2024.
[^41]: Conrad Hofmeyr. [API Calling is to Sync Engines as jQuery is to React](https://www.powersync.com/blog/api-calling-is-to-sync-engines-as-jquery-is-to-react). *powersync.com*, November 2024. Archived at [perma.cc/2FP9-7WJJ](https://perma.cc/2FP9-7WJJ)
[^42]: Peter van Hardenberg and Martin Kleppmann. [PushPin: Towards Production-Quality Peer-to-Peer Collaboration](https://martin.kleppmann.com/papers/pushpin-papoc20.pdf). At *7th Workshop on Principles and Practice of Consistency for Distributed Data* (PaPoC), April 2020. [doi:10.1145/3380787.3393683](https://doi.org/10.1145/3380787.3393683)
[^43]: Leonard Kawell, Jr., Steven Beckhardt, Timothy Halvorsen, Raymond Ozzie, and Irene Greif. [Replicated document management in a group communication system](https://dl.acm.org/doi/pdf/10.1145/62266.1024798). At *ACM Conference on Computer-Supported Cooperative Work* (CSCW), September 1988. [doi:10.1145/62266.1024798](https://doi.org/10.1145/62266.1024798)
[^44]: Ricky Pusch. [Explaining how fighting games use delay-based and rollback netcode](https://words.infil.net/w02-netcode.html). *words.infil.net* and *arstechnica.com*, October 2019. Archived at [perma.cc/DE7W-RDJ8](https://perma.cc/DE7W-RDJ8)
[^45]: Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, Gunavardhan Kakulapati, Avinash Lakshman, Alex Pilchin, Swaminathan Sivasubramanian, Peter Vosshall, and Werner Vogels. [Dynamo: Amazon’s Highly Available Key-Value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf). At *21st ACM Symposium on Operating Systems Principles* (SOSP), October 2007. [doi:10.1145/1323293.1294281](https://doi.org/10.1145/1323293.1294281)
[^46]: Marc Shapiro, Nuno Preguiça, Carlos Baquero, and Marek Zawirski. [A Comprehensive Study of Convergent and Commutative Replicated Data Types](https://inria.hal.science/inria-00555588v1/document). INRIA Research Report no. 7506, January 2011.
[^47]: Chengzheng Sun and Clarence Ellis. [Operational Transformation in Real-Time Group Editors: Issues, Algorithms, and Achievements](https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=aef660812c5a9c4d3f06775f9455eeb090a4ff0f). At *ACM Conference on Computer Supported Cooperative Work* (CSCW), November 1998. [doi:10.1145/289444.289469](https://doi.org/10.1145/289444.289469)
[^48]: Joseph Gentle and Martin Kleppmann. [Collaborative Text Editing with Eg-walker: Better, Faster, Smaller](https://arxiv.org/abs/2409.14252). At *20th European Conference on Computer Systems* (EuroSys), March 2025. [doi:10.1145/3689031.3696076](https://doi.org/10.1145/3689031.3696076)
[^49]: Dharma Shukla. [Azure Cosmos DB: Pushing the frontier of globally distributed databases](https://azure.microsoft.com/en-us/blog/azure-cosmos-db-pushing-the-frontier-of-globally-distributed-databases/). *azure.microsoft.com*, September 2018. Archived at [perma.cc/UT3B-HH6R](https://perma.cc/UT3B-HH6R)
[^50]: David K. Gifford. [Weighted Voting for Replicated Data](https://www.cs.cmu.edu/~15-749/READINGS/required/availability/gifford79.pdf). At *7th ACM Symposium on Operating Systems Principles* (SOSP), December 1979. [doi:10.1145/800215.806583](https://doi.org/10.1145/800215.806583)
[^51]: Heidi Howard, Dahlia Malkhi, and Alexander Spiegelman. [Flexible Paxos: Quorum Intersection Revisited](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.OPODIS.2016.25). At *20th International Conference on Principles of Distributed Systems* (OPODIS), December 2016. [doi:10.4230/LIPIcs.OPODIS.2016.25](https://doi.org/10.4230/LIPIcs.OPODIS.2016.25)
[^52]: Joseph Blomstedt. [Bringing Consistency to Riak](https://vimeo.com/51973001). At *RICON West*, October 2012.
[^53]: Peter Bailis, Shivaram Venkataraman, Michael J. Franklin, Joseph M. Hellerstein, and Ion Stoica. [Quantifying eventual consistency with PBS](http://www.bailis.org/papers/pbs-vldbj2014.pdf). *The VLDB Journal*, volume 23, pages 279–302, April 2014. [doi:10.1007/s00778-013-0330-1](https://doi.org/10.1007/s00778-013-0330-1)
[^54]: Colin Breck. [Shared-Nothing Architectures for Server Replication and Synchronization](https://blog.colinbreck.com/shared-nothing-architectures-for-server-replication-and-synchronization/). *blog.colinbreck.com*, December 2019. Archived at [perma.cc/48P3-J6CJ](https://perma.cc/48P3-J6CJ)
[^55]: Jeffrey Dean and Luiz André Barroso. [The Tail at Scale](https://cacm.acm.org/research/the-tail-at-scale/). *Communications of the ACM*, volume 56, issue 2, pages 74–80, February 2013. [doi:10.1145/2408776.2408794](https://doi.org/10.1145/2408776.2408794)
[^56]: Peng Huang, Chuanxiong Guo, Lidong Zhou, Jacob R. Lorch, Yingnong Dang, Murali Chintalapati, and Randolph Yao. [Gray Failure: The Achilles’ Heel of Cloud-Scale Systems](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/06/paper-1.pdf). At *16th Workshop on Hot Topics in Operating Systems* (HotOS), May 2017. [doi:10.1145/3102980.3103005](https://doi.org/10.1145/3102980.3103005)
[^57]: Leslie Lamport. [Time, Clocks, and the Ordering of Events in a Distributed System](https://www.microsoft.com/en-us/research/publication/time-clocks-ordering-events-distributed-system/). *Communications of the ACM*, volume 21, issue 7, pages 558–565, July 1978. [doi:10.1145/359545.359563](https://doi.org/10.1145/359545.359563)
[^58]: D. Stott Parker Jr., Gerald J. Popek, Gerard Rudisin, Allen Stoughton, Bruce J. Walker, Evelyn Walton, Johanna M. Chow, David Edwards, Stephen Kiser, and Charles Kline. [Detection of Mutual Inconsistency in Distributed Systems](https://pages.cs.wisc.edu/~remzi/Classes/739/Papers/parker83detection.pdf). *IEEE Transactions on Software Engineering*, volume SE-9, issue 3, pages 240–247, May 1983. [doi:10.1109/TSE.1983.236733](https://doi.org/10.1109/TSE.1983.236733)
[^59]: Nuno Preguiça, Carlos Baquero, Paulo Sérgio Almeida, Victor Fonte, and Ricardo Gonçalves. [Dotted Version Vectors: Logical Clocks for Optimistic Replication](https://arxiv.org/abs/1011.5808). arXiv:1011.5808, November 2010.
[^60]: Giridhar Manepalli. [Clocks and Causality - Ordering Events in Distributed Systems](https://www.exhypothesi.com/clocks-and-causality/). *exhypothesi.com*, November 2022. Archived at [perma.cc/8REU-KVLQ](https://perma.cc/8REU-KVLQ)
[^61]: Sean Cribbs. [A Brief History of Time in Riak](https://speakerdeck.com/seancribbs/a-brief-history-of-time-in-riak). At *RICON*, October 2014. Archived at [perma.cc/7U9P-6JFX](https://perma.cc/7U9P-6JFX)
[^62]: Russell Brown. [Vector Clocks Revisited Part 2: Dotted Version Vectors](https://riak.com/posts/technical/vector-clocks-revisited-part-2-dotted-version-vectors/). *riak.com*, November 2015. Archived at [perma.cc/96QP-W98R](https://perma.cc/96QP-W98R)
[^63]: Carlos Baquero. [Version Vectors Are Not Vector Clocks](https://haslab.wordpress.com/2011/07/08/version-vectors-are-not-vector-clocks/). *haslab.wordpress.com*, July 2011. Archived at [perma.cc/7PNU-4AMG](https://perma.cc/7PNU-4AMG)
[^64]: Reinhard Schwarz and Friedemann Mattern. [Detecting Causal Relationships in Distributed Computations: In Search of the Holy Grail](https://disco.ethz.ch/courses/hs08/seminar/papers/mattern4.pdf). *Distributed Computing*, volume 7, issue 3, pages 149–174, March 1994. [doi:10.1007/BF02277859](https://doi.org/10.1007/BF02277859)
