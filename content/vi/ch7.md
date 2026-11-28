---
translationKey: ddia-v2-ch7
title: Phân mảnh
book_kind: chapter
book_number: "7"
book_part: II
weight: 207
breadcrumbs: false
---

<a id="ch_sharding"></a>

![](/map/ch06.png)

> *Rõ ràng là ta phải thoát ra khỏi lối mòn của những chuỗi lệnh máy tính, đừng để máy tính bị trói buộc vào đó. Ta phải phát biểu định nghĩa, chỉ rõ thứ tự ưu tiên, mô tả dữ liệu. Ta phải làm rõ các mối quan hệ, thay vì viết ra các thủ tục.*
>
> Grace Murray Hopper, *Management and the Computer of the Future* (1962)

Cơ sở dữ liệu phân tán thường phân bố dữ liệu trên các node theo hai cách:

1. Giữ bản sao của cùng một dữ liệu trên nhiều node: đây là *sao chép* (*replication*), mà ta đã bàn ở [Chương 6](/ch6#ch_replication).
2. Nếu không muốn mỗi node lưu toàn bộ dữ liệu, ta có thể tách tập dữ liệu lớn thành các *shard* (*shard*) hay *partition* (*partition*) nhỏ hơn, rồi lưu các shard khác nhau trên các node khác nhau. Đó là chủ đề của chương này.

Thông thường, mỗi mẩu dữ liệu (mỗi bản ghi, dòng hay tài liệu) thuộc về đúng một shard. Có nhiều cách để làm điều này, và chương này sẽ bàn sâu một số cách. Thực tế, mỗi shard là một cơ sở dữ liệu nhỏ của riêng nó, dù một số cơ sở dữ liệu hỗ trợ cả các thao tác chạm đến nhiều shard cùng lúc.

Sharding thường được kết hợp với sao chép (*replication*) để bản sao của mỗi shard được lưu trên nhiều node. Nghĩa là dù mỗi bản ghi chỉ thuộc về một shard, nó vẫn có thể nằm trên nhiều node khác nhau để chịu lỗi.

Một node có thể lưu nhiều shard. Nếu dùng mô hình sao chép đơn leader (*single-leader replication*), sự kết hợp giữa sharding và sao chép (*replication*) có thể trông như {{< xref fig="7-1" page="/ch7" anchor="fig_sharding_replicas" >}}Hình 7-1{{< /xref >}}. Leader của mỗi shard được gán cho một node, còn các follower được gán cho các node khác. Mỗi node có thể là leader của một số shard và đồng thời là follower của các shard khác, nhưng mỗi shard vẫn chỉ có một leader.

{{< fig num="7-1" id="fig_sharding_replicas" src="/fig/ddia_0701.png" caption="Kết hợp sao chép (*replication*) và sharding: mỗi node đóng vai trò leader của một số shard và follower của các shard khác." class="ddia-figure ddia-figure--wide" width="2658" height="1380" />}}

Mọi điều ta đã bàn ở [Chương 6](/ch6#ch_replication) về sao chép (*replication*) cơ sở dữ liệu đều áp dụng tương tự cho việc sao chép (*replication*) các shard. Phần lớn trường hợp, việc chọn sơ đồ sharding độc lập với việc chọn sơ đồ sao chép (*replication*), nên để đơn giản, chương này sẽ bỏ qua sao chép (*replication*).

> [!TIP] Sharding và partitioning
>
> Thứ mà chương này gọi là *shard* có rất nhiều tên khác nhau tùy phần mềm: Kafka gọi là *partition*, CockroachDB gọi là *range*, HBase và TiDB gọi là *region*, Bigtable và YugabyteDB gọi là *tablet*, Cassandra, ScyllaDB và Riak gọi là *vnode*, còn Couchbase gọi là *vBucket* — đây mới chỉ là vài ví dụ.
>
> Một số cơ sở dữ liệu coi partitioning và sharding là hai khái niệm khác nhau. Ví dụ trong PostgreSQL, partitioning là chia một bảng lớn thành nhiều tệp lưu trên cùng một máy (có vài lợi ích, chẳng hạn xóa cả một partition cực nhanh), còn sharding là chia tập dữ liệu ra nhiều máy [^1] [^2]. Ở nhiều hệ thống khác, partitioning chỉ là một tên gọi khác của sharding.
>
> Từ *partitioning* khá dễ hiểu, còn *sharding* thì cái tên hơi bất ngờ. Một giả thuyết cho rằng nó bắt nguồn từ game nhập vai trực tuyến *Ultima Online*, trong đó một viên pha lê ma thuật vỡ thành nhiều mảnh, và mỗi mảnh phản chiếu một bản sao của thế giới game [^3]. Sau đó *shard* được dùng để chỉ một trong các máy chủ game chạy song song, rồi lan sang cơ sở dữ liệu. Một giả thuyết khác là *shard* vốn là từ viết tắt của *System for Highly Available Replicated Data* — nghe nói đó là một cơ sở dữ liệu từ thập niên 1980, nhưng chi tiết đã chìm vào quên lãng.
>
> Nhân tiện, partitioning không liên quan gì đến *network partition* (còn gọi là netsplit), một loại lỗi xảy ra trong mạng giữa các node. Ta sẽ bàn ở [Chương 9](/ch9#ch_distributed).

## Ưu và nhược điểm của sharding {#sec_sharding_reasons}

Lý do chính để shard một cơ sở dữ liệu là *khả năng mở rộng* (*scalability*): đây là giải pháp khi khối lượng dữ liệu hoặc thông lượng ghi lớn đến mức một node đơn lẻ không chịu nổi, vì nó cho phép trải dữ liệu và việc ghi ra nhiều node. (Nếu nút thắt là thông lượng đọc thì chưa chắc cần sharding — bạn có thể dùng *mở rộng đọc* (*read scaling*) như mô tả ở [Chương 6](/ch6#ch_replication).)

Thật ra, sharding là một trong những công cụ chính để đạt *mở rộng theo chiều ngang* (*horizontal scaling*, còn gọi là kiến trúc *scale-out*), như đã bàn ở ["Kiến trúc chia sẻ bộ nhớ, chia sẻ đĩa và shared-nothing"](/ch2#sec_introduction_shared_nothing): thay vì mua máy to hơn, ta tăng dung lượng bằng cách thêm nhiều máy (nhỏ hơn). Nếu chia được khối lượng công việc sao cho mỗi shard đảm nhận phần xấp xỉ bằng nhau, ta có thể gán các shard cho các máy khác nhau để xử lý song song dữ liệu và truy vấn của chúng.

Sao chép (*replication*) hữu ích ở mọi quy mô vì nó giúp chịu lỗi và chạy ngoại tuyến, còn sharding là giải pháp nặng nề, chủ yếu phù hợp với quy mô lớn. Nếu dữ liệu và thông lượng ghi vẫn xử lý được trên một máy (ngày nay một máy đơn lẻ mạnh hơn nhiều so với tưởng tượng!), thường nên tránh sharding và dùng cơ sở dữ liệu một shard.

Lý do là sharding thường làm tăng độ phức tạp. Thường bạn phải chọn một *partition key* (*partition key*) để quyết định bản ghi nào vào shard nào; các bản ghi cùng partition key sẽ vào cùng một shard [^4]. Lựa chọn này rất quan trọng: nếu biết bản ghi ở shard nào, việc truy cập nhanh, nhưng nếu không biết thì bạn phải tìm kiếm kém hiệu quả trên mọi shard, và sau này rất khó đổi sơ đồ sharding.

Vì vậy sharding thường hợp với dữ liệu khóa-giá trị, nơi có thể shard thẳng theo khóa. Dữ liệu quan hệ khó hơn, vì có thể bạn cần tìm theo *chỉ mục phụ* (*secondary index*) hoặc join các bản ghi nằm rải rác ở nhiều shard. Ta sẽ bàn thêm ở ["Sharding và chỉ mục phụ (*secondary index*)"](/ch7#sec_sharding_secondary_indexes).

Một vấn đề nữa: một lần ghi có thể cần cập nhật các bản ghi liên quan ở nhiều shard khác nhau. Giao dịch trên một node khá phổ biến (xem [Chương 8](/ch8#ch_transactions)), nhưng để đảm bảo nhất quán giữa nhiều shard thì cần *giao dịch phân tán* (*distributed transaction*). Như [Chương 8](/ch8#ch_transactions) sẽ giải thích, một số cơ sở dữ liệu hỗ trợ giao dịch phân tán, nhưng chúng thường chậm hơn nhiều so với giao dịch trên một node, có thể thành nút thắt của cả hệ thống, và một số hệ thống hoàn toàn không hỗ trợ.

Một số hệ thống còn dùng sharding ngay trên một máy đơn lẻ, thường bằng cách chạy một tiến trình đơn luồng cho mỗi lõi CPU để tận dụng song song của CPU, hoặc để tận dụng kiến trúc *truy cập bộ nhớ không đồng nhất* (*non-uniform memory access*, NUMA), trong đó một số vùng nhớ ở gần một CPU nhất định hơn các CPU khác [^5]. Chẳng hạn, Redis, VoltDB và FoundationDB dùng một tiến trình trên mỗi lõi và dựa vào sharding để trải tải ra các lõi CPU trong cùng một máy [^6].

### Sharding cho hệ thống multitenant {#sec_sharding_multitenancy}

Các sản phẩm phần mềm dạng dịch vụ (SaaS) và dịch vụ đám mây thường là *multitenant* (*multitenant*), trong đó mỗi tenant là một khách hàng. Một tenant có thể có nhiều tài khoản người dùng, nhưng mỗi tenant có một tập dữ liệu độc lập, tách biệt với các tenant khác. Chẳng hạn, trong dịch vụ email marketing, mỗi doanh nghiệp đăng ký thường là một tenant riêng, vì dữ liệu đăng ký bản tin, dữ liệu gửi thư... của các doanh nghiệp không liên quan đến nhau.

Đôi khi hệ thống multitenant được hiện thực bằng sharding: có thể gán cho mỗi tenant một shard riêng, hoặc gộp nhiều tenant nhỏ vào một shard lớn hơn. Các shard này có thể là các cơ sở dữ liệu tách biệt về mặt vật lý (như đã nhắc ở ["Storage engine nhúng"](/ch4#sidebar_embedded)), hoặc là các phần có thể quản lý riêng của một cơ sở dữ liệu logic lớn hơn [^7]. Dùng sharding để hiện thực multitenancy có những ưu điểm sau:

Cô lập tài nguyên
: Nếu một tenant thực hiện thao tác tốn nhiều tính toán, thì khi nó nằm ở shard khác, các tenant khác ít bị ảnh hưởng hiệu năng hơn.

Cô lập quyền truy cập
: Nếu logic kiểm soát truy cập có lỗ hổng, thì khi dữ liệu của các tenant được cô lập về mặt vật lý, khả năng vô tình cho tenant này truy cập dữ liệu của tenant khác sẽ thấp hơn.

Kiến trúc dựa trên cell
: Sharding không chỉ dùng ở tầng lưu trữ dữ liệu mà còn có thể dùng để chia các dịch vụ chạy mã ứng dụng. Trong *cell-based architecture*, các dịch vụ và lưu trữ phục vụ một nhóm tenant cụ thể được đóng gói thành một *cell* (*cell*) khép kín, và các cell khác nhau có thể chạy gần như độc lập với nhau. Cách này giúp *cô lập lỗi* (*fault isolation*): lỗi trong một cell chỉ ảnh hưởng đến cell đó, còn tenant ở các cell khác không bị tác động [^8].

Sao lưu và khôi phục theo từng tenant
: Sao lưu riêng từng shard của tenant cho phép khôi phục trạng thái của một tenant từ bản sao lưu mà không ảnh hưởng đến các tenant khác. Điều này hữu ích khi tenant vô tình xóa hoặc ghi đè dữ liệu quan trọng [^9].

Tuân thủ quy định
: Các quy định về quyền riêng tư dữ liệu như GDPR trao cho cá nhân quyền truy cập và xóa toàn bộ dữ liệu được lưu về mình. Nếu dữ liệu của mỗi người nằm ở một shard riêng, việc này chỉ cần thao tác xuất và xóa dữ liệu đơn giản trên shard tương ứng [^10].

Lưu trú dữ liệu
: Nếu quy định về lưu trú dữ liệu yêu cầu dữ liệu của một tenant phải nằm trong một khu vực pháp lý nhất định, thì cơ sở dữ liệu nhận biết khu vực có thể gán shard của tenant đó vào khu vực chỉ định.

Triển khai dần các thay đổi schema
: Việc di chuyển schema (đã bàn ở ["Tính linh hoạt của schema trong mô hình tài liệu"](/ch3#sec_datamodels_schema_flexibility)) có thể triển khai dần, mỗi lần một tenant. Nhờ vậy ta phát hiện sự cố trước khi nó lan ra mọi tenant, giảm rủi ro, dù khó thực hiện dưới dạng giao dịch [^11].

Những thách thức chính của việc dùng sharding để hiện thực multitenancy là:

* Cách này giả định dữ liệu của mỗi tenant đủ nhỏ để nằm gọn trên một node. Nếu một tenant lớn đến mức một máy không chứa nổi, bạn vẫn phải shard bên trong tenant đó, và ta quay lại bài toán shard để mở rộng [^12].
* Nếu có nhiều tenant nhỏ, tạo shard riêng cho từng tenant có thể tốn kém quá mức. Bạn có thể gộp nhiều tenant nhỏ vào một shard lớn hơn, nhưng khi tenant lớn dần, sẽ gặp vấn đề di chuyển nó từ shard này sang shard khác.
* Nếu sau này cần hỗ trợ các tính năng liên kết dữ liệu giữa các tenant, việc join dữ liệu qua nhiều shard sẽ khiến chúng khó hiện thực hơn.

## Sharding dữ liệu khóa-giá trị {#sec_sharding_key_value}

Giả sử bạn có một lượng lớn dữ liệu và muốn shard nó. Làm sao quyết định lưu bản ghi nào trên node nào?

Mục tiêu của sharding là phân bố đều dữ liệu và tải truy vấn trên các node. Nếu mỗi node gánh phần dữ liệu và tải công bằng, thì về lý thuyết 10 node sẽ xử lý được lượng dữ liệu và thông lượng đọc/ghi gấp 10 lần một node đơn lẻ (tạm bỏ qua sao chép (*replication*)). Ngoài ra, khi thêm hoặc bớt node, ta muốn có thể *cân bằng lại* (*rebalance*) tải để nó vẫn phân bố đều trên 11 node sau khi thêm, hoặc 9 node còn lại sau khi bớt.

Nếu sharding không công bằng, tức một số shard chứa nhiều dữ liệu hoặc truy vấn hơn các shard khác, ta gọi đó là *lệch tải* (*skew*). Lệch tải làm giảm mạnh hiệu quả của sharding. Trong trường hợp cực đoan, toàn bộ tải có thể dồn vào một shard, khiến 9 trong 10 node nhàn rỗi và nút thắt nằm ở một node duy nhất đang bận. Shard có tải cao bất tương xứng gọi là *hot shard* (*hot shard*) hay *hot spot* (*hot spot*). Nếu một khóa có tải đặc biệt cao (ví dụ tài khoản người nổi tiếng trên mạng xã hội), ta gọi nó là *hot key* (*hot key*).

Vì vậy ta cần một thuật toán nhận partition key của bản ghi làm đầu vào và cho biết bản ghi đó thuộc shard nào. Trong kho khóa-giá trị, partition key thường là khóa, hoặc phần đầu của khóa. Trong mô hình quan hệ, nó có thể là một cột của bảng (không nhất thiết phải là khóa chính). Thuật toán này cũng phải cho phép cân bằng lại, để giảm bớt hot spot.

### Sharding theo khoảng khóa {#sec_sharding_key_range}

Một cách shard là gán cho mỗi shard một khoảng liên tục của partition key (từ một giá trị nhỏ nhất đến một giá trị lớn nhất), giống các tập của bộ bách khoa toàn thư giấy, như trong {{< xref fig="7-2" page="/ch7" anchor="fig_sharding_encyclopedia" >}}Hình 7-2{{< /xref >}}. Ở ví dụ này, tiêu đề mục từ là partition key. Nếu biết ranh giới giữa các khoảng, bạn dễ dàng xác định shard chứa mục từ: tìm tập có khoảng khóa bao phủ tiêu đề đó, rồi lấy đúng cuốn sách trên giá.

{{< fig num="7-2" id="fig_sharding_encyclopedia" src="/fig/ddia_0702.png" caption="Bộ bách khoa toàn thư in được shard theo khoảng khóa." class="ddia-figure ddia-figure--wide" width="2880" height="1023" />}}

Các khoảng khóa không nhất thiết có độ rộng bằng nhau, vì dữ liệu có thể phân bố không đều. Chẳng hạn, trong {{< xref fig="7-2" page="/ch7" anchor="fig_sharding_encyclopedia" >}}Hình 7-2{{< /xref >}}, tập 1 gồm các từ bắt đầu bằng A và B, nhưng tập 12 gồm các từ bắt đầu bằng T, U, V, W, X, Y và Z. Nếu cứ máy móc quy định mỗi tập hai chữ cái thì một số tập sẽ dày hơn hẳn các tập khác. Để phân bố dữ liệu đều, ranh giới shard phải được điều chỉnh theo dữ liệu.

Ranh giới shard có thể do quản trị viên chọn thủ công, hoặc do cơ sở dữ liệu tự chọn. Chẳng hạn, Vitess (lớp sharding cho MySQL) dùng sharding theo khoảng khóa thủ công; còn Bigtable, phiên bản mã nguồn mở HBase, tùy chọn sharding theo khoảng của MongoDB, CockroachDB, RethinkDB và FoundationDB dùng cách tự động [^6]. YugabyteDB hỗ trợ cả tách tablet thủ công lẫn tự động.

Trong mỗi shard, các khóa được lưu theo thứ tự, ví dụ dùng B-tree hoặc SSTable (xem [Chương 4](/ch4#ch_storage)). Nhờ vậy quét theo khoảng rất dễ, và bạn có thể coi khóa như một chỉ mục ghép (*concatenated index*) để lấy nhiều bản ghi liên quan trong một truy vấn (xem ["Chỉ mục đa chiều (*multidimensional index*) và chỉ mục (*index*) toàn văn"](/ch4#sec_storage_multidimensional)). Chẳng hạn, một ứng dụng lưu dữ liệu từ mạng cảm biến với khóa là dấu thời gian của phép đo. Quét theo khoảng rất hữu ích ở đây, vì nó cho phép dễ dàng lấy mọi số đo của một tháng.

Nhược điểm của sharding theo khoảng khóa là dễ tạo hot shard nếu có nhiều lượt ghi dồn vào các khóa liền kề nhau. Chẳng hạn, nếu khóa là dấu thời gian thì các shard ứng với các khoảng thời gian, ví dụ mỗi tháng một shard. Đáng tiếc, nếu các cảm biến ghi dữ liệu vào cơ sở dữ liệu ngay khi đo, toàn bộ lượt ghi sẽ dồn vào cùng một shard (shard của tháng hiện tại), khiến nó bị quá tải vì ghi trong khi các shard khác ngồi chơi [^13].

Để tránh vấn đề này trong cơ sở dữ liệu cảm biến, phần đầu của khóa không nên chỉ là dấu thời gian. Chẳng hạn, bạn có thể thêm ID cảm biến vào trước dấu thời gian, để khóa được sắp xếp trước tiên theo ID cảm biến rồi mới đến dấu thời gian. Khi có nhiều cảm biến hoạt động đồng thời, tải ghi sẽ được trải đều hơn trên các shard. Cái giá phải trả là khi cần lấy số đo của nhiều cảm biến trong một khoảng thời gian, giờ bạn phải thực hiện một truy vấn khoảng riêng cho từng cảm biến.

#### Cân bằng lại dữ liệu shard theo khoảng khóa {#rebalancing-key-range-sharded-data}

Khi mới thiết lập cơ sở dữ liệu, chưa có dữ liệu nên chưa có cơ sở để vạch ranh giới khoảng khóa. Một số cơ sở dữ liệu, như HBase và MongoDB, cho phép cấu hình một tập shard ban đầu trên cơ sở dữ liệu rỗng, việc này gọi là *pre-splitting* (*pre-splitting*). Để làm vậy, bạn phải biết trước đại khái các khóa sẽ phân bố ra sao để chọn được ranh giới hợp lý [^14].

Về sau, khi dữ liệu và thông lượng ghi tăng, hệ thống sharding theo khoảng khóa sẽ tách shard hiện có thành hai hoặc nhiều shard nhỏ hơn, mỗi shard mới giữ một khoảng con liên tục của khoảng khóa ban đầu. Các shard nhỏ này sau đó có thể được trải ra nhiều node. Ngược lại, nếu nhiều dữ liệu bị xóa, một vài shard liền kề đã nhỏ đi cũng có thể cần gộp thành một shard lớn hơn. Quá trình này tương tự điều xảy ra ở tầng trên cùng của B-tree (xem ["B-tree"](/ch4#sec_storage_b_trees)).

Với cơ sở dữ liệu tự quản lý ranh giới shard, việc tách shard thường được kích hoạt bởi:

* shard đạt đến kích thước cấu hình (ví dụ mặc định của HBase là 10 GB), hoặc
* ở một số hệ thống, thông lượng ghi liên tục vượt một ngưỡng nào đó. Vì thế, một hot shard cũng có thể bị tách dù không lưu nhiều dữ liệu, để tải ghi của nó được phân bố đều hơn.

Ưu điểm của sharding theo khoảng khóa là số shard thích ứng với lượng dữ liệu. Dữ liệu ít thì chỉ cần vài shard, chi phí nhỏ; dữ liệu khổng lồ thì kích thước mỗi shard vẫn bị giới hạn ở mức tối đa cấu hình được [^15].

Nhược điểm là việc tách shard rất tốn kém: phải ghi lại toàn bộ dữ liệu của nó vào các tệp mới, tương tự thao tác nén (compaction) trong engine lưu trữ dựa trên log. Shard cần tách thường đang chịu tải cao, và chi phí tách có thể làm nó quá tải hơn nữa, thậm chí sập hẳn.

### Sharding theo hash của khóa {#sec_sharding_hash}

Sharding theo khoảng khóa hữu ích khi bạn muốn các partition key liền kề (nhưng khác nhau) nằm cùng một shard; dấu thời gian là một ví dụ. Nếu bạn không quan tâm các partition key có liền kề hay không (ví dụ ID tenant trong ứng dụng multitenant), cách thường dùng là trước hết băm partition key, rồi ánh xạ giá trị hash vào shard.

Một hàm hash tốt biến dữ liệu lệch thành dữ liệu phân bố đều. Giả sử có hàm hash 32-bit nhận đầu vào là chuỗi. Mỗi khi nhận một chuỗi mới, nó trả về một số trông như ngẫu nhiên giữa 0 và 2³² − 1. Ngay cả khi các chuỗi đầu vào rất giống nhau, hash của chúng vẫn phân bố đều trong khoảng đó (dù cùng đầu vào luôn cho cùng đầu ra).

Hàm hash dùng cho sharding không cần mạnh về mật mã: chẳng hạn MongoDB dùng MD5, còn Cassandra và ScyllaDB dùng Murmur3. Nhiều ngôn ngữ lập trình có sẵn hàm hash đơn giản cho bảng băm, nhưng chúng có thể không phù hợp với sharding. Ví dụ, `Object.hashCode()` của Java và `Object#hash` của Ruby có thể cho các giá trị hash khác nhau với cùng một khóa ở các tiến trình khác nhau, nên không dùng được cho sharding [^16].

#### Hash modulo số node {#hash-modulo-number-of-nodes}

Sau khi băm khóa, làm sao chọn shard để lưu? Có thể bạn nghĩ ngay đến việc lấy giá trị hash *modulo* số node trong hệ thống (nhiều ngôn ngữ dùng toán tử `%`). Chẳng hạn, *hash*(*key*) % 10 trả về số từ 0 đến 9 (nếu viết hash ở dạng thập phân thì hash % 10 chính là chữ số cuối). Với 10 node đánh số từ 0 đến 9, đây có vẻ là cách đơn giản để gán mỗi khóa cho một node.

Vấn đề của cách *mod N* là khi số node *N* thay đổi, hầu hết các khóa phải chuyển từ node này sang node khác. {{< xref fig="7-3" page="/ch7" anchor="fig_sharding_hash_mod_n" >}}Hình 7-3{{< /xref >}} minh họa khi tăng từ ba lên bốn node. Trước khi cân bằng lại, node 0 lưu các khóa có hash 0, 3, 6, 9... Sau khi thêm node thứ tư, khóa có hash 3 chuyển sang node 3, khóa có hash 6 chuyển sang node 2, khóa có hash 9 chuyển sang node 1, và cứ thế.

{{< fig num="7-3" id="fig_sharding_hash_mod_n" src="/fig/ddia_0703.png" caption="Gán khóa cho node bằng cách băm khóa rồi lấy modulo số node. Đổi số node khiến nhiều khóa phải chuyển từ node này sang node khác." class="ddia-figure ddia-figure--wide" width="2658" height="1314" />}}

Phép *mod N* dễ tính, nhưng dẫn đến việc cân bằng lại cực kỳ kém hiệu quả vì phải di chuyển quá nhiều bản ghi giữa các node một cách không cần thiết. Ta cần cách chỉ di chuyển lượng dữ liệu tối thiểu cần thiết.

#### Số shard cố định {#fixed-number-of-shards}

Một giải pháp đơn giản và phổ biến là tạo số shard nhiều hơn hẳn số node, rồi gán nhiều shard cho mỗi node. Ví dụ, cơ sở dữ liệu chạy trên cụm 10 node có thể được chia ngay từ đầu thành 1.000 shard, mỗi node nhận 100 shard. Khóa được lưu vào shard số *hash*(*key*) % 1.000, và hệ thống ghi lại riêng shard nào nằm trên node nào.

Khi thêm node vào cụm, hệ thống có thể gán lại một số shard từ các node hiện có sang node mới cho đến khi các shard lại phân bố đều. {{< xref fig="7-4" page="/ch7" anchor="fig_sharding_rebalance_fixed" >}}Hình 7-4{{< /xref >}} minh họa quá trình này. Khi bớt node thì làm ngược lại.

{{< fig num="7-4" id="fig_sharding_rebalance_fixed" src="/fig/ddia_0704.png" caption="Thêm node mới vào cụm cơ sở dữ liệu mà mỗi node có nhiều shard." class="ddia-figure ddia-figure--wide" width="2658" height="1436" />}}

Trong mô hình này, chỉ có nguyên shard được di chuyển giữa các node, rẻ hơn tách shard. Số shard không đổi, và việc khóa thuộc shard nào cũng không đổi. Thứ duy nhất thay đổi là shard nằm trên node nào. Thay đổi này không tức thời — truyền nhiều dữ liệu qua mạng cần thời gian — nên mọi lượt đọc/ghi diễn ra trong lúc truyền vẫn dùng ánh xạ shard-node cũ.

Số shard thường được chọn là một số có nhiều ước, để tập dữ liệu chia đều được cho nhiều kích cỡ cụm khác nhau, chẳng hạn không đòi hỏi số node là lũy thừa của 2 [^4]. Thậm chí có thể tính đến sự khác biệt phần cứng trong cụm: gán nhiều shard hơn cho node mạnh hơn để chúng gánh tỉ lệ tải lớn hơn.

Cách sharding này được dùng trong Citus (lớp sharding cho PostgreSQL), Riak, Elasticsearch và Couchbase, cùng các hệ thống khác. Nó hoạt động tốt miễn là bạn ước lượng chính xác số shard cần có khi tạo cơ sở dữ liệu lần đầu. Sau đó có thể thêm bớt node dễ dàng, nhưng số node không thể vượt quá số shard.

Nếu số shard cấu hình ban đầu hóa ra không phù hợp — ví dụ hệ thống đã lớn đến mức cần nhiều node hơn số shard — thì phải thực hiện việc sharding lại rất tốn kém. Nó đòi hỏi tách từng shard và ghi ra các tệp mới, đồng thời cần nhiều dung lượng đĩa dư. Một số hệ thống không cho phép shard lại khi cơ sở dữ liệu vẫn đang nhận ghi, nên khó đổi số shard mà không có thời gian ngừng hoạt động.

Chọn đúng số shard khó nếu tổng kích thước tập dữ liệu biến động mạnh (ví dụ ban đầu nhỏ nhưng có thể tăng gấp nhiều lần). Vì mỗi shard chứa một tỉ lệ cố định trên tổng dữ liệu, kích thước shard tăng tỉ lệ với tổng dữ liệu trong cụm. Shard quá lớn thì việc cân bằng lại và phục hồi sau khi node hỏng đều rất tốn kém. Còn shard quá nhỏ thì phát sinh quá nhiều chi phí quản lý. Hiệu năng tốt nhất khi kích thước shard "vừa phải", không quá lớn cũng không quá nhỏ — điều này khó đạt được khi số shard cố định còn kích thước tập dữ liệu thì liên tục thay đổi.

#### Sharding theo khoảng hash {#sharding-by-hash-range}

Nếu không thể đoán trước cần bao nhiêu shard, tốt hơn là dùng sơ đồ cho phép số shard dễ dàng thích ứng với khối lượng công việc. Sharding theo khoảng khóa ở trên có tính chất này, nhưng dễ tạo hot spot khi nhiều lượt ghi dồn vào các khóa liền kề. Một cách khắc phục là kết hợp sharding theo khoảng khóa với hàm hash, sao cho mỗi shard chứa một khoảng *giá trị hash* thay vì một khoảng *khóa*.

{{< xref fig="7-5" page="/ch7" anchor="fig_sharding_hash_range" >}}Hình 7-5{{< /xref >}} dùng hàm hash 16-bit trả về số từ 0 đến 65.535 = 2¹⁶ − 1 (các hash thực tế thường có ít nhất 32 bit). Ngay cả khi khóa đầu vào rất giống nhau (ví dụ các dấu thời gian liên tiếp), hash của chúng vẫn phân bố đều trong khoảng này. Khi đó ta có thể gán cho mỗi shard một khoảng giá trị hash: ví dụ 0 đến 16.383 cho shard 0, 16.384 đến 32.767 cho shard 1, v.v.

{{< fig num="7-5" id="fig_sharding_hash_range" src="/fig/ddia_0705.png" caption="Gán cho mỗi shard một khoảng liên tục các giá trị hash." class="ddia-figure ddia-figure--panorama" width="2658" height="832" />}}

Giống sharding theo khoảng khóa, trong sharding theo khoảng hash, shard cũng có thể bị tách khi quá lớn hoặc quá tải. Thao tác này vẫn tốn kém nhưng có thể thực hiện khi cần, nên số shard thích ứng với lượng dữ liệu thay vì cố định từ trước.

Nhược điểm so với sharding theo khoảng khóa là không thể thực hiện truy vấn khoảng hiệu quả trên partition key, vì các khóa trong một khoảng giờ nằm rải rác ở mọi shard. Tuy nhiên, nếu khóa gồm hai cột trở lên và partition key chỉ là cột đầu tiên, bạn vẫn có thể truy vấn khoảng hiệu quả trên các cột còn lại: miễn là mọi bản ghi trong truy vấn khoảng có cùng partition key thì chúng sẽ nằm cùng một shard.

> [!TIP] Partitioning và truy vấn khoảng trong kho dữ liệu
>
> Các kho dữ liệu (data warehouse) như BigQuery, Snowflake và Delta Lake cũng hỗ trợ kiểu lập chỉ mục (*index*) tương tự, dù thuật ngữ khác. Ví dụ trong BigQuery, partition key quyết định bản ghi thuộc partition nào, còn các "cột cluster" (cluster columns) quyết định thứ tự sắp xếp bản ghi trong partition. Snowflake tự động gán bản ghi vào các "micro-partition" nhưng cho phép người dùng định nghĩa cluster key cho bảng. Delta Lake hỗ trợ cả gán partition thủ công lẫn tự động, và cũng hỗ trợ cluster key. Việc phân cụm (cluster) dữ liệu không chỉ cải thiện hiệu năng quét theo khoảng mà còn cải thiện tỉ lệ nén và hiệu quả lọc.

YugabyteDB và DynamoDB dùng sharding theo khoảng hash [^17], và MongoDB cũng cung cấp nó như một tùy chọn. Cassandra và ScyllaDB dùng một biến thể của cách này, như {{< xref fig="7-6" page="/ch7" anchor="fig_sharding_cassandra" >}}Hình 7-6{{< /xref >}}: chúng chia không gian giá trị hash thành các khoảng với số lượng tỉ lệ thuận với số node (trong {{< xref fig="7-6" page="/ch7" anchor="fig_sharding_cassandra" >}}Hình 7-6{{< /xref >}} mỗi node có 3 khoảng, nhưng giá trị mặc định thực tế là 8 khoảng mỗi node ở Cassandra và 256 khoảng mỗi node ở ScyllaDB), với ranh giới giữa các khoảng được chọn ngẫu nhiên. Cách này khiến một số khoảng lớn hơn số khác, nhưng khi mỗi node có nhiều khoảng, sự mất cân đối đó có xu hướng triệt tiêu lẫn nhau [^15] [^18].

{{< fig num="7-6" id="fig_sharding_cassandra" src="/fig/ddia_0706.png" caption="Cassandra và ScyllaDB chia khoảng giá trị hash có thể có (ở đây là 0–1023) thành các khoảng liên tục với ranh giới ngẫu nhiên, và gán nhiều khoảng cho mỗi node." class="ddia-figure ddia-figure--standard" width="2658" height="2012" />}}

Khi thêm hoặc bớt node, ranh giới khoảng được thêm hoặc xóa tương ứng, và shard được tách hoặc gộp [^19]. Trong ví dụ ở {{< xref fig="7-6" page="/ch7" anchor="fig_sharding_cassandra" >}}Hình 7-6{{< /xref >}}, khi node 3 được thêm vào, node 1 chuyển một phần của hai khoảng của nó sang node 3, và node 2 chuyển một phần của một khoảng sang node 3. Nhờ vậy node mới nhận được phần dữ liệu khá công bằng mà không phải truyền dữ liệu không cần thiết giữa các node.

#### Consistent hashing {#sec_sharding_consistent_hashing}

Thuật toán *consistent hashing* (*consistent hashing*) là một hàm hash ánh xạ khóa vào một số shard xác định và thỏa hai tính chất:

1. số khóa ánh xạ vào mỗi shard xấp xỉ bằng nhau, và
2. khi số shard thay đổi, số khóa phải di chuyển giữa các shard là ít nhất có thể.

Lưu ý rằng "consistent" (nhất quán) ở đây không liên quan gì đến nhất quán bản sao (xem [Chương 6](/ch6#ch_replication)) hay nhất quán ACID (xem [Chương 8](/ch8#ch_transactions)), mà mô tả xu hướng của một khóa ở lại trong cùng shard càng nhiều càng tốt.

Thuật toán sharding của Cassandra và ScyllaDB tương tự định nghĩa gốc của consistent hashing [^20], nhưng cũng đã có nhiều thuật toán consistent hashing khác được đề xuất [^21], chẳng hạn *highest random weight* (*highest random weight*, còn gọi là *rendezvous hashing*) [^22] và *jump consistent hash* (*jump consistent hash*) [^23]. Với thuật toán của Cassandra, thêm một node sẽ tách một số shard hiện có thành các khoảng con; với rendezvous hashing hay jump consistent hash, node mới nhận từng khóa riêng lẻ mà trước đó nằm rải rác trên mọi node khác. Cách nào phù hợp hơn tùy thuộc vào ứng dụng.

### Khối lượng công việc lệch và giảm nhẹ hot spot {#sec_sharding_skew}

Consistent hashing đảm bảo các khóa được phân bố đều trên các node, nhưng không có nghĩa tải thực tế cũng phân bố đều. Nếu khối lượng công việc bị lệch nặng — tức lượng dữ liệu dưới một số partition key lớn hơn nhiều so với các khóa khác, hoặc tốc độ yêu cầu với một số khóa cao hơn nhiều — thì vẫn có thể có máy chủ quá tải trong khi máy khác gần như rảnh.

Ví dụ trên mạng xã hội, một người nổi tiếng với hàng triệu người theo dõi thực hiện một hành động có thể gây ra cơn bão hoạt động [^24], dẫn đến lượng đọc/ghi khổng lồ vào cùng một khóa (partition key có thể là ID người dùng của người nổi tiếng, hoặc ID của sự kiện mà mọi người đang bình luận).

Tình huống này cần chiến lược sharding linh hoạt hơn [^25] [^26]. Nếu hệ thống định nghĩa shard theo khoảng khóa (hoặc khoảng hash), bạn có thể đặt riêng một hot key vào một shard, thậm chí cấp cho nó một máy chuyên dụng [^27].

Bạn cũng có thể bù lệch tải ở tầng ứng dụng. Chẳng hạn, nếu biết một khóa rất nóng, một cách đơn giản là thêm một số ngẫu nhiên vào đầu hoặc cuối khóa. Chỉ cần số ngẫu nhiên hai chữ số thập phân là có thể chia đều các lượt ghi vào khóa đó thành 100 khóa khác nhau, và chúng sẽ phân bố vào các shard khác nhau.

Tuy nhiên, khi các lượt ghi đã bị chia ra nhiều khóa, mọi lượt đọc phải trả thêm chi phí: phải đọc dữ liệu từ cả 100 khóa rồi gộp kết quả. Do hot key được chia nhỏ, lượng đọc mỗi shard phải chịu không giảm; chỉ có tải ghi được giảm. Kỹ thuật này còn cần thêm công việc ghi sổ: chỉ nên thêm số ngẫu nhiên cho số ít khóa nóng; với đại đa số khóa có thông lượng ghi thấp thì chỉ thêm chi phí vô ích. Do đó, bạn cũng cần theo dõi khóa nào đã bị chia, và có quy trình chuyển khóa thường thành hot key cần xử lý đặc biệt.

Vấn đề còn phức tạp hơn vì tải thay đổi theo thời gian. Chẳng hạn, một bài đăng mạng xã hội đột nhiên nổi như cồn có thể chịu tải cao trong vài ngày, sau đó nhiều khả năng lại yên ắng. Ngoài ra, có khóa nóng về ghi, có khóa nóng về đọc, và hai loại cần chiến lược xử lý khác nhau.

Một số hệ thống (đặc biệt là dịch vụ đám mây thiết kế cho quy mô lớn) có thể tự động xử lý hot shard; ví dụ Amazon gọi cơ chế này là *heat management* (*heat management*) [^28] hay *adaptive capacity* (*adaptive capacity*) [^17]. Chi tiết cách các hệ thống này hoạt động nằm ngoài phạm vi cuốn sách.

### Vận hành: cân bằng lại tự động hay thủ công {#sec_sharding_operations}

Về cân bằng lại có một câu hỏi quan trọng mà ta đã lướt qua: nó diễn ra tự động hay thủ công?

Một số hệ thống tự quyết định khi nào tách shard và khi nào di chuyển shard từ node này sang node khác mà không cần con người can thiệp, trong khi số khác yêu cầu quản trị viên cấu hình shard tường minh. Cũng có cách dung hòa: ví dụ Couchbase và Riak tự sinh đề xuất phân bổ shard, nhưng cần quản trị viên xác nhận (commit) thì mới có hiệu lực.

Cân bằng lại hoàn toàn tự động rất tiện vì công việc vận hành bảo trì thường ngày ít hơn, và hệ thống như vậy thậm chí có thể tự co giãn theo khối lượng công việc. Các cơ sở dữ liệu đám mây như DynamoDB tuyên bố có thể tự động thêm bớt shard trong vài phút để đáp ứng tải tăng giảm mạnh [^17] [^29].

Tuy nhiên, quản lý shard tự động cũng có thể khó đoán. Cân bằng lại là thao tác đắt đỏ, vì nó phải định tuyến lại yêu cầu và di chuyển lượng lớn dữ liệu giữa các node. Nếu làm không cẩn thận, quá trình này có thể làm quá tải mạng hoặc node và làm chậm các yêu cầu khác. Hệ thống còn phải tiếp tục xử lý ghi trong lúc cân bằng lại; nếu đã gần thông lượng ghi tối đa, tốc độ tách shard thậm chí có thể không theo kịp tốc độ ghi mới đến [^29].

Tự động hóa kiểu này khi kết hợp với phát hiện lỗi tự động có thể nguy hiểm. Chẳng hạn, một node quá tải và tạm thời phản hồi yêu cầu chậm. Các node khác kết luận node đó đã hỏng và tự động cân bằng lại cụm để chuyển tải đi khỏi nó. Điều này đặt thêm tải lên các node khác và mạng, khiến tình hình tệ hơn và có thể gây lỗi dây chuyền: các node khác cũng quá tải và bị nhầm là đã chết.

Vì vậy, để con người tham gia vào quy trình cân bằng lại là điều tốt. Nó chậm hơn quy trình hoàn toàn tự động, nhưng giúp ngăn ngừa các bất ngờ trong vận hành.

## Định tuyến yêu cầu {#sec_sharding_routing}

Ta đã bàn cách shard tập dữ liệu trên nhiều node, và cách cân bằng lại các shard khi thêm bớt node. Giờ đến vấn đề tiếp theo: nếu muốn đọc hoặc ghi một khóa cụ thể, làm sao biết phải kết nối đến node nào — tức địa chỉ IP và cổng nào?

Vấn đề này gọi là *định tuyến yêu cầu* (*request routing*), và rất giống *service discovery* (*service discovery*) đã bàn ở ["Bộ cân bằng tải, service discovery và service mesh"](/ch5#sec_encoding_service_discovery). Khác biệt lớn nhất: với các instance dịch vụ chạy mã ứng dụng, chúng thường không giữ trạng thái và bộ cân bằng tải có thể gửi yêu cầu đến instance bất kỳ. Còn với cơ sở dữ liệu được shard, yêu cầu cho một khóa chỉ có thể do node giữ bản sao của shard chứa khóa đó xử lý.

Do đó, định tuyến yêu cầu phải biết ánh xạ từ khóa đến shard và từ shard đến node. Nhìn chung có vài cách (như {{< xref fig="7-7" page="/ch7" anchor="fig_sharding_routing" >}}Hình 7-7{{< /xref >}}):

1. Cho phép client kết nối đến node bất kỳ (ví dụ qua bộ cân bằng tải round-robin). Nếu node đó tình cờ sở hữu shard mà yêu cầu liên quan, nó xử lý trực tiếp; nếu không, nó chuyển tiếp yêu cầu đến node đúng, nhận phản hồi rồi chuyển lại cho client.
2. Mọi yêu cầu của client trước hết đi đến một tầng định tuyến, tầng này xác định node nào nên xử lý từng yêu cầu rồi chuyển tiếp tương ứng. Tầng định tuyến tự nó không xử lý yêu cầu nào, chỉ đóng vai bộ cân bằng tải nhận biết shard.
3. Để client biết sơ đồ sharding và việc gán shard cho node. Khi đó client có thể kết nối thẳng đến node đúng mà không cần qua trung gian.

{{< fig num="7-7" id="fig_sharding_routing" src="/fig/ddia_0707.png" caption="Ba cách khác nhau để định tuyến yêu cầu đến đúng node." class="ddia-figure ddia-figure--wide" width="2658" height="1182" />}}

Trong mọi trường hợp đều có một số câu hỏi then chốt:

* Ai quyết định shard nào nằm trên node nào? Cách đơn giản nhất là một coordinator duy nhất quyết định, nhưng khi đó làm sao chịu lỗi nếu node chạy coordinator chết? Và nếu coordinator có thể failover sang node khác, làm sao ngăn chặn split-brain (xem ["Xử lý sự cố node"](/ch6#sec_replication_failover)) khiến hai coordinator đưa ra các phân bổ shard mâu thuẫn nhau?
* Thành phần chịu trách nhiệm định tuyến (có thể là một node cơ sở dữ liệu, tầng định tuyến hoặc client) biết thế nào về việc thay đổi phân bổ shard cho node?
* Khi một shard được chuyển từ node này sang node khác, có một khoảng thời gian chuyển giao: node mới đã tiếp quản, nhưng các yêu cầu gửi đến node cũ có thể vẫn đang trên đường. Xử lý các yêu cầu này thế nào?

Nhiều hệ thống dữ liệu phân tán dựa vào dịch vụ điều phối riêng như ZooKeeper hay etcd để theo dõi việc phân bổ shard, như {{< xref fig="7-8" page="/ch7" anchor="fig_sharding_zookeeper" >}}Hình 7-8{{< /xref >}}. Chúng dùng thuật toán đồng thuận (xem [Chương 10](/ch10#ch_consistency)) để chịu lỗi và ngăn split-brain. Mỗi node đăng ký với ZooKeeper, và ZooKeeper giữ ánh xạ đáng tin cậy từ shard đến node. Các bên khác, như tầng định tuyến hoặc client nhận biết shard, có thể đăng ký nhận thông tin này từ ZooKeeper. Bất cứ khi nào shard đổi chủ, hoặc node được thêm hay bị loại, ZooKeeper thông báo cho tầng định tuyến để thông tin định tuyến luôn cập nhật.

{{< fig num="7-8" id="fig_sharding_zookeeper" src="/fig/ddia_0708.png" caption="Dùng ZooKeeper để theo dõi việc gán shard cho node." class="ddia-figure ddia-figure--wide" width="2658" height="1163" />}}

Ví dụ, HBase và SolrCloud dùng ZooKeeper để quản lý phân bổ shard, còn Kubernetes dùng etcd để theo dõi mỗi instance dịch vụ đang chạy ở đâu. MongoDB có kiến trúc tương tự, nhưng dựa vào *config server* (*config server*) của riêng nó và dùng tiến trình nền *mongos* làm tầng định tuyến. Kafka, YugabyteDB và TiDB dùng giao thức đồng thuận Raft tích hợp sẵn để thực hiện chức năng điều phối này.

Cassandra, ScyllaDB và Riak theo cách khác: chúng lan truyền các thay đổi trạng thái cụm giữa các node bằng *giao thức gossip* (*gossip protocol*). Cách này cung cấp tính nhất quán yếu hơn nhiều so với giao thức đồng thuận, nên có thể xảy ra split-brain, khi các phần khác nhau của cụm có phân bổ node khác nhau cho cùng một shard. Cơ sở dữ liệu không leader chấp nhận được điều này vì chúng thường chỉ cung cấp đảm bảo nhất quán yếu (xem ["Giới hạn của nhất quán quorum"](/ch6#sec_replication_quorum_limitations)).

Dù dùng tầng định tuyến hay gửi yêu cầu đến node ngẫu nhiên, client vẫn cần tìm được địa chỉ IP để kết nối. Địa chỉ IP ít thay đổi hơn việc gán shard cho node, nên thường chỉ cần dùng DNS.

Phần định tuyến yêu cầu ở trên chủ yếu bàn về việc tìm shard cho một khóa đơn lẻ, phù hợp nhất với cơ sở dữ liệu OLTP được shard. Cơ sở dữ liệu phân tích cũng thường được shard, nhưng cách thực thi truy vấn rất khác: thay vì chạy trong một shard, truy vấn thường phải tổng hợp và join dữ liệu từ nhiều shard song song. Ta sẽ bàn các kỹ thuật thực thi truy vấn song song này ở ["JOIN và GROUP BY"](/ch11#sec_batch_join).

## Sharding và chỉ mục phụ (*secondary index*) {#sec_sharding_secondary_indexes}

Các sơ đồ sharding đã bàn cho đến giờ đều dựa vào việc client biết partition key của bản ghi cần truy cập. Điều này dễ nhất với mô hình dữ liệu khóa-giá trị, nơi partition key là phần đầu (hoặc toàn bộ) của khóa chính, nên có thể dùng nó để xác định shard và định tuyến yêu cầu đọc/ghi đến node chịu trách nhiệm khóa đó.

Mọi thứ phức tạp hơn nhiều khi có chỉ mục phụ (*secondary index*) (xem thêm ["Chỉ mục (*index*) nhiều cột và chỉ mục phụ (*secondary index*)"](/ch4#sec_storage_index_multicolumn)). Chỉ mục phụ (*secondary index*) thường không định danh duy nhất một bản ghi mà dùng để tìm nơi một giá trị cụ thể xuất hiện: ví dụ tìm mọi hành động của người dùng `123`, mọi bài viết chứa từ `hogwash`, hoặc mọi chiếc xe màu `red`.

Kho khóa-giá trị thường không có chỉ mục phụ (*secondary index*), nhưng chúng là tính năng nền tảng của cơ sở dữ liệu quan hệ, rất phổ biến trong cơ sở dữ liệu tài liệu, và là *lý do tồn tại* của các công cụ tìm kiếm toàn văn như Solr và Elasticsearch. Rắc rối của chỉ mục phụ (*secondary index*) là chúng không ánh xạ gọn gàng vào shard. Có hai cách chính để shard cơ sở dữ liệu có chỉ mục phụ (*secondary index*): chỉ mục cục bộ (*local index*) và chỉ mục toàn cục (*global index*).

### Chỉ mục phụ (*secondary index*) cục bộ {#id166}

Giả sử bạn đang vận hành một trang rao bán xe cũ (như {{< xref fig="7-9" page="/ch7" anchor="fig_sharding_local_secondary" >}}Hình 7-9{{< /xref >}}). Mỗi tin đăng có một ID duy nhất, dùng làm partition key để shard (ví dụ ID từ 0 đến 499 vào shard 0, ID từ 500 đến 999 vào shard 1, v.v.).

Nếu muốn cho người dùng tìm xe và lọc theo màu và hãng, bạn cần chỉ mục phụ (*secondary index*) trên `color` và `make` (trong cơ sở dữ liệu tài liệu chúng là trường, trong cơ sở dữ liệu quan hệ là cột). Sau khi khai báo chỉ mục (*index*), cơ sở dữ liệu tự động duy trì nó. Chẳng hạn, mỗi khi một chiếc xe đỏ được thêm vào, shard chứa nó tự động thêm ID của xe vào danh sách ID của mục chỉ mục (*index*) `color:red`. Như đã nói ở [Chương 4](/ch4#ch_storage), danh sách ID này còn gọi là *postings list* (*postings list*).

{{< fig num="7-9" id="fig_sharding_local_secondary" src="/fig/ddia_0709.png" caption="Chỉ mục phụ (*secondary index*) cục bộ: mỗi shard chỉ lập chỉ mục (*index*) các bản ghi trong shard của chính nó." class="ddia-figure ddia-figure--wide" width="2658" height="1260" />}}

> [!WARNING] Cảnh báo
>
> Nếu cơ sở dữ liệu chỉ hỗ trợ mô hình khóa-giá trị, bạn có thể bị cám dỗ tự hiện thực chỉ mục phụ (*secondary index*) ở mã ứng dụng bằng cách tạo ánh xạ từ giá trị sang ID. Nếu đi đường này, bạn phải hết sức cẩn thận để chỉ mục (*index*) luôn nhất quán với dữ liệu gốc. Race condition và lỗi ghi chập chờn (khi một số thay đổi được lưu còn số khác thì không) rất dễ khiến hai bên lệch nhau — xem ["Nhu cầu về giao dịch nhiều đối tượng"](/ch8#sec_transactions_need).

Với cách lập chỉ mục (*index*) này, mỗi shard hoàn toàn độc lập: nó duy trì chỉ mục phụ (*secondary index*) riêng, chỉ bao phủ các bản ghi trong shard đó, không quan tâm shard khác lưu dữ liệu gì. Mỗi lần ghi vào cơ sở dữ liệu — thêm, xóa hay cập nhật bản ghi — bạn chỉ cần xử lý shard chứa bản ghi đó. Vì vậy loại chỉ mục phụ (*secondary index*) này được gọi là *chỉ mục cục bộ* (*local index*). Trong lĩnh vực truy xuất thông tin, nó còn gọi là *chỉ mục phân vùng theo tài liệu* (*document-partitioned index*) [^30].

Khi đọc từ chỉ mục phụ (*secondary index*) cục bộ, nếu đã biết partition key của bản ghi cần tìm, bạn chỉ cần tìm trên shard tương ứng. Nếu chỉ cần kết quả *một phần* chứ không cần đầy đủ, bạn cũng có thể gửi yêu cầu đến shard bất kỳ.

Nhưng nếu cần toàn bộ kết quả mà không biết trước partition key của chúng, bạn phải gửi truy vấn đến mọi shard rồi gộp các kết quả trả về, vì các bản ghi khớp có thể nằm rải rác ở mọi shard. Trong {{< xref fig="7-9" page="/ch7" anchor="fig_sharding_local_secondary" >}}Hình 7-9{{< /xref >}}, cả shard 0 và shard 1 đều có xe đỏ.

Cách truy vấn cơ sở dữ liệu được shard này có thể khiến truy vấn đọc trên chỉ mục phụ (*secondary index*) khá tốn kém. Ngay cả khi truy vấn các shard song song, nó vẫn dễ bị khuếch đại độ trễ đuôi (xem ["Ứng dụng các chỉ số thời gian phản hồi"](/ch2#sec_introduction_slo_sla)). Nó cũng hạn chế khả năng mở rộng của ứng dụng: thêm shard giúp chứa nhiều dữ liệu hơn, nhưng nếu mọi truy vấn vẫn phải do tất cả shard xử lý thì thông lượng truy vấn không tăng theo.

Dù vậy, chỉ mục phụ (*secondary index*) cục bộ vẫn được dùng rộng rãi [^31] — MongoDB, Riak, Cassandra [^32], Elasticsearch [^33], SolrCloud và VoltDB [^34] đều dùng chúng.

### Chỉ mục phụ (*secondary index*) toàn cục {#id167}

Thay vì để mỗi shard có chỉ mục phụ (*secondary index*) cục bộ riêng, ta có thể xây một *chỉ mục toàn cục* (*global index*) bao phủ dữ liệu của mọi shard. Tuy nhiên, không thể chỉ lưu chỉ mục (*index*) này trên một node, vì nó dễ trở thành nút thắt và làm mất ý nghĩa của sharding. Chỉ mục toàn cục (*global index*) cũng phải được shard, nhưng có thể shard theo cách khác với chỉ mục (*index*) khóa chính.

{{< xref fig="7-10" page="/ch7" anchor="fig_sharding_global_secondary" >}}Hình 7-10{{< /xref >}} minh họa dạng của nó: ID các xe đỏ từ mọi shard được liệt kê dưới mục `color:red` của chỉ mục (*index*), nhưng bản thân chỉ mục (*index*) được shard sao cho các màu bắt đầu từ chữ *a* đến *r* vào shard 0, và các màu bắt đầu từ *s* đến *z* vào shard 1. Chỉ mục (*index*) theo hãng xe cũng được shard tương tự, với ranh giới giữa *f* và *h*.

{{< fig num="7-10" id="fig_sharding_global_secondary" src="/fig/ddia_0710.png" caption="Chỉ mục phụ (*secondary index*) toàn cục phản ánh dữ liệu từ mọi shard, và bản thân nó được shard theo giá trị được lập chỉ mục (*index*)." class="ddia-figure ddia-figure--wide" width="2658" height="1129" />}}

Loại chỉ mục (*index*) này còn gọi là *phân vùng theo term* (*term-partitioned*) [^30]. Nhớ lại ở ["Tìm kiếm toàn văn"](/ch4#sec_storage_full_text) rằng trong tìm kiếm toàn văn, *term* (*term*) là từ khóa có thể tìm trong văn bản. Ở đây ta tổng quát hóa thành mọi giá trị có thể tìm trong chỉ mục phụ (*secondary index*).

Chỉ mục toàn cục (*global index*) dùng term làm partition key, nên khi tìm một term hay giá trị, ta xác định được ngay cần truy vấn shard nào. Như trước, một shard có thể chứa một khoảng term liên tục (như {{< xref fig="7-10" page="/ch7" anchor="fig_sharding_global_secondary" >}}Hình 7-10{{< /xref >}}), hoặc có thể gán term vào shard dựa trên hash của term.

Ưu điểm của chỉ mục toàn cục (*global index*) là nếu truy vấn chỉ có một điều kiện (như *color = red*), chỉ cần đọc một shard để lấy postings list. Tuy nhiên, nếu bạn muốn không chỉ ID mà cả bản ghi đầy đủ, bạn vẫn phải đọc mọi shard chịu trách nhiệm lưu các ID đó.

Nếu truy vấn có nhiều điều kiện hay nhiều term (ví dụ tìm xe theo màu và hãng, hoặc tìm nhiều từ cùng xuất hiện trong một văn bản), các term này nhiều khả năng nằm ở các shard khác nhau. Để tính phép AND logic của hai điều kiện, hệ thống phải tìm các ID có mặt trong cả hai postings list. Nếu list ngắn thì không khó, nhưng nếu dài, việc gửi chúng qua mạng để tính giao có thể chậm [^30].

Một khó khăn khác của chỉ mục phụ (*secondary index*) toàn cục là việc ghi phức tạp hơn chỉ mục cục bộ (*local index*): ghi một bản ghi có thể ảnh hưởng nhiều shard của chỉ mục (*index*) (mỗi term trong tài liệu có thể nằm ở một shard khác). Do đó khó giữ cho chỉ mục phụ (*secondary index*) đồng bộ với dữ liệu gốc. Một cách là dùng giao dịch phân tán để cập nhật nguyên tử shard chứa bản ghi chính và các shard chứa chỉ mục phụ (*secondary index*) của nó (xem [Chương 8](/ch8#ch_transactions)).

CockroachDB, TiDB và YugabyteDB dùng chỉ mục phụ (*secondary index*) toàn cục; DynamoDB hỗ trợ cả chỉ mục phụ (*secondary index*) cục bộ lẫn toàn cục. Trong DynamoDB, các lượt ghi được phản ánh vào chỉ mục toàn cục (*global index*) một cách bất đồng bộ, nên kết quả đọc từ chỉ mục toàn cục (*global index*) có thể đã cũ (tương tự ["Vấn đề của replication lag"](/ch6#sec_replication_lag)). Dù vậy, chỉ mục toàn cục (*global index*) vẫn hữu ích nếu thông lượng đọc cao hơn thông lượng ghi và postings list không quá dài.

## Tóm tắt {#summary}

Trong chương này, ta đã khám phá các cách chia tập dữ liệu lớn thành các tập con nhỏ hơn. Khi lượng dữ liệu rất lớn, lưu trữ và xử lý trên một máy không còn khả thi, và sharding trở nên cần thiết.

Mục tiêu của sharding là phân bố đều dữ liệu và tải truy vấn trên nhiều máy, tránh hot spot (node có tải bất tương xứng). Điều này đòi hỏi chọn sơ đồ sharding phù hợp với dữ liệu, và cân bằng lại các shard khi node được thêm vào hoặc bị loại khỏi cụm.

Ta đã bàn hai cách sharding chính:

* *Sharding theo khoảng khóa*: các khóa được sắp xếp, và mỗi shard sở hữu mọi khóa từ một giá trị nhỏ nhất đến một giá trị lớn nhất. Lưu trữ có thứ tự cho phép truy vấn khoảng hiệu quả, nhưng có nguy cơ tạo hot spot nếu ứng dụng thường xuyên truy cập các khóa gần nhau trong thứ tự sắp xếp.

  Với cách này, shard thường được cân bằng lại động bằng cách tách khoảng khóa thành hai khoảng con khi shard quá lớn.
* *Sharding theo hash*: hàm hash được áp dụng cho từng khóa, và mỗi shard sở hữu một khoảng giá trị hash (hoặc có thể dùng thuật toán consistent hashing khác để ánh xạ hash vào shard). Cách này phá vỡ thứ tự của khóa, làm truy vấn khoảng kém hiệu quả, nhưng có thể phân bố tải đều hơn.

  Khi shard theo hash, thường tạo trước một số shard cố định, gán nhiều shard cho mỗi node, và di chuyển nguyên shard từ node này sang node khác khi thêm bớt node. Cũng có thể tách shard như với khoảng khóa.

Cách làm phổ biến là dùng phần đầu của khóa làm partition key (tức dùng để xác định shard), rồi sắp xếp các bản ghi trong shard theo phần còn lại của khóa. Như vậy vẫn có thể truy vấn khoảng hiệu quả giữa các bản ghi có cùng partition key.

Ta cũng bàn về tương tác giữa sharding và chỉ mục phụ (*secondary index*). Chỉ mục phụ (*secondary index*) cũng cần được shard, có hai cách:

* *Chỉ mục phụ (*secondary index*) cục bộ*: chỉ mục phụ (*secondary index*) được lưu cùng shard với khóa chính và giá trị. Nhờ vậy khi ghi chỉ cần cập nhật một shard, nhưng khi tra cứu chỉ mục phụ (*secondary index*) phải đọc mọi shard.
* *Chỉ mục phụ (*secondary index*) toàn cục*: được shard riêng theo giá trị được lập chỉ mục (*index*). Mục chỉ mục phụ (*secondary index*) có thể tham chiếu bản ghi từ mọi shard của khóa chính. Khi ghi bản ghi, có thể cần cập nhật nhiều shard của chỉ mục phụ (*secondary index*); nhưng khi đọc postings list thì chỉ cần truy cập một shard (việc lấy bản ghi thực tế vẫn cần đọc nhiều shard).

Cuối cùng, ta đã bàn cách định tuyến truy vấn đến đúng shard, và cách dịch vụ điều phối theo dõi việc gán shard cho node.

Về thiết kế, mỗi shard phần lớn hoạt động độc lập — chính điều này cho phép cơ sở dữ liệu được shard mở rộng ra nhiều máy. Tuy nhiên, các thao tác cần ghi vào nhiều shard có thể rất khó: chẳng hạn, nếu ghi vào một shard thành công nhưng shard khác thất bại thì sao? Ta sẽ trả lời câu hỏi này trong các chương tiếp theo.

### Tài liệu tham khảo

[^1]: Claire Giordano. [Understanding partitioning and sharding in Postgres and Citus](https://www.citusdata.com/blog/2023/08/04/understanding-partitioning-and-sharding-in-postgres-and-citus/). *citusdata.com*, August 2023. Archived at [perma.cc/8BTK-8959](https://perma.cc/8BTK-8959) 
[^2]: Brandur Leach. [Partitioning in Postgres, 2022 edition](https://brandur.org/fragments/postgres-partitioning-2022). *brandur.org*, October 2022. Archived at [perma.cc/Z5LE-6AKX](https://perma.cc/Z5LE-6AKX) 
[^3]: Raph Koster. [Database “sharding” came from UO?](https://www.raphkoster.com/2009/01/08/database-sharding-came-from-uo/) *raphkoster.com*, January 2009. Archived at [perma.cc/4N9U-5KYF](https://perma.cc/4N9U-5KYF) 
[^4]: Garrett Fidalgo. [Herding elephants: Lessons learned from sharding Postgres at Notion](https://www.notion.com/blog/sharding-postgres-at-notion). *notion.com*, October 2021. Archived at [perma.cc/5J5V-W2VX](https://perma.cc/5J5V-W2VX) 
[^5]: Ulrich Drepper. [What Every Programmer Should Know About Memory](https://www.akkadia.org/drepper/cpumemory.pdf). *akkadia.org*, November 2007. Archived at [perma.cc/NU6Q-DRXZ](https://perma.cc/NU6Q-DRXZ) 
[^6]: Jingyu Zhou, Meng Xu, Alexander Shraer, Bala Namasivayam, Alex Miller, Evan Tschannen, Steve Atherton, Andrew J. Beamon, Rusty Sears, John Leach, Dave Rosenthal, Xin Dong, Will Wilson, Ben Collins, David Scherer, Alec Grieser, Young Liu, Alvin Moore, Bhaskar Muppana, Xiaoge Su, and Vishesh Yadav. [FoundationDB: A Distributed Unbundled Transactional Key Value Store](https://www.foundationdb.org/files/fdb-paper.pdf). At *ACM International Conference on Management of Data* (SIGMOD), June 2021. [doi:10.1145/3448016.3457559](https://doi.org/10.1145/3448016.3457559) 
[^7]: Marco Slot. [Citus 12: Schema-based sharding for PostgreSQL](https://www.citusdata.com/blog/2023/07/18/citus-12-schema-based-sharding-for-postgres/). *citusdata.com*, July 2023. Archived at [perma.cc/R874-EC9W](https://perma.cc/R874-EC9W) 
[^8]: Robisson Oliveira. [Reducing the Scope of Impact with Cell-Based Architecture](https://docs.aws.amazon.com/pdfs/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/reducing-scope-of-impact-with-cell-based-architecture.pdf). AWS Well-Architected white paper, Amazon Web Services, September 2023. Archived at [perma.cc/4KWW-47NR](https://perma.cc/4KWW-47NR) 
[^9]: Gwen Shapira. [Things DBs Don’t Do - But Should](https://www.thenile.dev/blog/things-dbs-dont-do). *thenile.dev*, February 2023. Archived at [perma.cc/C3J4-JSFW](https://perma.cc/C3J4-JSFW) 
[^10]: Malte Schwarzkopf, Eddie Kohler, M. Frans Kaashoek, and Robert Morris. [Position: GDPR Compliance by Construction](https://cs.brown.edu/people/malte/pub/papers/2019-poly-gdpr.pdf). At *Towards Polystores that manage multiple Databases, Privacy, Security and/or Policy Issues for Heterogenous Data* (Poly), August 2019. [doi:10.1007/978-3-030-33752-0\_3](https://doi.org/10.1007/978-3-030-33752-0_3) 
[^11]: Gwen Shapira. [Introducing pg\_karnak: Transactional schema migration across tenant databases](https://www.thenile.dev/blog/distributed-ddl). *thenile.dev*, November 2024. Archived at [perma.cc/R5RD-8HR9](https://perma.cc/R5RD-8HR9) 
[^12]: Arka Ganguli, Guido Iaquinti, Maggie Zhou, and Rafael Chacón. [Scaling Datastores at Slack with Vitess](https://slack.engineering/scaling-datastores-at-slack-with-vitess/). *slack.engineering*, December 2020. Archived at [perma.cc/UW8F-ALJK](https://perma.cc/UW8F-ALJK) 
[^13]: Ikai Lan. [App Engine Datastore Tip: Monotonically Increasing Values Are Bad](https://ikaisays.com/2011/01/25/app-engine-datastore-tip-monotonically-increasing-values-are-bad/). *ikaisays.com*, January 2011. Archived at [perma.cc/BPX8-RPJB](https://perma.cc/BPX8-RPJB) 
[^14]: Enis Soztutar. [Apache HBase Region Splitting and Merging](https://www.cloudera.com/blog/technical/apache-hbase-region-splitting-and-merging.html). *cloudera.com*, February 2013. Archived at [perma.cc/S9HS-2X2C](https://perma.cc/S9HS-2X2C) 
[^15]: Eric Evans. [Rethinking Topology in Cassandra](https://www.youtube.com/watch?v=Qz6ElTdYjjU). At *Cassandra Summit*, June 2013. Archived at [perma.cc/2DKM-F438](https://perma.cc/2DKM-F438) 
[^16]: Martin Kleppmann. [Java’s hashCode Is Not Safe for Distributed Systems](https://martin.kleppmann.com/2012/06/18/java-hashcode-unsafe-for-distributed-systems.html). *martin.kleppmann.com*, June 2012. Archived at [perma.cc/LK5U-VZSN](https://perma.cc/LK5U-VZSN) 
[^17]: Mostafa Elhemali, Niall Gallagher, Nicholas Gordon, Joseph Idziorek, Richard Krog, Colin Lazier, Erben Mo, Akhilesh Mritunjai, Somu Perianayagam, Tim Rath, Swami Sivasubramanian, James Christopher Sorenson III, Sroaj Sosothikul, Doug Terry, and Akshat Vig. [Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service](https://www.usenix.org/conference/atc22/presentation/elhemali). At *USENIX Annual Technical Conference* (ATC), July 2022. 
[^18]: Brandon Williams. [Virtual Nodes in Cassandra 1.2](https://www.datastax.com/blog/virtual-nodes-cassandra-12). *datastax.com*, December 2012. Archived at [perma.cc/N385-EQXV](https://perma.cc/N385-EQXV) 
[^19]: Branimir Lambov. [New Token Allocation Algorithm in Cassandra 3.0](https://www.datastax.com/blog/new-token-allocation-algorithm-cassandra-30). *datastax.com*, January 2016. Archived at [perma.cc/2BG7-LDWY](https://perma.cc/2BG7-LDWY) 
[^20]: David Karger, Eric Lehman, Tom Leighton, Rina Panigrahy, Matthew Levine, and Daniel Lewin. [Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web](https://people.csail.mit.edu/karger/Papers/web.pdf). At *29th Annual ACM Symposium on Theory of Computing* (STOC), May 1997. [doi:10.1145/258533.258660](https://doi.org/10.1145/258533.258660) 
[^21]: Damian Gryski. [Consistent Hashing: Algorithmic Tradeoffs](https://dgryski.medium.com/consistent-hashing-algorithmic-tradeoffs-ef6b8e2fcae8). *dgryski.medium.com*, April 2018. Archived at [perma.cc/B2WF-TYQ8](https://perma.cc/B2WF-TYQ8) 
[^22]: David G. Thaler and Chinya V. Ravishankar. [Using name-based mappings to increase hit rates](https://www.cs.kent.edu/~javed/DL/web/p1-thaler.pdf). *IEEE/ACM Transactions on Networking*, volume 6, issue 1, pages 1–14, February 1998. [doi:10.1109/90.663936](https://doi.org/10.1109/90.663936) 
[^23]: John Lamping and Eric Veach. [A Fast, Minimal Memory, Consistent Hash Algorithm](https://arxiv.org/abs/1406.2294). *arxiv.org*, June 2014. 
[^24]: Samuel Axon. [3% of Twitter’s Servers Dedicated to Justin Bieber](https://mashable.com/archive/justin-bieber-twitter). *mashable.com*, September 2010. Archived at [perma.cc/F35N-CGVX](https://perma.cc/F35N-CGVX) 
[^25]: Gerald Guo and Thawan Kooburat. [Scaling services with Shard Manager](https://engineering.fb.com/2020/08/24/production-engineering/scaling-services-with-shard-manager/). *engineering.fb.com*, August 2020. Archived at [perma.cc/EFS3-XQYT](https://perma.cc/EFS3-XQYT) 
[^26]: Sangmin Lee, Zhenhua Guo, Omer Sunercan, Jun Ying, Thawan Kooburat, Suryadeep Biswal, Jun Chen, Kun Huang, Yatpang Cheung, Yiding Zhou, Kaushik Veeraraghavan, Biren Damani, Pol Mauri Ruiz, Vikas Mehta, and Chunqiang Tang. [Shard Manager: A Generic Shard Management Framework for Geo-distributed Applications](https://dl.acm.org/doi/pdf/10.1145/3477132.3483546). *28th ACM SIGOPS Symposium on Operating Systems Principles* (SOSP), pages 553–569, October 2021. [doi:10.1145/3477132.3483546](https://doi.org/10.1145/3477132.3483546) 
[^27]: Scott Lystig Fritchie. [A Critique of Resizable Hash Tables: Riak Core & Random Slicing](https://www.infoq.com/articles/dynamo-riak-random-slicing/). *infoq.com*, August 2018. Archived at [perma.cc/RPX7-7BLN](https://perma.cc/RPX7-7BLN) 
[^28]: Andy Warfield. [Building and operating a pretty big storage system called S3](https://www.allthingsdistributed.com/2023/07/building-and-operating-a-pretty-big-storage-system.html). *allthingsdistributed.com*, July 2023. Archived at [perma.cc/6S7P-GLM4](https://perma.cc/6S7P-GLM4) 
[^29]: Rich Houlihan. [DynamoDB adaptive capacity: smooth performance for chaotic workloads (DAT327)](https://www.youtube.com/watch?v=kMY0_m29YzU). At *AWS re:Invent*, November 2017. 
[^30]: Christopher D. Manning, Prabhakar Raghavan, and Hinrich Schütze. [*Introduction to Information Retrieval*](https://nlp.stanford.edu/IR-book/). Cambridge University Press, 2008. ISBN: 978-0-521-86571-5, available online at [nlp.stanford.edu/IR-book](https://nlp.stanford.edu/IR-book/) 
[^31]: Michael Busch, Krishna Gade, Brian Larson, Patrick Lok, Samuel Luckenbill, and Jimmy Lin. [Earlybird: Real-Time Search at Twitter](https://cs.uwaterloo.ca/~jimmylin/publications/Busch_etal_ICDE2012.pdf). At *28th IEEE International Conference on Data Engineering* (ICDE), April 2012. [doi:10.1109/ICDE.2012.149](https://doi.org/10.1109/ICDE.2012.149) 
[^32]: Nadav Har’El. [Indexing in Cassandra 3](https://github.com/scylladb/scylladb/wiki/Indexing-in-Cassandra-3). *github.com*, April 2017. Archived at [perma.cc/3ENV-8T9P](https://perma.cc/3ENV-8T9P) 
[^33]: Zachary Tong. [Customizing Your Document Routing](https://www.elastic.co/blog/customizing-your-document-routing/). *elastic.co*, June 2013. Archived at [perma.cc/97VM-MREN](https://perma.cc/97VM-MREN) 
[^34]: Andrew Pavlo. [H-Store Frequently Asked Questions](https://hstore.cs.brown.edu/documentation/faq/). *hstore.cs.brown.edu*, October 2013. Archived at [perma.cc/X3ZA-DW6Z](https://perma.cc/X3ZA-DW6Z) 
